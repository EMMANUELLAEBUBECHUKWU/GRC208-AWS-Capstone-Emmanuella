# Project Manifest

## GRC208 AWS Integrated GRC Platform - Complete File Inventory

**Author:** Emmanuella Ebubechukwu  
**Student ID:** 2025/GRC/10041  
**GRC208 Capstone | ICDFA | March 31, 2026**

-----

## Overview

This document provides a complete inventory of every file in the GRC208 AWS Integrated GRC Platform repository.

**Repository:** https://github.com/EMMANUELLAEBUBECHUKWU/GRC208-AWS-Capstone-Emmanuella  
**AWS Account:** 851417232379 (Personal Free Tier)  
**Deployed:** March 31, 2026  
**Total Files:** 18 (excluding screenshots folder contents)  
**Total Screenshots:** 23

-----

## Documentation Files (6)

### README.md

**Purpose:** Main project overview and entry point  
**Contents:** Deployment environment details, key improvement over Learner Lab, architecture summary, compliance frameworks table, repository structure, quick-start commands, test results, and author information  
**Key update:** Reflects personal AWS Free Tier account (851417232379) and confirmed AWS Config delivery channel operation

### DEPLOYMENT_GUIDE.md

**Purpose:** Complete step-by-step deployment walkthrough  
**Contents:** Background on mandatory migration directive, prerequisites, five-phase deployment commands, Free Tier template modifications, challenges encountered and resolutions, validation steps, and deployment checklist  
**Key update:** Documents all Free Tier compatibility fixes, dedicated IAM role creation, and AWS Config delivery channel activation

### BEST_PRACTICES.md

**Purpose:** Implementation best practices applied during deployment  
**Contents:** Ten practice areas covering IaC, Free Tier awareness, network security, dedicated IAM roles, AWS Config, audit logging, continuous monitoring, disaster recovery, billing management, and documentation  
**Key update:** Includes Free Tier awareness section based on challenges encountered during deployment

### AWS_SERVICES_GUIDE.md

**Purpose:** Per-service documentation for every AWS service used  
**Contents:** CloudFormation, VPC, IAM, Lambda, RDS, S3, DynamoDB, Config, CloudTrail, CloudWatch, EventBridge - each with actual deployed values, integration details, and deployment observations  
**Key update:** Includes dedicated IAM roles section and full AWS Config deployment details

### architecture_design.md

**Purpose:** System architecture documentation and design decisions  
**Contents:** Network topology, security group design, data flow architecture, full database schema, four design decisions with rationale and consequences, technology stack table, and deployment strategy  
**Key update:** Documents Free Tier template modifications as a formal design decision

### PROJECT_MANIFEST.md

**Purpose:** This file - complete inventory of all repository files  
**Contents:** Every file listed with purpose, contents, and key updates

-----

## Infrastructure Files (2)

### cloudformation-network-stack.yaml

**Purpose:** CloudFormation template for network infrastructure  
**Contents:** VPC, two public subnets, two private subnets, Internet Gateway, NAT Gateway, route tables, and four security groups  
**Status:** Successfully deployed - CREATE_COMPLETE  
**Stack:** grc-capstone-network-stack

### cloudformation-database-stack.yaml

**Purpose:** CloudFormation template for database and storage layer  
**Contents:** RDS MySQL 8.0 (Free Tier compatible settings), two S3 buckets with AES256 encryption, three DynamoDB tables  
**Status:** Successfully deployed - CREATE_COMPLETE  
**Stack:** grc-capstone-database-stack  
**Key modification:** Template was rewritten from the original instructor template to remove Free Tier incompatible settings (gp3 storage, encryption at rest, backup retention, KMS keys)

-----

## Application Code Files (4)

### lambda_compliance_monitor.py

**Purpose:** Main Lambda compliance monitoring function  
**Contents:** Python 3.11 handler retrieving compliance data, calculating percentage, classifying risk level, writing to DynamoDB and RDS  
**Deployed as:** grc-compliance-monitor (256 MB, 60s timeout, VPC-connected)  
**Result:** State Active, HTTP 200 on invocation

### grc-dashboard.jsx

**Purpose:** React GRC compliance dashboard frontend  
**Lines:** 400+  
**Deployed via:** ECS Fargate

### grc-dashboard.css

**Purpose:** Dashboard styling  
**Lines:** 500+

### test_cases.py

**Purpose:** Comprehensive test suite  
**Tests:** 22 across 8 categories  
**Result:** All 22 passing - `Ran 22 tests in 0.002s - OK`

-----

## Data & Configuration Files (4)

### sample_data.sql

**Purpose:** Database initialisation script  
**Statements:** 24 executed, zero errors  
**Loaded via:** grc-db-loader Lambda (VPC-connected, pymysql dependency)

### requirements.txt

**Purpose:** Python dependency manifest

### deploy.sh

**Purpose:** Deployment automation script

### architecture-diagram.md

**Purpose:** Architecture diagrams in markdown format

-----

## Additional Files (2)

### DELIVERY_SUMMARY.md

**Purpose:** Formal capstone delivery evidence document

### .gitignore

**Purpose:** Standard git configuration

-----

## Screenshots Folder (23 files)

|Screenshot                              |Content                                           |
|----------------------------------------|--------------------------------------------------|
|01-aws-identity-confirmation.png        |aws sts get-caller-identity - Account 851417232379|
|02-project-files-cloudshell.png         |ls output showing all 11 project files            |
|03-network-stack-create-complete.png    |Network stack CREATE_COMPLETE                     |
|04-network-stack-outputs.png            |VPC, subnet, and security group IDs               |
|05-database-stack-create-complete.png   |Database stack CREATE_COMPLETE                    |
|06-database-stack-outputs.png           |RDS endpoint, S3 buckets, DynamoDB tables         |
|07-lambda-function-active.png           |Lambda State: Active                              |
|08-lambda-test-invocation.png           |HTTP 200, compliance monitoring completed         |
|09-aws-config-recorder-active.png       |recording: true, lastStatus: SUCCESS              |
|10-cloudtrail-logging-active.png        |IsLogging: True                                   |
|11-eventbridge-cloudwatch-configured.png|EventBridge and CloudWatch created                |
|12-db-loader-lambda-invoked.png         |DB loader Lambda invoked                          |
|13-database-loaded-successfully.png     |executed: 24, errors: []                          |
|14-all-22-tests-passing.png             |Ran 22 tests in 0.002s - OK                       |
|15-rds-snapshot-created.png             |grc-initial-snapshot available                    |
|16-full-deployment-verification.png     |All resources confirmed via CLI                   |
|17-both-stacks-console.png              |CloudFormation console - both stacks              |
|18-lambda-console.png                   |Lambda console - grc-compliance-monitor           |
|19-rds-console.png                      |RDS console - grc-capstone-db                     |
|20-cloudtrail-console.png               |CloudTrail console - grc-trail Logging            |
|21-dynamodb-tables-console.png          |DynamoDB - all 3 tables                           |
|22-s3-buckets-console.png               |S3 - all 4 GRC buckets                            |
|23-config-recorder-console.png          |AWS Config - Recording is on, 596 resource types  |

-----

*Last updated: March 31, 2026 - Emmanuella Ebubechukwu*
