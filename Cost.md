## 1. Executive Summary & Budget Target Compliance (`[REQ-COST-01]`)

The AI-Assisted Clinical Trial Screening Platform is designed with a fully serverless, pay-per-use architecture that eliminates fixed 24/7 idle compute and database cluster fees. 

* **Governing Requirement:** `[REQ-COST-01]` (Total AWS infrastructure spend under $5,000.00 / year / ~$416.66 / month).
* **Projected Baseline Operational Spend:** **$105.00 – $240.00 / month** (**$1,260.00 – $2,880.00 / year**).
* **Budget Headroom:** 42% – 75% under the annual budget ceiling, providing substantial buffer for volume scaling.

---

## 2. Baseline Sizing & Operational Scale Assumptions

All quantitative pricing calculations derive strictly from the operational metrics defined in `Requirements.md`:

| Sizing Parameter | Value | Reference / Notes |
| :--- | :--- | :--- |
| **Monthly Patient Screenings** | 100 patients / month | `Requirements.md` Section 2 |
| **Patient Record Volume** | 2 PDFs / patient (avg 100 pages / file) | 200 pages / patient = **20,000 pages / month** |
| **Average Patient File Size** | 25 MB – 50 MB / PDF | 50 MB – 100 MB total per patient record |
| **Monthly Ingestion Data Volume** | 100 patients × 75 MB avg = **7.5 GB / month** | S3 Standard Ingestion |
| **Study Protocol Onboarding** | 2 protocols / month (avg 100 pages each) | **200 pages / month** OCR volume |
| **RAG Queries per Screening** | 12 criteria / study × 5 retrieved chunks | 60 retrieval queries per patient screening |
| **Clinical Review API Access** | ~100 reviews / month × 50 requests/review | ~5,000 API calls & Presigned S3 fetches |

---

## 3. Detailed Component Cost Derivations (`[REQ-COST-01]`)

### 3.1 Document Processing — Amazon Textract (`[REQ-F-02, REQ-F-07]`)
* **Monthly Workload:** 20,000 patient record pages + 200 protocol pages = 20,200 pages / month.
* **Pricing Rates:** Raw text detection ($0.0015/page) and Table/Form analysis ($0.015/page) for structured sections; blended effective rate is ~$0.0035 / page.
* **Monthly Calculation:** 20,200 pages × $0.0035/page = **$70.70 / month**.
* **Estimated Monthly Range:** **$60.00 – $110.00 / month**.

---

### 3.2 AI, Embeddings & Generative Reasoning — Amazon Bedrock (`[REQ-F-03, REQ-F-09, REQ-F-12, REQ-F-14]`)

#### A. Embedding Generation (`[REQ-F-09]` — Amazon Titan Text Embeddings v2):
* **Monthly Tokens:** 20,200 pages × ~450 words = ~9,090,000 words = ~12,120,000 input tokens.
* **Pricing Rate:** $0.00002 per 1,000 tokens.
* **Monthly Calculation:** (12,120,000 / 1,000) × $0.00002 = **$0.24 / month**.

#### B. Protocol Rule Extraction (`[REQ-F-03]` — Amazon Nova Pro / Claude 3.5 Sonnet):
* **Monthly Workload:** 2 protocols × 100 pages = 130,000 input tokens; ~12,000 output tokens.
* **Estimated Monthly Subtotal:** **$0.15 – $0.60 / month**.

#### C. Patient Screening Reasoning Agent (`[REQ-F-12, REQ-F-14]`):
* **Input Context Tokens:** 100 patients × 12 criteria × 5 chunks = 2,400,000 input context tokens / month.
* **Output Verdict Tokens:** 100 patients × 1,500 output tokens = 150,000 output tokens / month.
* **Token Cost:** Input (2.4M × $0.0008–$0.003/1k = $1.92–$7.20) + Output (150k × $0.0032–$0.015/1k = $0.48–$2.25).
* **Agent Orchestration & RAG Tool Calls:** ~$15.00 – $40.00 / month.
* **Estimated Monthly Subtotal:** **$18.00 – $50.00 / month**.

---

### 3.3 Storage & Vector Management (`[REQ-F-01, REQ-F-05, REQ-F-08, REQ-F-10]`)
* **Amazon S3 Storage:** 10 GB new data / month (~120 GB in Year 1) × $0.023/GB = ~$2.76 / month.
* **S3 API Requests:** PUT, GET, and LIST operations = ~$0.50 / month.
* **Serverless Vector Storage:** Vector index storage and read/write units in Bedrock Knowledge Bases = ~$10.00 – $35.00 / month.
* **Estimated Monthly Subtotal:** **$13.00 – $38.00 / month**.

---

### 3.4 Database & Cache — Amazon DynamoDB (`[REQ-F-04, REQ-F-15]`)
* **Billing Mode:** On-Demand Capacity Mode.
* **Write Requests:** ~1,500 writes / month (protocols and verdicts) = negligible (< $0.05).
* **Read Requests:** ~10,000 reads / month = negligible (< $0.05).
* **Storage:** Under 1 GB total structured protocol and verdict data = negligible (< $0.25).
* **Estimated Monthly Subtotal:** **$1.00 – $5.00 / month**.

---

### 3.5 Orchestration & API — Step Functions, Lambda & API Gateway (`[REQ-F-02, REQ-F-06, REQ-SEC-02]`)
* **AWS Step Functions:** 100 patient screening executions + 2 protocol onboarding executions (~1,530 state transitions / month) = $0.04 / month.
* **AWS Lambda:** ~3,000 invocations / month across OCR handlers, parsing, and review APIs = $0.20 / month.
* **Amazon API Gateway:** ~5,000 REST API requests / month = $0.02 / month.
* **Estimated Monthly Subtotal:** **$2.00 – $7.00 / month**.

---

### 3.6 Security, Governance & Auditing (`[REQ-SEC-01, REQ-SEC-04, REQ-SEC-05]`)
* **AWS KMS:** 1 Customer-Managed Key ($1.00) + ~60,000 cryptographic operations ($0.18) = $1.18 / month.
* **Amazon Cognito:** User directory with under 50 clinical staff MAUs (Free Tier covers up to 50,000 MAUs) = $0.00 / month.
* **AWS CloudTrail & CloudWatch Logs:** S3 Data Events + 10 GB/month log ingestion + 7-year log retention = $10.00 – $30.00 / month.
* **Estimated Monthly Subtotal:** **$11.00 – $30.00 / month**.

---

## 4. Total Cost Summary Table

| Category | Primary AWS Services | Baseline Low (USD/mo) | Baseline High (USD/mo) | Governing REQ IDs |
| :--- | :--- | :--- | :--- | :--- |
| **Document Processing** | Amazon Textract (Async OCR) | $60.00 | $110.00 | `[REQ-F-02, REQ-F-07]` |
| **AI & Vector Reasoning** | Amazon Bedrock (Agent, KB, Titan, Nova/Claude) | $18.00 | $50.00 | `[REQ-F-03, REQ-F-09, REQ-F-12, REQ-F-14]` |
| **Storage & Vectors** | Amazon S3 + Bedrock Vector Store | $13.00 | $38.00 | `[REQ-F-01, REQ-F-05, REQ-F-08, REQ-F-10]` |
| **Database Tier** | Amazon DynamoDB (On-Demand) | $1.00 | $5.00 | `[REQ-F-04, REQ-F-15]` |
| **Orchestration & API** | Step Functions + Lambda + API Gateway | $2.00 | $7.00 | `[REQ-F-06, REQ-SEC-02]` |
| **Security & Auditing** | AWS KMS + CloudWatch + CloudTrail + Cognito | $11.00 | $30.00 | `[REQ-SEC-01, REQ-SEC-04, REQ-SEC-05]` |
| **Total Monthly Spend** | | **$105.00** | **$240.00** | `[REQ-COST-01]` |
| **Total Annual Spend** | | **$1,260.00** | **$2,880.00** | **Budget Ceiling: $5,000.00** |

---

## Archive

### Initial Sizing Estimate (Superseded)
* **Monthly Range:** ~$160.00 – $345.00 / month
* **Annual Range:** ~$1,920.00 – $4,140.00 / year
* **Superseded Date:** 2026-08-25
* **Reason:** Replaced with exact formulaic unit-cost derivations aligned to `[REQ-COST-01]`.
