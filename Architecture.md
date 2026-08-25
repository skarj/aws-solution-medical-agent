## 1. High-Level System Architecture

The AI-Assisted Clinical Trial Screening Platform employs an asynchronous, event-driven, serverless architecture on AWS. The design eliminates 24/7 idle infrastructure costs while providing strict multi-tenant data isolation and HIPAA compliance.

```mermaid
graph LR
    subgraph Client_Tier ["User and Client Interface"]
        WEB["Clinical Investigator Portal"]
    end

    subgraph Ingestion_Tier ["Storage and Async Ingestion"]
        S3["Amazon S3 Buckets"]
        SFN["AWS Step Functions State Machines"]
        OCR["Amazon Textract OCR"]
    end

    subgraph AI_Engine_Tier ["RAG and AI Reasoning Engine"]
        KB["Bedrock Knowledge Base and Vectors"]
        AGENT["Bedrock Reasoning Agent"]
    end

    subgraph Persistence_Tier ["Structured Persistence and Audit"]
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
    subgraph Coordinator_Tier ["Clinical Investigator Action"]
        USER["Clinical Investigator"]
        S3_PROTO["S3: protocol-data-upload"]
        EV_BRIDGE["Amazon EventBridge"]
    end

    subgraph Step_Functions_Tier ["AWS Step Functions Orchestration"]
        SFN_PROTO["Protocol Onboarding State Machine"]
        LAMBDA_STRUCT["Lambda: Rule Extraction Task"]
        SQS_DLQ["Amazon SQS Dead Letter Queue"]
    end

    subgraph External_Services ["Managed AI and Persistence Services"]
        TEXTRACT["Amazon Textract StartDocumentAnalysis"]
        BEDROCK_LLM["Amazon Bedrock Nova / Claude Sonnet"]
        DDB_PROTO["DynamoDB: study-protocols Table"]
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
    
    state "Start Textract Async OCR Task" as StartProtocolOcr
    state "Wait for Textract Completion Callback" as WaitForProtocolOcr
    state "Check OCR Status" as CheckProtocolOcr <<choice>>
    state "Execute Lambda Rule Structurer Task" as LambdaExtractRules
    state "Invoke Bedrock Model JSON Enforcement" as BedrockRulePrompt
    state "PutItem to study-protocols Table" as PersistProtocolRules
    state "Route to SQS DLQ & Alert Admin" as ProtocolErrorState

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
1. **Document Ingestion (`[REQ-F-01]`):** The Clinical Investigator uploads `protocol.pdf` to `s3://protocol-data-upload/{StudyID}/`.
2. **State Machine Execution Trigger (`[REQ-F-02]`):** S3 ObjectCreated event routes via Amazon EventBridge to launch the Protocol Onboarding Step Functions state machine with input `{StudyID, Bucket, Key}`.
3. **Asynchronous Layout & Form OCR (`[REQ-F-02]`):** Step Functions starts Amazon Textract `StartDocumentAnalysis` with `TABLES` and `FORMS` feature types, pausing execution until Textract completes.
4. **Structured Rule Extraction (`[REQ-F-03]`):** Step Functions executes a Lambda task passing the extracted text to an LLM on Amazon Bedrock (Amazon Nova Pro / Anthropic Claude 3.5 Sonnet) enforcing the itemized `study-protocols` JSON schema.
5. **Rule Persistence (`[REQ-F-04]`):** Structured rules are written to the `study-protocols` DynamoDB table under primary key `StudyID`.

---

## 3. Workflow 2: Automated Patient Screening Pipeline

This workflow processes multi-file patient records (up to 150 MB total, digital or scanned faxes/records) and performs RAG-augmented deterministic reasoning against protocol criteria using an AWS Step Functions state machine.

### 3.1 Component Flow Diagram

```mermaid
graph TD
    subgraph Ingestion_Trigger ["Multi-File Ingestion"]
        CLINICIAN["Clinical Investigator"]
        S3_PATIENT["S3: patient-data-upload/{PatientID}/"]
        EV_BRIDGE["Amazon EventBridge"]
    end

    subgraph State_Machine ["AWS Step Functions Orchestration"]
        SFN_START["Execution Start: PatientID, StudyID"]
        MAP_OCR["Map State: Parallel OCR"]
        S3_EXTRACT["S3: patient-extracted-data"]
        KB_INGEST["Bedrock KB StartIngestionJob"]
        INVOKE_AGENT["Invoke Single Bedrock Agent"]
        SQS_DLQ["Amazon SQS Dead Letter Queue"]
    end

    subgraph AI_RAG_Services ["AI and Vector Engine"]
        TEXTRACT_ASYNC["Amazon Textract Async Jobs"]
        TITAN_EMB["Titan Embeddings v2 and Vector Store"]
        DDB_PROTO["DynamoDB: study-protocols"]
        BEDROCK_AGENT["Amazon Bedrock Reasoning Agent"]
    end

    subgraph Persistence ["Verdict Storage"]
        DDB_VERDICT["DynamoDB: patient-verdicts Table"]
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
    
    state "Parallel Map State: Process Patient PDFs" as MapStateOcr {
        [*] --> LaunchAsyncOcr: Item = PDF S3 Key
        state "Start Textract Async Job" as LaunchAsyncOcr
        state "Wait for Completion Token" as WaitForOcrToken
        LaunchAsyncOcr --> WaitForOcrToken: StartDocumentAnalysis
        WaitForOcrToken --> [*]: Single PDF OCR Success
    }
    
    state "Save Extracted Text to S3" as SaveExtractedText
    state "Trigger Bedrock KB Ingestion" as StartKbIngest
    state "Poll KB Ingestion Job Status" as PollKbStatus
    state "Check Ingestion Status" as CheckKbChoice <<choice>>
    state "Invoke Single Bedrock Agent" as InvokeAgentTask
    state "PutItem to patient-verdicts Table" as PersistVerdictTable
    state "Route to SQS DLQ & Alert Reviewer" as ScreeningErrorState

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
1. **Multi-File Trigger (`[REQ-F-05, REQ-F-06]`):** The Clinical Investigator uploads 1 or more PDF records to `s3://patient-data-upload/{PatientID}/`. S3 ObjectCreated triggers an EventBridge rule executing the Step Functions state machine.
2. **Parallel Asynchronous OCR (`[REQ-F-07]`):** Step Functions executes a dynamic `Map` state over all uploaded files. Each iteration executes `StartDocumentAnalysis` with an SQS/SNS completion callback.
3. **Raw Text Storage & Ingestion (`[REQ-F-08, REQ-F-09]`):** Extracted text is saved in `s3://patient-extracted-data/{PatientID}/`. Step Functions triggers `StartIngestionJob` on Amazon Bedrock Knowledge Bases.
4. **Vector Embedding & Chunk Metadata (`[REQ-F-10, REQ-F-11]`):** Bedrock Knowledge Bases automatically chunks the text (512 tokens with 20% overlap), generates embeddings using `amazon.titan-embed-text-v2`, and indexes chunks with metadata (`patient_id`, `source_filename`, `page_number`).
5. **Deterministic Single-Agent Reasoning (`[REQ-F-12, REQ-F-13, REQ-F-14]`):** A single Bedrock Agent receives `{PatientID, StudyID}`, fetches active criteria from DynamoDB `study-protocols`, queries Bedrock Knowledge Base filtered by `metadata.patient_id == {PatientID}` (`[REQ-NF-02]`), and populates the structured JSON verdict (`MET`, `NOT_MET`, `UNCERTAIN`) with citations and exact quotes.
6. **Persistence & Auditing (`[REQ-F-15]`):** Output verdict written to `patient-verdicts` with default status `PENDING`.

---

## 4. Workflow 3: Human-in-the-Loop Clinical Review Interface

This workflow describes the Clinical Investigator review process for reviewing AI-suggested eligibility verdicts alongside original patient documentation.

```mermaid
graph TD
    subgraph Client_App ["Clinical Investigator Dashboard"]
        DOCTOR["Clinical Investigator"]
        UI_VIEW["React / Next.js Web UI"]
    end

    subgraph API_Edge ["API and Security Gateway"]
        COGNITO["Amazon Cognito MFA / RBAC"]
        APIGW["Amazon API Gateway REST"]
        LAMBDA_REV["AWS Lambda: Review API Handler"]
    end

    subgraph Data_Sources ["Protected Storage and Verdict Data"]
        DDB_VERDICT["DynamoDB: patient-verdicts"]
        S3_PATIENT["S3: patient-data-upload Encrypted"]
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

### 4.1 Reviewer Interaction & Binding Signoff:
1. **Authenticated Access (`[REQ-SEC-04]`):** The Clinical Investigator logs into the portal using Amazon Cognito with Multi-Factor Authentication.
2. **Review Dashboard Loading (`[REQ-F-16]`):** The Web UI requests the verdict payload from API Gateway and receives short-lived (15-minute) S3 presigned URLs for the patient's original PDF records (`[REQ-SEC-01]`).
3. **Interactive Citation Navigation (`[REQ-F-17]`):** Selecting any criterion or extracted evidence quote in the checklist instantly scrolls the embedded PDF viewer to the cited page and highlights the source text.
4. **Binding Determination (`[REQ-F-18]`):** The Clinical Investigator confirms or overrides the AI recommendation, inputs clinical notes, and clicks `Approve`, `Reject`, or `Manual Override`, writing an immutable audit record to `patient-verdicts`.
