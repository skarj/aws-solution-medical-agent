# architecture.md

## 1. Workflow 1: Protocol Onboarding Pipeline

This workflow parses a single study protocol PDF (30–100 pages, text or scanned) once during trial setup and stores its inclusion/exclusion rules as structured data.

[Coordinator Uploads Protocol PDF] ---> [S3: protocol-data-upload]
                                              |
                                  (S3 Event Notification)
                                              v
                               [Amazon Textract Async OCR]
                                              |
                                     (Extracted Raw Text)
                                              v
                                 [Amazon Bedrock LLM Task]
                                              |
                                  (Parses Inclusion/Exclusion)
                                              v
                                [DynamoDB: Study_Protocols Table]

1. **Document Upload:** Coordinator uploads protocol.pdf into s3://protocol-data-upload/{StudyID}/.
2. **Text Extraction:** An S3 Event Notification triggers an AWS Lambda function that calls Amazon Textract `StartDocumentAnalysis` to handle scanned or digital pages.
3. **Structured Extraction:** Upon Textract completion, Lambda routes the raw text to an LLM in Amazon Bedrock with an extraction prompt.
4. **Data Persistence:** The LLM returns structured JSON rules, which Lambda saves to the StudyProtocols DynamoDB table under StudyID.

---

## 2. Workflow 2: Automated Patient Screening Pipeline

This workflow processes multi-file patient medical records (up to 150 MB each, scanned or digital) and performs AI evaluation against protocol rules.

[Upload 1+ Patient PDFs] ---> [S3: patient-data-upload/{PatientID}/]
                                         |
                             (Step Functions Map State)
                                         v
                   +------------------------------------------+
                   | Parallel Textract Async Processing Jobs  |
                   +------------------------------------------+
                                         |
                                (Text Output Saved)
                                         v
                     [Bedrock Knowledge Base + S3 Vectors]
                          (Tagged with PatientID)
                                         |
                                         v
                        [Bedrock Agent Screening Task]
                     (Queries Unified Patient Vector Space)
                                         |
                                         v
                          [DynamoDB: PatientVerdicts]

### Execution Steps
1. **Multi-File Upload:** Clinical staff upload 1 or more patient PDFs to s3://patient-data-upload/{PatientID}/.
2. **Step Functions Trigger:** An S3 Event Notification launches a Step Functions state machine execution containing PatientID and StudyID.
3. **Parallel OCR (Map State):** Step Functions executes a Map State, launching asynchronous Amazon Textract jobs for each uploaded PDF simultaneously. Step Functions pauses using Task Tokens until Textract emits completion notifications via Amazon SNS.
4. **Indexing via Bedrock Knowledge Bases:**
   * Parsed text files are written to s3://patient-extracted-data/.
   * Step Functions calls `StartIngestionJob` on Amazon Bedrock Knowledge Bases.
   * Bedrock Knowledge Bases automatically chunks the text, generates Titan embeddings, and writes the vector index into Amazon S3 Vectors.
   * Chunks maintain metadata tags for patient_id, source_filename, and page_number.
5. **Single Bedrock Agent Evaluation:**
   * A Lambda step invokes a single Amazon Bedrock Agent.
   * The agent fetches the active protocol rules from the Study_Protocols DynamoDB table.
   * The agent queries Amazon S3 Vectors for matching evidence across all files for that patient_id.
   * The foundation model enforces a structured JSON Schema output containing rule determinations, direct quotes, file names, and page numbers.
6. **Verdict Persistence:** Step Functions writes the JSON response to the Patient_Verdicts DynamoDB table and sends a notification to the Web UI.

---

## 3. Workflow 3: Human-in-the-Loop Web UI

[Clinical Reviewer] ---> [React / Next.js Web UI] ---> [API Gateway / Lambda]
                                                              |
                                                +-------------+-------------+
                                                v                           v
                                      [Read Patient PDFs]        [Fetch Verdict & Rules]
                                       (S3 Presigned URL)          (DynamoDB Lookups)

1. **Dashboard Rendering:** The UI fetches screening results from Patient_Verdicts via API Gateway and Lambda.
2. **Dual Pane Display:** The original patient PDF renders on the left pane via S3 presigned URLs, while the AI checklist displays on the right pane.
3. **Interactive Citations:** Clicking any evidence quote in the checklist passes source_filename and page_citation to the viewer, scrolling to the exact page and highlighting the source text.
4. **Clinical Decision Audit:** The reviewer confirms or overrides the AI findings, clicks Approve or Reject, and saves an immutable audit record to DynamoDB.


