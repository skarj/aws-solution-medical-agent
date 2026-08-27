## 1. High-Level System Architecture

The AI-Assisted Clinical Trial Screening Platform employs an asynchronous, event-driven, serverless architecture deployed in AWS Region **`us-west-2` (Oregon)** (`[REQ-OPS-01]`). The design eliminates 24/7 idle infrastructure costs while providing strict multi-tenant data isolation, HIPAA compliance, and payload resiliency using the S3 Claim-Check pattern.

```mermaid
graph LR
    WEB["Clinical Investigator Portal"]
    S3["Amazon S3 Buckets"]
    SFN["AWS Step Functions State Machines"]
    OCR["Amazon Textract OCR"]
    CONSOLIDATE["Full-Text Document Consolidation"]
    AGENT["Bedrock Reasoning Agent"]
    DDB["Amazon DynamoDB Tables"]

    WEB -->|"Uploads Records and Protocols"| S3
    S3 -->|"Triggers State Machines"| SFN
    SFN -->|"Executes Async OCR"| OCR
    OCR -->|"Merges Pages into Full-Text Document"| CONSOLIDATE
    CONSOLIDATE -->|"Delivers Complete Document Context"| AGENT
    AGENT -->|"Saves Rules and Verdicts"| DDB
    DDB -->|"Displays AI Checklist"| WEB
```

---

## 2. Workflow 1: Protocol Onboarding & Rule Extraction Pipeline

This workflow parses a clinical study protocol PDF (30–100 pages) during trial initialization and extracts itemized inclusion and exclusion criteria into structured DynamoDB records using an AWS Step Functions state machine.

### 2.1 Component Flow Diagram

```mermaid
graph TD
    USER["Clinical Investigator"]
    S3_PROTO["S3: protocol-data-upload"]
    EV_BRIDGE["Amazon EventBridge"]
    SFN_PROTO["Protocol Onboarding State Machine"]
    S3_PROTO_EXTRACTED["S3: protocol-extracted-data"]
    LAMBDA_STRUCT["Lambda: Rule Extraction Task"]
    SQS_DLQ["Amazon SQS Dead Letter Queue"]
    TEXTRACT["Amazon Textract StartDocumentAnalysis"]
    BEDROCK_LLM["Amazon Bedrock Anthropic Claude Sonnet 5"]
    DDB_PROTO["DynamoDB: study-protocols Table"]

    USER -->|"Uploads protocol.pdf"| S3_PROTO
    S3_PROTO -->|"S3 ObjectCreated Event"| EV_BRIDGE
    EV_BRIDGE -->|"Triggers Execution StudyID"| SFN_PROTO
    SFN_PROTO -->|"StartDocumentAnalysis Async"| TEXTRACT
    SFN_PROTO -->|"Polls GetDocumentAnalysis on Wait Loop"| TEXTRACT
    TEXTRACT -->|"Writes Extracted Text on SUCCEEDED"| S3_PROTO_EXTRACTED
    SFN_PROTO -->|"Invokes Task with S3 Pointer"| LAMBDA_STRUCT
    LAMBDA_STRUCT -->|"Reads Extracted Text"| S3_PROTO_EXTRACTED
    LAMBDA_STRUCT -->|"Prompts with JSON Schema"| BEDROCK_LLM
    BEDROCK_LLM -->|"Returns Structured Rules JSON"| LAMBDA_STRUCT
    LAMBDA_STRUCT -->|"PutItem StudyID, Rules"| DDB_PROTO
    SFN_PROTO -.->|"On Task Failure / Retry Exceeded"| SQS_DLQ
```

### 2.2 Protocol Onboarding State Machine (`stateDiagram-v2`)

```mermaid
stateDiagram-v2
    [*] --> StartProtocolExecution: S3 Event Trigger (StudyID)
    
    state "Start Textract Async OCR Job" as StartProtocolOcr
    state "Wait 5s Polling Interval" as WaitForProtocolOcr
    state "Poll GetDocumentAnalysis Status" as PollProtocolOcr
    state "Check OCR Status" as CheckProtocolOcr <<choice>>
    state "Save Extracted Text to S3 protocol-extracted-data" as SaveProtocolText
    state "Execute Lambda Rule Structurer Task" as LambdaExtractRules
    state "Invoke Bedrock Model JSON Enforcement" as BedrockRulePrompt
    state "PutItem to study-protocols Table" as PersistProtocolRules
    state "Route to SQS DLQ & Alert Admin" as ProtocolErrorState

    StartProtocolExecution --> StartProtocolOcr
    StartProtocolOcr --> WaitForProtocolOcr: StartDocumentAnalysis(Tables) Returns JobId
    WaitForProtocolOcr --> PollProtocolOcr: Elapsed 5s
    PollProtocolOcr --> CheckProtocolOcr: GetDocumentAnalysis(JobId)
    
    CheckProtocolOcr --> SaveProtocolText: OCR Status == SUCCEEDED
    CheckProtocolOcr --> WaitForProtocolOcr: OCR Status == IN_PROGRESS
    PollProtocolOcr --> PollProtocolOcr: Catch [Throttling - Retry with Backoff]
    CheckProtocolOcr --> ProtocolErrorState: OCR Status == FAILED / Retries Exhausted

    SaveProtocolText --> LambdaExtractRules: Write Extracted Text to S3
    LambdaExtractRules --> BedrockRulePrompt: Send Formatted Markdown Text
    BedrockRulePrompt --> PersistProtocolRules: Valid Structured JSON Rules Returned
    BedrockRulePrompt --> LambdaExtractRules: Catch [Throttling / Retry with Backoff]
    BedrockRulePrompt --> ProtocolErrorState: Catch [Schema Violation / Fatal Error]

    PersistProtocolRules --> [*]: Protocol Onboarding Succeeded
    ProtocolErrorState --> [*]: Execution Terminated (DLQ Captured)
```

### 2.3 Detailed Pipeline Stages:
1. **Document Ingestion (`[REQ-F-01]`):** The Clinical Investigator uploads `protocol.pdf` to `s3://protocol-data-upload/{StudyID}/`.
2. **State Machine Execution Trigger (`[REQ-F-02]`):** S3 ObjectCreated event routes via Amazon EventBridge to launch the Protocol Onboarding Step Functions state machine with input metadata `{StudyID, Bucket, Key}`.
3. **Asynchronous Table-Preserving OCR (`[REQ-F-03]`):** Step Functions starts Amazon Textract `StartDocumentAnalysis` with the `TABLES` feature type only (`FORMS` was dropped from `[REQ-F-03]` on 2026-08-25 to bring Textract spend within `[REQ-COST-01]`), then polls `GetDocumentAnalysis` on a `Wait`/`Choice` loop (5-second interval) until the job reports `SUCCEEDED`, `FAILED`, or the retry budget is exhausted. Amazon Textract has no native Step Functions `.sync`/`.waitForTaskToken` service integration, so polling — not a callback — is the supported asynchronous pattern.
4. **Structured Rule Extraction & Payload Safety (`[REQ-F-04, REQ-F-05]`):** Textract writes OCR output to the intermediate `protocol-extracted-data` S3 bucket immediately upon job completion. Following the Claim-Check pattern to prevent exceeding the Step Functions 256 KB state limit, Step Functions passes only the S3 object pointer to an AWS Lambda task, which reads the extracted text and sends it to **Anthropic Claude Sonnet 5** on Amazon Bedrock enforcing the itemized `study-protocols` JSON schema.
5. **Rule Persistence (`[REQ-F-06]`):** Structured rules are written to the `study-protocols` DynamoDB table under primary key `StudyID`.

---

## 3. Workflow 2: Automated Patient Screening Pipeline

This workflow processes multi-file patient records (up to 150 MB total, digital or scanned faxes/records) and performs RAG-augmented deterministic reasoning against protocol criteria using an AWS Step Functions state machine.

### 3.1 Component Flow Diagram

```mermaid
graph TD
    CLINICIAN["Clinical Investigator"]
    S3_PATIENT["S3: patient-data-upload/{PatientID}/"]
    EV_BRIDGE["Amazon EventBridge"]
    SFN_START["Execution Start: PatientID, StudyID"]
    MAP_OCR["Map State: Parallel OCR"]
    S3_EXTRACT["S3: patient-extracted-data"]
    LAMBDA_CONSOLIDATE["Lambda: Patient Text Consolidator"]
    S3_CONSOLIDATED["S3: patient-consolidated-text"]
    LAMBDA_SCREEN["Lambda: patient-screening-handler"]
    SQS_DLQ["Amazon SQS Dead Letter Queue"]
    TEXTRACT_ASYNC["Amazon Textract Async Jobs"]
    DDB_PROTO["DynamoDB: study-protocols"]
    BEDROCK_AGENT["Amazon Bedrock Agent: Anthropic Claude Sonnet 5"]
    DDB_VERDICT["DynamoDB: patient-verdicts Table"]

    CLINICIAN -->|"Uploads Multi-PDF Records"| S3_PATIENT
    S3_PATIENT -->|"ObjectCreated Event"| EV_BRIDGE
    EV_BRIDGE -->|"Triggers Execution"| SFN_START
    SFN_START --> MAP_OCR
    MAP_OCR -->|"Parallel StartDocumentAnalysis Async"| TEXTRACT_ASYNC
    MAP_OCR -->|"Polls GetDocumentAnalysis Per Item"| TEXTRACT_ASYNC
    TEXTRACT_ASYNC -->|"Extracted Text on SUCCEEDED"| S3_EXTRACT
    S3_EXTRACT -->|"Reads All Per-File Extracted Text"| LAMBDA_CONSOLIDATE
    LAMBDA_CONSOLIDATE -->|"Writes Ordered, Page-Annotated Full Text"| S3_CONSOLIDATED
    S3_CONSOLIDATED -->|"Claim-Check Pointer"| LAMBDA_SCREEN
    LAMBDA_SCREEN -->|"Pull Protocol Rules"| DDB_PROTO
    LAMBDA_SCREEN -->|"Invoke With Complete Document Text"| BEDROCK_AGENT
    BEDROCK_AGENT -->|"Structured Verdict JSON"| LAMBDA_SCREEN
    LAMBDA_SCREEN -->|"PutItem Verdict"| DDB_VERDICT
    MAP_OCR -.->|"On Failure Retries Exceeded"| SQS_DLQ
    LAMBDA_SCREEN -.->|"On Unhandled Error"| SQS_DLQ
```

### 3.2 Patient Screening State Machine (`stateDiagram-v2`)

```mermaid
stateDiagram-v2
    [*] --> InitScreeningExecution: S3 Event Trigger (PatientID, StudyID)
    
    state "Parallel Map State: Process Patient PDFs" as MapStateOcr {
        [*] --> LaunchAsyncOcr: Item = PDF S3 Key
        state "Start Textract Async Job" as LaunchAsyncOcr
        state "Wait 5s Polling Interval" as WaitForOcrPoll
        state "Poll GetDocumentAnalysis Status" as PollOcrStatus
        state "Check OCR Status" as CheckOcrStatus <<choice>>
        LaunchAsyncOcr --> WaitForOcrPoll: StartDocumentAnalysis Returns JobId
        WaitForOcrPoll --> PollOcrStatus: Elapsed 5s
        PollOcrStatus --> CheckOcrStatus: GetDocumentAnalysis(JobId)
        CheckOcrStatus --> [*]: OCR Status == SUCCEEDED
        CheckOcrStatus --> WaitForOcrPoll: OCR Status == IN_PROGRESS
    }
    
    state "Save Extracted Text to S3" as SaveExtractedText
    state "Execute Lambda Text Consolidator Task" as ConsolidateText
    state "Write Full-Text Document to S3 (Claim-Check)" as SaveConsolidatedText
    state "Execute Lambda patient-screening-handler Task" as InvokeAgentTask
    state "PutItem to patient-verdicts Table" as PersistVerdictTable
    state "Route to SQS DLQ & Alert Reviewer" as ScreeningErrorState

    InitScreeningExecution --> MapStateOcr: Validate Input Metadata
    MapStateOcr --> SaveExtractedText: All Files Completed Successfully
    MapStateOcr --> MapStateOcr: Catch [RateLimit / Retry with Exponential Backoff]
    MapStateOcr --> ScreeningErrorState: Catch [Fatal OCR Error / 3x Retries Exceeded]

    SaveExtractedText --> ConsolidateText: Write Extracted Text to S3
    ConsolidateText --> SaveConsolidatedText: Merge All Files into Ordered, Page-Annotated Text
    ConsolidateText --> ScreeningErrorState: Catch [Consolidation Failure]

    SaveConsolidatedText --> InvokeAgentTask: Pass S3 Pointer (Claim-Check)
    InvokeAgentTask --> PersistVerdictTable: Structured Verdict JSON (MET, NOT_MET, UNCERTAIN)
    InvokeAgentTask --> InvokeAgentTask: Catch [Bedrock Throttling / Retry with Backoff]
    InvokeAgentTask --> ScreeningErrorState: Catch [Agent Invocation Failure]

    PersistVerdictTable --> [*]: Screening Succeeded (Status = PENDING Review)
    ScreeningErrorState --> [*]: Execution Failed (Captured in DLQ)
```

### 3.3 Execution Steps & State Specifications:
1. **Multi-File Trigger (`[REQ-F-07, REQ-F-08]`):** The Clinical Investigator uploads 1 or more PDF records to `s3://patient-data-upload/{PatientID}/`. S3 ObjectCreated triggers an EventBridge rule executing the Step Functions state machine.
2. **Parallel Asynchronous OCR (`[REQ-F-09]`):** Step Functions executes a dynamic `Map` state over all uploaded files. Each iteration calls `StartDocumentAnalysis`, then polls `GetDocumentAnalysis` on a `Wait`/`Choice` loop (5-second interval) per item until `SUCCEEDED` or `FAILED` — Amazon Textract has no native Step Functions callback integration, so each Map iteration polls independently rather than waiting on a shared notification.
3. **Raw Text Storage & Full-Document Consolidation (`[REQ-F-10, REQ-F-11]`):** Extracted text is saved in `s3://patient-extracted-data/{PatientID}/`, one object per uploaded file. Step Functions invokes the `Lambda: Patient Text Consolidator` task, which reads every per-file object for the `PatientID`, concatenates them in a stable order with explicit source-filename and page-number boundary markers, and writes a single ordered full-text document to `s3://patient-consolidated-text/{PatientID}/`. No chunking or embedding occurs — this replaces the prior Bedrock Knowledge Base ingestion/vector-indexing step per the 2026-08-27 RAG-to-full-document-reasoning decision (`[REQ-F-11], [REQ-NF-05]`).
4. **Citation Metadata Preservation (`[REQ-F-13]`):** Every page boundary within the consolidated document retains its `patient_id`, `source_filename`, and `page_number` tags inline, so the reasoning step can still cite an exact source location for every claim without a vector index.
5. **Deterministic Full-Document Single-Agent Reasoning (`[REQ-F-14, REQ-F-15, REQ-F-16, REQ-NF-04, REQ-NF-05]`):** Step Functions invokes `Lambda: patient-screening-handler` with `{PatientID, StudyID}` and the `[REQ-F-11]` S3 Claim-Check pointer (respecting the Step Functions 256 KB state limit, since the full consolidated text — roughly 130,000 tokens for an average 200-page patient record set — would exceed it). The Lambda reads the complete consolidated text from S3, fetches active criteria from DynamoDB `study-protocols`, and invokes a single Bedrock Agent powered by **Anthropic Claude Sonnet 5** (mandated by `[REQ-F-14]`) with the entire patient document passed as direct reasoning input — no retrieval step, no similarity search. Claude Sonnet 5's verified 1,000,000-token context window (`docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5.html`) comfortably holds the full document at baseline scale. The agent populates the structured JSON verdict (`MET`, `NOT_MET`, `UNCERTAIN`) with citations and exact quotes, setting `UNCERTAIN` rather than inferring when no supporting text exists (`[REQ-NF-04]`). As with the protocol-extraction Lambda in Workflow 1, the agent's `foundationModel` is set to the Geographic (US) cross-Region inference profile `us.anthropic.claude-sonnet-5` (via `CreateAgent`'s `foundationModel` field) rather than a bare model ID, since Claude Sonnet 5 has no `us-west-2` In-Region support — permitted by the `[REQ-OPS-01]` relaxation; see `Security.md`. **Open risk (flagged, not yet validated):** processing ~130,000 input tokens per screening (versus the prior ~24,000-token retrieved-chunk average) has not been latency-tested against the `[REQ-NF-01]` 10-minute end-to-end SLA; confirm actual Bedrock inference latency at this input size before production deployment.
6. **Persistence & Auditing (`[REQ-F-17]`):** `Lambda: patient-screening-handler` writes the output verdict to `patient-verdicts` with default status `PENDING`.

---

## 4. Workflow 3: Human-in-the-Loop Clinical Review Interface

This workflow describes the Clinical Investigator review process for reviewing AI-suggested eligibility verdicts alongside original patient documentation.

```mermaid
graph TD
    DOCTOR["Clinical Investigator"]
    UI_VIEW["React / Next.js Web UI"]
    COGNITO["Amazon Cognito MFA / RBAC"]
    APIGW["Amazon API Gateway REST"]
    LAMBDA_REV["AWS Lambda: Review API Handler"]
    DDB_VERDICT["DynamoDB: patient-verdicts"]
    S3_PATIENT["S3: patient-data-upload Encrypted"]

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
2. **Review Dashboard Loading (`[REQ-F-18]`):** The Web UI requests the verdict payload from API Gateway and receives short-lived (15-minute) S3 presigned URLs for the patient's original PDF records (`[REQ-SEC-01]`).
3. **Interactive Citation Navigation (`[REQ-F-19]`):** Selecting any criterion or extracted evidence quote in the checklist instantly scrolls the embedded PDF viewer to the cited page and highlights the source text.
4. **Binding Determination (`[REQ-F-20]`):** The Clinical Investigator confirms or overrides the AI recommendation, inputs clinical notes, and clicks `Approve`, `Reject`, or `Manual Override`, writing an immutable audit record to `patient-verdicts`.
