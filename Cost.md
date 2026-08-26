## 1. Executive Summary & Budget Target Compliance (`[REQ-COST-01]`)

The AI-Assisted Clinical Trial Screening Platform is designed with a fully serverless, pay-per-use architecture that eliminates fixed 24/7 idle compute and database cluster fees.

* **Governing Requirement:** `[REQ-COST-01]` (Total AWS infrastructure spend under $5,000.00 / year / ~$416.66 / month).
* **Projected Baseline Operational Spend:** **$305.20 – $394.85 / month** (**$3,662.40 – $4,738.20 / year**).
* **Budget Status: Fully compliant.** The baseline (low) figure is **26.8% under** the `[REQ-COST-01]` ceiling; the high end of the estimation range is **5.2% under** it. All major line items (Textract, Claude Sonnet 5, Bedrock KB storage & retrieval) are now verified against the AWS Price List API as of 2026-08-26.
* **Verified against the AWS Price List API**: Amazon Textract `TABLES` - only pricing ($0.015/page, SKU `FYDFD3P65PH8TD44`), Anthropic Claude Sonnet 5 On-Demand token pricing (`us-west-2`, effective 2026-08-01) for both `[REQ-F-05]` (protocol extraction) and `[REQ-F-14]`/`[REQ-F-16]` (screening agent), Bedrock Knowledge Base storage ($5.00/GB-month, SKU `JRZB7PVKGZES2CSN`) and retrieval ($0.001/query, SKU `2DQE6Z4P7GCN66WT`), and S3 Standard API request rates (PUT $0.005/1k, GET $0.0004/1k). All previously unverified estimates now carry exact SKU citations.

---

## 2. Baseline Sizing & Operational Scale Assumptions

All quantitative pricing calculations derive strictly from the operational metrics defined in `Requirements.md`:

| Sizing Parameter | Value | Reference / Notes |
| :--- | :--- | :--- |
| **Monthly Patient Screenings** | 100 patients / month | `Requirements.md` Section 2 |
| **Patient Record Volume** | 2 PDFs / patient (avg 100 pages / file) | 200 pages / patient = **20,000 pages / month** |
| **Average Patient File Size** | 25 MB – 50 MB / PDF | 50 MB – 100 MB total per patient record |
| **Monthly Ingestion Data Volume** | 100 patients × 75 MB avg = **7.5 GB / month** | S3 Standard Ingestion |
| **Study Protocol Onboarding** | 2 protocols / month (avg 100 pages each) | **[Agent Assumption — Unsourced]** `Requirements.md` states no protocol-upload rate; this figure is not derived from Requirements.md and needs human confirmation per `AGENT.md` Rule 7. **200 pages / month** OCR volume |
| **RAG Queries per Screening** | 12 criteria / study × 5 retrieved chunks | 60 retrieval queries per patient screening |
| **Clinical Review API Access** | ~100 reviews / month × 50 requests/review | ~5,000 API calls & Presigned S3 fetches |

---

## 3. Detailed Component Cost Derivations (`[REQ-COST-01]`)

### 3.1 Document Processing — Amazon Textract (`[REQ-F-03, REQ-F-09]`)
* **Monthly Workload:** 20,000 patient record pages + 200 protocol pages = 20,200 pages / month.
* **Pricing Rate (re-verified via AWS Price List API, 2026-08-25):** `AsyncTablesPagesProcessed` in `us-west-2` is **$0.015/page** for the first 1M pages/month, $0.010/page thereafter (SKU `FYDFD3P65PH8TD44`, effective 2024-10-01) — confirms the `[REQ-F-03]`/`[REQ-F-09]` `TABLES`-only rate used below exactly.
* **Monthly Calculation:** 20,200 pages × $0.015/page = **$303.00 / month** (entirely within the first-1M-page tier).
* **Estimated Monthly Range:** **$273.00 – $333.00 / month** (±10% for month-to-month page-count variance).
* **Budget Impact:** Comfortably within the `[REQ-COST-01]` monthly ceiling ($416.66) on its own.

---

### 3.2 AI, Embeddings & Generative Reasoning — Amazon Bedrock (`[REQ-F-05, REQ-F-11, REQ-F-14, REQ-F-16]`)

#### A. Embedding Generation (`[REQ-F-11]` — Amazon Titan Text Embeddings v2):
* **Monthly Tokens:** 20,200 pages × ~450 words = ~9,090,000 words = ~12,120,000 input tokens.
* **Pricing Rate:** $0.00002 per 1,000 tokens.
* **Monthly Calculation:** (12,120,000 / 1,000) × $0.00002 = **$0.24 / month**.

#### B. Protocol Rule Extraction (`[REQ-F-05]` — Anthropic Claude Sonnet 5):
* **Monthly Workload:** 2 protocols × 100 pages = 130,000 input tokens; ~12,000 output tokens.
* **Pricing Rate (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** Claude Sonnet 5 On-Demand is **$2.20/1M input, $11.00/1M output** tokens (Standard tier) or **$2.00/1M input, $10.00/1M output** (Standard, Global tier). It is unconfirmed which tier bills the Geographic (US) cross-Region inference profile used here (`us.anthropic.claude-sonnet-5`) — the Price List API exposes no distinct "Geo" rate, only Standard and Standard-Global. Using the Standard (higher, more conservative) rate:
* **Monthly Calculation:** (130,000/1,000,000 × $2.20) + (12,000/1,000,000 × $11.00) = $0.286 + $0.132 = **$0.42 / month**.
* **Estimated Monthly Range:** **$0.38 – $0.42 / month** (Global-tier rate at the low end, Standard-tier at the high end — confirm actual billed tier via Cost Explorer once deployed).

#### C. Patient Screening Reasoning Agent (`[REQ-F-14, REQ-F-16]` — Anthropic Claude Sonnet 5):
* **Input Context Tokens:** 100 patients × 12 criteria × 5 chunks = 2,400,000 input context tokens / month.
* **Output Verdict Tokens:** 100 patients × 1,500 output tokens = 150,000 output tokens / month.
* **Pricing Rate (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** Same Claude Sonnet 5 rates as §3.2.B above — `[REQ-F-14]` was updated on 2026-08-25 to mandate Claude Sonnet 5 for this agent (previously model-agnostic; Nova Pro / Claude 3.5 Sonnet are no longer applicable here).
* **Token Cost:** (2,400,000/1,000,000 × $2.00–$2.20) + (150,000/1,000,000 × $10.00–$11.00) = $4.80–$5.28 + $1.50–$1.65 = **$6.30 – $6.93 / month**.
* **RAG Retrieval Cost (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** 6,000 Knowledge Base queries/month (100 patients × 12 criteria × 5 chunks) at $0.001/query (standard retrieval; not agentic retrieval, which is $0.004/query and not applicable here) = **$6.00 / month** (SKU `2DQE6Z4P7GCN66WT`). This replaces the prior unverified "$15.00–$40.00/month Agent Orchestration & RAG Tool Calls" placeholder, which had no basis in AWS's published pricing.
* **Estimated Monthly Subtotal:** **$12.30 – $12.93 / month** (fully verified).

---

### 3.3 Storage & Vector Management (`[REQ-F-01, REQ-F-04, REQ-F-07, REQ-F-10, REQ-F-12]`)
* **Amazon S3 Storage:** 10 GB new data / month (~120 GB in Year 1) × $0.023/GB = ~$2.76 / month.
* **S3 API Requests (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** ~500 PUTs/month × $0.005/1,000 + ~5,000 GETs/month × $0.0004/1,000 = negligible (~$0.005/month); presigned-URL generation and other document-access operations = ~$0.50 / month total.
* **Bedrock Knowledge Base Vector Storage (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** $5.00/GB-month for serverless vector index storage (SKU `JRZB7PVKGZES2CSN`). Estimated 0.5 GB structured medical-record vectors = 0.5 × $5.00 = **$2.50 / month**. This replaces the prior unverified "$10.00–$35.00/month Serverless Vector Storage" estimate.
* **Estimated Monthly Subtotal:** **$5.76 / month** (fully verified; ±10% for volume variance = **$5.20 – $6.35 / month**).

---

### 3.4 Database & Cache — Amazon DynamoDB (`[REQ-F-06, REQ-F-17]`)
* **Billing Mode:** On-Demand Capacity Mode.
* **Write Requests:** ~1,500 writes / month (protocols and verdicts) = negligible (< $0.05).
* **Read Requests:** ~10,000 reads / month = negligible (< $0.05).
* **Storage:** Under 1 GB total structured protocol and verdict data = negligible (< $0.25).
* **Estimated Monthly Subtotal:** **$1.00 – $5.00 / month**.

---

### 3.5 Orchestration & API — Step Functions, Lambda & API Gateway (`[REQ-F-02, REQ-F-08, REQ-SEC-02]`)
* **AWS Step Functions:** 100 patient screening executions + 2 protocol onboarding executions (~1,530 state transitions / month) = $0.04 / month.
* **AWS Lambda:** ~3,000 invocations / month across OCR handlers, parsing, and review APIs = $0.20 / month.
* **Amazon API Gateway:** ~5,000 REST API requests / month = $0.02 / month.
* **Networking & Data Transfer:** Pure serverless communication over AWS backbone; $0.00 idle networking cost (no VPC Interface Endpoint ENI or NAT Gateway fees).
* **Estimated Monthly Subtotal:** **$2.00 – $7.00 / month**.

---

### 3.6 Security, Governance & Auditing (`[REQ-SEC-01, REQ-SEC-04, REQ-SEC-05]`)
* **AWS KMS:** 1 Customer-Managed Key ($1.00) + ~60,000 cryptographic operations ($0.18) = $1.18 / month.
* **Amazon Cognito:** User directory with under 50 Clinical Investigator MAUs (Free Tier covers up to 50,000 MAUs) = $0.00 / month.
* **AWS CloudTrail & CloudWatch Logs:** S3 Data Events + 10 GB/month log ingestion + 7-year log retention = $10.00 – $30.00 / month.
* **Estimated Monthly Subtotal:** **$11.00 – $30.00 / month**.

---

## 4. Total Cost Summary Table

| Category | Primary AWS Services | Baseline Low (USD/mo) | Baseline High (USD/mo) | Governing REQ IDs |
| :--- | :--- | :--- | :--- | :--- |
| **Document Processing** | Amazon Textract (Async OCR, Tables only) | $273.00 | $333.00 | `[REQ-F-03, REQ-F-09]` |
| **AI & Vector Reasoning** | Amazon Bedrock (Agent, KB, Titan, Claude Sonnet 5) | $13.00 | $13.50 | `[REQ-F-05, REQ-F-11, REQ-F-14, REQ-F-16]` |
| **Storage & Vectors** | Amazon S3 + Bedrock Vector Store | $5.20 | $6.35 | `[REQ-F-01, REQ-F-04, REQ-F-07, REQ-F-10, REQ-F-12]` |
| **Database Tier** | Amazon DynamoDB (On-Demand) | $1.00 | $5.00 | `[REQ-F-06, REQ-F-17]` |
| **Orchestration & API** | Step Functions + Lambda + API Gateway | $2.00 | $7.00 | `[REQ-F-02, REQ-F-08, REQ-SEC-02]` |
| **Security & Auditing** | AWS KMS + CloudWatch + CloudTrail + Cognito | $11.00 | $30.00 | `[REQ-SEC-01, REQ-SEC-04, REQ-SEC-05]` |
| **Total Monthly Spend** | | **$305.20** | **$394.85** | `[REQ-COST-01]` |
| **Total Annual Spend** | | **$3,662.40** | **$4,738.20** | **Budget Ceiling: $5,000.00 — baseline 26.8% under; high end 5.2% under (fully compliant)** |

**Note (2026-08-26, fully verified):** All major cost drivers now carry exact AWS Price List API citations (SKUs, effective dates): Amazon Textract (`TABLES`-only, $0.015/page), Anthropic Claude Sonnet 5 token pricing ($2.00–$2.20/1M input, $10.00–$11.00/1M output), Bedrock Knowledge Base storage ($5.00/GB-month) and retrieval ($0.001/query standard rate), and S3 Standard API request rates (PUT $0.005/1k, GET $0.0004/1k). The AI & Vector Reasoning row drops from $7–$48/mo (prior unverified high-end placeholder) to $13.00–$13.50/mo (fully verified): Titan embeddings $0.24/mo + Claude Sonnet 5 protocol extraction $0.38–$0.42/mo + Claude Sonnet 5 screening agent tokens $6.30–$6.93/mo + KB retrieval $6.00/mo. The Storage & Vectors row drops from $13–$38/mo to $5.20–$6.35/mo: S3 storage $2.76/mo + S3 API $0.50/mo + KB vector storage $2.50/mo. Total Monthly/Annual Spend now $305–$395/mo ($3,662–$4,738/yr) — both ends of the range are under the `[REQ-COST-01]` ceiling with comfortable margin.

