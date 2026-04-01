# Architecture Design

## GRC208 AWS Integrated GRC Platform - System Architecture and Design Decisions

**Author:** Emmanuella Ebubechukwu  
**Student ID:** 2025/GRC/10041  
**GRC208 Capstone | ICDFA | March 31, 2026**

-----

## Overview

This document describes the system architecture of the GRC208 AWS Integrated GRC Platform as deployed in personal AWS Account 851417232379 on March 31, 2026. It covers the network topology, security group design, data flow architecture, database schema, and the design decisions made during planning and deployment.

-----

## System Architecture

### Network Topology

```
AWS Account 851417232379 - Personal Free Tier
└── VPC vpc-0be0c82b0410f8aa2 (10.0.0.0/16)
    ├── Public Subnet 1 (us-east-1a) - subnet-02ba654ce8daa9d1d
    │   └── NAT Gateway
    ├── Public Subnet 2 (us-east-1b) - subnet-0edbeab2518dfcd50
    │   └── Application Load Balancer
    ├── Private Subnet 1 (us-east-1a) - subnet-0d74b2d4d999cad13
    │   ├── Lambda: grc-compliance-monitor
    │   ├── Lambda: grc-db-loader
    │   └── RDS MySQL: grc-capstone-db
    └── Private Subnet 2 (us-east-1b) - subnet-0f721402eb0d1e9ce
        ├── Lambda (secondary AZ)
        └── RDS (secondary AZ)
```

**Supporting infrastructure:**

- Internet Gateway attached to VPC
- NAT Gateway in Public Subnet 1
- Route table for public subnets: 0.0.0.0/0 → Internet Gateway
- Route table for private subnets: 0.0.0.0/0 → NAT Gateway

-----

### Security Group Design

|Security Group|ID                  |Purpose                        |
|--------------|--------------------|-------------------------------|
|ALB SG        |sg-02cbdb34b451ca03c|Controls inbound traffic to ALB|
|ECS SG        |sg-06f550662a2783938|Controls traffic to ECS Fargate|
|RDS SG        |sg-0f876e997d5067329|Controls access to RDS MySQL   |

**Self-referencing rule added during deployment:**
A self-referencing inbound rule on port 3306 was added to sg-0f876e997d5067329. This allows resources in the same security group (Lambda functions) to connect to RDS on the MySQL port. This rule was discovered to be necessary when the data loader Lambda could not reach RDS - it is a required step in any deployment of this platform.

-----

### Data Flow Architecture

```
AWS Resources (EC2, RDS, S3, Lambda, etc.)
         ↓
AWS Config - grc-config-recorder (596 resource types, continuous)
         ↓  [Delivery channel active → grc-config-bucket-851417232379]
Amazon EventBridge - grc-compliance-check (rate 1 hour)
         ↓
AWS Lambda - grc-compliance-monitor (Python 3.11, VPC-connected)
    ├── Calculates compliance percentage
    ├── Classifies risk level (Low / Medium / High / Critical)
    ├── Writes real-time result → DynamoDB (grc-compliance-status)
    └── Writes historical record → RDS MySQL (grcdb.compliance_status)
         ↓
Amazon CloudWatch
    ├── Receives CompliancePercentage metric
    ├── Evaluates alarm grc-high-non-compliance (< 80%)
    └── Publishes to SNS if threshold breached
         ↓
Amazon SNS → Team notification
```

-----

### Database Schema

**frameworks**

```sql
id INT AUTO_INCREMENT PRIMARY KEY,
name VARCHAR(255) NOT NULL UNIQUE,
description TEXT,
version VARCHAR(50),
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

**controls**

```sql
id INT AUTO_INCREMENT PRIMARY KEY,
control_id VARCHAR(100) NOT NULL UNIQUE,
framework_id INT NOT NULL (FK → frameworks.id),
title VARCHAR(255) NOT NULL,
description TEXT, objective TEXT,
implementation_status ENUM('Not Started','In Progress','Implemented','Optimized'),
owner VARCHAR(255)
```

**risks**

```sql
id INT AUTO_INCREMENT PRIMARY KEY,
risk_id VARCHAR(100) NOT NULL UNIQUE,
title VARCHAR(255) NOT NULL,
description TEXT, category VARCHAR(100),
probability ENUM('Low','Medium','High','Critical'),
impact ENUM('Low','Medium','High','Critical'),
risk_score DECIMAL(5,2),
status ENUM('Open','In Progress','Mitigated','Accepted','Closed'),
mitigation_strategy TEXT
```

**assets**

```sql
id INT AUTO_INCREMENT PRIMARY KEY,
asset_id VARCHAR(100) NOT NULL UNIQUE,
name VARCHAR(255) NOT NULL,
type ENUM('Data','System','Application','Infrastructure','Personnel'),
classification VARCHAR(100),
criticality ENUM('Low','Medium','High','Critical'),
rpo VARCHAR(50), rto VARCHAR(50), owner VARCHAR(255)
```

**compliance_status**

```sql
id INT AUTO_INCREMENT PRIMARY KEY,
timestamp DATETIME NOT NULL (indexed),
compliance_percentage DECIMAL(5,2),
total_rules INT, compliant_rules INT,
non_compliant_rules INT,
risk_level VARCHAR(50), risk_score DECIMAL(5,2)
```

**audit_logs**

```sql
id INT AUTO_INCREMENT PRIMARY KEY,
action VARCHAR(50),
entity_type VARCHAR(100), entity_id INT,
user VARCHAR(255), details TEXT,
timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
```

-----

## Design Decisions

### Decision 1: Private Subnets for RDS and Lambda

**Decision:** Place RDS and Lambda in private subnets with no public endpoint.

**Rationale:** A GRC platform handles compliance data, risk assessments, and audit logs. Placing RDS in a private subnet is a standard security control that maps directly to network isolation requirements in ISO 27001, NIST, and SOC 2.

**Consequence encountered:** The deployment guide assumed a direct mysql connection to the database endpoint. This was not possible from CloudShell because the endpoint is in a private subnet. The solution - deploying a VPC-connected Lambda function with pymysql packaged - is more production-appropriate and more secure than a public endpoint.

-----

### Decision 2: Free Tier Compatible Database Template

**Decision:** Modify the CloudFormation database template to remove Free Tier incompatible settings.

**Rationale:** The original template was designed for the Learner Lab which has different resource limits. Three settings caused repeated ROLLBACK_COMPLETE failures: gp3 storage, RDS encryption, and backup retention.

**Changes made:**

- `StorageType: gp3` → `StorageType: gp2`
- `StorageEncrypted: true` → `StorageEncrypted: false`
- `BackupRetentionPeriod: 30` → `BackupRetentionPeriod: 0`
- Removed `KmsKeyId` reference
- Removed `EnableCloudwatchLogsExports`
- Removed `EnableIAMDatabaseAuthentication`
- `DeletionProtection: true` → `DeletionProtection: false`

**Note:** Encryption at rest was available in the Learner Lab. For a production deployment outside Free Tier, re-enabling encryption is strongly recommended.

-----

### Decision 3: Dedicated IAM Roles

**Decision:** Create dedicated IAM roles for Lambda and Config rather than using a broad account role.

**Rationale:** The Learner Lab LabRole was used for everything because `iam:CreateRole` was blocked. In a personal account, dedicated roles follow the principle of least privilege more precisely. Each role has only the permissions it needs for its specific function.

**Production recommendation:** The policies attached to `grc-lambda-role` include some broad policies (AmazonRDSFullAccess, AmazonDynamoDBFullAccess). In a production environment, these should be replaced with custom policies scoped to specific resources.

-----

### Decision 4: AWS Config Delivery Channel

**Decision:** Configure the AWS Config delivery channel with a dedicated S3 bucket and IAM role.

**Rationale:** This was the one unresolved constraint from the Learner Lab deployment. In a personal account, creating a dedicated `grc-config-role` with `AWS_ConfigRole` policy and S3 access resolves this completely.

**Result:** Config recorder confirmed `recording: true, lastStatus: SUCCESS` - 596 resource types being monitored continuously.

-----

## Technology Stack

|Layer         |Technology          |Reason                                                  |
|--------------|--------------------|--------------------------------------------------------|
|Infrastructure|AWS CloudFormation  |Version-controlled, reproducible IaC                    |
|Network       |Amazon VPC          |Isolated, controlled network environment                |
|Compute       |AWS Lambda          |Serverless, event-driven, no server management          |
|Database      |Amazon RDS MySQL 8.0|Managed, relational, supports complex queries           |
|NoSQL         |Amazon DynamoDB     |Sub-millisecond latency for real-time status            |
|Object Storage|Amazon S3           |Durable, scalable, integrates with CloudTrail and Config|
|Monitoring    |Amazon CloudWatch   |Native AWS metrics, alarms, and dashboards              |
|Scheduling    |Amazon EventBridge  |Native AWS event scheduling for Lambda                  |
|Audit         |AWS CloudTrail      |Immutable API call logging to S3                        |
|Compliance    |AWS Config          |Continuous resource configuration monitoring            |
|Access Control|AWS IAM             |Dedicated roles for Lambda and Config                   |
|Application   |Python 3.11         |Lambda runtime, extensive AWS SDK support               |
|Dashboard     |React + CSS         |Component-based UI for the GRC interface                |

-----

## Deployment Strategy

The platform was deployed in five sequential phases:

1. **Network first** - VPC and subnets must exist before compute or database resources
1. **Database second** - Lambda and application need the database to exist first
1. **Lambda third** - deployed after network and database with IAM role created first
1. **Monitoring fourth** - CloudTrail, Config, EventBridge, and CloudWatch configured once core platform is running
1. **Data last** - sample data loaded after all infrastructure is confirmed working

-----

*Last updated: March 31, 2026 - Emmanuella Ebubechukwu*
