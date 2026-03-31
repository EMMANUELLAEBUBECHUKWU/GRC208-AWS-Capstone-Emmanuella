# GRC208 Deployment Guide

## AWS Integrated GRC Platform — Personal AWS Free Tier Account

**Author:** Emmanuella Ebubechukwu  
**Student ID:** 2025/GRC/10041  
**Deployed:** March 31, 2026  
**Environment:** Personal AWS Free Tier Account — AWS CloudShell  
**Account:** 851417232379 | **Region:** us-east-1

-----

## Background

This deployment was carried out following a mandatory directive from the ICDFA Directorate of Training requiring all GRC Engineering Batch A2025 students to migrate their capstone from the AWS Academy Learner Lab to a personal AWS Free Tier account. The directive cited security hardening measures in the Learner Lab that limited access required for capstone work.

This personal account deployment resolves the one known constraint from the Learner Lab — the AWS Config delivery channel is now fully operational.

-----

## Overview

Complete five-phase deployment of the GRC208 AWS Integrated GRC Platform, carried out entirely via AWS CloudShell. No local installation was required. All infrastructure was provisioned using CloudFormation Infrastructure as Code templates adapted for AWS Free Tier compatibility.

**Total deployment time:** Approximately 90 minutes including troubleshooting and fixes.

-----

## Prerequisites

### AWS Environment

- Personal AWS Free Tier account
- $10 billing alert configured (mandatory ICDFA requirement)
- AWS CloudShell open in the console

### Verify Your Identity

```bash
aws sts get-caller-identity
```

Expected output:

```json
{
    "UserId": "851417232379",
    "Account": "851417232379",
    "Arn": "arn:aws:iam::851417232379:root"
}
```

### Clone Repository and Copy Files

```bash
git clone https://github.com/EMMANUELLAEBUBECHUKWU/GRC208-AWS-Capstone-Emmanuella.git
cd GRC208-AWS-Capstone-Emmanuella
```

-----

## Phase 1: Network Infrastructure (~8 minutes)

```bash
aws cloudformation create-stack \
  --stack-name grc-capstone-network-stack \
  --template-body file://cloudformation-network-stack.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=grc-capstone \
  --region us-east-1
```

Monitor until CREATE_COMPLETE:

```bash
aws cloudformation describe-stacks \
  --stack-name grc-capstone-network-stack \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

**Resources created:**

|Resource          |Value                              |
|------------------|-----------------------------------|
|VPC               |vpc-0be0c82b0410f8aa2 (10.0.0.0/16)|
|Public Subnet 1   |subnet-02ba654ce8daa9d1d           |
|Public Subnet 2   |subnet-0edbeab2518dfcd50           |
|Private Subnet 1  |subnet-0d74b2d4d999cad13           |
|Private Subnet 2  |subnet-0f721402eb0d1e9ce           |
|RDS Security Group|sg-0f876e997d5067329               |
|ECS Security Group|sg-06f550662a2783938               |
|ALB Security Group|sg-02cbdb34b451ca03c               |

-----

## Phase 2: Database Infrastructure (~14 minutes)

### Important — Free Tier Template Modifications

The original CloudFormation database template required three modifications for AWS Free Tier compatibility:

1. `StorageType` changed from `gp3` to `gp2` — Free Tier does not support gp3
1. `StorageEncrypted` set to `false` — Free Tier does not support RDS encryption at rest
1. `BackupRetentionPeriod` set to `0` — Free Tier does not allow backup retention on unencrypted instances
1. `KmsKeyId` removed — not applicable without encryption
1. `EnableCloudwatchLogsExports` removed — caused Free Tier conflict
1. `EnableIAMDatabaseAuthentication` removed — caused Free Tier conflict
1. `DeletionProtection` set to `false` — allows stack cleanup

These changes are already applied in the `cloudformation-database-stack.yaml` file in this repository.

```bash
aws cloudformation create-stack \
  --stack-name grc-capstone-database-stack \
  --template-body file://cloudformation-database-stack.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=grc-capstone \
    ParameterKey=DBUsername,ParameterValue=grcadmin \
    ParameterKey=DBPassword,ParameterValue=GrcPass2026! \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

Monitor until CREATE_COMPLETE:

```bash
aws cloudformation describe-stacks \
  --stack-name grc-capstone-database-stack \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

**Resources created:**

|Resource           |Value                                                   |
|-------------------|--------------------------------------------------------|
|RDS Endpoint       |grc-capstone-db.cu3mygyw4dpy.us-east-1.rds.amazonaws.com|
|Database Name      |grcdb                                                   |
|Evidence Bucket    |grc-capstone-evidence-bucket-851417232379               |
|Reports Bucket     |grc-capstone-compliance-reports-851417232379            |
|Compliance Table   |grc-compliance-status                                   |
|Controls Table     |grc-controls                                            |
|Risk Register Table|grc-risk-register                                       |

-----

## Phase 3: Lambda Functions (~5 minutes)

Unlike the Learner Lab deployment, a dedicated IAM role can be created in a personal AWS account.

### Create IAM Role

```bash
aws iam create-role \
  --role-name grc-lambda-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'
```

### Attach Policies

```bash
aws iam attach-role-policy --role-name grc-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam attach-role-policy --role-name grc-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess

aws iam attach-role-policy --role-name grc-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonRDSFullAccess

aws iam attach-role-policy --role-name grc-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole

aws iam attach-role-policy --role-name grc-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AWSConfigUserAccess
```

### Deploy Lambda Function

```bash
zip -r lambda_compliance_monitor.zip lambda_compliance_monitor.py

aws lambda create-function \
  --function-name grc-compliance-monitor \
  --runtime python3.11 \
  --role arn:aws:iam::851417232379:role/grc-lambda-role \
  --handler lambda_compliance_monitor.lambda_handler \
  --zip-file fileb://lambda_compliance_monitor.zip \
  --timeout 60 \
  --memory-size 256 \
  --region us-east-1

aws lambda update-function-configuration \
  --function-name grc-compliance-monitor \
  --vpc-config SubnetIds=subnet-0d74b2d4d999cad13,subnet-0f721402eb0d1e9ce,SecurityGroupIds=sg-0f876e997d5067329 \
  --region us-east-1
```

Verify Lambda is active:

```bash
aws lambda get-function \
  --function-name grc-compliance-monitor \
  --region us-east-1 \
  --query 'Configuration.State' \
  --output text
```

Test invocation:

```bash
aws lambda invoke \
  --function-name grc-compliance-monitor \
  --region us-east-1 \
  response.json && cat response.json
```

Expected: `{"statusCode": 200, "body": "{\"message\": \"Compliance monitoring completed\"}"}`

-----

## Phase 4: Monitoring & Config (~8 minutes)

### AWS Config — Fully Operational

This phase resolves the constraint from the Learner Lab. A dedicated config role is created with full permissions.

```bash
# Create S3 bucket for Config
aws s3 mb s3://grc-config-bucket-851417232379 --region us-east-1

# Create Config IAM role
aws iam create-role \
  --role-name grc-config-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "config.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name grc-config-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWS_ConfigRole

aws iam attach-role-policy \
  --role-name grc-config-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Configure recorder
aws configservice put-configuration-recorder \
  --configuration-recorder name=grc-config-recorder,roleARN=arn:aws:iam::851417232379:role/grc-config-role \
  --recording-group allSupported=true,includeGlobalResourceTypes=true \
  --region us-east-1

# Configure delivery channel
aws configservice put-delivery-channel \
  --delivery-channel name=grc-delivery-channel,s3BucketName=grc-config-bucket-851417232379 \
  --region us-east-1

# Start recording
aws configservice start-configuration-recorder \
  --configuration-recorder-name grc-config-recorder \
  --region us-east-1

# Verify recording
aws configservice describe-configuration-recorder-status \
  --region us-east-1
```

Expected: `"recording": true, "lastStatus": "SUCCESS"`

### CloudTrail

```bash
aws s3 mb s3://grc-cloudtrail-logs-851417232379 --region us-east-1

aws s3api put-bucket-policy \
  --bucket grc-cloudtrail-logs-851417232379 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AWSCloudTrailAclCheck",
        "Effect": "Allow",
        "Principal": {"Service": "cloudtrail.amazonaws.com"},
        "Action": "s3:GetBucketAcl",
        "Resource": "arn:aws:s3:::grc-cloudtrail-logs-851417232379"
      },
      {
        "Sid": "AWSCloudTrailWrite",
        "Effect": "Allow",
        "Principal": {"Service": "cloudtrail.amazonaws.com"},
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::grc-cloudtrail-logs-851417232379/AWSLogs/851417232379/*"
      }
    ]
  }'

aws cloudtrail create-trail \
  --name grc-trail \
  --s3-bucket-name grc-cloudtrail-logs-851417232379 \
  --region us-east-1

aws cloudtrail start-logging --name grc-trail --region us-east-1

aws cloudtrail get-trail-status \
  --name grc-trail --region us-east-1 \
  --query '{Logging:IsLogging}' --output table
```

### EventBridge

```bash
aws events put-rule \
  --name grc-compliance-check \
  --schedule-expression "rate(1 hour)" \
  --state ENABLED --region us-east-1

aws events put-targets \
  --rule grc-compliance-check \
  --targets Id=1,Arn=arn:aws:lambda:us-east-1:851417232379:function:grc-compliance-monitor \
  --region us-east-1
```

### CloudWatch

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name grc-high-non-compliance \
  --alarm-description "Alert when compliance drops below 80%" \
  --metric-name CompliancePercentage \
  --namespace GRC \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --region us-east-1

aws cloudwatch put-dashboard \
  --dashboard-name GRC-Platform \
  --dashboard-body '{"widgets":[{"type":"metric","x":0,"y":0,"width":12,"height":6,"properties":{"region":"us-east-1","metrics":[["GRC","CompliancePercentage"]],"title":"Compliance Percentage","period":300,"stat":"Average","view":"timeSeries"}},{"type":"metric","x":12,"y":0,"width":12,"height":6,"properties":{"region":"us-east-1","metrics":[["AWS/Lambda","Invocations","FunctionName","grc-compliance-monitor"]],"title":"Lambda Invocations","period":300,"stat":"Sum","view":"timeSeries"}}]}' \
  --region us-east-1
```

-----

## Phase 5: Data Initialisation (~5 minutes)

RDS is in a private subnet. Data is loaded via a VPC-connected Lambda function.

```bash
# Create data loader script
cat > db_loader_script.py << 'EOF'
import pymysql
import json

def lambda_handler(event, context):
    try:
        connection = pymysql.connect(
            host='grc-capstone-db.cu3mygyw4dpy.us-east-1.rds.amazonaws.com',
            user='grcadmin',
            password='GrcPass2026!',
            database='grcdb',
            connect_timeout=30
        )
        cursor = connection.cursor()
        with open('sample_data.sql', 'r') as f:
            sql_content = f.read()
        statements = [s.strip() for s in sql_content.split(';') if s.strip()]
        executed = 0
        errors = []
        for statement in statements:
            try:
                cursor.execute(statement)
                executed += 1
            except Exception as e:
                errors.append(str(e))
        connection.commit()
        cursor.close()
        connection.close()
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Database loaded successfully',
                'executed': executed,
                'errors': errors
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'message': 'Connection failed', 'error': str(e)})
        }
EOF

# Package with pymysql and sample data
mkdir package
pip install pymysql -t ./package
cp db_loader_script.py ./package/
cp sample_data.sql ./package/
cd package && zip -r ../db_loader.zip . && cd ..

# Add self-referencing security group rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-0f876e997d5067329 \
  --protocol tcp --port 3306 \
  --source-group sg-0f876e997d5067329 \
  --region us-east-1

# Deploy data loader Lambda
aws lambda create-function \
  --function-name grc-db-loader \
  --runtime python3.11 \
  --role arn:aws:iam::851417232379:role/grc-lambda-role \
  --handler db_loader_script.lambda_handler \
  --zip-file fileb://db_loader.zip \
  --timeout 300 --memory-size 256 \
  --vpc-config SubnetIds=subnet-0d74b2d4d999cad13,subnet-0f721402eb0d1e9ce,SecurityGroupIds=sg-0f876e997d5067329 \
  --region us-east-1

# Invoke data loader
sleep 10 && aws lambda invoke \
  --function-name grc-db-loader \
  --region us-east-1 \
  --cli-read-timeout 310 \
  db_response.json && cat db_response.json
```

Expected: `{"message": "Database loaded successfully", "executed": 24, "errors": []}`

### Create RDS Snapshot

```bash
aws rds create-db-snapshot \
  --db-instance-identifier grc-capstone-db \
  --db-snapshot-identifier grc-initial-snapshot \
  --region us-east-1
```

-----

## Validation

### Run Full Test Suite

```bash
python3 test_cases.py
```

Expected: `Ran 22 tests in 0.002s — OK`

### Verify All Resources

```bash
aws cloudformation describe-stacks --stack-name grc-capstone-network-stack --region us-east-1 --query 'Stacks[0].StackStatus' --output text
aws cloudformation describe-stacks --stack-name grc-capstone-database-stack --region us-east-1 --query 'Stacks[0].StackStatus' --output text
aws lambda get-function --function-name grc-compliance-monitor --region us-east-1 --query 'Configuration.State' --output text
aws cloudtrail get-trail-status --name grc-trail --region us-east-1 --query 'IsLogging' --output text
aws dynamodb list-tables --region us-east-1 --output table
aws s3 ls | grep grc
```

-----

## Deployment Checklist

- [x] Personal AWS Free Tier account created
- [x] $10 billing alert configured
- [x] AWS CLI identity confirmed — Account 851417232379
- [x] Network stack deployed — CREATE_COMPLETE
- [x] Database stack deployed — CREATE_COMPLETE (Free Tier template applied)
- [x] IAM role grc-lambda-role created with 5 policies
- [x] Lambda function deployed and VPC-connected — State: Active
- [x] IAM role grc-config-role created
- [x] AWS Config recorder configured — Recording: True, lastStatus: SUCCESS
- [x] Config delivery channel activated — S3 bucket grc-config-bucket-851417232379
- [x] CloudTrail enabled — IsLogging: True
- [x] EventBridge hourly rule created
- [x] CloudWatch alarm configured
- [x] CloudWatch dashboard deployed
- [x] Database initialised — 24 statements, zero errors
- [x] RDS backup snapshot created
- [x] All 22 unit tests passing

-----

## Troubleshooting — Challenges Encountered During Deployment

### Challenge 1: RDS CloudFormation Rollback — Free Tier Incompatible Settings

**Error:** `The specified backup retention period exceeds the maximum available to free tier customers`

**Cause:** The original CloudFormation template was designed for the Learner Lab which has different resource limits. Three settings were incompatible with Free Tier: `StorageType: gp3`, `StorageEncrypted: true`, and `BackupRetentionPeriod: 30`.

**Resolution:** Rewrote the database CloudFormation template with Free Tier compatible settings — `StorageType: gp2`, `StorageEncrypted: false`, `BackupRetentionPeriod: 0`, and removed KMS encryption references.

### Challenge 2: RDS in Private Subnet — Cannot Connect Directly from CloudShell

**Error:** MySQL connection hanging with no output.

**Cause:** RDS is correctly placed in a private subnet with no public endpoint. Direct mysql client connection from CloudShell cannot reach the database.

**Resolution:** Created a dedicated `db_loader_script.py` Lambda function with pymysql packaged, connected to the VPC private subnets, with a self-referencing security group rule on port 3306. The Lambda loaded 24 SQL statements successfully with zero errors.

### Challenge 3: AWS Config Command Syntax

**Error:** `Found invalid choice 'put-config-recorder'`

**Cause:** The deployment guide used abbreviated command names that are not valid in the AWS CLI.

**Resolution:** Used correct full command names: `put-configuration-recorder`, `start-configuration-recorder`.

### Challenge 4: CloudShell Session Timeout

**Cause:** CloudShell sessions expire after a period of inactivity.

**Resolution:** Reconnected and continued from where the session ended. All files persisted in CloudShell storage between sessions.

-----

*Last updated: March 31, 2026 — Emmanuella Ebubechukwu*
