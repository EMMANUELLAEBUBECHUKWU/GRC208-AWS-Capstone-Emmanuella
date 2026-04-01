# Best Practices

## AWS Integrated GRC Platform — Implementation Guide

**Author:** Emmanuella Ebubechukwu  
**Student ID:** 2025/GRC/10041  
**GRC208 Capstone | ICDFA | March 31, 2026**

-----

## Overview

This document captures the implementation best practices I applied during the deployment of the GRC208 AWS Integrated GRC Platform on a personal AWS Free Tier account. These reflect the AWS Well-Architected Framework pillars, real decisions I made during deployment, and lessons I learned from working through live challenges throughout this project.

-----

## 1. Infrastructure as Code

I defined every resource in this platform as CloudFormation YAML rather than clicking through the console. The VPC, subnets, NAT Gateway, RDS instance, S3 buckets, and DynamoDB tables are all in two template files that can be redeployed from scratch with a single CLI command.

When the original database template proved incompatible with AWS Free Tier, I updated the code and redeployed cleanly. This is far easier than trying to fix manually created resources.

What I applied:

- Two CloudFormation stacks defining all infrastructure: grc-capstone-network-stack and grc-capstone-database-stack
- Templates stored in my GitHub repository and version controlled
- Stack outputs used to pass values between stacks rather than hardcoding resource IDs
- Database template rewritten for Free Tier compatibility before redeployment

-----

## 2. Free Tier Awareness

Before deploying any CloudFormation template on AWS Free Tier, I learned the hard way that the template needs to be reviewed against Free Tier limits. The database stack rolled back three times before I identified and fixed all the incompatible settings.

Free Tier restrictions I encountered:

- RDS does not support StorageType gp3, only gp2
- RDS does not support encryption at rest without upgrading the account
- RDS does not allow BackupRetentionPeriod above 0 on unencrypted instances
- KMS key creation for RDS encryption is not available on Free Tier

The fix was to rewrite the entire database section of the CloudFormation template with these settings removed or changed. After that the stack deployed cleanly on the first attempt.

-----

## 3. Network Security

I placed RDS and Lambda in private subnets with no public endpoint. This is the correct security decision for a GRC platform that handles compliance data, risk records, and audit logs.

What I applied:

- Public subnets only for the ALB and NAT Gateway
- Private subnets for RDS, Lambda, and ECS Fargate
- NAT Gateway for outbound-only internet access from private resources
- Self-referencing inbound rule on port 3306 so only Lambda can reach RDS

When the deployment guide assumed a direct mysql connection from CloudShell, I could not do that because RDS had no public endpoint. The solution was to deploy a VPC-connected Lambda with pymysql packaged. That turned out to be a better approach anyway since it kept the database completely private throughout.

-----

## 4. Dedicated IAM Roles

In the Learner Lab, I had to use the LabRole for everything because iam:CreateRole was blocked. In this personal account, I created two dedicated IAM roles with specific permissions for each service.

Roles I created:

- grc-lambda-role with AWSLambdaBasicExecutionRole, AmazonDynamoDBFullAccess, AmazonRDSFullAccess, AWSLambdaVPCAccessExecutionRole, and AWSConfigUserAccess
- grc-config-role with AWS_ConfigRole and AmazonS3FullAccess

No hardcoded AWS credentials exist anywhere in the codebase. Lambda assumes grc-lambda-role at execution time and the credentials are temporary and auto-rotated by AWS.

-----

## 5. AWS Config Delivery Channel

This was the one unresolved constraint from the Learner Lab deployment. In my personal account I was able to complete it fully.

I created the grc-config-role with the correct permissions, configured the grc-config-recorder for 596 resource types with continuous recording, set up the delivery channel pointing to S3 bucket grc-config-bucket-851417232379, and started the recorder.

The result confirmed via CLI: recording: true, lastStatus: SUCCESS.

This is the most significant improvement over the Learner Lab version of this deployment.

-----

## 6. Audit Logging

I configured CloudTrail grc-trail during Phase 4 before loading any data. Every API call in Account 851417232379 from that point forward is captured and delivered to S3.

What I applied:

- CloudTrail trail created and started, IsLogging: True confirmed via CLI
- Logs delivered to S3 bucket grc-cloudtrail-logs-851417232379
- Bucket policy scoped to allow writes only from the CloudTrail service
- Database audit_logs table records every Create, Update, and Delete with user identity and timestamp

-----

## 7. Continuous Monitoring

The EventBridge rule grc-compliance-check fires every hour on a rate(1 hour) schedule. The compliance Lambda runs automatically 24 times per day without any human steps. This is what makes compliance continuous rather than a quarterly exercise.

What I applied:

- EventBridge hourly schedule triggering grc-compliance-monitor
- CloudWatch alarm grc-high-non-compliance alerting at less than 80% compliance
- DynamoDB storing real-time compliance status for fast dashboard reads
- RDS storing historical compliance records for trend reporting

-----

## 8. Disaster Recovery

After completing Phase 5, I created RDS snapshot grc-initial-snapshot as a baseline recovery point. The CloudFormation templates in GitHub mean the full infrastructure can be redeployed at any time. S3 versioning on all three GRC buckets means deleted or overwritten objects remain recoverable.

-----

## 9. Billing Management

Before deploying any resources, I set up the mandatory $10 billing alert with two thresholds: 80% actual spend ($8) and 100% forecasted spend ($10). This was required by the ICDFA mandatory directive and is also simply good practice before deploying cloud resources on any account.

-----

## 10. Documentation

I wrote the documentation in this repository to reflect what I actually deployed, not what I planned to deploy. Every resource ID, every challenge, every resolution, and every constraint is real. The account ID is mine, the endpoint addresses are from my deployment, and the problems described are ones I actually encountered and worked through.

-----

*Last updated: March 31, 2026 — Emmanuella Ebubechukwu*
