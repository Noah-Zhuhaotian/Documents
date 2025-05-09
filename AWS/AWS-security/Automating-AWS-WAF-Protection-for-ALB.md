# Automating AWS WAF Protection for Application Load Balancers Using AWS Systems Manager

## Introduction

Web Application Firewalls (WAFs) are critical security components that help protect web applications from common vulnerabilities like SQL injection, cross-site scripting, and other OWASP Top 10 threats. In AWS environments, ensuring that all Application Load Balancers (ALBs) are properly protected with AWS WAF is an essential security practice. However, maintaining this protection at scale can be challenging, especially when changes are made outside of your deployment process.

This article shows how to implement an automated solution using AWS Systems Manager Automation documents that ensures AWS WAF is always enabled for all ALBs in your AWS account, even if it's accidentally removed after deployment.

## The Solution Architecture

Our approach consists of three key components:

1. **CloudFormation templates** to deploy ALBs with AWS WAF enabled by default
2. **AWS Config rules** to detect ALBs without WAF protection
3. **AWS Systems Manager Automation documents** to automatically reattach WAF to unprotected ALBs

Let's dive into each component.

## Part 1: CloudFormation Template for ALB with WAF

First, we'll create a CloudFormation template that deploys an ALB with AWS WAF already attached. This template will be used as part of each application stack's deployment process.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CloudFormation template for ALB with AWS WAF'

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC where the ALB will be deployed
  
  Subnets:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Subnets for the ALB
  
  ApplicationName:
    Type: String
    Description: Name of the application

Resources:
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub ${ApplicationName}-alb
      Subnets: !Ref Subnets
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Type: application
      Tags:
        - Key: Application
          Value: !Ref ApplicationName

  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for ALB
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - CidrIp: 0.0.0.0/0
          IpProtocol: tcp
          FromPort: 80
          ToPort: 80
        - CidrIp: 0.0.0.0/0
          IpProtocol: tcp
          FromPort: 443
          ToPort: 443

  # WAF Web ACL
  WAFWebACL:
    Type: AWS::WAFv2::WebACL
    Properties:
      Name: !Sub ${ApplicationName}-web-acl
      Scope: REGIONAL
      DefaultAction:
        Allow: {}
      VisibilityConfig:
        SampledRequestsEnabled: true
        CloudWatchMetricsEnabled: true
        MetricName: !Sub ${ApplicationName}-WebACL
      Rules:
        - Name: AWSManagedRulesCommonRuleSet
          Priority: 0
          OverrideAction:
            None: {}
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: AWSManagedRulesCommonRuleSet
          Statement:
            ManagedRuleGroupStatement:
              VendorName: AWS
              Name: AWSManagedRulesCommonRuleSet

  # Association between ALB and WAF WebACL
  WebACLAssociation:
    Type: AWS::WAFv2::WebACLAssociation
    Properties:
      ResourceArn: !Ref ApplicationLoadBalancer
      WebACLArn: !GetAtt WAFWebACL.Arn

Outputs:
  ALBArn:
    Description: ARN of the Application Load Balancer
    Value: !Ref ApplicationLoadBalancer
  
  WAFWebACLArn:
    Description: ARN of the WAF Web ACL
    Value: !GetAtt WAFWebACL.Arn
```

This template creates:
- An Application Load Balancer
- A Security Group for the ALB
- A WAF Web ACL with AWS Managed Rules
- An association between the ALB and WAF Web ACL

## Part 2: AWS Config Rule to Detect Unprotected ALBs

Next, we'll implement an AWS Config custom rule to detect ALBs that don't have AWS WAF enabled. This rule will use an AWS Config managed rule, eliminating the need for Lambda functions.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'AWS Config rule to check WAF on ALBs'

Resources:
  # AWS Config Managed Rule
  ALBWAFConfigRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: alb-waf-enabled
      Description: Checks if ALBs are associated with a WAF Web ACL
      Source:
        Owner: AWS
        SourceIdentifier: ALB_WAF_ENABLED
      Scope:
        ComplianceResourceTypes:
          - AWS::ElasticLoadBalancingV2::LoadBalancer
```

This template uses the AWS managed Config rule `ALB_WAF_ENABLED`, which automatically checks if your Application Load Balancers have AWS WAF enabled. Using a managed rule simplifies the implementation and ensures that the rule stays up-to-date with AWS best practices.

If you need more customization, you can alternatively use a custom rule with AWS Lambda, but the managed rule is sufficient for most use cases and eliminates the need to maintain custom code.

## Part 3: Automated Remediation with AWS Systems Manager Automation

Instead of using Lambda for remediation, we'll leverage AWS Systems Manager Automation documents to automatically attach a default WAF Web ACL to any ALB that is found to be unprotected. This approach provides better visibility, tracking, and easier maintenance.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Automated remediation for ALBs without WAF using Systems Manager'

Parameters:
  DefaultWebACLArn:
    Type: String
    Description: ARN of the default WAF Web ACL to attach to unprotected ALBs

Resources:
  # IAM Role for Systems Manager Automation
  SSMAutomationRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ssm.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMAutomationRole
      Policies:
        - PolicyName: WAFRemediation
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - wafv2:AssociateWebACL
                  - wafv2:GetWebACL
                  - wafv2:ListResourcesForWebACL
                  - elasticloadbalancing:DescribeLoadBalancers
                Resource: '*'

  # Systems Manager Automation Document
  AttachWAFToALBDocument:
    Type: AWS::SSM::Document
    Properties:
      DocumentType: Automation
      Name: AttachWAFToALB
      Content:
        schemaVersion: '0.3'
        assumeRole: !GetAtt SSMAutomationRole.Arn
        description: 'Attaches WAF WebACL to an unprotected ALB'
        parameters:
          ALBArn:
            type: String
            description: 'ARN of the ALB to protect'
          WebACLArn:
            type: String
            description: 'ARN of the WAF WebACL to attach'
            default: !Ref DefaultWebACLArn
        mainSteps:
          - name: CheckCurrentWAFStatus
            action: aws:executeAwsApi
            inputs:
              Service: wafv2
              Api: GetWebACLForResource
              ResourceArn: '{{ ALBArn }}'
            onFailure: 'step:AssociateWAFWithALB'
            nextStep: ValidateWAFAssociation
            isEnd: false
          
          - name: AssociateWAFWithALB
            action: aws:executeAwsApi
            inputs:
              Service: wafv2
              Api: AssociateWebACL
              ResourceArn: '{{ ALBArn }}'
              WebACLArn: '{{ WebACLArn }}'
            isEnd: false
            nextStep: ValidateWAFAssociation
          
          - name: ValidateWAFAssociation
            action: aws:executeAwsApi
            inputs:
              Service: wafv2
              Api: GetWebACLForResource
              ResourceArn: '{{ ALBArn }}'
            onSuccess: VerifyCorrectWebACL
            onFailure: Fail
            isEnd: false
          
          - name: VerifyCorrectWebACL
            action: aws:branch
            inputs:
              Choices:
                - NextStep: Success
                  Variable: '{{ ValidateWAFAssociation.WebACLSummary.ARN }}'
                  StringEquals: '{{ WebACLArn }}'
              Default: Fail
          
          - name: Success
            action: aws:executeScript
            inputs:
              Runtime: python3.8
              Handler: success_handler
              Script: |-
                def success_handler(events, context):
                  return {
                    'status': 'success',
                    'message': 'WAF WebACL successfully attached to ALB'
                  }
            isEnd: true
          
          - name: Fail
            action: aws:executeScript
            inputs:
              Runtime: python3.8
              Handler: failure_handler
              Script: |-
                def failure_handler(events, context):
                  return {
                    'status': 'failed',
                    'message': 'Failed to attach WAF WebACL to ALB'
                  }
            isEnd: true

  # AWS Config Remediation Configuration
  ConfigRemediation:
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: alb-waf-enabled
      ResourceType: AWS::ElasticLoadBalancingV2::LoadBalancer
      TargetId: AttachWAFToALB
      TargetType: SSM_DOCUMENT
      TargetVersion: '1'
      Parameters:
        ALBArn:
          ResourceValue:
            Value: RESOURCE_ID
        WebACLArn:
          StaticValue:
            Values: 
              - !Ref DefaultWebACLArn
      Automatic: true
      MaximumAutomaticAttempts: 5
      RetryAttemptSeconds: 60
```

## CloudWatch Events for Real-Time Protection

For even faster protection, we can set up a CloudWatch Event that triggers the Systems Manager Automation document whenever a WAF association is removed:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CloudWatch Events for WAF disassociation with SSM Automation'

Parameters:
  DefaultWebACLArn:
    Type: String
    Description: ARN of the default WAF Web ACL to attach to unprotected ALBs

Resources:
  WAFDisassociationEventRule:
    Type: AWS::Events::Rule
    Properties:
      Name: waf-disassociation-event
      Description: "Rule to detect WAF disassociation from resources"
      EventPattern:
        source:
          - aws.wafv2
        detail-type:
          - "AWS API Call via CloudTrail"
        detail:
          eventSource:
            - wafv2.amazonaws.com
          eventName:
            - DisassociateWebACL
      State: ENABLED
      Targets:
        - Arn: !Sub arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:automation-definition/AttachWAFToALB:$DEFAULT
          Id: WAFDisassociationTarget
          RoleArn: !GetAtt EventsInvokeSSMRole.Arn
          InputTransformer:
            InputPathsMap:
              resourceArn: "$.detail.requestParameters.resourceArn"
            InputTemplate: |
              {
                "ALBArn": <resourceArn>,
                "WebACLArn": "${DefaultWebACLArn}"
              }

  # IAM Role for EventBridge to invoke SSM Automation
  EventsInvokeSSMRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: events.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: SSMAutomationExecution
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ssm:StartAutomationExecution
                Resource: !Sub arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:automation-definition/AttachWAFToALB:*
```

## Implementation and Deployment

To deploy this solution:

1. Create a default WAF Web ACL that will be used as the fallback protection
2. Deploy the CloudFormation templates in the following order:
   - Default WAF WebACL template
   - AWS Config Rule template
   - Systems Manager Automation document template
   - CloudWatch Events template
3. Update your application deployment process to use the ALB with WAF template

## Testing the Solution

To verify that the solution works as expected:

1. Deploy an ALB with WAF using the template
2. Manually remove the WAF association through the AWS Console
3. Wait for a few minutes and check if the WAF was automatically reattached

## Monitoring and Alerting

Set up monitoring to keep track of the system's health:

1. Create CloudWatch Alarms for the AWS Config rule to alert when non-compliant resources are detected
2. Monitor Systems Manager Automation executions through the AWS Systems Manager console
3. Use AWS Systems Manager OpsCenter to track and resolve operational issues
4. Create a dashboard to visualize WAF protection status across all ALBs

One major advantage of using Systems Manager Automation over Lambda is the built-in execution history and tracking capabilities. You can easily view all automation executions, their statuses, and detailed logs directly in the AWS Systems Manager console.

## Conclusion

By implementing this automated solution, you can ensure that all Application Load Balancers in your AWS account are always protected by AWS WAF, even if someone inadvertently removes the protection after deployment. This approach leverages AWS native services like CloudFormation, AWS Config, and Lambda to establish a self-healing infrastructure that maintains your security posture with minimal manual intervention.

Remember that this is just one layer of your defense-in-depth strategy. It should be complemented with other security controls like proper IAM policies, security groups, and continuous monitoring to provide comprehensive protection for your applications.

## Next Steps

Consider these enhancements to further improve your security posture:

1. Implement custom WAF rules specific to your applications
2. Add tagging strategies to identify which WAF rules should be applied to different ALBs
3. Extend the solution to protect other resources like API Gateway and CloudFront distributions
4. Implement compliance reporting to track protection status over time
5. Create additional Systems Manager Automation documents for other security controls
6. Integrate with AWS Security Hub for centralized security management

## Advantages of Using Systems Manager Automation

Using AWS Systems Manager Automation for this solution offers several key benefits:

1. **Visual workflow representation**: The step-by-step automation flow is clearly visible in the AWS console, making it easier to understand and troubleshoot.

2. **Built-in approval process**: You can add approval steps to the automation if you want manual verification before some critical actions.

3. **Detailed execution tracking**: Every execution of the automation document is tracked with comprehensive logs, making auditing and troubleshooting easier.

4. **Integration with other AWS services**: Systems Manager integrates seamlessly with AWS Config, CloudWatch Events, and other AWS services.

5. **Centralized operation management**: Through Systems Manager OpsCenter, you can track and manage all automation executions in one place.

6. **Simplified maintenance**: Updating the automation logic is as simple as modifying the document, without having to redeploy Lambda functions.

With these tools and practices in place, you can confidently deploy and maintain secure applications on AWS while ensuring consistent WAF protection across your entire environment.