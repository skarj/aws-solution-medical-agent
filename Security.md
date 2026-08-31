## 1. Security & Governance Overview

The AI-Assisted Clinical Trial Screening Platform processes Protected Health Information (PHI) and clinical trial protocol intellectual property. The security posture adheres strictly to the **HIPAA Security and Privacy Rules**, the **AWS Well-Architected Framework (Security Pillar)**, and the signed **AWS Business Associate Addendum (BAA)**.

---

## 2. Cryptographic Controls & Data Protection

### 2.1 Encryption at Rest (`[REQ-SEC-01]`)
* **AWS KMS Customer-Managed Keys (CMK):** All persistent and transient data storage layers are encrypted using a dedicated KMS Customer-Managed Key (`alias/medical-study-screening-cmk`) with automated annual key rotation enabled.
* **Amazon S3 Buckets:** Default bucket encryption is set to `aws:kms` utilizing the CMK. S3 Bucket Policies explicitly deny unencrypted uploads (`s3:PutObject` without `x-amz-server-side-encryption: aws:kms`).
* **Amazon DynamoDB:** Tables (`study-protocols`, `patient-verdicts`) are created with customer-managed KMS CMK encryption at rest.
* **Consolidated Patient Text Store:** The `patient-consolidated-text` S3 bucket (full-document text used for deterministic reasoning, `[REQ-F-11]`) is encrypted using the same KMS CMK. No vector store exists in this architecture.

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
All compute resources (Lambda, Step Functions) operate under dedicated execution roles with zero wildcard permissions on data actions.

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
        "arn:aws:lambda:*:*:function:patient-text-consolidator",
        "arn:aws:lambda:*:*:function:patient-screening-handler"
      ]
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

#### Protocol Rule Structurer Lambda Execution Role Policy (Excerpt) — Scoped to `[REQ-F-05]` Mandated Model:
This role is intentionally distinct from the Patient Screening Handler role below, so each Lambda's model access is scoped to its own workload. `[REQ-F-05]` mandates **Anthropic Claude Sonnet 5** exclusively for one-time protocol rule extraction. Least-privilege IAM enforces that mandate at the policy level, rather than relying solely on prompt/application logic.

Claude Sonnet 5 has no `bedrock-runtime` In-Region support in *any* AWS Region and can only be invoked via a Geographic (US) or Global cross-Region inference profile. `[REQ-OPS-01]` was relaxed to permit the Geographic (US) profile specifically (see `Requirements.md`). Per AWS's documented IAM pattern for Geographic cross-Region inference, the policy below grants `bedrock:InvokeModel` on both the inference-profile ARN and the underlying foundation-model ARN in the source Region and each destination Region the profile can route to, scoped with a `Condition` on `bedrock:InferenceProfileArn` so the foundation-model grant cannot be used outside that profile. The destination-Region list (`us-east-1`, `us-east-2`) is carried over from the equivalent table published for Claude Sonnet 4.5, since Claude Sonnet 5's own model-card page did not expose a per-source-Region breakdown at review time — **reconfirm this list against the live Bedrock console or `GetInferenceProfile` before production deployment**, and note AWS's Claude Sonnet 5 documentation describes the US profile as keeping data within "US and Canada," so a `ca-central-1` destination should also be verified/excluded if strict US-only residency is required.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowClaudeSonnet5GeoUsInferenceProfile",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": [
        "arn:aws:bedrock:us-west-2:*:inference-profile/us.anthropic.claude-sonnet-5"
      ]
    },
    {
      "Sid": "AllowClaudeSonnet5FoundationModelAcrossGeoUsDestinations",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": [
        "arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-sonnet-5",
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-5",
        "arn:aws:bedrock:us-east-2::foundation-model/anthropic.claude-sonnet-5"
      ],
      "Condition": {
        "StringEquals": {
          "bedrock:InferenceProfileArn": "arn:aws:bedrock:us-west-2:*:inference-profile/us.anthropic.claude-sonnet-5"
        }
      }
    },
    {
      "Sid": "AllowDynamoDBWriteProtocolRules",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/study-protocols"
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

No separate Bedrock Agent execution role exists: `[REQ-F-14]` specifies a direct `bedrock-runtime:InvokeModel` call rather than a Bedrock Agent, so the screening step's model-invocation grant lives on the `patient-screening-handler` Lambda's own execution role (below) instead of on an agent principal.

#### Screening Trigger Handler Lambda Execution Role Policy (New, `[REQ-F-07, REQ-F-08]`):
Invoked by the `POST /patients/{PatientID}/screenings` API route once the Clinical Investigator confirms all patient files are staged. Lists the authoritative file set for the patient and starts exactly one Step Functions execution, replacing a raw S3 `ObjectCreated` auto-trigger that would otherwise fire once per uploaded file. This role omits KMS grants: `s3:ListBucket` returns object key names and metadata only (no decryption of object content), and `states:StartExecution` has no KMS interaction — granting `kms:Decrypt` here would be unused, unjustified access on a data-handling system, so it's deliberately excluded (unlike every other Lambda role in this section, which does touch encrypted object/item content and needs it).
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListPatientUploadPrefix",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::patient-data-upload",
      "Condition": {
        "StringLike": { "s3:prefix": "*/" }
      }
    },
    {
      "Sid": "AllowStartPatientScreeningExecution",
      "Effect": "Allow",
      "Action": ["states:StartExecution"],
      "Resource": "arn:aws:states:us-west-2:*:stateMachine:PatientScreeningStateMachine"
    }
  ]
}
```

#### Patient Text Consolidator Lambda Execution Role Policy (New, `[REQ-F-11]`):
Reads all per-file Textract output for a patient and writes the single ordered, page-annotated full-text document consumed by the reasoning step.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadExtractedPatientText",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::patient-extracted-data/*"
    },
    {
      "Sid": "AllowWriteConsolidatedPatientText",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::patient-consolidated-text/*"
    },
    {
      "Sid": "AllowKMSDecrypt",
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "arn:aws:kms:*:*:key/*"
    }
  ]
}
```

#### Patient Screening Handler Lambda Execution Role Policy (`[REQ-F-14, REQ-F-15, REQ-F-17, REQ-NF-06]`):
Reads the consolidated full-text document and active protocol rules, invokes Claude Sonnet 5 directly with both as reasoning input, mechanically verifies every returned citation against the source text (`[REQ-NF-06]`), and persists the resulting verdict. The Bedrock grant uses the same two-statement Geographic (US) cross-Region inference-profile pattern as the Protocol Rule Structurer role above (and carries the same unresolved reconfirmation caveats on the destination-Region list and `ca-central-1` residency question). Citation verification needs no additional grant — it compares the model's output against the consolidated text this role already reads from S3.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadConsolidatedPatientText",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::patient-consolidated-text/*"
    },
    {
      "Sid": "AllowDynamoDBReadProtocols",
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:Query"],
      "Resource": "arn:aws:dynamodb:*:*:table/study-protocols"
    },
    {
      "Sid": "AllowDynamoDBWriteVerdicts",
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem", "dynamodb:UpdateItem"],
      "Resource": "arn:aws:dynamodb:*:*:table/patient-verdicts"
    },
    {
      "Sid": "AllowClaudeSonnet5GeoUsInferenceProfileForScreening",
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel"],
      "Resource": ["arn:aws:bedrock:us-west-2:*:inference-profile/us.anthropic.claude-sonnet-5"]
    },
    {
      "Sid": "AllowClaudeSonnet5FoundationModelAcrossGeoUsDestinationsForScreening",
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel"],
      "Resource": [
        "arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-sonnet-5",
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-5",
        "arn:aws:bedrock:us-east-2::foundation-model/anthropic.claude-sonnet-5"
      ],
      "Condition": {
        "StringEquals": {
          "bedrock:InferenceProfileArn": "arn:aws:bedrock:us-west-2:*:inference-profile/us.anthropic.claude-sonnet-5"
        }
      }
    },
    {
      "Sid": "AllowKMSDecrypt",
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
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
* **AWS CloudTrail:** Multi-region CloudTrail trail enabled with log file validation. Data events are captured for all S3 bucket accesses (`s3:GetObject`, `s3:PutObject`), KMS key operations (`kms:Decrypt`), and — per `[REQ-SEC-08]` — item-level operations on the `study-protocols` and `patient-verdicts` DynamoDB tables (`GetItem`, `Query`, `PutItem`, `UpdateItem`). The DynamoDB data events close a gap in the S3/KMS-only scope: `patient-verdicts` items hold direct PHI excerpts (`citations[].quote`, `Requirements.md` §5.2) and `study-protocols` holds trial protocol IP, so item-level reads and writes against them belong in the audit trail alongside document access.
* **CloudWatch Logs Retention:** All application, Step Functions, and API Gateway logs are encrypted with KMS and retained for a minimum of 7 years in compliance with clinical trial regulatory mandates (FDA 21 CFR Part 11 / HIPAA).
* **Clinical Review Signoff Auditing:** Reviewer decisions (`Approve`, `Reject`, `Manual Override`), timestamp (ISO 8601), user identity (Cognito sub), and clinician notes are stored immutably in `patient-verdicts`.
* **Conditional-Write Immutability Guard:** `patient-screening-handler`'s automated verdict write uses a conditional `PutItem` (`ConditionExpression`: item does not exist, or `reviewer_signoff.status = PENDING`), so a re-run of the screening pipeline for the same `{PatientID, StudyID}` cannot silently overwrite a Clinical Investigator's already-recorded `APPROVED`/`REJECTED` determination. This is the mechanism backing the "immutable" claim above against automated writes; the reviewer-facing signoff API (`[REQ-F-20]`) is unaffected and retains full `Approve`/`Reject`/`Manual Override` authority, since that is intentional human authority, not an automated overwrite.
