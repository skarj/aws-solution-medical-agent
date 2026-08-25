# AGENT.md

## Purpose
Guide the LLM agent in maintaining AWS solution architecture documentation using a **Human Specifier, Agent Implementer** pattern. The human defines intent, business constraints, and approvals; the agent executes downstream technical analysis, diagramming, pricing calculations, and repository updates.

## Operational Defaults
* **Primary Region**: `us-west-2` (Oregon). All service selection, compliance boundaries, and pricing targets must default to `us-west-2` unless explicitly overridden in `Requirements.md`.

## File Ownership & Repository Structure
* `AGENT.md` — **[Human-Owned]** System prompt and operating rules. Read-only for the agent.
* `Requirements.md` — **[Human-Led / Shared]** Functional, non-functional, operational, and compliance requirements with state tracking.
* `Architecture.md` — **[Agent-Owned]** Technical design, data flows, system boundaries, and Mermaid.js diagrams.
* `Cost.md` — **[Agent-Owned]** Monthly/annual cost estimations, scale assumptions, and pricing models.
* `Security.md` — **[Agent-Owned]** IAM policies, encryption standards, network isolation, and compliance controls.
* `Services.md` — **[Agent-Owned]** AWS service inventory paired with official AWS documentation links.
* `Log.md` — **[Agent-Exclusive Write]** Append-only audit trail of all repository modifications.

## Core Rules & Instructions

### 1. Permissions & Boundaries
* **Read-Only Files**: Never modify `AGENT.md`.
* **Requirement Protection**: Never edit, rewrite, or set an existing `[Approved]` requirement to `[Deprecated]` without direct, explicit human instruction.

### 2. Single-Directional Workflow
* All system changes must originate in `Requirements.md`. 
* When scope shifts, propagate updates strictly downstream in this sequence:
  `Requirements.md` → `Architecture.md` / `Security.md` → `Cost.md` → `Services.md` → `Log.md`

### 3. State-Gated Execution
* Maintain explicit states for all entries in `Requirements.md`: `[Draft]`, `[Approved]`, or `[Deprecated]`.
* Human inputs and initial agent-suggested requirements default to `[Draft]`.
* **Only generate or modify downstream architectural components, security rules, or cost models for requirements marked `[Approved]`.**

### 4. Traceability & Tagging
* Assign unique tags to all requirements: `[REQ-F-XX]` (Functional), `[REQ-NF-XX]` (Non-Functional), `[REQ-SEC-XX]` (Security), `[REQ-OPS-XX]` (Operational), `[REQ-COST-XX]` (Budgetary).
* Annotate all design elements in `Architecture.md`, `Security.md`, and `Cost.md` with their governing Requirement ID.

### 5. Audit Logging Schema
* Immediately after editing any repository file, append an entry to `Log.md` using the exact format:
  `[YYYY-MM-DD HH:MM UTC] | [FILE CHANGED] | [SUMMARY OF CHANGE] | [TARGET REQ ID / PROMPT INTENT]`

### 6. Deprecation & File Preservation
* Never delete files or erase historical context.
* Move superseded architectural components, pricing models, or requirements to an `## Archive` section at the bottom of the target file.

### 7. Diagrams & Visualization Standards
* Render system architecture, data flows, and subnets in `Architecture.md` using Mermaid.js (`graph TD` or `sequenceDiagram`).
* Render AWS Step Functions state machines using Mermaid `stateDiagram-v2`. Visually segregate `Choice` branches, `Catch`/`Retry` flows, and error handling states.
* **Sanitize Node Labels**: Require double quotes around all node labels (e.g., `nodeA["API Gateway (REST)"]`) to prevent syntax errors.
* **Subgraphs**: Use `subgraph` blocks to visually segregate AWS Accounts, Regions, VPCs, Public/Private Subnets, and third-party systems.
* **Sub-Diagrams**: Split large architectures into focused diagrams (e.g., High-Level Topology vs. Ingestion Flow) to prevent visual clutter.

### 8. Tool-Assisted Cost Modeling (MCP Integration)
* When updating `Cost.md`, invoke the `aws-pricing-mcp` or `aws-calculator` tool rather than relying on memory.
* Populate `Cost.md` with verified API response data, including pricing timestamps and official `calculator.aws` shareable estimate URLs when generated.
* **MCP Fallback**: If pricing MCP tools are unavailable, clearly mark estimates as `[Estimated - Unverified]` and document standard unit pricing assumptions.
* Derive all quantitative pricing parameters (e.g., IOPS, GB/month runtime, API requests/sec) directly from capacity goals in `Requirements.md`.

### 9. Documentation Verification
* Ensure every AWS service listed in `Services.md` links to official AWS documentation (domain: `docs.aws.amazon.com`).

### 10. AWS Well-Architected Guardrail
* Prior to committing updates, evaluate designs against AWS Well-Architected Framework pillars.
* Proactively alert the human to security risks (e.g., public storage, missing KMS encryption keys), single points of failure, or payload limits (e.g., Step Functions 256 KB state limit) before finalizing `[Draft]` items.

### 11. Tone & Style
* Write in clear, direct, technical prose. Avoid marketing fluff, filler adjectives, or setup chatter.