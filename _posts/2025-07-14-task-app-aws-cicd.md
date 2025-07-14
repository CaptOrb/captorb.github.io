## CI/CD for Spring Boot + React on AWS: Automating Deployments with GitHub Actions

### Objective
In my previous blog post, I detailed the architecture decisions, CloudFormation setup, and security considerations for deploying a Spring Boot + React app on AWS. Building on that foundation, this post will focus on creating a fully automated CI/CD pipeline. The pipeline will automatically run tests and deploy the application every time code is pushed to GitHub, streamlining the development workflow and ensuring faster, reliable deployments.

### Revisiting the Cloud Architecture
Recall that the cloud infrastrucure for this application is deployed via CloudFormation. The backend is to deployed to a EC2 backed ECS and the Frontend is deployed to S3 and served through CloudFront.

![Cloud Architecture Diagram](/images/cloud_arch_task.png)


### Creating the CI/CD pipeline
Using GitHub Actions, I created a CI/CD pipeline that is trigged when I push to my aws_deploy branch. First, I build the frontend and run the tests using Maven. The tests use JUnit and Mockito. Notice below I divided this pipeline into several sections namely "Checkout code, Set up JDK, etc. This makes it easier to visualise what parts of the pipeline have succeeded or failed allowing for easier debugging.

#### Security Considerations



### Adjusting the CloudFormation template for CI/CD
edits needed to cloudformation.
GitHubActionsRole - sts:AssumeRoleWithWebIdentity -oidc
GitHubActionsDeployPolicy with

{% raw %}
```yaml
  Action:
    - s3:PutObject
  Resource:
    - !GetAtt FrontendBucket.Arn
    - !Sub "${FrontendBucket.Arn}/*"

  - Sid: AllowCloudFrontInvalidation
  Effect: Allow
  Action:
    - cloudfront:CreateInvalidation
  Resource:
    - !Sub "arn:aws:cloudfront::${AWS::AccountId}:distribution/${CloudFrontDistribution}"
  - Sid: AllowEcrLogin
  Effect: Allow
  Action:
    - ecr:GetAuthorizationToken
  Resource: "*"

  - Sid: AllowEcsAndEcrDeployments
  Effect: Allow
  Action:
    - ecs:UpdateService
    - ecs:DescribeServices
    - ecr:BatchCheckLayerAvailability
    - ecr:InitiateLayerUpload
    - ecr:UploadLayerPart
    - ecr:CompleteLayerUpload
    - ecr:PutImage
  Resource:
    - !Sub "arn:aws:ecs:${AWS::Region}:${AWS::AccountId}:cluster/${ECSCluster}"
    - !Sub "arn:aws:ecs:${AWS::Region}:${AWS::AccountId}:service/${ECSCluster}/*"
    - !Sub "arn:aws:ecr:${AWS::Region}:${AWS::AccountId}:repository/taskapp"
```
{% endraw %}

#### Building and Testing the Backend
{% raw %}
```yaml
  build-backend:
    name: Build & Test Backend
    runs-on: ubuntu-24.04-arm
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Clean and Build Backend with Maven
        run: |
          cd backend
          mvn clean install -DskipTests

      - name: Run Tests with Maven
        run: |
          cd backend
          mvn test
```
{% endraw %}

#### Deploying the backend
{% raw %}
```yaml
  build-deploy-backend:
    name: Build & Deploy Backend to ECS
    runs-on: ubuntu-24.04-arm
    needs: build-backend
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHub_Actions_Role
          aws-region: eu-west-1
 
      - name: Build Docker image (native ARM64)
        run: |
          docker build -t taskapp:latest ./backend
      - name: Login to Amazon ECR
        run: |
          aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.eu-west-1.amazonaws.com

      - name: Tag and Push the Docker Image to ECR
        run: |
          docker tag taskapp:latest ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.eu-west-1.amazonaws.com/taskapp:latest
          docker push ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.eu-west-1.amazonaws.com/taskapp:latest

      - name: Update ECS Service
        run: |
          aws ecs update-service \
            --cluster ${{ secrets.ECS_CLUSTER_NAME }} \
            --service ${{ secrets.ECS_SERVICE_NAME }} \
            --force-new-deployment
```
{% endraw %}

do comparison of pre native arm64 vs native


## Lessons learnt
- Use OIDC instead of access keys for better security


## Conclusion
I really enjoyed completing this project and deepening my understanding of AWS infrastructure. It gave me hands-on experience with IaC, container orchestration, and secure cloud deployment



