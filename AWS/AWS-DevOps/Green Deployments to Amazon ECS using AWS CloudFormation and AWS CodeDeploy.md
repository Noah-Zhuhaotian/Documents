# Blue/Green Deployments to Amazon ECS using AWS CloudFormation and AWS CodeDeploy

### Overview
For running critical container-based applications on AWS, we can use ECS to deploy safely and change  infrastructure with minimal downtime. We can leverage AWS CodeDeploy and AWS CloudFormation to fullfill the requirements. AWS CloudFormation natively supports performing Blue/Green deployments on ECS using a CodeDeploy Blue/Green hook. one of them is the inability to use CloudFormation nested stacks, and another is the inability to update application and infrastructure changes in a single deployment.

### Architecture
The diagram below shows a reference CICD pipeline for orchestrating a Blue/Green deployment for an ECS application. In this reference architecture, we assume that you are deploying both infrastructure and application changes through the same pipeline.<br/><br/>
![Blue/Green Architecture](/assets/images/AWS/blue-green.png)

The pipeline consists of the following stages:
1. **Source:** In the source stage, CodePipeline pulls the code from the source repository, such as AWS CodeCommit or GitHub, and stages the changes in S3.
2. **Build:** In the build stage, you use CodeBuild to package CloudFormation templates, perform static analysis for the application code as well as the application infrastructure templates, run unit tests, build the application code, and generate and publish the application container image to ECR. These steps can be performed using a series of CodeBuild steps as described in the reference pipeline above.
3. **Deploy Infrastructure:** In the deploy stage, you leverage CodePipeline’s CloudFormation deploy action to deploy or update the application infrastructure. In this stage, the entire application infrastructure is set up using CloudFormation nested stacks. This includes the components required to perform Blue/Green deployments on ECS using CodeDeploy, such as the ECS Cluster, ECS Service, Task definition, Application Load Balancer (ALB) listeners, target groups, CodeDeploy application, deployment group, and others.
4. **Deploy Application:** In the deploy application stage, you use the CodePipeline ECS-to-CodeDeploy action to deploy your application changes using CodeDeploy’s blue/green deployment capability. By leveraging CodeDeploy, you can automate the blue/green deployment workflow for your applications running on ECS, including testing of your application after deployment and automated rollbacks in case of failed deployments. CodeDeploy also offers different ways to switch traffic for your application during a blue/green deployment by supporting Linear, Canary, and All-at-once traffic shifting options. 

### Considerations
Some considerations that you may need to account for when implementing the above reference pipeline

1. **Creating the CodeDeploy deployment group using CloudFormation**
- For performing Blue/Green deployments using CodeDeploy on ECS, CloudFormation currently does not support creating the CodeDeploy components directly as these components are created and managed by CloudFormation through the `AWS::CodeDeploy::BlueGreen` hook. To work around this, you can leverage a CloudFormation custom resource implemented through an AWS Lambda function, to create the CodeDeploy Deployment group with the required configuration.

2. **Generating the required code deploy artifacts (`appspec.yml` and `taskdef.json`)**
- For leveraging the CodeDeployToECS action in CodePipeline, there are two input files (`appspec.yml` and `taskdef.json`) that are needed. These files/artifacts are used by CodePipeline to create a CodeDeploy deployment that performs Blue/Green deployment on your ECS cluster. The AppSpec file specifies an Amazon ECS task definition for the deployment, a container name and port mapping used to route traffic, and the Lambda functions that run after deployment lifecycle hooks. The container name must be a container in your Amazon ECS task definition. The `taskdef.json` is used by CodePipeline to dynamically generate a new revision of the task definition with the updated application container image in ECR. This is an optional capability supported by the CodeDeployToECS action where it can automatically replace a place holder value (for example `IMAGE1_NAME`) for ImageUri in the `taskdef.json` with the Uri of the updated container Image. In the reference solution we do not use this capability as our `taskdef.json` contains the latest ImageUri that we plan to deploy. To create this `taskdef.json`, you can leverage CodeBuild to dynamically build the `taskdef.json` from the latest task definition ARN. Below are sample CodeBuild buildspec commands that creates the `taskdef.json` from ECS task definition.
```yaml
build:
  commands:
    # Create appspec.yml for CodeDeploy deployment
    - python iac/code-deploy/scripts/update-appspec.py --taskArn ${TASKDEF_ARN} --hooksLambdaArn ${HOOKS_LAMBDA_ARN} --inputAppSpecFile 'iac/code-deploy/appspec.yml' --outputAppSpecFile '/tmp/appspec.yml'
    # Create taskdefinition for CodeDeploy deployment
    - aws ecs describe-task-definition --task-definition ${TASKDEF_ARN} --region ${AWS_REGION} --query taskDefinition >> taskdef.json
  artifacts:
    files:
        - /tmp/appspec.yml
        - /tmp/taskdef.json
    discard-paths: yes
```
- To generate the `appspec.yml`, you can leverage a python or shell script and a placeholder `appspec.yml` in your source repository to dynamically generate the updated `appspec.yml` file. For example, the below code snippet updates the placeholder values in an `appspec.yml` to generate an updated appspec.yml that is used in the deploy stage. In this example, we set the values of `AfterAllowTestTraffic` hook, the Container name, Container port values from task definition and Hooks Lambda ARN that is passed as input to the script.
```bash
  contents = yaml.safe_load(file)
  print(contents)
  response = ecs.describe_task_definition(taskDefinition=taskArn)
  contents['Hooks'][0]['AfterAllowTestTraffic'] = hooksLambdaArn
  contents['Resources'][0]['TargetService']['Properties']['LoadBalancerInfo']['ContainerName'] = response['taskDefinition']['containerDefinitions'][0]['name']
  contents['Resources'][0]['TargetService']['Properties']['LoadBalancerInfo']['ContainerPort'] = response['taskDefinition']['containerDefinitions'][0]['portMappings'][0]['containerPort']
  contents['Resources'][0]['TargetService']['Properties']['TaskDefinition'] = taskArn

  print('Updated appspec.yaml contents')
  yaml.dump(contents, outputFile)
```
1. **Updates to the ECS task definition**

- To perform Blue/Green deployments on your ECS cluster using CodeDeploy, the deployment controller on the ECS Service needs to be set to CodeDeploy. With this configuration, any time there is an update to the task definition on the ECS service (such as when building new application image), the update results in a failure. This essentially causes CloudFormation updates to the application infrastructure to fail when new application changes are deployed. To avoid this, you can implement a CloudFormation based custom resource that obtains the previous version of task definition. This prevents CloudFormation from updating the ECS Service with new task definition when the application container image is updated and ultimately from failing the stack update. Updates to ECS Services for new task revisions are performed using the CodeDeploy deployment as outlined in `#2` above. Using this mechanism, you can update the application infrastructure along with changes to the application code using a single pipeline while also leveraging CodeDeploy Blue/Green deployment.

4. **Passing configuration between different stages of the pipeline**
- To create an automated pipeline that builds your infrastructure and performs a blue/green deployment for your application, you will need the ability to pass configuration between different stages of your pipeline. For example, when you want to create the `taskdef.json` and `appspec.yml` as mentioned in step `#2`, you need the ARN of the existing task definition and ARN of the CodeDeploy hook Lambda. These components are created in different stages within your pipeline. To facilitate this, you can leverage CodePipeline’s variables and namespaces. For example, in the CodePipeline stage below, we set the value of `TASKDEF_ARN` and `HOOKS_LAMBDA_ARN` environment variables by fetching those values from a different stage in the same pipeline where we create those components. An alternate option is to use AWS System Manager Parameter Store to store and retrieve that information.
```YAML
- Name: BuildCodeDeployArtifacts
  Actions:
    - Name: BuildCodeDeployArtifacts
  	  ActionTypeId:
        Category: Build
    		Owner: AWS
    		Provider: CodeBuild
    		Version: "1"
  	  Configuration:
    		ProjectName: !Sub "${pApplicationName}-CodeDeployConfigBuild"
  		  EnvironmentVariables: '[{"name": "TASKDEF_ARN", "value": "#{DeployInfraVariables.oTaskDefinitionArn}", "type": "PLAINTEXT"},{"name": "HOOKS_LAMBDA_ARN", "value": "#{DeployInfraVariables.oAfterInstallHookLambdaArn}", "type": "PLAINTEXT"}]'
  	  InputArtifacts:
        - Name: Source
      OutputArtifacts:
        - Name: CodeDeployConfig
  	  RunOrder: 1
```