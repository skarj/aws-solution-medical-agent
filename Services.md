## AWS Services Catalog & Documentation Directory

The AI-Assisted Clinical Trial Screening Platform relies on the following official AWS services. Each service entry includes its functional role within the solution, governing requirement IDs, and verified official AWS documentation links.

---

### 1. Compute & Orchestration

| AWS Service | Architecture Role | Governing Requirements | Official Documentation |
| :--- | :--- | :--- | :--- |
| **AWS Step Functions** | Serverless state machines orchestrating protocol onboarding OCR/rule parsing and patient multi-file screening pipelines. | `[REQ-F-02]`, `[REQ-F-03]`, `[REQ-F-06]`, `[REQ-F-07]`, `[REQ-NF-01]`, `[REQ-NF-03]` | [AWS Step Functions Documentation](https://docs.aws.amazon.com/step-functions/) |
| **AWS Lambda** | Event-driven microservices for S3 event routing, rule parsing, API handlers, and Bedrock Agent actions. | `[REQ-F-03]`, `[REQ-F-12]`, `[REQ-SEC-04]` | [AWS Lambda Documentation](https://docs.aws.amazon.com/lambda/) |
| **Amazon API Gateway** | Managed REST API edge providing authenticated clinician access and presigned URL generation. | `[REQ-F-16]`, `[REQ-SEC-02]`, `[REQ-SEC-04]` | [Amazon API Gateway Documentation](https://docs.aws.amazon.com/apigateway/) |
| **Amazon EventBridge** | Serverless event bus routing S3 document upload events to Step Functions executions. | `[REQ-F-06]` | [Amazon EventBridge Documentation](https://docs.aws.amazon.com/eventbridge/) |

---

### 2. Document Intelligence & Generative AI

| AWS Service | Architecture Role | Governing Requirements | Official Documentation |
| :--- | :--- | :--- | :--- |
| **Amazon Textract** | Asynchronous OCR and layout/form/table extraction from multi-page scanned and digital medical PDFs. | `[REQ-F-02]`, `[REQ-F-07]` | [Amazon Textract Documentation](https://docs.aws.amazon.com/textract/) |
| **Amazon Bedrock (Agents)** | Single-agent deterministic reasoning engine pulling protocol criteria and evaluating patient records via RAG. | `[REQ-F-12]`, `[REQ-F-13]`, `[REQ-F-14]` | [Amazon Bedrock Agents Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html) |
| **Amazon Bedrock (Knowledge Bases)** | Managed RAG ingestion pipeline providing chunking, vector embedding, and hybrid retrieval. | `[REQ-F-09]`, `[REQ-F-10]`, `[REQ-F-11]`, `[REQ-NF-02]` | [Amazon Bedrock Knowledge Bases Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html) |
| **Amazon Titan Text Embeddings** | Foundation model generating vector embeddings for patient document text chunks. | `[REQ-F-09]` | [Amazon Titan Embeddings Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html) |
| **Amazon Bedrock (Foundation Models)** | High-accuracy LLM reasoning for rule structuring and JSON verdict enforcement (Amazon Nova Pro / Claude 3.5 Sonnet). | `[REQ-F-03]`, `[REQ-F-14]` | [Amazon Bedrock Foundation Models Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) |

---

### 3. Storage, Database & Messaging

| AWS Service | Architecture Role | Governing Requirements | Official Documentation |
| :--- | :--- | :--- | :--- |
| **Amazon Simple Storage Service (Amazon S3)** | Encrypted object storage for protocol PDFs, raw patient records, and extracted OCR text. | `[REQ-F-01]`, `[REQ-F-05]`, `[REQ-F-08]`, `[REQ-SEC-01]` | [Amazon S3 Documentation](https://docs.aws.amazon.com/s3/) |
| **Amazon DynamoDB** | Serverless NoSQL database storing structured study protocols (`study-protocols`) and patient verdicts (`patient-verdicts`). | `[REQ-F-04]`, `[REQ-F-15]`, `[REQ-F-18]`, `[REQ-SEC-01]` | [Amazon DynamoDB Documentation](https://docs.aws.amazon.com/dynamodb/) |
| **Amazon Simple Notification Service (Amazon SNS)** | Managed messaging topic receiving Textract asynchronous job completion notifications. | `[REQ-F-07]` | [Amazon SNS Documentation](https://docs.aws.amazon.com/sns/) |
| **Amazon Simple Queue Service (Amazon SQS)** | Dead Letter Queues (DLQ) capturing failed ingestion or inference tasks for retry handling. | `[REQ-NF-03]` | [Amazon SQS Documentation](https://docs.aws.amazon.com/sqs/) |

---

### 4. Security, Identity & Governance

| AWS Service | Architecture Role | Governing Requirements | Official Documentation |
| :--- | :--- | :--- | :--- |
| **AWS Key Management Service (AWS KMS)** | Customer-Managed Keys (CMK) managing cryptographic keys for S3, DynamoDB, and log encryption. | `[REQ-SEC-01]` | [AWS KMS Documentation](https://docs.aws.amazon.com/kms/) |
| **Amazon Cognito** | User directory and authentication service enforcing Multi-Factor Authentication (MFA) and RBAC. | `[REQ-SEC-04]` | [Amazon Cognito Documentation](https://docs.aws.amazon.com/cognito/) |
| **AWS CloudTrail** | Regulatory auditing service logging management and S3/KMS data events. | `[REQ-SEC-05]` | [AWS CloudTrail Documentation](https://docs.aws.amazon.com/cloudtrail/) |
| **Amazon CloudWatch** | Centralized logging, metrics collection, and alerting for Step Functions and Lambda execution. | `[REQ-SEC-05]` | [Amazon CloudWatch Documentation](https://docs.aws.amazon.com/cloudwatch/) |
