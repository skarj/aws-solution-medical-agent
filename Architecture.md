## 1. High-Level System Architecture

The AI-Assisted Clinical Trial Screening Platform employs an asynchronous, event-driven, serverless architecture on AWS. The design eliminates 24/7 idle infrastructure costs while providing strict multi-tenant data isolation and HIPAA compliance.

```mermaid
graph LR
    subgraph Client_Tier ["User and Client Interface (REQ-F-16)"]
        WEB["Clinical Web Dashboard"]
    end

    subgraph Ingestion_Tier ["Storage and Async Ingestion (REQ-F-01, REQ-F-02, REQ-F-05, REQ-F-06)"]
        S3["Amazon S3 Buckets"]
        SFN["AWS Step Functions State Machines"]
        OCR["Amazon Textract OCR"]
    end

    subgraph AI_Engine_Tier ["RAG and AI Reasoning Engine (REQ-F-03, REQ-F-09, REQ-F-12)"]
        KB["Bedrock Knowledge Base and Vectors"]
        AGENT["Bedrock Reasoning Agent"]
    end

    subgraph Persistence_Tier ["Structured Persistence and Audit (REQ-F-04, REQ-F-15)"]
        DDB["Amazon DynamoDB Tables"]
    end

    WEB -->|"Uploads Records and Protocols"| S3
    S3 -->|"Triggers State Machines"| SFN
    SFN -->|"Executes Async OCR"| OCR
    OCR -->|"Indexes Documents"| KB
    KB -->|"Retrieves Context"| AGENT
    AGENT -->|"Saves Rules and Verdicts"| DDB
    DDB -->|"Displays AI Checklist"| WEB
```

---

## 2. Workflow 1: Protocol Onboarding & Rule Extraction Pipeline

This workflow parses a clinical study protocol PDF (30–100 pages) during trial initialization and extracts itemized inclusion and exclusion criteria into structured DynamoDB records using an AWS Step Functions state machine.

### 2.1 Component Flow Diagram

```mermaid
graph TD
    subgraph Coordinator_Tier ["Study Coordinator Action (REQ-F-01)"]
        USER["Study Coordinator"]
        S3_PROTO["S3: protocol-data-upload (REQ-F-01)"]
        EV_BRIDGE["Amazon EventBridge (REQ-F-02)"]
    end

    subgraph Step_Functions_Tier ["AWS Step Functions Orchestration (REQ-F-02, REQ-F-03)"]
        SFN_PROTO["Step Functions: Protocol Onboarding State Machine (REQ-F-02)"]
        LAMBDA_STRUCT["AWS Lambda: Rule Extraction Task (REQ-F-03)"]
        SQS_DLQ["Amazon SQS Dead Letter Queue (REQ-NF-03)"]
    end

    subgraph External_Services ["AWS Managed AI and Persistence"]
        TEXTRACT["Amazon Textract StartDocumentAnalysis (REQ-F-02)"]
        BEDROCK_LLM["Amazon Bedrock Nova / Claude Sonnet (REQ-F-03)"]
        DDB_PROTO["DynamoDB: study-protocols Table (REQ-F-04)"]
    end

    USER -->|"Uploads protocol.pdf"| S3_PROTO
    S3_PROTO -->|"S3 ObjectCreated Event"| EV_BRIDGE
    EV_BRIDGE -->|"Triggers Execution StudyID"| SFN_PROTO
    SFN_PROTO -->|"StartDocumentAnalysis"| TEXTRACT
    TEXTRACT -->|"Returns Tables and Text"| SFN_PROTO
    SFN_PROTO -->|"Invokes Task with Extracted Text"| LAMBDA_STRUCT
    LAMBDA_STRUCT -->|"Prompts with JSON Schema"| BEDROCK_LLM
    BEDROCK_LLM -->|"Returns Structured Rules JSON"| LAMBDA_STRUCT
    LAMBDA_STRUCT -->|"PutItem StudyID, Rules"| DDB_PROTO
    SFN_PROTO -.->|"On Task Failure / Retry Exceeded"| SQS_DLQ
```

### 2.2 Protocol Onboarding State Machine (`stateDiagram-v2`)

```mermaid
stateDiagram-v2
    [*] --> StartProtocolExecution: S3 Event Trigger (StudyID)
    
    state "Start Textract Async OCR Task (REQ-F-02)" as StartProtocolOcr
    state "Wait for Textract Completion Callback" as WaitForProtocolOcr
    state "Check OCR Status" as CheckProtocolOcr <<choice>>
    state "Execute Lambda Rule Structurer Task (REQ-F-03)" as LambdaExtractRules
    state "Invoke Bedrock Model JSON Enforcement (REQ-F-03)" as BedrockRulePrompt
    state "PutItem to study-protocols Table (REQ-F-04)" as PersistProtocolRules
    state "Route to SQS DLQ & Alert Admin (REQ-NF-03)" as ProtocolErrorState

    StartProtocolExecution --> StartProtocolOcr
    StartProtocolOcr --> WaitForProtocolOcr: StartDocumentAnalysis(Tables, Forms)
    WaitForProtocolOcr --> CheckProtocolOcr: Task Token Received
    
    CheckProtocolOcr --> LambdaExtractRules: OCR Status == SUCCEEDED
    CheckProtocolOcr --> StartProtocolOcr: Catch [Transient Error - Retry with Backoff]
    CheckProtocolOcr --> ProtocolErrorState: OCR Status == FAILED / Retries Exhausted

    LambdaExtractRules --> BedrockRulePrompt: Send Formatted Markdown Text
    BedrockRulePrompt --> PersistProtocolRules: Valid Structured JSON Rules Returned
    BedrockRulePrompt --> LambdaExtractRules: Catch [Throttling / Retry with Backoff]
    BedrockRulePrompt --> ProtocolErrorState: Catch [Schema Violation / Fatal Error]

    PersistProtocolRules --> [*]: Protocol Onboarding Succeeded
    ProtocolErrorState --> [*]: Execution Terminated (DLQ Captured)
```

### 2.3 Detailed Pipeline Stages:
1. **Document Ingestion (`(REQ-F-01)`):** The study coordinator uploads `protocol.pdf` to `s3://protocol-data-upload/{StudyID}/`.
2. **State Machine Execution Trigger (`(REQ-F-02)`):** S3 ObjectCreated event routes via Amazon EventBridge to launch the Protocol Onboarding Step Functions state machine with input `{StudyID, Bucket, Key}`.
3. **Asynchronous Layout & Form OCR (`(REQ-F-02)`):** Step Functions starts Amazon Textract `StartDocumentAnalysis` with `TABLES` and `FORMS` feature types, pausing execution until Textract completes.
4. **Structured Rule Extraction (`(REQ-F-03)`):** Step Functions executes a Lambda task passing the extracted text to an LLM on Amazon Bedrock (Amazon Nova Pro / Anthropic Claude 3.5 Sonnet) enforcing the itemized `study-protocols` JSON schema.
5. **Rule Persistence (`(REQ-F-04)`):** Structured rules are written to the `study-protocols` DynamoDB table under primary key `StudyID`.

---

## 3. Workflow 2: Automated Patient Screening Pipeline

This workflow processes multi-file patient records (up to 150 MB total, digital or scanned faxes/records) and performs RAG-augmented deterministic reasoning against protocol criteria using an AWS Step Functions state machine.

### 3.1 Component Flow Diagram

```mermaid
graph TD
    subgraph Ingestion_Trigger ["Multi-File Ingestion (REQ-F-05, REQ-F-06)"]
        CLINICIAN["Clinical Staff"]
        S3_PATIENT["S3: patient-data-upload/{PatientID}/ (REQ-F-05)"]
        EV_BRIDGE["Amazon EventBridge (REQ-F-06)"]
    end

    subgraph State_Machine ["AWS Step Functions Orchestration (REQ-F-06, REQ-NF-01)"]
        SFN_START["Execution Start: PatientID, StudyID"]
        MAP_OCR["Map State: Parallel OCR (REQ-F-07)"]
        S3_EXTRACT["S3: patient-extracted-data (REQ-F-08)"]
        KB_INGEST["Bedrock KB StartIngestionJob (REQ-F-09)"]
        INVOKE_AGENT["Invoke Single Bedrock Agent (REQ-F-12)"]
        SQS_DLQ["Amazon SQS Dead Letter Queue (REQ-NF-03)"]
    end

    subgraph AI_RAG_Services ["AI and Vector Engine (REQ-F-10, REQ-F-13, REQ-F-14)"]
        TEXTRACT_ASYNC["Amazon Textract Async Jobs (REQ-F-07)"]
        TITAN_EMB["Titan Embeddings v2 and Vector Store (REQ-F-10, REQ-F-11)"]
        DDB_PROTO["DynamoDB: study-protocols (REQ-F-04)"]
        BEDROCK_AGENT["Amazon Bedrock Reasoning Agent (REQ-F-12, REQ-F-14)"]
    end

    subgraph Persistence ["Verdict Storage (REQ-F-15)"]
        DDB_VERDICT["DynamoDB: patient-verdicts Table (REQ-F-15)"]
    end

    CLINICIAN -->|"Uploads Multi-PDF Records"| S3_PATIENT
    S3_PATIENT -->|"ObjectCreated Event"| EV_BRIDGE
    EV_BRIDGE -->|"Triggers Execution"| SFN_START
    SFN_START --> MAP_OCR
    MAP_OCR -->|"Parallel StartDocumentAnalysis"| TEXTRACT_ASYNC
    TEXTRACT_ASYNC -->|"Extracted Text"| S3_EXTRACT
    S3_EXTRACT --> KB_INGEST
    KB_INGEST -->|"Chunking and Embeddings"| TITAN_EMB
    KB_INGEST --> INVOKE_AGENT
    INVOKE_AGENT -->|"Pull Protocol Rules"| DDB_PROTO
    INVOKE_AGENT -->|"RAG Query patient_id"| TITAN_EMB
    INVOKE_AGENT -->|"Execute Single-Agent Evaluation"| BEDROCK_AGENT
    BEDROCK_AGENT -->|"Structured Verdict JSON"| DDB_VERDICT
    MAP_OCR -.->|"On Failure Retries Exceeded"| SQS_DLQ
    INVOKE_AGENT -.->|"On Unhandled Error"| SQS_DLQ
```

### 3.2 Patient Screening State Machine (`stateDiagram-v2`)

```mermaid
stateDiagram-v2
    [*] --> InitScreeningExecution: S3 Event Trigger (PatientID, StudyID)
    
    state "Parallel Map State: Process Patient PDFs (REQ-F-07)" as MapStateOcr {
        [*] --> LaunchAsyncOcr: Item = PDF S3 Key
        state "Start Textract Async Job" as LaunchAsyncOcr
        state "Wait for Completion Token" as WaitForOcrToken
        LaunchAsyncOcr --> WaitForOcrToken: StartDocumentAnalysis
        WaitForOcrToken --> [*]: Single PDF OCR Success
    }
    
    state "Save Extracted Text to S3 (REQ-F-08)" as SaveExtractedText
    state "Trigger Bedrock KB Ingestion (REQ-F-09)" as StartKbIngest
    state "Poll KB Ingestion Job Status" as PollKbStatus
    state "Check Ingestion Status" as CheckKbChoice <<choice>>
    state "Invoke Single Bedrock Agent (REQ-F-12, REQ-F-14)" as InvokeAgentTask
    state "PutItem to patient-verdicts Table (REQ-F-15)" as PersistVerdictTable
    state "Route to SQS DLQ & Alert Reviewer (REQ-NF-03)" as ScreeningErrorState

    InitScreeningExecution --> MapStateOcr: Validate Input Metadata
    MapStateOcr --> SaveExtractedText: All Files Completed Successfully
    MapStateOcr --> MapStateOcr: Catch [RateLimit / Retry with Exponential Backoff]
    MapStateOcr --> ScreeningErrorState: Catch [Fatal OCR Error / 3x Retries Exceeded]

    SaveExtractedText --> StartKbIngest: Write Extracted Text to S3
    StartKbIngest --> PollKbStatus: StartIngestionJob Call
    PollKbStatus --> CheckKbChoice: Fetch Status After 10s Wait
    
    CheckKbChoice --> InvokeAgentTask: Status == COMPLETE
    CheckKbChoice --> PollKbStatus: Status == IN_PROGRESS
    CheckKbChoice --> ScreeningErrorState: Status == FAILED

    InvokeAgentTask --> PersistVerdictTable: Structured Verdict JSON (MET, NOT_MET, UNCERTAIN)
    InvokeAgentTask --> InvokeAgentTask: Catch [Bedrock Throttling / Retry with Backoff]
    InvokeAgentTask --> ScreeningErrorState: Catch [Agent Invocation Failure]

    PersistVerdictTable --> [*]: Screening Succeeded (Status = PENDING Review)
    ScreeningErrorState --> [*]: Execution Failed (Captured in DLQ)
```

### 3.3 Execution Steps & State Specifications:
1. **Multi-File Trigger (`(REQ-F-05, REQ-F-06)`):** Clinical staff upload 1 or more PDF records to `s3://patient-data-upload/{PatientID}/`. S3 ObjectCreated triggers an EventBridge rule executing the Step Functions state machine.
2. **Parallel Asynchronous OCR (`(REQ-F-07)`):** Step Functions executes a dynamic `Map` state over all uploaded files. Each iteration executes `StartDocumentAnalysis` with an SQS/SNS completion callback.
3. **Raw Text Storage & Ingestion (`(REQ-F-08, REQ-F-09)`):** Extracted text is saved in `s3://patient-extracted-data/{PatientID}/`. Step Functions triggers `StartIngestionJob` on Amazon Bedrock Knowledge Bases.
4. **Vector Embedding & Chunk Metadata (`(REQ-F-10, REQ-F-11)`):** Bedrock Knowledge Bases automatically chunks the text (512 tokens with 20% overlap), generates embeddings using `amazon.titan-embed-text-v2`, and indexes chunks with metadata (`patient_id`, `source_filename`, `page_number`).
5. **Deterministic Single-Agent Reasoning (`(REQ-F-12, REQ-F-13, REQ-F-14)`):** A single Bedrock Agent receives `{PatientID, StudyID}`, fetches active criteria from DynamoDB `study-protocols`, queries Bedrock Knowledge Base filtered by `metadata.patient_id == {PatientID}` (`(REQ-NF-02)`), and populates the structured JSON verdict (`MET`, `NOT_MET`, `UNCERTAIN`) with citations and exact quotes.
6. **Persistence & Auditing (`(REQ-F-15)`):** Output verdict written to `patient-verdicts` with default status `PENDING`.

---

## 4. Workflow 3: Human-in-the-Loop Clinical Review Interface

This workflow describes the clinical reviewer workflow for reviewing AI-suggested eligibility verdicts alongside original patient documentation.

```mermaid
graph TD
    subgraph Client_App ["Clinical Reviewer Dashboard (REQ-F-16)"]
        DOCTOR["Licensed Clinical Reviewer"]
        UI_VIEW["React / Next.js Web UI (REQ-F-16)"]
    end

    subgraph API_Edge ["API and Security Gateway (REQ-SEC-02, REQ-SEC-04)"]
        COGNITO["Amazon Cognito MFA / RBAC (REQ-SEC-04)"]
        APIGW["Amazon API Gateway REST (REQ-SEC-02)"]
        LAMBDA_REV["AWS Lambda: Review API Handler"]
    end

    subgraph Data_Sources ["Protected Storage and Verdict Data (REQ-SEC-01)"]
        DDB_VERDICT["DynamoDB: patient-verdicts (REQ-F-15, REQ-F-18)"]
        S3_PATIENT["S3: patient-data-upload Encrypted (REQ-F-05)"]
    end

    DOCTOR -->|"Authenticates via MFA and RBAC"| COGNITO
    COGNITO -->|"Issues Bearer JWT"| UI_VIEW
    UI_VIEW -->|"GET /verdicts/PatientID"| APIGW
    APIGW -->|"Invokes with Claims"| LAMBDA_REV
    LAMBDA_REV -->|"Reads Verdict and Rules"| DDB_VERDICT
    LAMBDA_REV -->|"Generates 15-min Presigned URL"| S3_PATIENT
    LAMBDA_REV -->|"Returns Payload and PDF URLs"| UI_VIEW
    UI_VIEW -->|"Renders Side-by-Side Dual Pane"| DOCTOR
    DOCTOR -->|"Clicks Citation to Scroll to PDF Page"| UI_VIEW
    DOCTOR -->|"Submits Determination Approve / Reject / Override"| UI_VIEW
    UI_VIEW -->|"POST /verdicts/PatientID/signoff"| APIGW
    APIGW --> LAMBDA_REV
    LAMBDA_REV -->|"Saves Signoff and Immutable Audit Trail"| DDB_VERDICT
```
