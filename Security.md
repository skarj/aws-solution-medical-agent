## 1. Security & Governance Overview

The AI-Assisted Clinical Trial Screening Platform processes Protected Health Information (PHI) and clinical trial protocol intellectual property. The security posture adheres strictly to the **HIPAA Security and Privacy Rules**, the **AWS Well-Architected Framework (Security Pillar)**, and the signed **AWS Business Associate Addendum (BAA)**.

---

## 2. Cryptographic Controls & Data Protection

### 2.1 Encryption at Rest (`[REQ-SEC-01]`)
* **AWS KMS Customer-Managed Keys (CMK):** All persistent and transient data storage layers are encrypted using a dedicated KMS Customer-Managed Key (`alias/medical-study-screening-cmk`) with automated annual key rotation enabled.
* **Amazon S3 Buckets:** Default bucket encryption is set to `aws:kms` utilizing the CMK. S3 Bucket Policies explicitly deny unencrypted uploads (`s3:PutObject` without `x-amz-server-side-encryption: aws:kms`).
* **Amazon DynamoDB:** Tables (`study-protocols`, `patient-verdicts`) are created with customer-managed KMS CMK encryption at rest.
* **Vector Store:** Vector embeddings, chunk indices, and metadata are encrypted using the same KMS CMK.

#### S3 KMS Enforcement Bucket Policy:
```json
{
  "Version": "2012-10-17",
  "Id": "DenyNonKMSEncryptedUploads",
  "Statement": [
    {
      "Sid": "DenyUnEncryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::patient-data-upload/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

### 2.2 Encryption in Transit (`[REQ-SEC-02]`)
* **Transport Layer Security (TLS 1.3):** All external and internal communications require TLS 1.3 (with fallback to TLS 1.2 minimum).
* **S3 Secure Transport Enforcement:** S3 Bucket Policies enforce HTTPS across all endpoints via `aws:SecureTransport` condition checks.

#### S3 TLS Enforcement Policy:
```json
{
  "Sid": "EnforceTLSRequestsOnly",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::patient-data-upload",
    "arn:aws:s3:::patient-data-upload/*"
  ],
  "Condition": {
    "Bool": {
      "aws:SecureTransport": "false"
    }
  }
}
```

---

## 3. Identity, Access Management & RBAC (`[REQ-SEC-04]`)

### 3.1 User Authentication & Authorization
* **Amazon Cognito User Pool:** Clinical personnel authenticate via Amazon Cognito with mandatory Multi-Factor Authentication (MFA - SMS/TOTP).
* **RBAC - ClinicalInvestigator:** Single unified operational role with end-to-end authority for study protocol onboarding, multi-file patient record uploading, and eligibility verdict review and binding determination signoff, scoped to authorized `StudyID`s via custom token claims.
* **RBAC - ComplianceAuditor:** Read-only access to immutable CloudTrail audit logs, review determinations, and regulatory compliance dashboards.

### 3.2 Service Least-Privilege IAM Policies
All compute resources (Lambda, Step Functions, Bedrock Agents) operate under dedicated execution roles with zero wildcard permissions on data actions.

#### Step Functions State Machine Execution Role Policy (Excerpt):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowTextractAsyncJobs",
      "Effect": "Allow",
      "Action": [
        "textract:StartDocumentAnalysis",
        "textract:GetDocumentAnalysis"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowLambdaTaskInvocations",
      "Effect": "Allow",
      "Action": [
        "lambda:InvokeFunction"
      ],
      "Resource": [
        "arn:aws:lambda:*:*:function:protocol-rule-structurer",
        "arn:aws:lambda:*:*:function:patient-screening-handler"
      ]
    },
    {
      "Sid": "AllowBedrockKBIngestion",
      "Effect": "Allow",
      "Action": [
        "bedrock:StartIngestionJob",
        "bedrock:GetIngestionJob"
      ],
      "Resource": "arn:aws:bedrock:*:*:knowledge-base/*"
    },
    {
      "Sid": "AllowSqsDlqMessages",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage"
      ],
      "Resource": "arn:aws:sqs:*:*:*-dlq"
    }
  ]
}
```

#### Bedrock Agent Execution Role Policy (Excerpt):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBedrockModelInvocation",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:Retrieve"
      ],
      "Resource": [
        "arn:aws:bedrock:*:*:foundation-model/amazon.nova-pro-v1:0",
        "arn:aws:bedrock:*:*:foundation-model/anthropic.claude-3-5-sonnet-*",
        "arn:aws:bedrock:*:*:knowledge-base/*"
      ]
    },
    {
      "Sid": "AllowDynamoDBReadProtocols",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/study-protocols"
    },
    {
      "Sid": "AllowDynamoDBWriteVerdicts",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/patient-verdicts"
    },
    {
      "Sid": "AllowKMSDecrypt",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:*:*:key/*"
    }
  ]
}
```

---

## 4. Network Security & Data Transport Protection (`[REQ-SEC-02]`)

* **Serverless Network Isolation:** The entire platform runs on managed serverless AWS services with zero compute instances (EC2), public database ports, or ingress listeners exposed to the internet.
* **Internal AWS Backbone Communication:** Microservices (Lambda, Step Functions) communicate with Amazon Bedrock, Amazon Textract, DynamoDB, S3, and AWS KMS across AWS's secure internal cloud network, authenticated via IAM Signature Version 4 (SigV4) and encrypted via TLS 1.3.
* **No Direct Internet Exposure:** Ingestion buckets and databases are private by default, with public access blocks enforced at the S3 account and bucket levels.
* **Time-Limited Presigned URLs for PDF Viewing:** Patient medical records are never exposed via public URLs. The clinical review API generates short-lived S3 Presigned GET URLs with a maximum 15-minute Time-To-Live (TTL), scoped strictly to the authenticated clinical reviewer session.

---

## 5. HIPAA Compliance, Auditing & Traceability (`[REQ-SEC-03, REQ-SEC-05]`)

### 5.1 Business Associate Addendum (BAA) Alignment (`[REQ-SEC-03]`)
* All AWS services utilized in this architecture (Amazon S3, Amazon Textract, Amazon Bedrock, Amazon DynamoDB, AWS Step Functions, AWS Lambda, Amazon Cognito, AWS KMS, AWS CloudTrail, Amazon CloudWatch) are **HIPAA-eligible** services under the AWS Business Associate Addendum (BAA).
* Data processing and model invocation execute strictly within customer-defined AWS Regions; Bedrock does not log or retain customer prompt/completion data for foundational model training.

### 5.2 Immutable Audit Trail (`[REQ-SEC-05]`)
* **AWS CloudTrail:** Multi-region CloudTrail trail enabled with log file validation. Data events are captured for all S3 bucket accesses (`s3:GetObject`, `s3:PutObject`) and KMS key operations (`kms:Decrypt`).
* **CloudWatch Logs Retention:** All application, Step Functions, and API Gateway logs are encrypted with KMS and retained for a minimum of 7 years in compliance with clinical trial regulatory mandates (FDA 21 CFR Part 11 / HIPAA).
* **Clinical Review Signoff Auditing:** Reviewer decisions (`Approve`, `Reject`, `Manual Override`), timestamp (ISO 8601), user identity (Cognito sub), and clinician notes are stored immutably in `patient-verdicts`.
