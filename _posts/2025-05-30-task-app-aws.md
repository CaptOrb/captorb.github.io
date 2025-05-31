## Deploying a Spring Boot + React App on AWS Using CloudFormation (ECS-EC2, ALB, S3)

### Objective
I wanted to gain experience with industry-standard AWS deployment practices, so I decided to rework the infrastructure behind my existing Task Manager app. The app itself which was built with a Spring Boot REST API backend and a React frontend remains unchanged. What’s different is how it’s deployed.

Previously, the app was deployed to Oracle Cloud using Docker Compose and a CI/CD pipeline. This time, I wanted to create an AWS-centric environment. A key constraint was to minimise the cost and as much as possible, stay within free tier limits.

## Cloud Architecture

As illustrated in the diagram below, the user interacts with CloudFront, which intelligently routes requests for static content to an S3 bucket and API calls to the Application Load Balancer (ALB). Note the VPC and Internet Gateway are omitted here for brevity.

![Cloud Architecture](/images/cloud_arch_task.png)

### Design Decisions
#### Backend

#### Lambda vs ECS-EC2 vs Fargate
 AWS Lambda (Serverless)

- Fully managed, scales automatically with demand
- Cold starts can be significant for Java applications like Spring Boot

#### ECS with EC2 Launch Type

- Full control over EC2 instance
- Can use a free tier EC2 instance to minimise costs
- Requires manual management of instance lifecycle and scaling policies

#### Amazon ECS with Fargate 

- No need to manage EC2 instances
- Seamless scaling
- Does not appear to be free tier eligible

Decision: I chose ECS (EC2) as it offers a balance between cost, flexibility, and suitability for a containerised Spring Boot app. It allows me to stay within the AWS Free Tier while still simulating a production-like environment.  ECS also provides container orchestration capabilities such as service discovery, load balancing, health checks, and auto-recovery of failed containers.

### Database
I chose Amazon RDS with MySQL to meet the application's original requirements without modifying the codebase.

#### Why RDS?
- Managed database which has many benefits such as automated backups, patching, and scaling
- Simplifies setup and management compared to self-hosted alternatives
- Can stay within Free Tier limits if you use a db.t3.micro instance

#### Why not self-hosted MySQL in ECS?

- Adds unnecessary operational overhead (no managed backups, scaling or failure recovery)
- Increase complexity without offering substantial benefits for this project

#### Application Load Balancer (ALB)

While the primary role of a load balancer is to distribute incoming traffic, AWS Application Load Balancer (ALB) offers additional benefits such as:


* **Health Checks:** ALB provides robust health checks at the application layer, allowing it to monitor the actual responsiveness of the application endpoint. This ensures that traffic is only sent to truly healthy instances, improving application availability

- No longer need to worry about EC2 instance IPs changing (e.g. on restart) as we are using the public DNS of the ALB

![Cloud Architecture](/images/asg-health.PNG)


It's worth noting that while I limited the number of instances to reduce costs, the setup can be easily scaled to increase availability and performance if needed.

### Networking:

Using a NAT Gateway in eu-west-1 would cost **~$34.56/month** + data charges which is far too expensive for a proof-of-concept. Instead, I used a NAT instance to keep the costs down. 

It's worth noting that while a NAT Gateway is fully managed and more scalable, using a NAT instance can be significantly cheaper even after accounting for the cost of an extra EC2 instance (since NAT functionality requires one instance to act as the NAT). 

Although this setup would normally exceed the AWS Free Tier limit, I was able to offset this charge entirely as AWS approved my $300 free credits application.

As the ECS (EC2) backend is inside a private subnet in order to prevent direct internet access to it for security reasons, the NAT instance is needed to pull Docker images from ECR (Elastic Container Registry).

### Frontend

The main choices for the frontend was to either deploy it in ECS-EC2 like the backend or instead use S3 + CloudFront. The latter was the obvious choice as it has the benefits of:
 - DDOS protection
 - Decoupling of frontend and backend logic
- TLS support through ACM (AWS Certificate Manager)
- Use of a CDN to cache content in edge locations while remaining a low cost option (effectively free) for a low traffic application such as mine.

ECS-EC2, on the other hand, would have introduced additional operational overhead with no clear benefits especially since I wasn’t using server-side rendering (SSR).

## Deployment

As mentioned previously, one of my goals was to be able to deploy this app without having to modify the existing code base. A key enabler of this was the use of an Infrastructure as Code (IAC) approach which allows the  provisioning IT infrastructure through code, rather through graphical interfaces. This code becomes a version-controlled, repeatable template that can be deployed multiple times in different environments.

I chose to use CloudFormation for my IAC. Which provides the following benefits:

- The application's underlying code doesn't have to know it's using AWS, avoiding vendor lock in

- The application code does not need to call the AWS SDK to fetch secrets (e.g. db password). ECS injects them, which again keeps the app portable

- Can roll back changes in case of failure

- Reproducible deployments
 
Here we fetch the secrets from Secret Manager in the `ECSTaskDefinition`, avoiding hard-coding sensitive information in the template.

{% raw %}
```yaml
Environment:
- Name: SPRING_DATASOURCE_USERNAME
  Value: !Sub "{{resolve:secretsmanager:/taskapp/db-credentials:SecretString:username}}"
- Name: SPRING_DATASOURCE_PASSWORD
  Value: !Sub "{{resolve:secretsmanager:/taskapp/db-credentials:SecretString:password}}"
- Name: JWT_SECRET
  Value: !Sub "{{resolve:secretsmanager:/taskapp/db-credentials:SecretString:jwt}}"
```
{% endraw %}

As I am using a t4g.small instance for my EC2 backed ECS which runs on ARM it was necessary to cross compile the docker image for the backend to ARM64 before pushing the image to ECR (Elastic Container Registry), so it could run on the instance. ECR is essentially just a private Docker image repository.

Given the full template is over 600 lines, here’s a concise summary of the main components:

- **Networking**: Sets up a VPC with public and private subnets across two Availability Zones, an Internet Gateway, and route tables. It also includes a NAT instance (as an alternative to a NAT Gateway) to allow private subnets to access the internet

- **Security Groups and IAM roles**: More on this later

- **ECS Task Definition and Service**: Defines an ECS task definition for the taskapp-container using a Docker image from ECR, An ECS service is created to run and maintain the desired count of tasks on the ECS cluster, integrating with the ALB.

- **ALB**: Creates an Application Load Balancer (ALB), a target group for the ECS service, and an ALB listener to forward HTTP traffic to the ECS tasks.

- **RDS (MySQL)**: Database: Provisions a MySQL RDS instance

- **S3 + CloudFront** for React Frontend: Sets up an S3 bucket to host the static React frontend, along with a bucket policy to allow CloudFront access

- **Outputs**: Outputs the CloudFront URL for the frontend, can be used as the CNAME target when configuring a custom subdomain (e.g., app.example.com).

One challenge when using CloudFormation at first was that if the stack failed to create, it would delete the logs making troubleshooting difficult, but there is an option to retain the logs even after stack deletion.

### Security Considerations

As much as possible, I tried to adhere to AWS security best practices for my deployment.


**Secrets Manager**
- Extremely important to not hardcode sensitive secrets in CloudFormation, there have been cases of compromised secrets through this, leading to AWS bills in the 10's of thousands! 
- Database secrets and JWT token are encrypted by AES-256 and fetched from the SecretsManager

**Session Manager vs Instance Connect**
- AWS recommends using Session Manager instead of Instance Connect
- Allows SSH port to be closed, reducing the attack surface

**CloudFront**
- DDOS protection
- ViewerProtocolPolicy: redirect-to-https forces secure access
- Security headers set to the AWS managed security policy
- S3 access control	Using OriginAccessControlId, only CloudFront can access S3


![CloudFrontHeaders](/images/cloudfront.PNG)

**We can see Security Headers such as HSTS and X-Frame Option being set from the managed security policy**

**RDS** 
- The database is not publicly accessible, reducing the attack surface
- RDSToECSTaskIngressRule: Allows TCP 3306 from **ECSTaskSecurityGroup**. Restricts database access solely to the ECS tasks that need to connect to it
- RDS in contained within a **private subnet**
- Secrets obtained from Secrets Manager

**VPC & Networking**:
Provides a dedicated, isolated network

**Security Token Service (STS)**
- STS: AssumeRole provides temporary credentials eliminating the need to store access keys in code. This eliminates the risk of credentials accidentally being exposed (e.g., in Git repos or logs)
- AssumeRolePolicyDocument defines who can assume the role

```yaml
  ECSInstanceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

**AWS Certificate Manager (ACM)**
- TLS certificate generated in ACM through DNS validation
- ACM can autorenew the certificate
- The certificate can be attached to CloudFront, enabling HTTPS for the frontend with a custom domain

## Lessons learnt
- Use correct AMI
- Be careful to research the cost of using each service
- Setup billing alarms to keep an eye on costs incurred 
- Avoid CloudFormation stack rollback on failure: Enable log retention and manual deletion for better debugging
- Certificate for CloudFront must be created in **us-east-1** region
- Debugging Tip: Early Error Detection with ECS Task Status and Health Checks

## Future Enhancements
- Add CI/CD pipeline to upload frontend code to S3 to invalidate the cache and upload backend to ECR
- In a production environment with real users, enable CloudTrail and GuardDuty for increased security (cost involved)
- If cost wasn't an issue, use NAT Gateway instead of a NAT instance
- Could use the Cloud Development Kit (CDK) to create the CloudFormation template

## Conclusion
I really enjoyed completing this project and deepening my understanding of AWS infrastructure. It gave me hands-on experience with IaC, container orchestration, and secure cloud deployment



