## 1. Executive Summary & Budget Target Compliance (`[REQ-COST-01]`)

The AI-Assisted Clinical Trial Screening Platform is designed with a fully serverless, pay-per-use architecture that eliminates fixed 24/7 idle compute and database cluster fees.

* **Governing Requirement:** `[REQ-COST-01]` (Total AWS infrastructure spend under $5,000.00 / year / ~$416.66 / month).
* **Projected Baseline Operational Spend:** **$317.81 – $409.26 / month** (**$3,813.72 – $4,911.12 / year**).
* **Budget Status: Compliant, but margin has compressed.** The baseline (low) figure is **23.7% under** the `[REQ-COST-01]` ceiling; the high end of the estimation range is only **1.8% under** it (down from 5.2% prior to this revision). This drop is driven entirely by the 2026-08-27 decision to replace RAG with full-document deterministic reasoning (`[REQ-F-11], [REQ-F-15]`): removing Bedrock Knowledge Base retrieval/embedding costs saved money, but feeding the full ~130,000-token patient document to Claude Sonnet 5 on every screening (vs. ~24,000 tokens of retrieved chunks) costs more net. **Flagged risk:** at $409.26/mo high end, normal month-to-month volume variance could push actual spend over the `[REQ-COST-01]` ceiling; recommend re-confirming actual token usage against Cost Explorer soon after production launch.
* **Verified against the AWS Price List API**: Amazon Textract `TABLES` - only pricing ($0.015/page, SKU `FYDFD3P65PH8TD44`), Anthropic Claude Sonnet 5 On-Demand token pricing (`us-west-2`, effective 2026-08-01) for both `[REQ-F-05]` (protocol extraction) and `[REQ-F-14]`/`[REQ-F-16]` (screening agent), and S3 Standard API request rates (PUT $0.005/1k, GET $0.0004/1k). Bedrock Knowledge Base storage/retrieval pricing no longer applies as of 2026-08-27 (RAG removed).

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
| **Full-Document Reasoning Input per Screening** | 200 pages / patient × ~650 tokens / page (rate derived from `[REQ-F-05]`'s protocol-extraction sizing below) | **~130,000 input tokens / patient screening** — replaces RAG chunk retrieval per the 2026-08-27 `[REQ-F-11], [REQ-F-15]` decision |
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

### 3.2 AI & Generative Reasoning — Amazon Bedrock (`[REQ-F-05, REQ-F-11, REQ-F-14, REQ-F-16]`)

**2026-08-27 change:** Removed the Amazon Titan Text Embeddings v2 subsection (previously ~$0.24/month) — no embeddings are generated under full-document deterministic reasoning (`[REQ-F-11]` superseded; RAG removed per direct human instruction).

#### A. Protocol Rule Extraction (`[REQ-F-05]` — Anthropic Claude Sonnet 5):
* **Monthly Workload:** 2 protocols × 100 pages = 130,000 input tokens; ~12,000 output tokens.
* **Pricing Rate (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** Claude Sonnet 5 On-Demand is **$2.20/1M input, $11.00/1M output** tokens (Standard tier) or **$2.00/1M input, $10.00/1M output** (Standard, Global tier). It is unconfirmed which tier bills the Geographic (US) cross-Region inference profile used here (`us.anthropic.claude-sonnet-5`) — the Price List API exposes no distinct "Geo" rate, only Standard and Standard-Global. Using the Standard (higher, more conservative) rate:
* **Monthly Calculation:** (130,000/1,000,000 × $2.20) + (12,000/1,000,000 × $11.00) = $0.286 + $0.132 = **$0.42 / month**.
* **Estimated Monthly Range:** **$0.38 – $0.42 / month** (Global-tier rate at the low end, Standard-tier at the high end — confirm actual billed tier via Cost Explorer once deployed).

#### B. Patient Screening Reasoning Agent (`[REQ-F-14, REQ-F-16]` — Anthropic Claude Sonnet 5):
* **2026-08-27 change:** Input sizing replaced RAG-retrieved chunks with the full consolidated patient document per `[REQ-F-11], [REQ-F-15]`, verified against Claude Sonnet 5's 1,000,000-token context window (`docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-sonnet-5.html`).
* **Input Context Tokens:** 100 patients × ~130,000 tokens/patient (full-document text, §2 sizing table) = **13,000,000 input tokens / month**.
* **Output Verdict Tokens:** 100 patients × 1,500 output tokens = 150,000 output tokens / month (unchanged — verdict JSON size is unaffected by input size).
* **Pricing Rate (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** Same Claude Sonnet 5 rates as §3.2.A above — `[REQ-F-14]` mandates Claude Sonnet 5 for this agent.
* **Token Cost:** (13,000,000/1,000,000 × $2.00–$2.20) + (150,000/1,000,000 × $10.00–$11.00) = $26.00–$28.60 + $1.50–$1.65 = **$27.50 – $30.25 / month**.
* **RAG Retrieval Cost:** **$0.00 / month** — removed 2026-08-27; no Knowledge Base queries occur under full-document deterministic reasoning.
* **Estimated Monthly Subtotal:** **$27.50 – $30.25 / month** (up from the prior $12.30–$12.93/month RAG-based figure; the ~5x input-token increase outweighs the removed $6.00/month retrieval cost).

---

### 3.3 Storage (`[REQ-F-01, REQ-F-04, REQ-F-07, REQ-F-10, REQ-F-11]`)
* **2026-08-27 change:** Removed Bedrock Knowledge Base vector storage (previously $2.50/month) — no vector index exists under full-document deterministic reasoning. The new `patient-consolidated-text` bucket (`[REQ-F-11]`) adds a second copy of already-extracted plain text per patient; text is negligible in size relative to the source PDFs already dominating the S3 storage figure below, so no separate line item is added — flagged as a minor, immaterial rounding assumption rather than a verified zero.
* **Amazon S3 Storage:** 10 GB new data / month (~120 GB in Year 1) × $0.023/GB = ~$2.76 / month.
* **S3 API Requests (verified via AWS Price List API, `us-west-2`, effective 2026-08-01):** ~500 PUTs/month × $0.005/1,000 + ~5,000 GETs/month × $0.0004/1,000 = negligible (~$0.005/month); presigned-URL generation and other document-access operations = ~$0.50 / month total.
* **Estimated Monthly Subtotal:** **$3.26 / month** (±10% for volume variance = **$2.93 – $3.59 / month**).

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
| **AI Reasoning** | Amazon Bedrock (Agent, Claude Sonnet 5, full-document input) | $27.88 | $30.67 | `[REQ-F-05, REQ-F-11, REQ-F-14, REQ-F-16]` |
| **Storage** | Amazon S3 (raw, extracted, and consolidated text) | $2.93 | $3.59 | `[REQ-F-01, REQ-F-04, REQ-F-07, REQ-F-10, REQ-F-11]` |
| **Database Tier** | Amazon DynamoDB (On-Demand) | $1.00 | $5.00 | `[REQ-F-06, REQ-F-17]` |
| **Orchestration & API** | Step Functions + Lambda + API Gateway | $2.00 | $7.00 | `[REQ-F-02, REQ-F-08, REQ-SEC-02]` |
| **Security & Auditing** | AWS KMS + CloudWatch + CloudTrail + Cognito | $11.00 | $30.00 | `[REQ-SEC-01, REQ-SEC-04, REQ-SEC-05]` |
| **Total Monthly Spend** | | **$317.81** | **$409.26** | `[REQ-COST-01]` |
| **Total Annual Spend** | | **$3,813.72** | **$4,911.12** | **Budget Ceiling: $5,000.00 — baseline 23.7% under; high end only 1.8% under (compliant, tight margin)** |

**Note (2026-08-27, RAG replaced with full-document deterministic reasoning per `[REQ-F-11], [REQ-F-15]`, direct human instruction):** Removed Bedrock Knowledge Base storage ($5.00/GB-month) and retrieval ($0.001/query) and Amazon Titan embedding costs entirely — no vector index exists. The AI Reasoning row rises from $13.00–$13.50/mo to $27.88–$30.67/mo: Claude Sonnet 5 protocol extraction $0.38–$0.42/mo (unchanged) + Claude Sonnet 5 screening agent tokens $27.50–$30.25/mo (up from $6.30–$6.93/mo — full ~130,000-token patient document per screening replaces ~24,000 tokens of retrieved chunks). The Storage row drops from $5.20–$6.35/mo to $2.93–$3.59/mo (KB vector storage removed). Net effect: Total Monthly/Annual Spend rises from $305.20–$394.85/mo ($3,662.40–$4,738.20/yr) to **$317.81–$409.26/mo ($3,813.72–$4,911.12/yr)** — still compliant with `[REQ-COST-01]`, but the high-end margin has compressed from 5.2% to **1.8%** under the ceiling. Recommend validating actual token usage against AWS Cost Explorer shortly after production launch, since normal volume variance could now approach the ceiling.

