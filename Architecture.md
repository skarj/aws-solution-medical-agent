# Architecture.md

## 1. High-Level System Architecture

The AI-Assisted Clinical Trial Screening Platform employs an asynchronous, event-driven, serverless architecture on AWS. The design eliminates 24/7 idle infrastructure costs while providing strict multi-tenant data isolation and HIPAA compliance.

```mermaid
graph TD
    subgraph Client_Tier ["Client & Clinical Review Tier"]
        CR["Clinical Reviewer"]
        UI["Web UI: React / Next.js [REQ-F-16]"]
        COG["Amazon Cognito User Pool [REQ-SEC-04]"]
    end

    subgraph API_Edge ["API & Edge Security Tier"]
        APIGW["Amazon API Gateway (REST API) [REQ-SEC-02]"]
        AUTH_LAMBDA["Lambda Authorizer [REQ-SEC-04]"]
    end

    subgraph Storage_Tier ["S3 Storage Tier (Encrypted with KMS CMK) [REQ-SEC-01]"]
        S3_PROTO["S3: protocol-data-upload [REQ-F-01]"]
        S3_PATIENT["S3: patient-data-upload [REQ-F-05]"]
        S3_EXTRACT["S3: patient-extracted-data [REQ-F-08]"]
    end

    subgraph Async_Ingestion ["Asynchronous Ingestion & OCR Tier"]
        SFN["AWS Step Functions (Patient Workflow) [REQ-F-06, REQ-NF-03]"]
        TEXTRACT["Amazon Textract (StartDocumentAnalysis) [REQ-F-02, REQ-F-07]"]
        SNS_TOPIC["Amazon SNS (Textract Completion) [REQ-F-07]"]
        SQS_DLQ["Amazon SQS (Dead Letter Queue) [REQ-NF-03]"]
    end

    subgraph AI_RAG_Tier ["Bedrock AI & Vector Search Tier"]
        LAMBDA_EXTRACT["AWS Lambda (Rule Extraction) [REQ-F-03]"]
        KB_INGEST["Bedrock Knowledge Bases Ingestion [REQ-F-09]"]
        TITAN_EMB["Amazon Titan Text Embeddings [REQ-F-09]"]
        VEC_STORE["Serverless Vector Store [REQ-F-10, REQ-F-11]"]
        BEDROCK_AGENT["Amazon Bedrock Single Agent [REQ-F-12, REQ-F-14]"]
    end

    subgraph Persistence_Tier ["Persistence & Audit Tier"]
        DDB_PROTO["DynamoDB: study-protocols [REQ-F-04]"]
        DDB_VERDICT["DynamoDB: patient-verdicts [REQ-F-15, REQ-F-18]"]
        KMS["AWS KMS Customer-Managed Key [REQ-SEC-01]"]
        CW_TRAIL["CloudWatch & CloudTrail [REQ-SEC-05]"]
    end

    %% Client Interactions
    CR -->|"1. Authenticates (MFA/RBAC)"| COG
    CR -->|"2. Interacts with UI"| UI
    UI -->|"3. HTTPS / TLS 1.3"| APIGW
    APIGW -.->|"Validates JWT"| AUTH_LAMBDA

    %% Protocol Flow
    UI -->|"Upload Protocol PDF"| S3_PROTO
    S3_PROTO -->|"S3 ObjectCreated Event"| LAMBDA_EXTRACT
    LAMBDA_EXTRACT -->|"Invoke Async OCR"| TEXTRACT
    LAMBDA_EXTRACT -->|"Extract Rules via Bedrock LLM"| BEDROCK_AGENT
    LAMBDA_EXTRACT -->|"Store Structured Rules"| DDB_PROTO

    %% Patient Flow
    UI -->|"Upload Multi-file Records"| S3_PATIENT
    S3_PATIENT -->|"Trigger State Machine"| SFN
    SFN -->|"Map State: Async OCR Jobs"| TEXTRACT
    TEXTRACT -->|"Emit Job Notification"| SNS_TOPIC
    SNS_TOPIC -->|"Resume Task Token"| SFN
    SFN -->|"Save Extracted Text"| S3_EXTRACT
    SFN -->|"Start Ingestion Job"| KB_INGEST
    KB_INGEST -->|"Generate Embeddings"| TITAN_EMB
    TITAN_EMB -->|"Persist Vector Chunks"| VEC_STORE
    SFN -->|"Trigger Agent Evaluation"| BEDROCK_AGENT
    BEDROCK_AGENT -->|"Read Protocol Rules"| DDB_PROTO
    BEDROCK_AGENT -->|"Filtered Vector Query (patient_id)"| VEC_STORE
    BEDROCK_AGENT -->|"Persist Structured Verdict"| DDB_VERDICT
    SFN -.->|"On Failure Retry Exhaustion"| SQS_DLQ

    %% Reviewer Flow
    APIGW -->|"Generate Presigned S3 GET URL"| S3_PATIENT
    APIGW -->|"Query Verdict & Protocols"| DDB_VERDICT
    UI -->|"Submit Clinician Determination"| DDB_VERDICT
```

---

## 2. Workflow 1: Protocol Onboarding & Rule Extraction Pipeline

This workflow parses a clinical study protocol PDF (30–100 pages) during trial initialization and extracts itemized inclusion and exclusion criteria into structured DynamoDB records.

```mermaid
sequenceDiagram
    autonumber
    actor Coordinator as "Study Coordinator"
    participant S3_Proto as "S3: protocol-data-upload [REQ-F-01]"
    participant Lambda_Ocr as "Lambda: Protocol Parser [REQ-F-02, REQ-F-03]"
    participant Textract as "Amazon Textract [REQ-F-02]"
    participant Bedrock_LLM as "Amazon Bedrock (Nova / Claude) [REQ-F-03]"
    participant DDB_Proto as "DynamoDB: study-protocols [REQ-F-04]"

    Coordinator->>S3_Proto: Upload protocol.pdf (Encrypted with KMS CMK)
    S3_Proto->>Lambda_Ocr: S3 ObjectCreated Event Notification
    Lambda_Ocr->>Textract: StartDocumentAnalysis(Tables, Forms, Text)
    Note over Lambda_Ocr,Textract: Asynchronous extraction for scanned/digital PDFs
    Textract-->>Lambda_Ocr: Return Document Hierarchy & Full Text
    Lambda_Ocr->>Bedrock_LLM: InvokeModel with System Extraction Prompt & JSON Schema
    Note over Bedrock_LLM: Extracts itemized inclusion/exclusion arrays with unique IDs
    Bedrock_LLM-->>Lambda_Ocr: Structured JSON Protocol Rules
    Lambda_Ocr->>DDB_Proto: PutItem (StudyID, Study_Name, Inclusion, Exclusion)
```

### Detailed Pipeline Stages:
1. **Document Ingestion (`[REQ-F-01]`):** The study coordinator uploads `protocol.pdf` to `s3://protocol-data-upload/{StudyID}/`.
2. **Text & Table OCR (`[REQ-F-02]`):** S3 Event Notification triggers the Protocol Ingestion Lambda function. For documents exceeding 5 pages or containing scanned text, Lambda calls Textract `StartDocumentAnalysis` with feature types `TABLES` and `FORMS`.
3. **Structured Rule Extraction (`[REQ-F-03]`):** Extracted markdown text is provided to Bedrock foundation model (Amazon Nova Pro / Anthropic Claude Sonnet 3.5) with a constrained system prompt enforcing the `study-protocols` JSON schema.
4. **Persistence (`[REQ-F-04]`):** Structured rules are written to the `study-protocols` DynamoDB table under primary key `Study_ID`.

---

## 3. Workflow 2: Automated Patient Screening Pipeline

This workflow processes multi-file patient records (up to 150 MB total, digital or scanned faxes/records) and performs RAG-augmented deterministic reasoning against protocol criteria.

```mermaid
graph TD
    subgraph Step_Functions_Execution ["AWS Step Functions State Machine [REQ-F-06, REQ-NF-01]"]
        START["Execution Start: {PatientID, StudyID}"]
        MAP_OCR["Map State (Concurrency: 10) [REQ-F-07]"]
        WAIT_OCR["Task Token / Wait State [REQ-F-07]"]
        STORE_RAW["Save Parsed Text to S3 [REQ-F-08]"]
        START_KB["Bedrock KB StartIngestionJob [REQ-F-09]"]
        WAIT_KB["Poll / Wait for KB Ingestion Complete"]
        INVOKE_AGENT["Invoke Single Bedrock Agent [REQ-F-12]"]
        SAVE_VERDICT["PutItem to patient-verdicts Table [REQ-F-15]"]
        DLQ_HANDLER["SQS DLQ Error Handler [REQ-NF-03]"]
    end

    subgraph Services ["Underlying AWS Services"]
        TXT["Amazon Textract (Async StartDocumentAnalysis) [REQ-F-07]"]
        S3_OUT["S3: patient-extracted-data [REQ-F-08]"]
        VEC["Vector Store (Bedrock KB / Serverless) [REQ-F-10, REQ-F-11]"]
        DDB_P["DynamoDB: study-protocols [REQ-F-04]"]
        DDB_V["DynamoDB: patient-verdicts [REQ-F-15]"]
    end

    START --> MAP_OCR
    MAP_OCR -->|"Start OCR Job per PDF"| TXT
    TXT -.->|"Completion Callback"| WAIT_OCR
    WAIT_OCR --> STORE_RAW
    STORE_RAW -->|"Write Text"| S3_OUT
    STORE_RAW --> START_KB
    START_KB -->|"Chunk, Embed & Index"| VEC
    START_KB --> WAIT_KB
    WAIT_KB --> INVOKE_AGENT
    INVOKE_AGENT -->|"Read Protocol Rules"| DDB_P
    INVOKE_AGENT -->|"Retrieve Evidence Chunks"| VEC
    INVOKE_AGENT --> SAVE_VERDICT
    SAVE_VERDICT -->|"Write Record"| DDB_V

    %% Error Transitions
    MAP_OCR -.->|"On 3x Retry Failure"| DLQ_HANDLER
    INVOKE_AGENT -.->|"On Error"| DLQ_HANDLER
```

### Execution Steps & State Specifications:
1. **Multi-File Trigger (`[REQ-F-05, REQ-F-06]`):** Clinical staff upload 1 or more PDF records to `s3://patient-data-upload/{PatientID}/`. S3 ObjectCreated triggers an EventBridge rule executing the Step Functions state machine.
2. **Parallel Asynchronous OCR (`[REQ-F-07]`):** Step Functions executes a dynamic `Map` state over all uploaded files. Each iteration executes `StartDocumentAnalysis` with an SQS/SNS completion callback.
3. **Raw Text Storage & Ingestion (`[REQ-F-08, REQ-F-09]`):** Extracted text is saved in `s3://patient-extracted-data/{PatientID}/`. Step Functions triggers `StartIngestionJob` on Amazon Bedrock Knowledge Bases.
4. **Vector Embedding & Chunk Metadata (`[REQ-F-10, REQ-F-11]`):** Bedrock Knowledge Bases automatically chunks the text (512 tokens with 20% overlap), generates embeddings using `amazon.titan-embed-text-v2`, and indexes chunks with metadata (`patient_id`, `source_filename`, `page_number`).
5. **Deterministic Single-Agent Reasoning (`[REQ-F-12, REQ-F-13, REQ-F-14]`):**
   * A single Bedrock Agent receives `{PatientID, StudyID}`.
   * Fetches active criteria from DynamoDB `study-protocols`.
   * For each criterion, executes vector retrieval against Bedrock Knowledge Base filtered by `metadata.patient_id == {PatientID}` (`[REQ-NF-02]`).
   * Evaluates evidence and populates JSON schema verdict (`MET`, `NOT_MET`, `UNCERTAIN`) with exact quotes and page numbers.
6. **Persistence & Auditing (`[REQ-F-15]`):** Output verdict written to `patient-verdicts` with default status `PENDING`.

---

## 4. Workflow 3: Human-in-the-Loop Clinical Review Interface

```mermaid
sequenceDiagram
    autonumber
    actor Clinician as "Licensed Clinical Reviewer [REQ-F-16]"
    participant WebUI as "React / Next.js Web UI [REQ-F-16]"
    participant APIGW as "Amazon API Gateway [REQ-SEC-02]"
    participant Lambda as "AWS Lambda (Review API)"
    participant DDB as "DynamoDB: patient-verdicts [REQ-F-15, REQ-F-18]"
    participant S3 as "S3: patient-data-upload [REQ-F-05, REQ-SEC-01]"

    Clinician->>WebUI: Open Patient Screening Dashboard
    WebUI->>APIGW: GET /verdicts/{PatientID}?studyId={StudyID} (Bearer JWT)
    APIGW->>Lambda: Route to Review Handler
    Lambda->>DDB: GetItem(Patient_ID, Study_ID)
    Lambda->>S3: GeneratePresignedUrl(patient-data-upload/{PatientID}/*.pdf, Expires=900s)
    Lambda-->>WebUI: Return Verdict JSON + Presigned S3 PDF URLs
    Note over WebUI,Clinician: UI renders side-by-side: PDF viewer on left, Criteria on right [REQ-F-16]
    Clinician->>WebUI: Click Criterion Evidence Quote [REQ-F-17]
    Note over WebUI: PDF Viewer automatically scrolls to page_citation and highlights evidence quote
    Clinician->>WebUI: Select Decision ("Approve" / "Reject" / "Override") + Notes [REQ-F-18]
    WebUI->>APIGW: POST /verdicts/{PatientID}/signoff (reviewer_signoff payload)
    APIGW->>Lambda: Process Clinical Determination
    Lambda->>DDB: UpdateItem (reviewer_signoff, status='APPROVED', timestamp=ISO8601)
    Lambda-->>WebUI: 200 OK (Audit Log Committed)
```

---

## 5. System Boundary & Network Isolation Diagram

```mermaid
graph TD
    subgraph Internet ["Public Network (TLS 1.3 Required) [REQ-SEC-02]"]
        CLIENT["Clinician Browser / HTTPS Client"]
    end

    subgraph AWS_Cloud ["AWS Cloud Environment (BAA Signed) [REQ-SEC-03]"]
        subgraph VPC ["Customer Amazon VPC (Isolated Subnets)"]
            subgraph Private_App_Subnets ["Private Application Subnets (No IGW Route)"]
                LAMBDA_SVCS["AWS Lambda Microservices"]
            end

            subgraph VPC_Endpoints ["VPC Interface Endpoints (AWS PrivateLink) [REQ-SEC-02]"]
                VPCE_S3["S3 Gateway Endpoint"]
                VPCE_DDB["DynamoDB Gateway Endpoint"]
                VPCE_BEDROCK["Bedrock VPC Interface Endpoint"]
                VPCE_TEXTRACT["Textract VPC Interface Endpoint"]
                VPCE_KMS["KMS VPC Interface Endpoint"]
            end
        end

        subgraph AWS_Managed_PaaS ["AWS Managed Serverless Services (KMS Encrypted) [REQ-SEC-01]"]
            BEDROCK_SVC["Amazon Bedrock (Agent + KB)"]
            TEXTRACT_SVC["Amazon Textract"]
            DDB_SVC["Amazon DynamoDB"]
            S3_SVC["Amazon S3 Buckets"]
            KMS_SVC["AWS KMS (Customer Managed Key)"]
        end
    end

    CLIENT -->|"HTTPS / TLS 1.3"| APIGW_EDGE["Amazon API Gateway"]
    APIGW_EDGE --> LAMBDA_SVCS
    LAMBDA_SVCS --> VPCE_S3 --> S3_SVC
    LAMBDA_SVCS --> VPCE_DDB --> DDB_SVC
    LAMBDA_SVCS --> VPCE_BEDROCK --> BEDROCK_SVC
    LAMBDA_SVCS --> VPCE_TEXTRACT --> TEXTRACT_SVC
    LAMBDA_SVCS --> VPCE_KMS --> KMS_SVC
```
