## 1. Executive Summary & Budget Target Compliance (`[REQ-COST-01]`)

The AI-Assisted Clinical Trial Screening Platform is designed with a fully serverless, pay-per-use architecture that eliminates fixed 24/7 idle compute and database cluster fees.

* **Governing Requirement:** `[REQ-COST-01]` (Total AWS infrastructure spend under $5,000.00 / year / ~$416.66 / month).
* **Projected Baseline Operational Spend:** **$167.61 – $228.03 / month** (**$2,011.32 – $2,736.36 / year**).
* **Budget Status: Compliant, comfortable margin.** The baseline (low) figure is **59.8% under** the `[REQ-COST-01]` ceiling; the high end of the estimation range is **45.3% under** it. Actual token usage should still be confirmed against Cost Explorer after production launch, but volume variance does not threaten the ceiling at this scale.
* **Verified against the AWS Price List API**: Amazon Textract `TABLES` - only pricing ($0.015/page, SKU `FYDFD3P65PH8TD44`), Anthropic Claude Sonnet 5 On-Demand token pricing (`us-west-2`, effective 2026-08-01) for both `[REQ-F-05]` (protocol extraction) and `[REQ-F-14]`/`[REQ-F-16]` (screening agent), and S3 Standard API request rates (PUT $0.005/1k, GET $0.0004/1k).

---

## 2. Baseline Sizing & Operational Scale Assumptions

All quantitative pricing calculations derive strictly from the operational metrics defined in `Requirements.md`:

| Sizing Parameter | Value | Reference / Notes |
| :--- | :--- | :--- |
| **Monthly Patient Screenings** | 50 patients / month | `Requirements.md` Section 2 |
| **Patient Record Volume** | 2 PDFs / patient (avg 100 pages / file) | 200 pages / patient = **10,000 pages / month** |
| **Average Patient File Size** | 25 MB – 50 MB / PDF | up to **150 MB per-patient total** (`Requirements.md` §2, `[REQ-NF-01], [REQ-F-22]`) — a single file can itself approach this ceiling |
| **Monthly Ingestion Data Volume** | 50 patients × 75 MB avg = **3.75 GB / month** | S3 Standard Ingestion |
| **Study Protocol Onboarding** | 2 protocols / month (avg 100 pages each) | **[Agent Assumption — Unsourced]** `Requirements.md` states no protocol-upload rate; this figure is not derived from Requirements.md and needs human confirmation per `AGENT.md` Rule 7. **200 pages / month** OCR volume |
| **Full-Document Reasoning Input per Screening** | 200 pages / patient × ~650 tokens / page (rate derived from `[REQ-F-05]`'s protocol-extraction sizing below) | **~130,000 input tokens / patient screening** (`[REQ-F-11], [REQ-F-15]`). **Corroborated by stage-0 POC measurement (2026-08-31):** real Bedrock usage on an actual 42-page protocol and 39-page patient record measured ~2.7 chars/token, working out to ~670–700 tokens/page for both documents — closely matching the ~650 tokens/page assumption already used here. |
| **Clinical Review API Access** | ~50 reviews / month × 50 requests/review | ~2,500 API calls & Presigned S3 fetches |

---

## 3. Detailed Component Cost Derivations (`[REQ-COST-01]`)

### 3.1 Document Processing — Amazon Textract (`[REQ-F-03, REQ-F-09]`)
* **Monthly Workload:** 10,000 patient record pages + 200 protocol pages = 10,200 pages / month.
* **Pricing Rate (re-verified via AWS Price List API, 2026-08-25):** `AsyncTablesPagesProcessed` in `us-west-2` is **$0.015/page** for the first 1M pages/month, $0.010/page thereafter (SKU `FYDFD3P65PH8TD44`, effective 2024-10-01) — confirms the `[REQ-F-03]`/`[REQ-F-09]` `TABLES`-only rate used below exactly.
* **Monthly Calculation:** 10,200 pages × $0.015/page = **$153.00 / month** (entirely within the first-1M-page tier).
* **Estimated Monthly Range:** **$137.70 – $168.30 / month** (±10% for month-to-month page-count variance).
* **Budget Impact:** Comfortably within the `[REQ-COST-01]` monthly ceiling ($416.66) on its own.

---

### 3.2 AI & Generative Reasoning — Amazon Bedrock (`[REQ-F-05, REQ-F-11, REQ-F-14, REQ-F-16]`)

No embeddings are generated in this architecture — full-document deterministic reasoning (`[REQ-F-11]`) requires no vector index.

#### A. Protocol Rule Extraction (`[REQ-F-05]` — Anthropic Claude Sonnet 5):
* **Monthly Workload:** 2 protocols × 100 pages = 130,000 input tokens; ~12,000 output tokens.
* **Pricing Rate (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** Claude Sonnet 5 On-Demand is **$2.20/1M input, $11.00/1M output** tokens (Standard tier) or **$2.00/1M input, $10.00/1M output** (Standard, Global tier). It is unconfirmed which tier bills the Geographic (US) cross-Region inference profile used here (`us.anthropic.claude-sonnet-5`) — the Price List API exposes no distinct "Geo" rate, only Standard and Standard-Global. Using the Standard (higher, more conservative) rate:
* **Monthly Calculation:** (130,000/1,000,000 × $2.20) + (12,000/1,000,000 × $11.00) = $0.286 + $0.132 = **$0.42 / month**.
* **Estimated Monthly Range:** **$0.38 – $0.42 / month** (Global-tier rate at the low end, Standard-tier at the high end — confirm actual billed tier via Cost Explorer once deployed).

#### B. Patient Screening Reasoning (`[REQ-F-14, REQ-F-16]` — Anthropic Claude Sonnet 5, direct `InvokeModel`):
* Input sizing reflects the full consolidated patient document (`[REQ-F-11], [REQ-F-15]`), verified against Claude Sonnet 5's 1,000,000-token context window (`docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5.html`).
* **Input Context Tokens:** 50 patients × ~130,000 tokens/patient (full-document text, §2 sizing table) = **6,500,000 input tokens / month**.
* **Output Verdict Tokens:** 50 patients × 1,500 output tokens = 75,000 output tokens / month.
* **Pricing Rate (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** Same Claude Sonnet 5 rates as §3.2.A above — `[REQ-F-14]` mandates Claude Sonnet 5 for this agent.
* **Token Cost:** (6,500,000/1,000,000 × $2.00–$2.20) + (75,000/1,000,000 × $10.00–$11.00) = $13.00–$14.30 + $0.75–$0.825 = **$13.75 – $15.13 / month**.
* **Estimated Monthly Subtotal:** **$13.75 – $15.13 / month**.

---

### 3.3 Storage (`[REQ-F-01, REQ-F-04, REQ-F-07, REQ-F-10, REQ-F-11]`)
* **Flagged assumption:** The `patient-consolidated-text` bucket (`[REQ-F-11]`) adds a second copy of already-extracted plain text per patient; text is negligible in size relative to the source PDFs already dominating the S3 storage figure below, so no separate line item is added here — an unverified rounding assumption, not a measured zero.
* **Amazon S3 Storage:** 6.25 GB new data / month (~75 GB in Year 1) × $0.023/GB = ~$1.73 / month — 3.75 GB from patient ingestion (§2) plus a ~2.5 GB/month protocol & overhead allowance.
* **S3 API Requests (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** ~250 PUTs/month × $0.005/1,000 + ~2,500 GETs/month × $0.0004/1,000 = negligible (~$0.002/month); presigned-URL generation and other document-access operations = ~$0.25 / month total.
* **Estimated Monthly Subtotal:** **$1.98 / month** (±10% for volume variance = **$1.78 – $2.18 / month**).

---

### 3.4 Database & Cache — Amazon DynamoDB (`[REQ-F-06, REQ-F-17]`)
* **Billing Mode:** On-Demand Capacity Mode.
* **Write Requests:** ~800 writes / month (protocols and verdicts) = negligible (< $0.05).
* **Read Requests:** ~5,000 reads / month = negligible (< $0.05).
* **Storage:** Under 1 GB total structured protocol and verdict data = negligible (< $0.25).
* **Estimated Monthly Subtotal:** **$1.00 – $5.00 / month**.

---

### 3.5 Orchestration & API — Step Functions, Lambda & API Gateway (`[REQ-F-02, REQ-F-07, REQ-F-08, REQ-SEC-02]`)
* **AWS Step Functions:** 50 patient screening executions + 2 protocol onboarding executions (~780 state transitions / month) = $0.02 / month. The `screening-trigger-handler` Lambda now guarantees exactly one execution per patient (`[REQ-F-07, REQ-F-08]`), so this figure no longer risks being doubled by duplicate per-file S3 event triggers.
* **AWS Lambda:** ~1,600 invocations / month across OCR handlers, parsing, the `screening-trigger-handler` finalize-upload trigger (~50/month, one per patient), and review APIs = $0.10 / month (the added trigger invocations are within the existing modeled volume, no change to this line).
* **Amazon API Gateway:** ~2,550 REST API requests / month, including the new `POST /patients/{PatientID}/screenings` route (~50/month) = $0.01 / month (no change to this line).
* **Networking & Data Transfer:** Pure serverless communication over AWS backbone; $0.00 idle networking cost (no VPC Interface Endpoint ENI or NAT Gateway fees).
* **Estimated Monthly Subtotal:** **$2.00 – $7.00 / month**.

---

### 3.6 Security, Governance & Auditing (`[REQ-SEC-01, REQ-SEC-04, REQ-SEC-05]`)
* **AWS KMS:** 1 Customer-Managed Key ($1.00) + ~32,000 cryptographic operations ($0.10) = $1.10 / month.
* **Amazon Cognito:** User directory with under 50 Clinical Investigator MAUs (Free Tier covers up to 50,000 MAUs) = $0.00 / month.
* **AWS CloudTrail & CloudWatch Logs:** S3 Data Events + 10 GB/month log ingestion + 7-year log retention = $10.00 – $30.00 / month.
* **CloudTrail DynamoDB Data Events (`[REQ-SEC-08]`):** ~5,800 item-level operations / month (~800 writes + ~5,000 reads, per §3.4) × $0.000001 / data event (verified via AWS Price List API, `us-west-2`, SKU `RDWGMK92MEBUVNWF`, effective 2026-07-01) = **~$0.01 / month** — absorbed within the range above, no change to the subtotal.
* **Estimated Monthly Subtotal:** **$11.00 – $30.00 / month**.

---

## 4. Total Cost Summary Table

| Category | Primary AWS Services | Baseline Low (USD/mo) | Baseline High (USD/mo) | Governing REQ IDs |
| :--- | :--- | :--- | :--- | :--- |
| **Document Processing** | Amazon Textract (Async OCR, Tables only) | $137.70 | $168.30 | `[REQ-F-03, REQ-F-09]` |
| **AI Reasoning** | Amazon Bedrock (direct InvokeModel, Claude Sonnet 5, full-document input) | $14.13 | $15.55 | `[REQ-F-05, REQ-F-11, REQ-F-14, REQ-F-16]` |
| **Storage** | Amazon S3 (raw, extracted, and consolidated text) | $1.78 | $2.18 | `[REQ-F-01, REQ-F-04, REQ-F-07, REQ-F-10, REQ-F-11]` |
| **Database Tier** | Amazon DynamoDB (On-Demand) | $1.00 | $5.00 | `[REQ-F-06, REQ-F-17]` |
| **Orchestration & API** | Step Functions + Lambda + API Gateway | $2.00 | $7.00 | `[REQ-F-02, REQ-F-08, REQ-SEC-02]` |
| **Security & Auditing** | AWS KMS + CloudWatch + CloudTrail + Cognito | $11.00 | $30.00 | `[REQ-SEC-01, REQ-SEC-04, REQ-SEC-05]` |
| **Total Monthly Spend** | | **$167.61** | **$228.03** | `[REQ-COST-01]` |
| **Total Annual Spend** | | **$2,011.32** | **$2,736.36** | **Budget Ceiling: $5,000.00 — baseline 59.8% under; high end 45.3% under (compliant, comfortable margin)** |

**Note:** AI Reasoning ($14.13–$15.55/mo) is Claude Sonnet 5 protocol extraction ($0.38–$0.42/mo, driven by protocol onboarding rate, not patient volume) plus full-document screening agent tokens ($13.75–$15.13/mo, `[REQ-F-11], [REQ-F-15]`); no vector index costs apply anywhere in this architecture. Document Processing remains the dominant cost driver; recommend validating actual token and page usage against AWS Cost Explorer after production launch.

