# GRC208 AWS Integrated GRC Platform

## Capstone Project — Emmanuella Ebubechukwu

**Student:** Emmanuella Ebubechukwu  
**Student ID:** 2025/GRC/10041  
**Programme:** GRC Engineering (CGRCE)  
**Institution:** International Cybersecurity and Digital Forensics Academy (ICDFA)  
**Course:** GRC208 — Governance, Risk, and Compliance Capstone  
**Submission Date:** March 31, 2026

-----

## Overview

This repository contains the complete deployment of an AWS-native Governance, Risk, and Compliance (GRC) platform, built and submitted as the GRC208 capstone project. The platform was deployed from scratch in a personal AWS Free Tier account using AWS CloudShell, Infrastructure as Code via CloudFormation, and a serverless Lambda-based compliance automation engine.

This deployment was completed following a mandatory directive from the ICDFA Directorate of Training requiring all GRC Engineering Batch A2025 students to migrate their capstone from the AWS Academy Learner Lab to a personal AWS Free Tier account within 48 hours, due to security hardening measures introduced in the Learner Lab environment.

-----

## Deployment Environment

|Detail           |Value                                                     |
|-----------------|----------------------------------------------------------|
|AWS Account      |851417232379                                              |
|Account Type     |Personal AWS Free Tier                                    |
|Region           |us-east-1                                                 |
|Deployment Method|AWS CloudShell                                            |
|Network          |VPC vpc-0be0c82b0410f8aa2 (10.0.0.0/16)                   |
|Database Endpoint|grc-capstone-db.cu3mygyw4dpy.us-east-1.rds.amazonaws.com  |
|Lambda Function  |grc-compliance-monitor (Python 3.11, 256 MB)              |
|IAM Role         |grc-lambda-role (custom role — full IAM permissions)      |
|Config Role      |grc-config-role (custom role for AWS Config)              |
|CloudTrail Trail |grc-trail (IsLogging: True)                               |
|AWS Config       |grc-config-recorder (Recording: True, lastStatus: SUCCESS)|
|Test Results     |22/22 passing                                             |
|Deployment Date  |March 31, 2026                                            |

-----

## Key Improvement Over Learner Lab

The previous deployment in the AWS Academy Learner Lab had one documented constraint — the AWS Config delivery channel could not be activated because the LabRole IAM did not have the required S3 write permissions. In this personal AWS account deployment, that constraint is fully resolved:

- A dedicated `grc-config-role` IAM role was created with the `AWS_ConfigRole` policy
- The Config delivery channel was successfully configured with S3 bucket `grc-config-bucket-851417232379`
- AWS Config recorder is confirmed **Recording: True** with **lastStatus: SUCCESS**
- 596 resource types are being continuously monitored

-----

## Architecture Summary

The platform is built on a multi-tier VPC architecture with public and private subnets across two availability zones.

**Network Layer**

- VPC (10.0.0.0/16) with two public and two private subnets
- NAT Gateway for outbound-only private subnet internet access
- Application Load Balancer in public subnets
- Four security groups with scoped least-privilege rules

**Data Layer**

- Amazon RDS MySQL 8.0 (grc-capstone-db, db.t3.micro): GRC relational data
- Amazon S3: evidence storage and compliance reports
- Amazon DynamoDB:  three tables for real-time compliance status
- AWS S3-based encryption (AES256):  Free Tier compatible

**Application Layer**

- AWS Lambda (grc-compliance-monitor): hourly compliance automation
- Amazon ECS Fargate: containerised GRC dashboard application
- AWS EventBridge: hourly schedule triggering compliance Lambda

**Security & Compliance Layer**

- AWS Config — grc-config-recorder, 596 resource types, continuous recording, delivery channel active
- AWS CloudTrail — grc-trail, all API calls logged to S3
- Amazon CloudWatch — alarm grc-high-non-compliance (<80% threshold)
- Amazon SNS — compliance threshold alert notifications

-----

## Compliance Frameworks Supported

|Framework                   |Focus Area                             |
|----------------------------|---------------------------------------|
|ISO 27001:2022              |Information Security Management System |
|NIST Cybersecurity Framework|Risk management and security controls  |
|PCI DSS 3.2.1               |Payment card data protection           |
|HIPAA                       |Health information privacy and security|
|GDPR                        |EU personal data protection            |
|SOC 2                       |Service organisation control assurance |

-----

## Repository Structure

```
GRC208-AWS-Capstone-Emmanuella/
├── README.md                              # This file
├── DEPLOYMENT_GUIDE.md                    # Step-by-step deployment walkthrough
├── BEST_PRACTICES.md                      # AWS and security best practices applied
├── AWS_SERVICES_GUIDE.md                  # Per-service documentation
├── architecture_design.md                 # System architecture and design decisions
├── PROJECT_MANIFEST.md                    # Complete file inventory
├── cloudformation-network-stack.yaml      # Network infrastructure IaC
├── cloudformation-database-stack.yaml     # Database infrastructure IaC (Free Tier optimised)
├── lambda_compliance_monitor.py           # Compliance monitoring Lambda
├── grc-dashboard.jsx                      # React GRC dashboard
├── grc-dashboard.css                      # Dashboard styling
├── test_cases.py                          # 22-test validation suite
├── sample_data.sql                        # Database initialisation data
├── requirements.txt                       # Python dependencies
├── deploy.sh                              # Deployment automation script
├── architecture-diagram.md               # Architecture diagrams
├── .gitignore                             # Git configuration
└── screenshots/                           # 23 deployment evidence screenshots
```

-----

## Quick Start

### Prerequisites

- Personal AWS account (Free Tier)
- AWS CloudShell (no local installation required)
- GitHub repository with project files

### Deployment Steps

```bash
# 1. Clone this repository into CloudShell
git clone https://github.com/EMMANUELLAEBUBECHUKWU/GRC208-AWS-Capstone-Emmanuella.git
cd GRC208-AWS-Capstone-Emmanuella

# 2. Deploy network infrastructure
aws cloudformation create-stack \
  --stack-name grc-capstone-network-stack \
  --template-body file://cloudformation-network-stack.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=grc-capstone \
  --region us-east-1

# 3. Deploy database infrastructure
aws cloudformation create-stack \
  --stack-name grc-capstone-database-stack \
  --template-body file://cloudformation-database-stack.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=grc-capstone \
    ParameterKey=DBUsername,ParameterValue=grcadmin \
    ParameterKey=DBPassword,ParameterValue=YourSecurePassword \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

For the complete five-phase deployment walkthrough, see <DEPLOYMENT_GUIDE.md>.

-----

## Test Results

All 22 unit tests pass. Run with:

```bash
python3 test_cases.py
```

|Category             |Tests |Result    |
|---------------------|------|----------|
|Compliance Monitoring|3     |✓ Pass    |
|Risk Assessment      |3     |✓ Pass    |
|Data Validation      |4     |✓ Pass    |
|Database Operations  |3     |✓ Pass    |
|Compliance Frameworks|2     |✓ Pass    |
|Audit Logging        |3     |✓ Pass    |
|Report Generation    |2     |✓ Pass    |
|Integration Workflows|2     |✓ Pass    |
|**Total**            |**22**|**✓ 100%**|

-----

## Author

**Emmanuella Ebubechukwu**  
GRC Engineering (CGRCE) | ICDFA  
Student ID: 2025/GRC/10041  
grc2510041@students.icdfa.edu.ng

*Submitted: March 31, 2026*
