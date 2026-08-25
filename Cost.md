## 1. Executive Summary & Budget Target Compliance (`[REQ-COST-01]`)

The AI-Assisted Clinical Trial Screening Platform is designed with a fully serverless, pay-per-use architecture that eliminates fixed 24/7 idle compute and database cluster fees.

* **Governing Requirement:** `[REQ-COST-01]` (Total AWS infrastructure spend under $5,000.00 / year / ~$416.66 / month).
* **Projected Baseline Operational Spend:** **$318.00 – $463.00 / month** (**$3,816.00 – $5,556.00 / year**).
* **Budget Status: Compliant at baseline, tight at high end.** `Requirements.md` was updated on 2026-08-25 to drop the `FORMS` Textract feature from `[REQ-F-03]`/`[REQ-F-09]` (TABLES-only now), which resolves the prior 2.9x–3.8x budget overrun. The baseline (low) figure is **23.7% under** the `[REQ-COST-01]` ceiling; the high end of the estimation range is **~11% over** it — driven mostly by volume-variance assumptions in the Bedrock, Storage, and Security line items, not Textract. Monitor actual page/patient volume against the §2 sizing assumptions.
* This does not change the cost total materially (Bedrock's `[REQ-F-05]` line is a small share of spend). The Claude Sonnet 5 token pricing used in §3.2.B below is carried over from the prior Claude 3.5 Sonnet estimate and remains **`[Estimated - Unverified]`** — AWS's public Bedrock pricing page did not expose Sonnet 5's per-token rate in this pass; re-verify before committing to a production cost figure.

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

### 3.1 Document Processing — Amazon Textract (`[REQ-F-03, REQ-F-09]`)
* **Monthly Workload:** 20,000 patient record pages + 200 protocol pages = 20,200 pages / month.
* **Pricing Rates (verified via `aws.amazon.com/textract/pricing/`, US West/Oregon, 2026-08-25):** `[REQ-F-03]` and `[REQ-F-09]` were updated on 2026-08-25 to request the `TABLES` feature type only (`FORMS` dropped to control cost). `AnalyzeDocument`/`StartDocumentAnalysis` with `TABLES` is **$0.015/page**.
* **Monthly Calculation:** 20,200 pages × $0.015/page = **$303.00 / month**.
* **Estimated Monthly Range:** **$273.00 – $333.00 / month** (±10% for month-to-month page-count variance).
* **Budget Impact:** Comfortably within the `[REQ-COST-01]` monthly ceiling ($416.66) on its own. Prior versions of this section (see `Log.md`, 2026-08-25 15:05 UTC) used an unverified $0.0035/page blended rate ($70.70/mo), then a corrected but since-superseded $0.065/page Tables+Forms rate ($1,313.00/mo); both are obsolete now that `[REQ-F-03]`/`[REQ-F-09]` specify `TABLES` only.

---

### 3.2 AI, Embeddings & Generative Reasoning — Amazon Bedrock (`[REQ-F-05, REQ-F-11, REQ-F-14, REQ-F-16]`)

#### A. Embedding Generation (`[REQ-F-11]` — Amazon Titan Text Embeddings v2):
* **Monthly Tokens:** 20,200 pages × ~450 words = ~9,090,000 words = ~12,120,000 input tokens.
* **Pricing Rate:** $0.00002 per 1,000 tokens.
* **Monthly Calculation:** (12,120,000 / 1,000) × $0.00002 = **$0.24 / month**.

#### B. Protocol Rule Extraction (`[REQ-F-05]` — Anthropic Claude Sonnet 5): `[Estimated - Unverified]`
* **Monthly Workload:** 2 protocols × 100 pages = 130,000 input tokens; ~12,000 output tokens.
* **Estimated Monthly Subtotal:** **$0.15 – $0.60 / month** — carried over from the prior Claude 3.5 Sonnet estimate; `Requirements.md` now pins Claude Sonnet 5 (`[REQ-F-05]`, 2026-08-25) and this has not been re-priced against Sonnet 5's published per-token rate, which was not exposed in the pricing page pass used for this review. Re-verify once Sonnet 5 pricing is confirmed (the `[REQ-F-05]`/`[REQ-OPS-01]` regional conflict itself is resolved — see §1).

#### C. Patient Screening Reasoning Agent (`[REQ-F-14, REQ-F-16]`):
* **Input Context Tokens:** 100 patients × 12 criteria × 5 chunks = 2,400,000 input context tokens / month.
* **Output Verdict Tokens:** 100 patients × 1,500 output tokens = 150,000 output tokens / month.
* **Token Cost:** Input (2.4M × $0.0008–$0.003/1k = $1.92–$7.20) + Output (150k × $0.0032–$0.015/1k = $0.48–$2.25).
* **Agent Orchestration & RAG Tool Calls:** ~$15.00 – $40.00 / month.
* **Estimated Monthly Subtotal:** **$18.00 – $50.00 / month**.

---

### 3.3 Storage & Vector Management (`[REQ-F-01, REQ-F-04, REQ-F-07, REQ-F-10, REQ-F-12]`)
* **Amazon S3 Storage:** 10 GB new data / month (~120 GB in Year 1) × $0.023/GB = ~$2.76 / month.
* **S3 API Requests:** PUT, GET, and LIST operations = ~$0.50 / month.
* **Serverless Vector Storage:** Vector index storage and read/write units in Bedrock Knowledge Bases = ~$10.00 – $35.00 / month.
* **Estimated Monthly Subtotal:** **$13.00 – $38.00 / month**.

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
| **AI & Vector Reasoning** | Amazon Bedrock (Agent, KB, Titan, Claude Sonnet 5) | $18.00 | $50.00 | `[REQ-F-05, REQ-F-11, REQ-F-14, REQ-F-16]` |
| **Storage & Vectors** | Amazon S3 + Bedrock Vector Store | $13.00 | $38.00 | `[REQ-F-01, REQ-F-04, REQ-F-07, REQ-F-10, REQ-F-12]` |
| **Database Tier** | Amazon DynamoDB (On-Demand) | $1.00 | $5.00 | `[REQ-F-06, REQ-F-17]` |
| **Orchestration & API** | Step Functions + Lambda + API Gateway | $2.00 | $7.00 | `[REQ-F-02, REQ-F-08, REQ-SEC-02]` |
| **Security & Auditing** | AWS KMS + CloudWatch + CloudTrail + Cognito | $11.00 | $30.00 | `[REQ-SEC-01, REQ-SEC-04, REQ-SEC-05]` |
| **Total Monthly Spend** | | **$318.00** | **$463.00** | `[REQ-COST-01]` |
| **Total Annual Spend** | | **$3,816.00** | **$5,556.00** | **Budget Ceiling: $5,000.00 — baseline compliant (23.7% under); high end ~11% over** |

**Note (2026-08-25, updated):** `Requirements.md` was revised to drop the Textract `FORMS` feature (`[REQ-F-03]`/`[REQ-F-09]` now `TABLES`-only) and to pin `[REQ-F-05]` to Anthropic Claude Sonnet 5. The Document Processing row is recomputed at $0.015/page (TABLES only), bringing Total Monthly/Annual Spend back within range of the `[REQ-COST-01]` ceiling at baseline. The prior $1,180–$1,450/mo Tables+Forms figure and the $60–$110/mo unverified blended-rate figure before that are both superseded — see `Log.md` for the full correction history. The remaining ~11% high-end overage is a volume-variance margin, not a structural conflict, and should be revisited once real usage data is available. Sonnet 5's per-token pricing remains unverified and should be confirmed before this line item is treated as final.

