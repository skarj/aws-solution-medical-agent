# Cost.md

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
* **Workload:** 20,000 patient record pages + 200 protocol pages = 20,200 pages / month.
* **Textract Async Processing Rates (US East):**
  * Raw Text Detection (`DetectDocumentText`): $0.0015 / page.
  * Document Analysis with Tables (`StartDocumentAnalysis`): $0.015 / page (for tabular lab results / structured sections, ~40% of pages).
  * Blended Effective Rate: ~$0.0035 / page.
* **Calculation:**
  $$\text{Textract Cost} = 20,200\text{ pages} \times \$0.0035 = \$70.70\text{ / month}$$
* **Monthly Range:** **$60.00 – $110.00**

---

### 3.2 AI, Embeddings & Generative Reasoning — Amazon Bedrock (`[REQ-F-03, REQ-F-09, REQ-F-12, REQ-F-14]`)

#### A. Embedding Generation (`[REQ-F-09]` — Amazon Titan Text Embeddings v2):
* Text volume: 20,200 pages × ~450 words = ~9,090,000 words = ~12,120,000 input tokens.
* Titan Embeddings rate: $0.00002 per 1,000 tokens.
* Calculation:
  $$\text{Embeddings Cost} = \frac{12,120,000}{1,000} \times \$0.00002 = \$0.24\text{ / month}$$

#### B. Protocol Rule Extraction (`[REQ-F-03]` — Amazon Nova Pro / Claude 3.5 Sonnet):
* 2 protocols × 100 pages = 130,000 input tokens; ~12,000 output tokens.
* Cost per run: ~$0.15 – $0.60 / month.

#### C. Patient Screening Reasoning Agent (`[REQ-F-12, REQ-F-14]`):
* 100 patients × 12 criteria × 5 chunks (2,000 context tokens/criterion) = 2,400,000 input context tokens / month.
* Output structured verdicts: 100 patients × 1,500 output tokens = 150,000 output tokens / month.
* Pricing (Blended Amazon Nova Pro / Anthropic Claude 3.5 Sonnet):
  * Input tokens: 2.4M tokens × $0.0008 – $0.003/1k tokens = $1.92 – $7.20 / month.
  * Output tokens: 150k tokens × $0.0032 – $0.015/1k tokens = $0.48 – $2.25 / month.
  * Agent RAG orchestration queries & multi-step tool calls: ~$15.00 – $40.00 / month.
* **Monthly AI Subtotal:** **$18.00 – $50.00**

---

### 3.3 Storage & Vector Management (`[REQ-F-01, REQ-F-05, REQ-F-08, REQ-F-10]`)
* **Amazon S3 Storage:**
  * Raw PDF uploads + extracted text: ~10 GB / month cumulative (~120 GB in Year 1).
  * S3 Standard: 120 GB × $0.023/GB = $2.76 / month.
  * S3 API Requests (PUT, GET, LIST): ~$0.50 / month.
* **Serverless Vector Storage (Bedrock Knowledge Bases / S3 Vectors):**
  * Vector index storage & vector read/write units: ~$10.00 – $35.00 / month.
* **Monthly Storage Subtotal:** **$13.00 – $38.00**

---

### 3.4 Database & Cache — Amazon DynamoDB (`[REQ-F-04, REQ-F-15]`)
* **Billing Mode:** On-Demand Capacity.
* **Write Requests:** ~500 protocol writes + ~1,000 verdict writes/month = 1,500 WCU = negligible (< $0.05).
* **Read Requests:** ~10,000 RCU/month = negligible (< $0.05).
* **Storage:** Under 1 GB total structured protocol and verdict data = negligible (< $0.25).
* **Monthly DynamoDB Subtotal:** **$1.00 – $5.00**

---

### 3.5 Orchestration & API — Step Functions, Lambda & API Gateway (`[REQ-F-06, REQ-SEC-02]`)
* **AWS Step Functions:** 100 workflow executions × 15 state transitions = 1,500 transitions/month = $0.04 / month.
* **AWS Lambda:** ~3,000 invocations/month across protocol parsing, OCR handlers, and API review endpoints = $0.20 / month.
* **Amazon API Gateway (REST API):** ~5,000 requests/month = $0.02 / month.
* **Monthly Orchestration Subtotal:** **$2.00 – $7.00**

---

### 3.6 Security, Governance & Auditing (`[REQ-SEC-01, REQ-SEC-04, REQ-SEC-05]`)
* **AWS KMS:** 1 Customer-Managed Key ($1.00) + ~60,000 cryptographic operations ($0.18) = $1.18 / month.
* **Amazon Cognito:** User Pool with under 50 clinical staff MAUs (Free Tier covers up to 50,000 MAUs) = $0.00 / month.
* **AWS CloudTrail & CloudWatch:** S3 Data Events + CloudWatch Log Ingestion (10 GB/month) + 7-year log retention archive = $10.00 – $30.00 / month.
* **Monthly Security Subtotal:** **$11.00 – $30.00**

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

## ## Archive

### Initial Sizing Estimate (Superseded)
* **Monthly Range:** ~$160.00 – $345.00 / month
* **Annual Range:** ~$1,920.00 – $4,140.00 / year
* **Superseded Date:** 2026-08-25
* **Reason:** Replaced with exact formulaic unit-cost derivations aligned to `[REQ-COST-01]`.
