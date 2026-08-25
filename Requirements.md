## 1. Executive Summary & Objective
The AI-Assisted Clinical Trial Screening Platform automates patient eligibility assessment by analyzing unstructured medical records and matching them against trial inclusion and exclusion criteria. 
The platform operates on a **Human-in-the-Loop (HITL)** architecture. The AI automates data extraction, protocol matching, and citation mapping, while licensed clinical personnel retain full authority over final trial enrollment decisions.

---

## 2. Scope & Technical Constraints
* **Monthly Volume:** ~100 patient screening requests per month.
* **Patient Records:** 1 or more PDF files per patient (scanned images, faxes, or searchable PDFs; file sizes up to 150 MB each).
* **Protocol Documents:**  1 protocol PDF per study (30–100 pages; searchable or scanned).
* **Operational Cost Target:** Under $5,000 / year total AWS infrastructure spend.

---

## 3. Functional Requirements

### FR-1: Protocol Onboarding & Structuring
* **FR-1.1:** The system shall accept a single study protocol PDF (30–100 pages) via a web portal into an encrypted S3 bucket (`protocol-data-upload`).
* **FR-1.2:** S3 upload events must automatically invoke Amazon Textract (asynchronous) to parse text, forms, and tables from digital or scanned/faxed protocol files.
* **FR-1.3:** An AWS Lambda function must pass parsed protocol text to an LLM on Amazon Bedrock (such as Amazon Nova Pro or Anthropic Claude Sonnet) to extract all inclusion and exclusion rules into an itemized JSON array.
* **FR-1.4:** Extracted protocol rules must be stored in an Amazon DynamoDB table (`study-protocols`), indexed by `StudyID`.

### FR-2: Patient Record Ingestion & Asynchronous OCR
* **FR-2.1:** The system shall support uploading multi-file patient medical records under a single `PatientID` directory into an encrypted S3 bucket (`patient-data-upload`).
* **FR-2.2:** Document uploads must initiate an AWS Step Functions state machine execution to manage long-running background tasks.
* **FR-2.3:** Step Functions must execute a Map State to run Amazon Textract asynchronously (`StartDocumentAnalysis`) across all patient PDFs in parallel.
* **FR-2.4:** Textract output files must be written to an intermediate S3 bucket (`patient-extracted-data`) before triggering the next ingestion step.

### FR-3: RAG Indexing & Vector Storage
* **FR-3.1:** The system shall invoke Amazon Bedrock Knowledge Bases (`StartIngestionJob`) to automatically chunk parsed patient text and generate embeddings using Amazon Titan Text Embeddings.
* **FR-3.2:** Embeddings and chunk metadata must be stored in **Amazon S3 Vectors** to avoid fixed 24/7 database instance fees.
* **FR-3.3:** Every stored vector chunk must maintain metadata tags for `patient_id`, `source_filename`, and `page_number` to enable document-level citations.

### FR-4: AI Reasoning & Eligibility Verdict Generation
* **FR-4.1:** The system shall run a single **Amazon Bedrock Agent** to perform eligibility analysis. Multi-agent setups are explicitly excluded to maintain lower latency and cost.
* **FR-4.2:** The agent must pull active inclusion and exclusion rules from DynamoDB and perform RAG queries against Amazon S3 Vectors filtered by `patient_id`.
* **FR-4.3:** The agent must evaluate rules across all uploaded patient documents simultaneously, using foundation model JSON Schema enforcement to output a structured verdict containing rule statuses (`MET`, `NOT_MET`, `UNCERTAIN`), direct quotes, source filenames, and page citations.
* **FR-4.4:** The generated verdict must be saved to the patient-verdicts` Amazon DynamoDB table.

### FR-5: Clinical Review Interface (Human-in-the-Loop)
* **FR-5.1:** The Web UI must present a side-by-side view: an interactive PDF viewer on the left, and an AI criteria checklist on the right.
* **FR-5.2:** Selecting any criterion or quote in the checklist must jump directly to the cited page and highlight text in the correct patient PDF.
* **FR-5.3:** Clinical staff must be provided with interactive controls (`Approve`, `Reject`, `Manual Override`, `Notes`) to log the binding determination.

---

## 4. Non-Functional Requirements

### NFR-1: Security & HIPAA Compliance
* **NFR-1.1:** All data at rest across Amazon S3, DynamoDB, and Amazon S3 Vectors must be encrypted using AWS KMS Customer-Managed Keys (CMK).
* **NFR-1.2:** All network communications must use TLS 1.3 encryption.
* **NFR-1.3:** The infrastructure must operate under a signed AWS Business Associate Addendum (BAA).
* **NFR-1.4:** Authentication and access control must be managed via Amazon Cognito using Role-Based Access Control (RBAC) and Multi-Factor Authentication (MFA).
* **NFR-1.5:** AWS CloudTrail and AWS CloudWatch must record all document access requests, API calls, and administrative actions for regulatory compliance.

### NFR-2: Performance & Reliability
* **NFR-2.1:** Background ingestion and screening of multi-file patient records (up to 150 MB total) must complete within 10 minutes.
* **NFR-2.2:** RAG vector search retrieval times must remain under 2 seconds per query.
* **NFR-2.3:** Step Functions must handle task retries, exponential backoffs, and dead-letter queues (DLQs) for failed Textract or Bedrock jobs.

---

## 5. Database Schema Specifications

### 5.1 Protocol Schema (`study-protocols` DynamoDB Table)
```json
{
  "Study_ID": "STRING (Partition Key)",
  "Study_Name": "STRING",
  "inclusion_criteria": [
    { "id": "STRING", "rule": "STRING" }
  ],
  "exclusion_criteria": [
    { "id": "STRING", "rule": "STRING" }
  ],
  "created_at": "STRING (ISO8601)"
}
```
### 5.2 Verdict Schema (`patient-verdicts` DynamoDB Table)

```JSON
{
  "Patient_ID": "STRING (Partition Key)",
  "Study_ID": "STRING (Sort Key)",
  "overall_recommendation": "ELIGIBLE | INELIGIBLE | MANUAL_REVIEW_REQUIRED",
  "criteria_evaluations": [
    {
      "criterion_id": "STRING",
      "type": "INCLUSION | EXCLUSION",
      "status": "MET | NOT_MET | UNCERTAIN",
      "evidence_quote": "STRING",
      "source_filename": "STRING",
      "page_citation": "NUMBER",
      "confidence_score": "NUMBER"
    }
  ],
  "reviewer_signoff": {
    "status": "PENDING | APPROVED | REJECTED",
    "reviewed_by": "STRING",
    "notes": "STRING",
    "timestamp": "STRING (ISO8601)"
  }
}
```
