# Deploying a Java Application to Amazon EC2 Using CodePipeline and CodeDeploy (with GitHub OAuth Token Managed by Secrets Manager)

### Introduction
In this article, let's walk through how to fully automate deploying a Java application (packaged as an RPM or a WAR) to a fleet of Amazon EC2 instances using AWS CodePipeline and CodeDeploy.
We will securely manage the GitHub OAuth token using AWS Secrets Manager to avoid storing sensitive information in plaintext.
We will cover:
- Preparing the EC2 environment (pre-baked AMI with CodeDeploy agent installed)
- Setting up IAM roles
- Building a CodePipeline
- Automating the build and deploy process
- Managing GitHub OAuth token securely with Secrets Manager


### Step 1: Prepare the Pre-baked AMI

Instead of manually installing the CodeDeploy agent on each instance, we automate the AMI creation using Packer.

Steps:
1. Launch a base Amazon Linux 2 EC2 instance.
2. Install CodeDeploy agent:
   ```bash
   sudo yum update -y
   sudo yum install ruby wget -y
   cd /home/ec2-user
   wget https://aws-codedeploy-${region}.s3.${region}.amazonaws.com/latest/install
   chmod +x ./install
   sudo ./install auto
   sudo systemctl start codedeploy-agent
   sudo systemctl enable codedeploy-agent
   ```
3. Create an AMI from this instance.

This AMI will be used in your Auto Scaling Group or Launch Template.

### Step 2: Set up IAM Roles
You need three IAM roles:
- **CodePipelineRole:** for managing the pipeline.
- **CodeBuildRole:** for building the application.
- **EC2 Instance Role:** for CodeDeploy agent to communicate with AWS services.

Here’s a brief of the permissions:
- `codepipeline.amazonaws.com` and `codebuild.amazonaws.com` trust policies.
- Permissions to access CodeDeploy, CodeBuild, S3, Secrets Manager, and EC2.

### Step 3: Securely Manage GitHub OAuth Token with Secrets Manager
Never store sensitive tokens in plaintext!
Instead:
1. Create a secret in Secrets Manager:
   ```bash
   aws secretsmanager create-secret \
   --name codepipeline-github-token \
   --description "GitHub OAuth Token for CodePipeline" \
   --secret-string '{"github-token":"your-real-oauth-token"}'
   ```

2. In CodePipeline, reference it securely:
   ```yaml
   OAuthToken: '{{resolve:secretsmanager:codepipeline-github-token:SecretString:github-token}}'
   ```

IAM roles need `secretsmanager:GetSecretValue` permission.

### Step 4: Create the CodePipeline
Here’s a basic structure of the `pipeline.yaml`:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CodePipeline with secure GitHub OAuth token handling.

Parameters:
  GitHubRepo:
    Type: String
    Description: GitHub repository name (e.g., 'yourname/repo')
  GitHubBranch:
    Type: String
    Default: main
    Description: GitHub branch
  GitHubOAuthSecretName:
    Type: String
    Description: Secrets Manager secret name
  CodeDeployApplicationName:
    Type: String
  CodeDeployDeploymentGroupName:
    Type: String
  S3BucketArtifacts:
    Type: String
  CodeBuildProjectName:
    Type: String

Resources:
  # CodePipelineRole
  # CodeBuildRole
  # CodeBuildProject
  # Pipeline
```
Notes:
- Source Stage pulls from GitHub (via secure token).
- Build Stage compiles the Java application.
- Deploy Stage deploys using CodeDeploy.

### Step 5: Buildspec for CodeBuild
Create a `buildspec.yml` file in your repository:
```yaml
version: 0.2

phases:
  install:
    commands:
      - echo Installing Maven...
      - yum install -y maven
  build:
    commands:
      - echo Building Java application...
      - mvn clean package
  post_build:
    commands:
      - echo Build completed

artifacts:
  files:
    - target/*.war
    - appspec.yml
    - scripts/deploy.sh
```

### Step 6: Deployment Scripts and AppSpec
- `appspec.yml` (for CodeDeploy):
  ```yaml
  version: 0.0
  os: linux
  files:
    - source: /
      destination: /opt/tomcat/webapps/
  hooks:
    AfterInstall:
      - location: scripts/deploy.sh
        timeout: 300
        runas: root
  ```
- `scripts/deploy.sh` (simple deploy script):
  ```bash
  #!/bin/bash
  echo "Deploying Java application..."
  systemctl restart tomcat
  ```


### Step 7: One-Click Deploy

Once everything is set:
- Upload your CloudFormation stack
- CodePipeline automatically triggers:
  - Pulls source from GitHub
  - Builds with CodeBuild
  - Deploys to EC2 with CodeDeploy
No manual steps required!

*Full Repository Structure Example*
```css
/ (root)
 ├── appspec.yml
 ├── buildspec.yml
 ├── scripts/
 │    └── deploy.sh
 └── src/
      └── main/
          └── java/
```

[Demo templetes]()

### Conclusion

Using AWS CodePipeline, CodeBuild, and CodeDeploy, you can create a fully automated, secure CI/CD pipeline for your Java applications.
By managing secrets properly with Secrets Manager, you enhance your system’s security and scalability.

This architecture can be expanded to support more complex build, test, and deployment flows.