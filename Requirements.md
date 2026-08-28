## 1. Executive Summary & Objective
The AI-Assisted Clinical Trial Screening Platform automates patient eligibility assessment by analyzing unstructured medical records and matching them against trial inclusion and exclusion criteria. 
The platform operates on a **Human-in-the-Loop (HITL)** architecture. The AI automates data extraction, protocol matching, and citation mapping, while the **Clinical Investigator** retains full authority over final trial enrollment decisions across protocol onboarding, patient record upload, and eligibility review.

---

## 2. Scope & Budget Constraints

* **Monthly Volume:** ~100 patient screening requests per month.
* **Patient Records:** 1 or more PDF files per patient (scanned images, faxes, or searchable PDFs; file sizes up to 150 MB each, average 2 PDFs / 100 pages per file).
* **Protocol Documents:** 1 protocol PDF per study (30–100 pages; searchable or scanned).
* **[Approved] [REQ-COST-01] Operational Budget Target:** Total AWS infrastructure spend must remain under $5,000 / year (~$416.66 / month) at baseline operational capacity.

---

## 3. Functional Requirements

### 3.1 Protocol Onboarding & Structuring
* **[Approved] [REQ-F-01]:** The system shall accept a single study protocol PDF (30–100 pages, searchable or scanned) uploaded by the Clinical Investigator via a web portal into an encrypted S3 bucket (`protocol-data-upload`).
* **[Approved] [REQ-F-02]:** Protocol PDF upload events must automatically initiate an AWS Step Functions state machine execution to orchestrate asynchronous background processing.
* **[Approved] [REQ-F-03]:** Step Functions shall invoke Amazon Textract (`aws-sdk:textract:startDocumentAnalysis`) with the TABLES feature enabled (`FeatureTypes=["TABLES"]`) to extract raw text and preserve table structures. Step Functions shall manage job completion natively using a Wait/Choice polling loop (`aws-sdk:textract:getDocumentAnalysis`) before proceeding.
* **[Approved] [REQ-F-04]:** Textract output files must be written to an intermediate S3 bucket (`protocol-extracted-data`) before triggering the next ingestion step.
* **[Approved] [REQ-F-05]:** Step Functions must execute a Lambda task passing the raw text from `protocol-extracted-data` S3 bucket to `Anthropic Claude Sonnet 5` on Amazon Bedrock to extract all inclusion and exclusion rules into an itemized JSON array.
	* **Model Selection Justification (Anthropic Claude Sonnet):** Protocol parsing is a high-stakes, one-time operation per study. Anthropic Claude Sonnet is mandated due to its superior clinical reasoning over 100-page context windows and strict JSON Schema compliance. Using Sonnet prevents misinterpretations or dropped criteria that would compromise all downstream patient evaluations, delivering maximum extraction accuracy for pennies per trial setup.
* **[Approved] [REQ-F-06]:** Extracted protocol rules must be stored in an Amazon DynamoDB table (`study-protocols`), indexed by `StudyID`. See §5.1 Protocol Schema.
	* **Architectural Justification (DynamoDB vs. Bedrock Knowledge Base):** Protocol rules are intentionally stored as structured records in DynamoDB rather than indexed into an Amazon Bedrock Knowledge Base (vector database). This guarantees deterministic, 100% evaluation coverage of all criteria per patient run (preventing vector similarity search from accidentally missing edge-case rules), eliminates redundant vector query costs during screening, and provides a static, schema-validated checklist directly to the Web UI for clinical review.

### 3.2 Patient Record Ingestion & Asynchronous OCR
* **[Approved] [REQ-F-07]:** The system shall support uploading multi-file patient medical records by the Clinical Investigator under a single `PatientID` directory into an encrypted S3 bucket (`patient-data-upload`).
* **[Approved] [REQ-F-08]:** Document uploads must initiate an AWS Step Functions state machine execution to manage long-running background tasks.
* **[Approved] [REQ-F-09]:** Step Functions shall route all patient PDF records through a Map state to execute Amazon Textract (`aws-sdk:textract:startDocumentAnalysis`) with `FeatureTypes=["TABLES"]` across files in parallel. Each parallel execution branch shall manage job completion using a native Step Functions Wait/Choice polling loop.
* **[Approved] [REQ-F-10]:** Textract output files must be written to an intermediate S3 bucket (`patient-extracted-data`) before triggering the next ingestion step.

### 3.3 Patient Document Consolidation for Full-Document Deterministic Reasoning
* **[Approved] [REQ-F-11]:** *(Revised 2026-08-27 — supersedes vector-embedding ingestion, per direct human instruction to replace RAG with full-document deterministic reasoning)* The system shall consolidate every per-file Textract output belonging to a patient's record set into a single ordered, page-annotated full-text document — concatenating all uploaded files in a stable order, with explicit source-file and page-boundary markers — and write it to an intermediate S3 bucket (`patient-consolidated-text`) prior to the reasoning step. No chunking, embedding generation, or vector indexing is performed.
* **[Deprecated] [REQ-F-12]:** ~~Embeddings and chunk metadata must be stored in `Amazon S3 Vectors` via `Amazon Bedrock Knowledge Bases` to eliminate fixed 24/7 idle database instance costs.~~ Superseded 2026-08-27 by revised `[REQ-F-11]`: no embeddings are generated under full-document deterministic reasoning. Deprecated per direct human instruction.
* **[Approved] [REQ-F-13]:** *(Revised 2026-08-27)* Every page within the consolidated full-text document must retain inline metadata identifying `patient_id`, `source_filename`, and `page_number`, so the reasoning agent's citations remain traceable to an exact source document and page (`[REQ-NF-04]`).

**Architectural Justification (Full-Document Deterministic Reasoning vs. RAG):** RAG depends on vector similarity search, which is probabilistic: it can fail to retrieve a chunk that is semantically dissimilar to the query embedding but still clinically decisive (a negation, a rare condition mentioned once, a lab value buried in a table row), silently dropping evidence rather than fabricating it. Because `[REQ-NF-04]` (no hallucination) and `[REQ-NF-05]` (no missing patient information) require complete, auditable coverage of every uploaded page, the platform passes the full consolidated patient text directly to the reasoning model instead of retrieving fragments. Anthropic Claude Sonnet 5 has a verified 1,000,000-token context window (128K max output) on Amazon Bedrock — confirmed via `docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5.html`, 2026-08-27 — comfortably exceeding the ~130,000-token full-text size of an average 200-page patient record set, so no truncation is required at baseline scale. This mirrors the deterministic-coverage rationale already applied to protocol rule storage over a vector Knowledge Base in `[REQ-F-06]`.

### 3.4 AI Reasoning & Eligibility Verdict Generation
* **[Approved] [REQ-F-14]:** The system shall run a single `Amazon Bedrock Agent` powered by `Anthropic Claude Sonnet 5` to perform eligibility analysis. Multi-agent setups are explicitly excluded to maintain simplicity and cost.
* **[Approved] [REQ-F-15]:** *(Revised 2026-08-27)* A dedicated Lambda task (`patient-screening-handler`) must pull active inclusion and exclusion rules from DynamoDB and read the complete consolidated patient full-text document from S3 (via the `[REQ-F-11]` Claim-Check pointer), then invoke the Bedrock Agent with the entire document as direct reasoning input. No similarity-based retrieval (RAG) is performed; every active rule must be evaluated against the entirety of the patient's uploaded record set on every screening run (`[REQ-NF-05]`).
* **[Approved] [REQ-F-16]:** The agent must evaluate rules across all uploaded patient documents simultaneously, using foundation model JSON Schema enforcement to output a structured verdict containing rule statuses (`MET`, `NOT_MET`, `UNCERTAIN`), direct quotes, source filenames, and page citations.
* **[Approved] [REQ-F-17]:** The generated verdict must be saved to the `patient-verdicts` Amazon DynamoDB table. See §5.1 Verdict Schema.
* **[Draft] [REQ-F-21]:** *(Added 2026-08-28, per human request following review of the AWS blog "AI Agents for Clinical Trial Screening")* `patient-screening-handler` must set `overall_recommendation` to `MANUAL_REVIEW_REQUIRED` whenever any entry in `criteria_evaluations` has `status = UNCERTAIN` or a `confidence_score` below a defined threshold **[OPEN PARAMETER — threshold value needs human input; no default assumed]**. Verdicts meeting this condition must be surfaced ahead of high-confidence `ELIGIBLE`/`INELIGIBLE` verdicts in the Clinical Investigator's review queue (`[REQ-F-18]`). This formalizes the `confidence_score`/`UNCERTAIN` fields already present in the `patient-verdicts` schema (§5.2) into an explicit escalation rule — today those fields are captured but nothing in `Architecture.md` specifies how they affect verdict routing or review priority.

### 3.5 Clinical Review Interface (Human-in-the-Loop)
* **[Approved] [REQ-F-18]:** The Web UI must present a unified Clinical Investigator dashboard with a side-by-side view: an interactive PDF viewer on the left, and an AI criteria checklist on the right.
* **[Approved] [REQ-F-19]:** Selecting any criterion or quote in the checklist must jump directly to the cited page and highlight text in the correct patient PDF.
* **[Approved] [REQ-F-20]:** The Clinical Investigator must be provided with interactive controls (`Approve`, `Reject`, `Manual Override`, `Notes`) to log the binding determination.

---

## 4. Non-Functional Requirements

### 4.1 Security & HIPAA Compliance
* **[Approved] [REQ-SEC-01]:** All data at rest across Amazon S3, DynamoDB, and Vector Storage must be encrypted using AWS KMS Customer-Managed Keys (CMK) with automated key rotation.
* **[Approved] [REQ-SEC-02]:** All network communications in transit must enforce TLS 1.3 encryption.
* **[Approved] [REQ-SEC-03]:** The AWS cloud infrastructure must operate under a signed AWS Business Associate Addendum (BAA).
* **[Approved] [REQ-SEC-04]:** Authentication and access control must be managed via Amazon Cognito User Pools with Role-Based Access Control (RBAC) and Multi-Factor Authentication (MFA) enforced.
* **[Approved] [REQ-SEC-05]:** AWS CloudTrail and Amazon CloudWatch must record all document access requests, API calls, and administrative actions for regulatory compliance and immutable audit logging.
* **[Draft] [REQ-SEC-06]:** Audit Trails stored in DynamoDB table.
* **[Draft] [REQ-SEC-07]:** *(Added 2026-08-28, per human request following review of the AWS blog "AI Agents for Clinical Trial Screening")* All Amazon Bedrock model invocations performing protocol rule extraction (`[REQ-F-05]`) or patient eligibility reasoning (`[REQ-F-14]`) must be filtered through an Amazon Bedrock Guardrail enforcing: PHI/PII content filtering, denied-topic boundaries restricting the model to eligibility-screening content, and a contextual grounding check that flags any generated claim not anchored in the supplied protocol/patient text. This provides a platform-level technical control complementing `[REQ-NF-04]`'s no-hallucination mandate, which today is enforced only via prompt-level JSON Schema instructions telling the model to emit `UNCERTAIN` rather than a verified, independent guardrail check.
* **[Draft] [REQ-SEC-08]:** *(Added 2026-08-28, per human request)* AWS CloudTrail must record data events for the `study-protocols` and `patient-verdicts` DynamoDB tables (`GetItem`, `Query`, `PutItem`, `UpdateItem`), in addition to the S3 object-level and KMS key-operation data events already required by `[REQ-SEC-05]`. `patient-verdicts` items contain direct PHI quotes (`evidence_quote`, §5.2) and `study-protocols` holds trial protocol IP; today's `[REQ-SEC-05]` audit scope (S3 `GetObject`/`PutObject`, KMS `Decrypt`) does not capture item-level reads/writes against these tables, leaving PHI-bearing record access outside the audit trail.

### 4.2 Performance & Reliability
* **[Approved] [REQ-NF-01]:** Background ingestion and screening of multi-file patient records (up to 150 MB total) must complete within `10 minutes`.
* **[Deprecated] [REQ-NF-02]:** ~~RAG vector search retrieval times must remain under 2 seconds per query.~~ Superseded 2026-08-27: no RAG queries occur under full-document deterministic reasoning (`[REQ-F-11], [REQ-F-15]`). Deprecated per direct human instruction. Full-document inference latency is covered by the existing end-to-end `[REQ-NF-01]` 10-minute SLA; the increased per-screening input size (~130,000 tokens vs. the prior ~24,000-token retrieved-chunk average) has not yet been latency-tested against that SLA — flagged as an open risk in `Architecture.md`.
* **[Approved] [REQ-NF-03]:** Step Functions must handle task retries, exponential backoffs, and dead-letter queues (DLQs) via Amazon SQS for failed Textract or Bedrock jobs.

### 4.3 Operational & Regional Deployment
* **[Approved] [REQ-OPS-01]:** All infrastructure, data storage, and vector indexing must be deployed and execute strictly within AWS Region **`us-west-2` (Oregon)**. Amazon Bedrock foundation model inference must originate from `us-west-2`. It must execute In-Region where the mandated model supports it, and where a mandated model has no `us-west-2` In-Region support, invocation via an AWS **Geographic (US) cross-Region inference profile** (`us.`-prefixed) is permitted. **Global** cross-Region inference is not permitted.

### 4.4 Observability
* **[Draft] [REQ-OBS-01]:** *(Expanded 2026-08-28 from placeholder text "Telemetry," per human request following review of the AWS blog "AI Agents for Clinical Trial Screening")* The system must continuously monitor eligibility-verdict quality against ground truth. A scheduled job shall compare each screening verdict's `overall_recommendation` against the corresponding `reviewer_signoff.status` once a Clinical Investigator has recorded a determination, computing a rolling agreement-rate metric published to Amazon CloudWatch. A CloudWatch Alarm must trigger when the rolling agreement rate drops below a defined threshold **[OPEN PARAMETER — threshold value needs human input; no default assumed]**, surfacing model-quality drift before it accumulates silently across screening runs.

### 4.5 Accuracy, Grounding & Data Completeness
* **[Approved] [REQ-NF-04]:** *(Approved 2026-08-27 — human resolved the RAG-vs-completeness conflict flagged against this Draft by selecting full-document deterministic reasoning, see `[REQ-F-11], [REQ-F-15]`)* The system must not hallucinate. Every criterion status (`MET`, `NOT_MET`, `UNCERTAIN`) and every clinical claim in a verdict must be grounded in a direct, verifiable quote traceable to a specific source document and page (`[REQ-F-16]`). If the foundation model cannot locate supporting text for a criterion, the status must be set to `UNCERTAIN` — the model must never infer, extrapolate, or fabricate a `MET`/`NOT_MET` status without a cited source quote.
* **[Approved] [REQ-NF-05]:** *(Approved 2026-08-27, same resolution as above)* No information within a patient's uploaded record set may be omitted from evaluation. The system must guarantee 100% page/document coverage per patient across all uploaded files for every screening run — every uploaded page must be read and considered against every active criterion, not merely the fragments a similarity search happens to return.

---

## 5. Database Schema Specifications

### 5.1 Protocol Schema (`study-protocols` DynamoDB Table)
```json
{
  "StudyID": "STRING (Partition Key)",
  "StudyName": "STRING",
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
```json
{
  "PatientID": "STRING (Partition Key)",
  "StudyID": "STRING (Sort Key)",
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
