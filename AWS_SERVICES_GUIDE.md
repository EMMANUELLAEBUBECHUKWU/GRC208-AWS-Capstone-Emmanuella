# AWS Services Guide

## GRC208 AWS Integrated GRC Platform - Per-Service Documentation

**Author:** Emmanuella Ebubechukwu  
**Student ID:** 2025/GRC/10041  
**GRC208 Capstone | ICDFA | March 31, 2026**

-----

## Overview

This document explains each AWS service used in the GRC208 platform - what it does, why it was chosen, how it integrates with the rest of the system, and what was observed during deployment. All details reflect the actual deployment in Account 851417232379, Region us-east-1, on March 31, 2026.

-----

## AWS CloudFormation

**Role:** Infrastructure as Code - all resources provisioned from YAML templates

Two stacks were deployed for this platform.

**grc-capstone-network-stack** creates:

- VPC vpc-0be0c82b0410f8aa2 (10.0.0.0/16)
- Two public subnets across us-east-1a and us-east-1b
- Two private subnets across the same availability zones
- Internet Gateway, NAT Gateway, and route tables
- Four security groups: ALB, ECS, RDS, and general

**grc-capstone-database-stack** creates:

- RDS MySQL 8.0 (db.t3.micro, Free Tier compatible)
- Two S3 buckets with AES256 encryption and versioning
- Three DynamoDB tables for compliance tracking
- Note: KMS encryption was removed from this template for Free Tier compatibility

**Why CloudFormation:** When the Learner Lab reset during the original deployment, IaC meant the full environment was rebuilt in under 20 minutes. The same discipline applied here - the template was fixed and redeployed cleanly after three rollback failures during Free Tier compatibility resolution.

-----

## Amazon VPC

**Role:** Network isolation and traffic control

|Resource          |Value                   |
|------------------|------------------------|
|VPC ID            |vpc-0be0c82b0410f8aa2   |
|CIDR              |10.0.0.0/16             |
|Public Subnet 1   |subnet-02ba654ce8daa9d1d|
|Public Subnet 2   |subnet-0edbeab2518dfcd50|
|Private Subnet 1  |subnet-0d74b2d4d999cad13|
|Private Subnet 2  |subnet-0f721402eb0d1e9ce|
|RDS Security Group|sg-0f876e997d5067329    |
|ECS Security Group|sg-06f550662a2783938    |
|ALB Security Group|sg-02cbdb34b451ca03c    |

A self-referencing inbound rule on port 3306 was added to the RDS security group, allowing only Lambda functions in the same security group to connect to the database.

-----

## AWS IAM

**Role:** Access control - dedicated roles for Lambda and Config

Two dedicated IAM roles were created in this deployment - an improvement over the Learner Lab where the LabRole was used for everything.

**grc-lambda-role**

- ARN: arn:aws:iam::851417232379:role/grc-lambda-role
- Policies: AWSLambdaBasicExecutionRole, AmazonDynamoDBFullAccess, AmazonRDSFullAccess, AWSLambdaVPCAccessExecutionRole, AWSConfigUserAccess

**grc-config-role**

- ARN: arn:aws:iam::851417232379:role/grc-config-role
- Policies: AWS_ConfigRole, AmazonS3FullAccess

Creating dedicated roles with specific policies is a significant security improvement over using the broad LabRole.

-----

## AWS Lambda

**Role:** Serverless compliance automation engine

**grc-compliance-monitor**

- Runtime: Python 3.11
- Memory: 256 MB
- Timeout: 60 seconds
- VPC: Connected to private subnets subnet-0d74b2d4d999cad13 and subnet-0f721402eb0d1e9ce
- Security group: sg-0f876e997d5067329
- Role: grc-lambda-role
- Trigger: EventBridge rule grc-compliance-check (rate 1 hour)
- Function ARN: arn:aws:lambda:us-east-1:851417232379:function:grc-compliance-monitor
- State: Active - confirmed via CLI
- Test result: HTTP 200, “Compliance monitoring completed”

**grc-db-loader**

- Runtime: Python 3.11
- Memory: 256 MB
- Timeout: 300 seconds
- VPC: Same private subnets as grc-compliance-monitor
- Handler: db_loader_script.lambda_handler
- Purpose: One-time execution to load sample_data.sql into private RDS
- Result: 24 SQL statements executed, zero errors

-----

## Amazon RDS MySQL

**Role:** Primary GRC data store

|Detail             |Value                                                   |
|-------------------|--------------------------------------------------------|
|Instance Identifier|grc-capstone-db                                         |
|Engine             |MySQL 8.0.40                                            |
|Instance Class     |db.t3.micro                                             |
|Endpoint           |grc-capstone-db.cu3mygyw4dpy.us-east-1.rds.amazonaws.com|
|Port               |3306                                                    |
|Database Name      |grcdb                                                   |
|Username           |grcadmin                                                |
|Storage Type       |gp2 (Free Tier compatible)                              |
|Encryption         |Disabled (Free Tier limitation)                         |
|Backup Retention   |0 (Free Tier limitation)                                |
|Backup Snapshot    |grc-initial-snapshot (created manually)                 |

**Six tables created:**

1. `frameworks` - six compliance frameworks loaded
1. `controls` - 30+ controls mapped across frameworks
1. `risks` - six risk records loaded
1. `assets` - asset inventory
1. `compliance_status` - time-series compliance history
1. `audit_logs` - complete change audit trail

-----

## Amazon S3

**Role:** Evidence storage, compliance reports, and audit log delivery

|Bucket                                      |Purpose                      |
|--------------------------------------------|-----------------------------|
|grc-capstone-evidence-bucket-851417232379   |Compliance artefacts         |
|grc-capstone-compliance-reports-851417232379|Generated reports            |
|grc-cloudtrail-logs-851417232379            |CloudTrail audit log delivery|
|grc-config-bucket-851417232379              |AWS Config delivery channel  |

All buckets use AES256 server-side encryption. S3 versioning is enabled on the evidence and reports buckets.

-----

## Amazon DynamoDB

**Role:** Real-time compliance status store

|Table                |Purpose                                       |
|---------------------|----------------------------------------------|
|grc-compliance-status|Real-time compliance percentage and risk score|
|grc-controls         |Control registry for fast dashboard queries   |
|grc-risk-register    |Live risk tracking                            |

All three tables use PAY_PER_REQUEST billing - no provisioned capacity charges.

-----

## AWS Config

**Role:** Continuous resource configuration monitoring

|Detail             |Value                         |
|-------------------|------------------------------|
|Recorder Name      |grc-config-recorder           |
|Role               |grc-config-role               |
|Recording Strategy |All supported resource types  |
|Resource Types     |596                           |
|Recording Frequency|Continuous                    |
|Delivery Channel   |grc-delivery-channel          |
|S3 Bucket          |grc-config-bucket-851417232379|
|Recording Status   |True                          |
|Last Status        |SUCCESS                       |

This is the most significant improvement over the Learner Lab deployment. The delivery channel is fully operational in this personal account because the `grc-config-role` has the required S3 write permissions - permissions that were blocked by the LabRole in the Learner Lab.

-----

## AWS CloudTrail

**Role:** Tamper-proof audit trail

|Detail    |Value                                                    |
|----------|---------------------------------------------------------|
|Trail Name|grc-trail                                                |
|S3 Bucket |grc-cloudtrail-logs-851417232379                         |
|IsLogging |True (confirmed via CLI)                                 |
|Scope     |All API calls in Account 851417232379                    |
|Trail ARN |arn:aws:cloudtrail:us-east-1:851417232379:trail/grc-trail|

-----

## Amazon CloudWatch

**Role:** Operational metrics, alerting, and dashboard

**Alarm:** `grc-high-non-compliance`

- Metric: CompliancePercentage
- Threshold: < 80%
- Period: 300 seconds

**Dashboard:** `GRC-Platform`

- Compliance percentage metric
- Lambda invocation count

-----

## Amazon EventBridge

**Role:** Automated hourly compliance scheduling

|Detail   |Value                                                          |
|---------|---------------------------------------------------------------|
|Rule Name|grc-compliance-check                                           |
|Schedule |rate(1 hour)                                                   |
|State    |ENABLED                                                        |
|Target   |grc-compliance-monitor Lambda                                  |
|Rule ARN |arn:aws:events:us-east-1:851417232379:rule/grc-compliance-check|

-----

*Last updated: March 31, 2026 - Emmanuella Ebubechukwu*
