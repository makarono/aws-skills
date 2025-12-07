---
name: aws-cloudformation
description: AWS CloudFormation expert for infrastructure as code using native templates. Use when working with CloudFormation templates (YAML/JSON), creating stacks, managing stack updates, implementing nested stacks, or when the user mentions CloudFormation, IaC templates, stack sets, change sets, or wants to define AWS infrastructure using native CloudFormation. Covers template structure, resource definitions, intrinsic functions, parameters, outputs, and stack management workflows.
---

# AWS CloudFormation Development

This skill provides comprehensive guidance for developing AWS infrastructure using CloudFormation templates, with integrated MCP servers for accessing latest AWS knowledge and CloudFormation utilities.

## AWS Documentation Requirement

**CRITICAL**: This skill requires AWS MCP tools for accurate, up-to-date AWS information.

### Before Answering AWS Questions

1. **Always verify** using AWS MCP tools (if available):
   - `mcp__aws-mcp__aws___search_documentation` or `mcp__*awsdocs*__aws___search_documentation` - Search AWS docs
   - `mcp__aws-mcp__aws___read_documentation` or `mcp__*awsdocs*__aws___read_documentation` - Read specific pages
   - `mcp__aws-mcp__aws___get_regional_availability` - Check service availability

2. **If AWS MCP tools are unavailable**:
   - Guide user to configure AWS MCP: See [AWS MCP Setup Guide](../../docs/aws-mcp-setup.md)
   - Help determine which option fits their environment:
     - Has uvx + AWS credentials → Full AWS MCP Server
     - No Python/credentials → AWS Documentation MCP (no auth)
   - If cannot determine → Ask user which option to use

## Integrated MCP Servers

This skill can leverage CloudFormation-specific MCP servers automatically configured with the plugin:

### AWS CloudFormation MCP Server
**When to use**: For CloudFormation-specific guidance and utilities
- Validate CloudFormation templates
- Get resource type recommendations
- Retrieve CloudFormation best practices
- Access intrinsic function guidance
- Validate template configurations
- Get help with CloudFormation-specific syntax

**Important**: Leverage this server for CloudFormation template validation and advanced CloudFormation operations.

## When to Use This Skill

Use this skill when:
- Creating new CloudFormation templates or stacks
- Refactoring existing CloudFormation infrastructure
- Implementing nested stacks or stack sets
- Following AWS CloudFormation best practices
- Validating CloudFormation template configurations before deployment
- Verifying AWS service capabilities and regional availability
- Working with change sets and stack updates
- Implementing cross-stack references
- Managing CloudFormation drift detection

## Core CloudFormation Principles

### Template Structure

CloudFormation templates follow a standard structure with these key sections:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: A clear description of what this template creates

Parameters:
  # Input parameters for template customization

Mappings:
  # Static lookup tables for template logic

Conditions:
  # Conditional logic for resource creation

Resources:
  # AWS resources to create (REQUIRED section)

Outputs:
  # Values to export for cross-stack references or display
```

### Resource Naming

**CRITICAL**: Use CloudFormation's automatic naming or parameter-driven naming for flexibility.

**Why**: Proper naming enables:
- **Reusable templates**: Deploy the same template multiple times without conflicts
- **Stack isolation**: Each stack gets uniquely identified resources automatically
- **Cross-account deployments**: Same template works across accounts
- **Easier updates**: CloudFormation handles resource replacement when needed

```yaml
# ✅ GOOD - Let CloudFormation generate unique names
Resources:
  MyFunction:
    Type: AWS::Lambda::Function
    Properties:
      # No FunctionName specified - CloudFormation generates: StackName-MyFunction-XXXXX
      Runtime: python3.11
      Handler: index.handler
      Code:
        ZipFile: |
          def handler(event, context):
              return {'statusCode': 200}

# ✅ ALSO GOOD - Parameter-driven naming for control
Parameters:
  FunctionNamePrefix:
    Type: String
    Default: my-app

Resources:
  MyFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub '${FunctionNamePrefix}-function-${AWS::StackName}'
      # ...

# ❌ BAD - Hard-coded names prevent reusability
Resources:
  MyFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: my-lambda  # Avoid this
      # ...
```

### Intrinsic Functions

Master CloudFormation's intrinsic functions for dynamic templates:

**Essential Functions**:
- `!Ref` - Reference parameters and resources
- `!GetAtt` - Get resource attributes
- `!Sub` - String substitution with variables
- `!Join` - Join strings with delimiter
- `!Select` - Select item from list
- `!Split` - Split string into list
- `!FindInMap` - Lookup values in mappings
- `!If` - Conditional value selection
- `!Equals`, `!Not`, `!And`, `!Or` - Conditional logic

```yaml
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AWS::StackName}-data-${AWS::AccountId}'
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentParameter
        - Key: StackId
          Value: !Ref AWS::StackId

  MyFunction:
    Type: AWS::Lambda::Function
    Properties:
      Environment:
        Variables:
          BUCKET_NAME: !Ref MyBucket
          BUCKET_ARN: !GetAtt MyBucket.Arn
          TABLE_NAME: !If [IsProduction, !Ref ProdTable, !Ref DevTable]
```

### Parameters and Validation

Use parameters with validation for safe, reusable templates:

```yaml
Parameters:
  EnvironmentType:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod
    Description: Environment type for resource configuration

  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium
    Description: EC2 instance type

  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16
    AllowedPattern: '^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])(\/([0-9]|[1-2][0-9]|3[0-2]))$'
    Description: CIDR block for VPC
```

### Outputs and Cross-Stack References

Export values for use in other stacks:

```yaml
Outputs:
  VpcId:
    Description: VPC ID for networking resources
    Value: !Ref MyVPC
    Export:
      Name: !Sub '${AWS::StackName}-VpcId'

  ApiEndpoint:
    Description: API Gateway endpoint URL
    Value: !Sub 'https://${MyApi}.execute-api.${AWS::Region}.amazonaws.com/prod'
    Export:
      Name: !Sub '${AWS::StackName}-ApiEndpoint'

# In another stack, import the value:
Resources:
  MySubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !ImportValue 'NetworkStack-VpcId'
```

### Pre-Deployment Validation

Use a **multi-layer validation strategy** for comprehensive CloudFormation quality checks:

#### Layer 1: Template Validation (Required)

```bash
# Validate template syntax
aws cloudformation validate-template \
  --template-body file://template.yaml

# Or using AWS CLI with local validation
cfn-lint template.yaml  # Install: pip install cfn-lint
```

#### Layer 2: Change Sets (Recommended)

Always preview changes before applying:

```bash
# Create change set to preview changes
aws cloudformation create-change-set \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --change-set-name my-changes \
  --capabilities CAPABILITY_IAM

# Review the change set
aws cloudformation describe-change-set \
  --stack-name my-stack \
  --change-set-name my-changes

# Execute if changes look good
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name my-changes
```

#### Layer 3: Drift Detection (Periodic)

Detect configuration drift from template:

```bash
# Detect drift
aws cloudformation detect-stack-drift --stack-name my-stack

# Check drift status
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <detection-id>

# View drift details
aws cloudformation describe-stack-resource-drifts \
  --stack-name my-stack
```

## Workflow Guidelines

### Development Workflow

1. **Design**: Plan infrastructure resources and relationships
2. **Verify AWS Services**: Use AWS Documentation MCP to confirm service availability and features
   - Check regional availability for all required services
   - Verify service limits and quotas
   - Confirm latest resource type specifications
3. **Implement**: Write CloudFormation templates following best practices
   - Use CloudFormation MCP server for resource type recommendations
   - Reference CloudFormation best practices via MCP tools
4. **Validate**: Run template validation (see above)
5. **Create Change Set**: Preview changes before deployment
6. **Review**: Examine change set for correctness and safety
7. **Deploy**: Execute change set or create stack
8. **Verify**: Confirm resources are created correctly
9. **Monitor**: Set up drift detection and stack monitoring

### Stack Organization

**Nested Stacks**: Break complex infrastructure into manageable pieces

```yaml
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/network-template.yaml
      Parameters:
        VpcCIDR: !Ref VpcCIDR
        EnvironmentType: !Ref EnvironmentType

  ApplicationStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/app-template.yaml
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
        SubnetIds: !GetAtt NetworkStack.Outputs.PrivateSubnetIds
```

**Stack Sets**: Deploy to multiple accounts/regions

- Use for multi-account governance
- Centralized template management
- Automated deployments across organization

### Update Strategies

**Safe Update Practices**:

1. **Use Change Sets**: Always preview changes
2. **Stack Policies**: Protect critical resources from updates
3. **Update Rollback**: Enable automatic rollback on failure
4. **DeletionPolicy**: Protect data with Retain or Snapshot

```yaml
Resources:
  MyDatabase:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot  # Create snapshot on stack deletion
    UpdateReplacePolicy: Snapshot  # Create snapshot on resource replacement
    Properties:
      # ...

  MyStatefulResource:
    Type: AWS::DynamoDB::Table
    DeletionPolicy: Retain  # Keep resource when stack is deleted
    Properties:
      # ...
```

## Using MCP Servers Effectively

### When to Use AWS Documentation MCP

**Always verify before implementing**:
- New AWS resource types or properties
- Service availability in target regions
- Resource property specifications
- Service limits and quotas
- Security best practices for AWS services

**Example scenarios**:
- "Check if AWS::Lambda::Function supports Python 3.13 runtime"
- "Verify AWS::RDS::DBInstance is available in eu-south-2"
- "What are the valid values for AWS::EC2::Instance InstanceType?"
- "Get latest AWS::S3::Bucket encryption options"

### When to Use CloudFormation MCP Server

**Leverage for CloudFormation-specific guidance**:
- CloudFormation resource type selection and usage
- Template syntax and structure
- Intrinsic function usage
- CloudFormation best practice patterns
- Resource property configurations
- CloudFormation-specific optimizations

**Example scenarios**:
- "Validate my CloudFormation template syntax"
- "What's the correct way to reference a Lambda function ARN?"
- "How to use !GetAtt with API Gateway?"
- "Best practices for CloudFormation nested stacks"

### MCP Usage Best Practices

1. **Verify First**: Always check AWS Documentation MCP before implementing new resources
2. **Regional Validation**: Check resource type availability in target deployment regions
3. **CloudFormation Guidance**: Use CloudFormation MCP for template-specific recommendations
4. **Stay Current**: MCP servers provide latest information beyond knowledge cutoff
5. **Combine Sources**: Use both skill patterns and MCP servers for comprehensive guidance

## CloudFormation Patterns Reference

For detailed CloudFormation patterns, anti-patterns, and architectural guidance, refer to the comprehensive reference:

**File**: `references/cloudformation-patterns.md`

This reference includes:
- Common CloudFormation patterns and their use cases
- Anti-patterns to avoid
- Security best practices with CloudFormation
- Cost optimization strategies
- Performance considerations
- Advanced template techniques

## Additional Resources

- **CloudFormation Patterns**: `references/cloudformation-patterns.md` - Detailed pattern library
- **AWS Documentation MCP**: Integrated for latest AWS information
- **CloudFormation MCP Server**: Integrated for CloudFormation-specific guidance

## GitHub Actions Integration

When GitHub Actions workflow files exist in the repository, ensure all checks defined in `.github/workflows/` pass before committing. This prevents CI/CD failures and maintains code quality standards.
