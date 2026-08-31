## Purpose

Point-in-time record of proof-of-concept testing performed against this architecture — what was tested, measurements taken, problems discovered, and which findings fed into which requirements. Unlike `Architecture.md`/`Security.md`/`Cost.md`/`Services.md` (current-state only, per `AGENT.md` Rule 11), this file is explicitly historical: each POC's results are appended as a dated, self-contained section and never rewritten to reflect later current-state changes — if a later POC or production run contradicts an earlier finding, that's a new section, not an edit to this one.

---

## Stage 0 — Full-document OCR + Bedrock reasoning smoke test (2026-08-31)

### Scope & method

Validated two open risks that had been flagged since 2026-08-27/28: whether full-document reasoning fits the `[REQ-NF-01]` 10-minute SLA, and whether verdict grounding quality (`[REQ-NF-04]`) is trustworthy. Tested with direct `boto3` calls to Textract and Bedrock `InvokeModel` — no Step Functions, Lambda, or Bedrock Agent wrapper — against:

- A real study protocol PDF (42 pages).
- A real redacted patient medical record (39 pages).

Run in a dedicated AWS account, separate from any production account, via the `us.anthropic.claude-sonnet-5` Geographic (US) cross-Region inference profile (`[REQ-OPS-01]`). Code and setup live in the external `medical-study-poc` repository (not part of this documentation repo, not intended to persist).

### Measurements

| Stage | Result |
| :--- | :--- |
| Protocol OCR (Textract, 42 pages) | 31.4s, 80,456 chars |
| Protocol rule extraction (Bedrock) | 11.5s, 29,786 input tokens, 770 output tokens |
| Patient record OCR (Textract, 39 pages) | 30.2s, 70,549 chars |
| Screening (Bedrock, full document + criteria) | 33.5–45.8s across runs, ~37,500 input tokens, 3,800–5,100 output tokens |
| **Total end-to-end (non-cached)** | **~111 seconds**, vs. the `[REQ-NF-01]` 600-second budget |
| Measured tokenization density | ~2.7 chars/token (both documents) — corroborates `Cost.md`'s existing ~650 tokens/page assumption |

**Caveat:** this covers one single-file patient record. `[REQ-NF-01]` also covers multi-file records up to 150 MB total (per `Requirements.md` §2) — that scenario, and the still-open `[REQ-NF-01]`-vs-§2 sizing conflict (flagged 2026-08-26, unresolved), were not exercised here.

### Problems found — OCR / Textract table extraction

- **Unrecoverable column merge, zero metadata signal.** Textract silently under-counted a table's columns (5 detected instead of 6), merging a `Taking?` header with `Authorizing Provider` into one `CELL` with `ColumnSpan=1` — no flag distinguishing this from an ordinary single-value cell. Tried and rejected two auto-split heuristics: `ColumnSpan`-gated splitting (never triggers — this case isn't flagged at all) and outlier-horizontal-gap detection (verified against raw geometry: the true gap in the merged case, ~0.006–0.008 normalized page-width units, is statistically indistinguishable from ordinary intra-sentence word spacing elsewhere in the same document, ~0.004–0.0064 — no safe threshold exists). **Not fixed** — accepted as a residual OCR-fidelity risk; the merged field stays fully readable, just not column-separated. Documented in `Architecture.md` Workflow 2 step 2.

### Problems found — grounding & citation quality

- **"Verbatim" quoting cannot be trusted at face value.** A mechanical check (does each quote appear as a literal, whitespace-normalized substring of the source?) surfaced three distinct ways the model doesn't literally quote even when the underlying facts are accurate:
   - Bridging a single logical list that Textract had split into two `TABLE` blocks across a page boundary (`...` mid-quote) — not fabrication, but exposed that a single `page_citation` field can't represent evidence spanning multiple pages.
   - Silently splicing two adjacent, related table rows (two allergy entries; a Creatinine result + an eGFR result) into one citation string with no disclosure — again not fabrication of facts, but a schema forcing an implicit multi-citation need into a single-quote field.
   - Non-deterministic preservation of inline annotation markers (`[JL.1]`/`[JL.²]`) — present verbatim in one run, silently dropped in another run of the same underlying fact.

   **Not fully fixed.** `Requirements.md` §5.2's schema now supports a `citations` array (multiple quotes/pages per criterion) and `[Draft] [REQ-NF-06]` mandates mechanical substring verification in production, forcing `MANUAL_REVIEW_REQUIRED` on any citation that still fails to verify — but the underlying tendency to reformat or combine source text remains a property of the model's behavior, not something eliminated.

### Final verdict quality (last run)

All 15 extracted criteria (4 inclusion, 11 exclusion) evaluated with full coverage (`[REQ-NF-05]`). Overall recommendation `INELIGIBLE`, driven by two well-grounded, verified-citation exclusions (hepatic insufficiency — MELD 26, esophageal varices; comorbid medical compromise — pancytopenia, portal hypertension, ascites). Several genuinely-absent criteria correctly returned `UNCERTAIN` rather than a guessed status.

### What changed in the governed docs as a result

- `Requirements.md`: `REQ-F-16` revised; new `[Draft] REQ-NF-06`; `REQ-SEC-07`/`REQ-SEC-08` cross-references updated; §5.2 Verdict Schema revised (`reasoning` field, `citations` array).
- `Architecture.md`: Workflow 2 step 2 (table-extraction implementation guidance + residual limitation), step 5 (latency risk resolved to validated, with scope caveat), step 6 (citation-verification gate added).
- `Cost.md`: token/page assumption corroborated with real measurement.

### Not tested / open for a future POC

- Multi-file patient records (only one file was tested).
- Record sizes approaching the 150 MB `[REQ-NF-01]` cap.
- Bedrock Guardrails (`[REQ-SEC-07]`) — discussed as a complementary control, not empirically exercised.
- Opus 5 vs. Sonnet 5 comparison — discussed, never run.
- `[REQ-OBS-01]`'s agreement-rate drift monitoring — needs multiple runs with reviewer signoff to be meaningful, out of scope for a single smoke test.
