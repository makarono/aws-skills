# CloudFormation Patterns and Best Practices

This document provides comprehensive patterns, anti-patterns, and best practices for AWS CloudFormation development.

## Table of Contents

1. [Template Organization Patterns](#template-organization-patterns)
2. [Resource Patterns](#resource-patterns)
3. [Security Patterns](#security-patterns)
4. [Cost Optimization Patterns](#cost-optimization-patterns)
5. [Common Anti-Patterns](#common-anti-patterns)
6. [Advanced Patterns](#advanced-patterns)

## Template Organization Patterns

### Pattern: Modular Templates with Nested Stacks

**Use Case**: Breaking down complex infrastructure into manageable, reusable components.

**Implementation**:

```yaml
# master-template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Master stack orchestrating infrastructure components

Parameters:
  EnvironmentType:
    Type: String
    AllowedValues: [dev, staging, prod]

Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub 'https://${TemplateBucket}.s3.amazonaws.com/network.yaml'
      Parameters:
        EnvironmentType: !Ref EnvironmentType
        VpcCIDR: 10.0.0.0/16
      Tags:
        - Key: Component
          Value: Network

  SecurityStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: !Sub 'https://${TemplateBucket}.s3.amazonaws.com/security.yaml'
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId

  ApplicationStack:
    Type: AWS::CloudFormation::Stack
    DependsOn:
      - NetworkStack
      - SecurityStack
    Properties:
      TemplateURL: !Sub 'https://${TemplateBucket}.s3.amazonaws.com/application.yaml'
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
        SecurityGroupId: !GetAtt SecurityStack.Outputs.AppSecurityGroupId
```

**Benefits**:
- Separation of concerns
- Reusable templates
- Easier testing and maintenance
- Parallel stack updates where possible

### Pattern: Parameter-Driven Configuration

**Use Case**: Creating flexible templates that work across environments.

```yaml
Parameters:
  EnvironmentType:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]

Mappings:
  EnvironmentConfig:
    dev:
      InstanceType: t3.micro
      MinSize: 1
      MaxSize: 2
      AlarmEmail: dev-team@example.com
    staging:
      InstanceType: t3.small
      MinSize: 2
      MaxSize: 4
      AlarmEmail: staging-alerts@example.com
    prod:
      InstanceType: t3.medium
      MinSize: 4
      MaxSize: 10
      AlarmEmail: prod-alerts@example.com

Conditions:
  IsProduction: !Equals [!Ref EnvironmentType, prod]
  CreateHighAvailability: !Or
    - !Equals [!Ref EnvironmentType, staging]
    - !Equals [!Ref EnvironmentType, prod]

Resources:
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !FindInMap [EnvironmentConfig, !Ref EnvironmentType, InstanceType]
      Monitoring: !If [IsProduction, true, false]

  BackupPlan:
    Type: AWS::Backup::BackupPlan
    Condition: IsProduction
    Properties:
      # Backup configuration only for production
```

## Resource Patterns

### Pattern: Lambda Function with Dependencies

**Use Case**: Deploying Lambda functions with proper IAM roles and environment configuration.

```yaml
Resources:
  FunctionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: DynamoDBAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:PutItem
                  - dynamodb:UpdateItem
                  - dynamodb:Query
                Resource: !GetAtt DataTable.Arn

  ProcessorFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.11
      Handler: index.handler
      Role: !GetAtt FunctionRole.Arn
      Timeout: 30
      MemorySize: 256
      Environment:
        Variables:
          TABLE_NAME: !Ref DataTable
          ENVIRONMENT: !Ref EnvironmentType
      Code:
        ZipFile: |
          import json
          import os
          import boto3

          dynamodb = boto3.resource('dynamodb')
          table = dynamodb.Table(os.environ['TABLE_NAME'])

          def handler(event, context):
              # Function logic here
              return {
                  'statusCode': 200,
                  'body': json.dumps('Success')
              }

  FunctionLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/aws/lambda/${ProcessorFunction}'
      RetentionInDays: 7

  DataTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
```

### Pattern: API Gateway with Lambda Integration

**Use Case**: Creating a RESTful API with Lambda backend.

```yaml
Resources:
  RestApi:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: !Sub '${AWS::StackName}-api'
      Description: API for processing requests
      EndpointConfiguration:
        Types:
          - REGIONAL

  ApiResource:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref RestApi
      ParentId: !GetAtt RestApi.RootResourceId
      PathPart: items

  ApiMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref RestApi
      ResourceId: !Ref ApiResource
      HttpMethod: POST
      AuthorizationType: NONE
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: POST
        Uri: !Sub 'arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${ProcessorFunction.Arn}/invocations'

  ApiDeployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn:
      - ApiMethod
    Properties:
      RestApiId: !Ref RestApi

  ApiStage:
    Type: AWS::ApiGateway::Stage
    Properties:
      RestApiId: !Ref RestApi
      DeploymentId: !Ref ApiDeployment
      StageName: prod
      TracingEnabled: true
      MethodSettings:
        - ResourcePath: /*
          HttpMethod: '*'
          LoggingLevel: INFO
          DataTraceEnabled: true
          MetricsEnabled: true

  LambdaApiPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref ProcessorFunction
      Action: lambda:InvokeFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${RestApi}/*/*/*'

Outputs:
  ApiEndpoint:
    Description: API Gateway endpoint URL
    Value: !Sub 'https://${RestApi}.execute-api.${AWS::Region}.amazonaws.com/prod'
    Export:
      Name: !Sub '${AWS::StackName}-ApiEndpoint'
```

### Pattern: VPC with Public and Private Subnets

**Use Case**: Creating a multi-AZ VPC with proper subnet configuration.

```yaml
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-vpc'

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-igw'

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Select [0, !Cidr [!Ref VpcCIDR, 6, 8]]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-public-subnet-1'

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Select [1, !Cidr [!Ref VpcCIDR, 6, 8]]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-public-subnet-2'

  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Select [2, !Cidr [!Ref VpcCIDR, 6, 8]]
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-private-subnet-1'

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Select [3, !Cidr [!Ref VpcCIDR, 6, 8]]
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-private-subnet-2'

  NatGatewayEIP1:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc

  NatGateway1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIP1.AllocationId
      SubnetId: !Ref PublicSubnet1

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-public-routes'

  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2

  PrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-private-routes-1'

  DefaultPrivateRoute1:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway1

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      SubnetId: !Ref PrivateSubnet1

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      SubnetId: !Ref PrivateSubnet2

Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${AWS::StackName}-VpcId'

  PublicSubnetIds:
    Description: Public subnet IDs
    Value: !Join [',', [!Ref PublicSubnet1, !Ref PublicSubnet2]]
    Export:
      Name: !Sub '${AWS::StackName}-PublicSubnetIds'

  PrivateSubnetIds:
    Description: Private subnet IDs
    Value: !Join [',', [!Ref PrivateSubnet1, !Ref PrivateSubnet2]]
    Export:
      Name: !Sub '${AWS::StackName}-PrivateSubnetIds'
```

## Security Patterns

### Pattern: Least Privilege IAM Roles

**Use Case**: Creating IAM roles with minimum necessary permissions.

```yaml
Resources:
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: SpecificResourceAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              # Specific DynamoDB table access only
              - Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:PutItem
                Resource: !GetAtt SpecificTable.Arn
              # Specific S3 bucket access only
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                Resource: !Sub '${DataBucket.Arn}/*'
              # Specific KMS key for encryption
              - Effect: Allow
                Action:
                  - kms:Decrypt
                  - kms:GenerateDataKey
                Resource: !GetAtt EncryptionKey.Arn
```

### Pattern: Encrypted Resources

**Use Case**: Ensuring data encryption at rest and in transit.

```yaml
Resources:
  EncryptionKey:
    Type: AWS::KMS::Key
    Properties:
      Description: Key for encrypting application data
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: kms:*
            Resource: '*'
          - Sid: Allow services to use the key
            Effect: Allow
            Principal:
              Service:
                - s3.amazonaws.com
                - dynamodb.amazonaws.com
                - logs.amazonaws.com
            Action:
              - kms:Decrypt
              - kms:GenerateDataKey
            Resource: '*'

  EncryptionKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: !Sub 'alias/${AWS::StackName}-encryption-key'
      TargetKeyId: !Ref EncryptionKey

  EncryptedBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !Ref EncryptionKey
            BucketKeyEnabled: true
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      VersioningConfiguration:
        Status: Enabled

  EncryptedTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      SSESpecification:
        SSEEnabled: true
        SSEType: KMS
        KMSMasterKeyId: !Ref EncryptionKey
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
```

### Pattern: Security Group with Minimal Access

**Use Case**: Creating security groups with explicit, minimal rules.

```yaml
Resources:
  ApplicationSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for application instances
      VpcId: !Ref VPC
      SecurityGroupIngress:
        # Only allow HTTPS from ALB security group
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
      SecurityGroupEgress:
        # Only allow HTTPS to specific endpoints
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          DestinationPrefixListId: !Ref S3PrefixList
          Description: S3 access via VPC endpoint
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          DestinationPrefixListId: !Ref DynamoDBPrefixList
          Description: DynamoDB access via VPC endpoint

  DatabaseSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for RDS database
      VpcId: !Ref VPC
      SecurityGroupIngress:
        # Only allow database port from application security group
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !Ref ApplicationSecurityGroup
          Description: PostgreSQL from application tier
      # No egress rules - database doesn't need outbound access
      SecurityGroupEgress: []
```

## Cost Optimization Patterns

### Pattern: Environment-Specific Sizing

**Use Case**: Right-sizing resources based on environment needs.

```yaml
Mappings:
  EnvironmentSize:
    dev:
      InstanceType: t3.micro
      DBInstanceClass: db.t3.micro
      CacheNodeType: cache.t3.micro
      DesiredCapacity: 1
    staging:
      InstanceType: t3.small
      DBInstanceClass: db.t3.small
      CacheNodeType: cache.t3.small
      DesiredCapacity: 2
    prod:
      InstanceType: t3.medium
      DBInstanceClass: db.r6g.large
      CacheNodeType: cache.r6g.large
      DesiredCapacity: 4

Resources:
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      MinSize: !FindInMap [EnvironmentSize, !Ref EnvironmentType, DesiredCapacity]
      MaxSize: !If
        - IsProduction
        - 10
        - !FindInMap [EnvironmentSize, !Ref EnvironmentType, DesiredCapacity]
      DesiredCapacity: !FindInMap [EnvironmentSize, !Ref EnvironmentType, DesiredCapacity]
```

### Pattern: Lifecycle Policies for Cost Savings

**Use Case**: Implementing S3 lifecycle policies to reduce storage costs.

```yaml
Resources:
  DataBucket:
    Type: AWS::S3::Bucket
    Properties:
      LifecycleConfiguration:
        Rules:
          - Id: TransitionToIA
            Status: Enabled
            Transitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
              - TransitionInDays: 90
                StorageClass: GLACIER
            ExpirationInDays: 365
          - Id: DeleteIncompleteMultipartUploads
            Status: Enabled
            AbortIncompleteMultipartUpload:
              DaysAfterInitiation: 7
          - Id: CleanupOldVersions
            Status: Enabled
            NoncurrentVersionTransitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
            NoncurrentVersionExpirationInDays: 90
```

### Pattern: Auto Scaling for Cost Efficiency

**Use Case**: Implementing auto-scaling to match capacity with demand.

```yaml
Resources:
  DynamoDBTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PROVISIONED
      ProvisionedThroughput:
        ReadCapacityUnits: 5
        WriteCapacityUnits: 5
      # ... other properties

  TableReadScalableTarget:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Properties:
      ServiceNamespace: dynamodb
      ResourceId: !Sub 'table/${DynamoDBTable}'
      ScalableDimension: dynamodb:table:ReadCapacityUnits
      MinCapacity: 5
      MaxCapacity: 100
      RoleARN: !Sub 'arn:aws:iam::${AWS::AccountId}:role/aws-service-role/dynamodb.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_DynamoDBTable'

  TableReadScalingPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Properties:
      ServiceNamespace: dynamodb
      ResourceId: !Sub 'table/${DynamoDBTable}'
      ScalableDimension: dynamodb:table:ReadCapacityUnits
      PolicyName: ReadAutoScalingPolicy
      PolicyType: TargetTrackingScaling
      TargetTrackingScalingPolicyConfiguration:
        TargetValue: 70.0
        PredefinedMetricSpecification:
          PredefinedMetricType: DynamoDBReadCapacityUtilization
```

## Common Anti-Patterns

### Anti-Pattern: Hard-Coded Resource Names

**Problem**: Prevents template reusability and parallel deployments.

```yaml
# ❌ BAD
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-app-data-bucket  # Hard-coded name

# ✅ GOOD
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      # Let CloudFormation generate unique name
      Tags:
        - Key: Purpose
          Value: ApplicationData
```

### Anti-Pattern: Overly Permissive IAM Policies

**Problem**: Violates principle of least privilege, increases security risk.

```yaml
# ❌ BAD
Resources:
  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      Policies:
        - PolicyName: TooMuchAccess
          PolicyDocument:
            Statement:
              - Effect: Allow
                Action: '*'  # All actions
                Resource: '*'  # All resources

# ✅ GOOD
Resources:
  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      Policies:
        - PolicyName: SpecificAccess
          PolicyDocument:
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:PutItem
                Resource: !GetAtt SpecificTable.Arn
```

### Anti-Pattern: Missing DeletionPolicy on Stateful Resources

**Problem**: Risk of data loss when stack is deleted or updated.

```yaml
# ❌ BAD
Resources:
  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      # No DeletionPolicy - database deleted with stack!

# ✅ GOOD
Resources:
  Database:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    UpdateReplacePolicy: Snapshot
    Properties:
      # Database settings
```

### Anti-Pattern: No Cross-Stack References for Shared Resources

**Problem**: Duplicating resources or tight coupling between stacks.

```yaml
# ❌ BAD - Duplicating VPC in every stack
Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

# ✅ GOOD - Reference shared VPC
Resources:
  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !ImportValue 'NetworkStack-VpcId'
```

## Advanced Patterns

### Pattern: Custom Resources with Lambda

**Use Case**: Implementing custom logic during stack lifecycle.

```yaml
Resources:
  CustomResourceFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.11
      Handler: index.handler
      Role: !GetAtt CustomResourceRole.Arn
      Timeout: 300
      Code:
        ZipFile: |
          import json
          import boto3
          import cfnresponse

          def handler(event, context):
              try:
                  request_type = event['RequestType']

                  if request_type == 'Create':
                      # Custom create logic
                      result = create_resource(event)
                  elif request_type == 'Update':
                      # Custom update logic
                      result = update_resource(event)
                  elif request_type == 'Delete':
                      # Custom delete logic
                      result = delete_resource(event)

                  cfnresponse.send(event, context, cfnresponse.SUCCESS, result)
              except Exception as e:
                  print(f"Error: {str(e)}")
                  cfnresponse.send(event, context, cfnresponse.FAILED, {})

  CustomResource:
    Type: Custom::MyCustomResource
    Properties:
      ServiceToken: !GetAtt CustomResourceFunction.Arn
      Property1: !Ref Parameter1
      Property2: !Ref Parameter2
```

### Pattern: Macros for Template Transformation

**Use Case**: Creating reusable template transformations.

```yaml
# Macro definition stack
Resources:
  MacroFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.11
      Handler: index.handler
      Role: !GetAtt MacroRole.Arn
      Code:
        ZipFile: |
          def handler(event, context):
              fragment = event['fragment']
              # Transform the template fragment
              transformed = transform_template(fragment)
              return {
                  'requestId': event['requestId'],
                  'status': 'SUCCESS',
                  'fragment': transformed
              }

  Macro:
    Type: AWS::CloudFormation::Macro
    Properties:
      Name: MyMacro
      FunctionName: !GetAtt MacroFunction.Arn

# Usage in templates
Transform: MyMacro
Resources:
  # Template resources that will be transformed
```

### Pattern: Stack Sets for Multi-Account Deployment

**Use Case**: Deploying infrastructure across multiple accounts and regions.

```yaml
# Create stack set via CLI
aws cloudformation create-stack-set \
  --stack-set-name baseline-security \
  --template-body file://security-baseline.yaml \
  --parameters \
    ParameterKey=AlertEmail,ParameterValue=security@example.com \
  --capabilities CAPABILITY_IAM

# Deploy to accounts and regions
aws cloudformation create-stack-instances \
  --stack-set-name baseline-security \
  --accounts 123456789012 234567890123 \
  --regions us-east-1 eu-west-1 \
  --operation-preferences \
    RegionConcurrencyType=PARALLEL \
    MaxConcurrentCount=2 \
    FailureToleranceCount=0
```

## Summary

These patterns provide a foundation for building robust, secure, and cost-effective CloudFormation templates. Always:

1. **Use parameters and conditions** for flexibility
2. **Implement least privilege** for IAM roles
3. **Encrypt sensitive data** at rest and in transit
4. **Add DeletionPolicy** to stateful resources
5. **Use nested stacks** for modularity
6. **Export outputs** for cross-stack references
7. **Implement auto-scaling** for cost optimization
8. **Validate templates** before deployment
9. **Use change sets** to preview updates
10. **Monitor for drift** regularly
