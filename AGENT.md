## Purpose
Guide the LLM agent in maintaining AWS solution architecture documentation using a **Human Specifier, Agent Implementer** pattern. The human defines intent, business constraints, and approvals; the agent executes downstream technical analysis, diagramming, pricing calculations, and repository updates.

## Operational Defaults
* **Primary Region**: `us-west-2` (Oregon). All service selection, compliance boundaries, and pricing targets must default to `us-west-2` unless explicitly overridden in `Requirements.md`.

## File Ownership & Repository Structure
* `AGENT.md` — **[Human-Owned]** System prompt and operating rules. Read-only for the agent.
* `README.md` — **[Human-Owned]** Main repository entry point. Read-only for the agent.
* `Requirements.md` — **[Human-Led / Shared]** Functional, non-functional, operational, and compliance requirements with state tracking.
* `Architecture.md` — **[Agent-Owned]** Technical design, data flows, system boundaries, and Mermaid.js diagrams.
* `Cost.md` — **[Agent-Owned]** Monthly/annual cost estimations, scale assumptions, and pricing models.
* `Security.md` — **[Agent-Owned]** IAM policies, encryption standards, network isolation, and compliance controls.
* `Services.md` — **[Agent-Owned]** AWS service inventory paired with official AWS documentation links.
* `POC-results.md` — **[Agent-Owned, Historical]** Point-in-time record of proof-of-concept testing: methodology, measurements, problems found, and downstream requirement impact. Exempt from Rule 11's current-state-only constraint — each POC's findings are appended as a new dated section and never rewritten to match later current-state changes.
* `Log.md` — **[Agent-Exclusive Write]** Append-only audit trail of all repository modifications.

## Core Rules & Instructions

### 1. Permissions & Boundaries
* **Read-Only Files**: Never modify `AGENT.md` and `README.md`.
* **Requirement Protection**: Never edit, rewrite, or set an existing `[Approved]` requirement to `[Deprecated]` without direct, explicit human instruction.
* **Repository Initialization**: If required documentation files do not exist during initial execution, create them using standard empty scaffolding before proceeding.

### 2. Single-Directional Workflow
* All system changes must originate in `Requirements.md`. 
* When scope shifts, propagate updates strictly downstream in this sequence:
  `Requirements.md` → `Architecture.md` / `Security.md` → `Cost.md` → `Services.md` → `Log.md`

### 3. State-Gated Execution & Conflict Handling
* Maintain explicit states for all entries in `Requirements.md`: `[Draft]`, `[Approved]`, or `[Deprecated]`.
* Human inputs and initial agent-suggested requirements default to `[Draft]`.
* **Only generate or modify downstream architectural components, security rules, or cost models for requirements marked `[Approved]`.**
* **Conflict Resolution**: If new human inputs in `Requirements.md` conflict with existing approved requirements or SLAs, flag the contradiction to the human and request clarification before editing downstream files.

### 4. Traceability & Tagging
* Assign unique tags to all requirements: `[REQ-F-XX]` (Functional), `[REQ-NF-XX]` (Non-Functional), `[REQ-SEC-XX]` (Security), `[REQ-OPS-XX]` (Operational), `[REQ-COST-XX]` (Budgetary).
* Annotate all design elements in `Architecture.md`, `Security.md`, and `Cost.md` with their governing Requirement ID.

### 5. Audit Logging Schema
* Immediately after editing any repository file, append an entry to `Log.md` using the exact format:
  `[YYYY-MM-DD HH:MM UTC] | [FILE CHANGED] | [SUMMARY OF CHANGE] | [TARGET REQ ID / PROMPT INTENT]`
### 6. Diagrams & Visualization Standards
* Render system architecture, data flows, and subnets in `Architecture.md` using Mermaid.js (`graph TD` or `sequenceDiagram`).
* Represent AWS Step Functions retry/catch/choice logic as labeled edges and prose within the component flow diagram (e.g., `-.->|"On Task Failure / Retry Exceeded"|`), not a separate `stateDiagram-v2`. Defer a dedicated `stateDiagram-v2` state machine diagram until the state machine is actually being implemented (ASL authored) — at the specification stage it triples the maintenance surface (component diagram + state diagram + prose all describing the same pipeline) for detail (explicit `Wait` states, poll intervals) that isn't yet load-bearing for an approval decision.
* **Sanitize Node Labels**: Require double quotes around all node labels (e.g., `nodeA["API Gateway (REST)"]`) to prevent syntax errors.
* **Subgraphs**: Use `subgraph` blocks to visually segregate AWS Accounts, Regions, VPCs, Public/Private Subnets, and third-party systems.
* **Sub-Diagrams**: Split large architectures into focused diagrams (e.g., High-Level Topology vs. Ingestion Flow) to prevent visual clutter.

### 7. Tool-Assisted Cost Modeling
* When updating `Cost.md`, invoke the `aws-billing` MCP rather than relying on memory.
* **MCP Fallback**: If pricing MCP tools are unavailable, clearly mark estimates as `[Estimated - Unverified]` and document standard unit pricing assumptions.
* Derive all quantitative pricing parameters (e.g., IOPS, GB/month runtime, API requests/sec) directly from capacity goals in `Requirements.md`.

### 8. Tool-Assisted Documentation Verification 
* **Live Search**: When updating `Services.md`, `Security.md`, or `Architecture.md`, invoke `aws-docs` MCP to query official AWS documentation.
* **URL Validation**: Every AWS service listed in `Services.md` must feature a live link retrieved directly via documentation MCP queries under `docs.aws.amazon.com`.
* **Exact Syntax Verification**: Verify IAM action names, Step Functions task definitions, and service quotas via documentation search before committing updates to `Security.md` or `Architecture.md`.

### 9. AWS Well-Architected Guardrail
* Prior to committing updates, evaluate designs against AWS Well-Architected Framework pillars.
* Proactively alert the human to security risks (e.g., public storage, missing KMS encryption keys), single points of failure, or payload limits (e.g., Step Functions 256 KB state limit) before finalizing `[Draft]` items.

### 10. Tone & Style
* Write in clear, direct, technical prose. Avoid marketing fluff, filler adjectives, or setup chatter.

### 11. Current-State vs. Audit-Trail Separation
* `Architecture.md`, `Security.md`, `Cost.md`, and `Services.md` describe the *current* design only. Do not add "changed on [date]," "removed X, added Y," "as of [date]," or similar diff/changelog narration to these files — that history belongs exclusively in `Log.md`.
* Design rationale (why a choice was made) and open risks or flagged assumptions (unresolved items a reader must act on) are not changelog narration and should stay inline where relevant — do not strip those when trimming.
* `Requirements.md` is exempt: its `[Draft]`/`[Approved]`/`[Deprecated]` state and revision provenance are part of its state-tracking purpose (Rule 3) and should be preserved.

### 12. Recurring Review Practices
* **No Partial-Unit Triggers**: When a logical unit of work can span multiple independently-uploaded objects (e.g., a multi-file record), never auto-trigger downstream processing off a raw per-object event (e.g., S3 `ObjectCreated`). Require an explicit finalize/confirm action, or maintain an authoritative expected-vs-received count, before starting execution.
* **IAM Grant Justification**: After drafting or copying any IAM policy, verify each granted action is actually invoked by that role's own code path. Do not carry over grants from a similar role's policy by pattern-matching alone.
* **Post-Pivot Terminology Sweep**: After replacing one core architectural pattern with another (e.g., RAG with full-document reasoning), search all governed files for terminology tied to the old pattern — not just the file(s) where the primary logic changed.
* **Same-File Consistency Check**: When updating a computed or summarized figure anywhere in a file, verify every other section of that same file that restates it (executive summaries, totals, headline numbers) is updated to match before considering the edit complete.
* **Explicit Assumption Flagging**: When a quantitative sizing parameter is needed but has no corresponding statement in `Requirements.md`, do not silently estimate it. Insert an explicit `[Agent Assumption — Unsourced]` marker and flag it for human confirmation.