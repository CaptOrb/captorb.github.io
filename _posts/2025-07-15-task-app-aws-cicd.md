## Secure CI/CD for Spring Boot + React on AWS with GitHub Actions and OIDC

## Objective
In my previous blog post, I discussed the architecture decisions, CloudFormation setup, and security considerations for deploying a Spring Boot + React app on AWS. 

Building on that foundation, this post will focus on creating a fully automated CI/CD pipeline which will automatically run tests and deploy the application every time code is pushed to GitHub.

## Revisiting the Cloud Architecture
Recall that the cloud infrastructure for this application is deployed via CloudFormation. The backend is deployed to an EC2-backed ECS and the Frontend is deployed to S3 and served through CloudFront.

![Cloud Architecture Diagram](/images/cloud_arch_task.png)


## Creating the CI/CD pipeline
I decided to use GitHub Actions instead of AWS tools such as CodePipeline, because it's both free and easy to use. 

Before setting up the pipeline, I needed to address some security considerations and update the CloudFormation template to grant the necessary permissions to allow a successful deployment to AWS.

### Security Considerations

A key security consideration was to use **OIDC (OpenID Connect)** instead of access keys for this deployment.

Malicious bots can automatically scan for the presence of AWS credentials in GitHub repositories resulting in account compromise, or malicious actors using your account to rack up tens of thousands of Euro. I didn't have to wait long to see such an attempt on my server. 

![AWS BOT](/images/aws_bot_check.PNG)
Note: a 200 response is returned because React handles routing client-side, the endpoint doesn't exist on the server.


The fact that AWS access keys are long lived and don't expire unless manually rotated poses security risks e.g. it requires storing the access key in GitHub Secrets and rotating it manually which is error prone and easy to neglect.

When using OIDC, GitHub  generates a temporary token which AWS then verifies. This token allows GitHub to assume an IAM role temporarily which limits the scope and duration of AWS access. For this purpose, an GitHubActionsRole IAM role exists in AWS.

### Adjusting the CloudFormation template for CI/CD
I started by amending my CloudFormation template to create a GitHub Actions IAM role that supports OIDC (OpenID Connect) authentication. 

This role uses the ``sts:AssumeRoleWithWebIdentity`` action, which allows GitHub Actions to securely assume the role without requiring long lived AWS credentials. The trust policy ensures the request is coming from my GitHub repository for security reasons. I have omitted the code for this for brevity.

To support deployments to ECS, update the ECS service with new task definitions, upload frontend assets to S3, and trigger a CloudFront cache invalidation for fresh content delivery, I also created a custom policy named GitHubActionsDeployPolicy. This policy grants the necessary permissions for my deployment pipeline, as shown below.

{% raw %}
```yaml
        - PolicyName: GitHubActionsDeployPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Sid: AllowS3Actions
                Effect: Allow
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

### GitHub Actions CI/CD



Using GitHub Actions, I created a CI/CD pipeline that is triggered when I push to my aws_deploy branch.

Notice below I divided this pipeline into several sections. This makes it easier to visualise what parts of the pipeline have succeeded or failed, allowing for easier debugging.

#### Building and Testing the Frontend
Building the frontend is very simple so I have omitted the code for brevity. It involves checking out the code, installing dependencies using ``npm ci`` and then building using ``npm run build``

As I deploy the build in separate stage, I use the ``actions/upload-artifact@v4`` action to make the build available in the deployment stage.

#### Deploying the Frontend
Deploying the frontend is relatively simple, I use **OIDC** to gain temporary credentials and assume the GitHub Actions Role, then use the frontend build which was generated in a prior stage and upload it to S3 and invalidate the CloudFront cache to ensure users see the latest version of my application.

{% raw %}
```yaml
  deploy-frontend:
    name: Deploy Frontend to AWS
    runs-on: ubuntu-latest
    needs: build-frontend
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4.1.1

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHub_Actions_Role
          aws-region: eu-west-1

      - name: Download frontend build
        uses: actions/download-artifact@v4
        with:
          name: frontend-dist
          path: frontend/dist

      - name: Upload build to S3
        run: |
          aws s3 cp frontend/dist s3://${{ secrets.AWS_S3_BUCKET }}/ --recursive

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DIST_ID }} \
            --paths "/*"
```
{% endraw %}
#### Building and Testing the Backend

This is quite straightforward. I use a ARM based runner as the EC2 instances I use are ARM based, checkout the code, set up the JDK and build the backend with Maven and finally run the backend tests using Maven. 

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

To deploy the backend, GitHub Actions uses OIDC to gain the necessary temporary permissions. The backend Docker image is then built, tagged, and pushed to **Amazon ECR (Elastic Container Registry)**. Finally the ECS Service is updated to ensure the latest version of the backend is fully deployed. Note that secrets such as **AWS_ACCOUNT_ID** are securely stored in **GitHub Secrets**.

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

#### Comparison of Docker Buildx vs native ARM 64 runner
Initially I was using Docker Buildx to cross compile for ARM since the EC2 instance has an ARM based architecture. This was very slow and resulted in the pipeline taking over 6 minutes to complete upon each push to GitHub. 

![CI/CD BuildX](/images/cicd-prenative.png)

As of January 2025, GitHub Actions now supports native ARM runners which reduced deployment time to roughly 2 minutes.
![CI/CD Native](/images/cicd-native.PNG)

## Lessons learnt
- Use OIDC instead of access keys for better security
- Use native GitHub Actions runner for faster deployments where possible


## Conclusion
I really enjoyed completing learning how to build secure CI/CD pipelines to deploy applications to AWS. It allowed me to optimise my workflow as well as cut deployment time in half.



