# SentinelAI — System Design (SDD-01)

*Cross-layer exploit-path reasoning for CI/CD security auditing*

**Sprint 0 Deliverable** · SentinelAI-Team · Graduate Capstone Project
Version 1.1 · Sprint 0 baseline · July 2026

---

## Document control

| Field | Detail |
|-------|--------|
| Project | SentinelAI — Automated Threat Modeling & SecOps Engine |
| Team | SentinelAI-Team (6 members) |
| Members | Mohamed Fathi Ibrahim, Mohamed Yasser Elsherbiny, Seif Eldin Medhat Farouk, George Mariey Demyan Beshara, Hana Mohamed Elsayeh, Mostafa Fouad Hendy |
| Sprint | Sprint 0 — Foundations, contracts & de-risk (Week 1) |
| Status | Baseline design, reflects three validated POC pipelines |
| Locked stack | C#/.NET · ASP.NET Core · Microsoft Agent Framework 1.0 · GPT-4o (Azure) / Claude (demo) · Qdrant · SQL Server · Angular · GitHub Actions + Docker |

> **What changed in v1.1.** Delivery moves to a **distributed-scan (D2)** model: scanners and graph-input extraction run in a GitHub Actions workflow at PR time, and the backend receives findings plus infrastructure/manifest artifacts — never the application source. Trigger surface is GitHub Actions; secret pre-scanning uses the Gitleaks CLI on the runner. The rule-mapping table is now a first-class SQL Server table.

---

## 1. Purpose & scope

This document is the Sprint 0 system-design baseline for SentinelAI. It fixes the high-level architecture, the responsibilities of every component, and the way components communicate, so that Sprint 1+ implementation builds against a stable picture. It reflects three validated proof-of-concept pipelines — knowledge ingestion, scan-time retrieval, and the resource graph — and the current locked technology stack.

> **Sprint 0 intent.** Sprint 0 does not ship features. Its output is the walking-skeleton design and the contracts that let the highest-risk components (multi-agent orchestration and the cross-layer graph) be built and de-risked first.

### 1.1 The one-sentence thesis

SentinelAI's value is **cross-layer attack-chain reasoning**: taking findings that each existing tool produces in isolation and reasoning about how they compose into a real exploit path — grounded in security knowledge, and forced to survive an adversarial rebuttal. The output is an honestly-framed prioritized **draft audit** for human review, not a verified verdict.

### 1.2 Delivery surface

SentinelAI is delivered primarily as a **GitHub Actions workflow step** that runs at pull-request time, because whole-system exploit-path reasoning needs the full change set and infrastructure graph, which only exists at PR time. Under the D2 model, the workflow runs the scanners and extracts the graph inputs on the runner, then sends findings plus infrastructure/manifest artifacts to the backend. An Angular web UI presents the adjudicated audit, and findings also post back to the pull request as comments.

---

## 2. The distributed-scan (D2) delivery model

The defining architectural decision of this baseline is **where the code is observed versus where it is reasoned over**. The workflow is where code lives and gets *observed* (scanned, graph-extracted); the backend is where those observations get *reasoned over* (graph construction, retrieval, debate). The application source never leaves the customer's runner.

### 2.1 Why D2

- **Clean sandbox boundary** — because application source never reaches the backend, there is no large code upload and no need for the backend to reach back into the repository. The backend's trust boundary stays narrow.
- **Strong privacy line** — the backend receives infrastructure definitions, dependency manifests, and scanner findings, *never* the C# application source.
- **Trivial retention** — with no source held, "purge after audit" is near-vacuous; the received bundle is deleted once the audit completes.
- **Centralized hard logic** — the two highest-risk, highest-value components (resource-graph construction and the agent debate) stay in the backend, in one place the team controls and can iterate on during the build.

### 2.2 What the backend needs (and why findings alone are not enough)

Findings decorate nodes; they do **not** carry edges. The resource graph's edges come from source artifacts — Terraform references, the Dockerfile, dependency manifests, and IAM policy statements. So the bundle must carry both the findings and the artifacts edges are drawn from.

| Graph need | Artifact shipped in the bundle | Consequence if omitted |
|------------|--------------------------------|------------------------|
| Infra spine | `terraform graph` DOT output (produced after `terraform init`) | Thin or empty spine |
| Spine fallback + role→resource edges | Terraform source files (esp. IAM policy statements) | No role→resource edge; risk of zero chains |
| Dependency → code seam | csproj + packages.lock.json | Broken dependency seam; empty SCA |
| Code → infra seam | Dockerfile | No image-name join |
| Node decoration / seeds | All five scanners' SARIF/JSON | Nodes unmarked; no hot chain seeds |

The bundle therefore reduces to: **the five scanners' findings + the `terraform graph` DOT + Terraform source + Dockerfile + dependency manifests.** This is the complete and minimal set required to reconstruct the cross-layer chain.

<!-- ### 2.3 Honest caveats of D2

- **Fragile cross-layer joins remain fragile.** The image-name (code→infra) and role→resource joins are convention-dependent and break on real repos (dynamic tags, registry prefixes) regardless of where the graph is built. The authored fixture keeps these joins clean; general-repo robustness is a stated limitation.
- **Client-produced findings are trusted.** Running scanners on the customer's runner means the backend trusts the SARIF it receives. This is acceptable for a controlled fixture; for anything broader it warrants a pinned composite Action and validation that incoming SARIF is well-formed and internally consistent. -->

---

## 3. Architecture overview

The system separates cleanly into layers spanning the runner and the backend. A scan is triggered by the workflow at PR time; the workflow observes the code and ships a bundle; the sandboxed backend turns that bundle into findings + a graph, reasons over it via the agent debate, and returns a draft audit surfaced in the UI and as PR comments.

### 3.1 Layered view

| Layer | Where | Components | Responsibility |
|-------|-------|-----------|----------------|
| Client / trigger | Runner | GitHub Actions workflow, Angular report UI, PR comment poster | Trigger a scan at PR time; observe code; present and post back the audit |
| Runner-side scan | Runner | Gitleaks CLI, 5 scanners, `terraform graph`, graph-input collector | Secret pre-scan, produce findings + graph inputs, package and upload |
| API & gateway | Backend | ASP.NET Core Web API, auth/RBAC, tenant isolation | Authenticate, authorize, isolate tenants, accept the bundle |
| Ingress security | Backend | Findings-text secret scan + redaction | Redact secrets before any content reaches an LLM |
| Scan pipeline (B) | Backend | SARIF/JSON normalization, rule-mapping table, resource graph, traversal, retrieval | Turn the bundle into findings + a graph, then query knowledge |
| Agent debate | Backend | Orchestrator + Red / Blue / Reporter agents (Agent Framework 1.0) | Assert, validate, and adjudicate cross-layer exploit chains |
| Data & external | Backend | Qdrant, SQL Server, Azure OpenAI, Langfuse, App Insights | Vector knowledge, relational + rule-mapping records, inference, observability |

### 3.2 End-to-end flow

1. **Workflow (runner):** checkout → Gitleaks CLI secret pre-scan → run five scanners (Semgrep, Roslyn, Dependency-Check, Trivy, Checkov) → `terraform init` + `terraform graph` → collect graph-input files → package findings + graph inputs → `POST /v1/scans` with a scoped machine token.
2. **API & gateway (backend):** authenticate, authorize, tenant-scope the job; return `202` + poll URL.
3. **Ingress security (backend):** secret-scan and redact findings text before anything reaches an LLM — a guaranteed gate, independent of the runner-side pre-scan.
4. **Scan pipeline (backend):** normalize SARIF/JSON into one unified finding set → resolve missing CWEs via the SQL rule-mapping table → build the resource graph from the shipped inputs → generate bounded candidate chains.
5. **Agent debate (backend):** Red asserts chains grounded in offense knowledge → Blue validates each link against the real configuration + defense knowledge → Reporter adjudicates.
6. **Return:** emit the prioritized draft audit with citations; the workflow polls and posts it back as PR comments; the Angular UI presents the full audit.
7. **Retention:** the received bundle is purged after the audit.

---

## 3A. The AI subsystem

The AI subsystem is the load-bearing part of SentinelAI — the whole thesis is cross-layer exploit-path reasoning, which rests entirely on the pieces described here. It is placed first among the component detail because everything else exists to feed it.

### 3A.1 The four agents

The system runs a four-agent debate, sequenced by an Orchestrator — not a single model answering in one shot. Each agent has a narrow, distinct job.

- **Orchestrator** — controls the whole debate. It sequences the turns (Red → Blue → Reporter), enforces the turn-cap so the loop always terminates, detects convergence, and routes each turn to the right model tier. It reads and writes the shared session state the other agents pass work through. It asserts nothing about security itself; it is the conductor.
- **Red Team agent** — the attacker. Given the bounded candidate chains from the graph plus retrieved **offense** knowledge (ATT&CK / CAPEC), it asserts ordered cross-layer exploit paths: for each hop it names the node, the technique that enables the transition, and the evidence. It reasons *within* the deterministically-generated candidate chains rather than roaming the whole graph freely — that boundedness is what stops it inventing edges.
- **Blue Team agent** — the defender and the false-positive reducer. It takes each link of a Red-asserted chain and tests it against the real configuration, using retrieved **defense** knowledge (OWASP / mitigations). The key mechanic: **breaking one link breaks the chain.** If Blue shows any single hop does not actually hold, the whole exploit path collapses. This is what suppresses the false positives that flat scanners drown in.
- **Reporter agent** — the adjudicator. It reads the debate transcript and assembles the surviving, validated chains into the prioritized draft audit, ranked by severity and exploitability, with every assertion carrying its citation. It enforces the "draft for human review, not a verdict" framing.

> **Why multi-agent at all.** The adversarial structure is the source of grounded, low-false-positive findings. A claim only survives if it withstands Blue's rebuttal **and** validates against the real configuration. A single model asked to "find attack paths" tends to hallucinate plausible-sounding chains; forcing every assertion through an adversary that can kill it on one broken link is what makes the output trustworthy. This is also why orchestration is the project's top technical risk.

### 3A.2 Bounded chaining

Chaining is the thesis, but it is deliberately constrained so it stays reproducible and affordable: candidate chains are capped at **3–4 hops**, seeded from the **highest-severity "hot" findings**, with the **turn-cap** enforcing termination. The Red agent reasons only inside candidates generated deterministically from **real graph edges**. This "deterministic candidates + free reasoning within bounds" middle path avoids three failure modes at once: hallucinated edges, cost blow-up, and non-reproducible benchmarks.

### 3A.3 The knowledge corpus (Qdrant, two collections)

The agents do not reason from memory; they retrieve grounded knowledge from a vector store split into two collections **by role**:

- **offense** — ATT&CK, CAPEC, exploitation-oriented knowledge. The Red agent queries this.
- **defense** — OWASP, mitigations, fixes. The Blue agent queries this.

A single knowledge chunk can route to offense, defense, or both depending on its content — a CVE record contributes exploitation context to offense **and** remediation context to defense. The split is by attacker/defender role, never by which scanner produced a finding, so a Checkov finding and a Dependency-Check finding resolve into the **same shared corpus** and can therefore chain together. Giving each tool its own index would rebuild the exact tool-silo problem the product exists to solve.

### 3A.4 The two-pipeline separation (the load-bearing invariant)

Knowledge is embedded **once, offline** (Pipeline A): canonical sources — NVD, ATT&CK, CAPEC, OWASP — are cleaned, chunked, embedded, and upserted into Qdrant. At scan time (Pipeline B), the scanned project is turned into findings + a graph, and those findings **query** the embedded knowledge. **The repository's own code is never embedded.** This distinction is the single most important idea in the AI subsystem; conflating the two pipelines is the classic way to waste days trying to vector-index Terraform.

### 3A.5 The retrieval decision tree

Matching a finding to the right knowledge chunk is a decision tree, not one search — the presence or absence of a clean identifier picks the path:

1. **Rule-mapping lookup (always first).** Every finding passes through the SQL rule-mapping table, resolving the tool's `check_id` to a CWE (and CVE where available). Exact lookup, never embedded.
2. **Exact payload filter.** If the finding now has a clean CVE or CWE, filter Qdrant by that exact ID — no similarity needed. Dependency findings almost always have a CVE; code findings usually carry a CWE, so both mostly resolve here.
3. **Semantic search.** If the finding has no clean ID — common for infra misconfigurations — build a short text query from the finding (its message + resource type), embed *that query string*, and run vector similarity to find related techniques and patterns. This is where embeddings do the real work.
4. **Hybrid refine.** Within the offense or defense collection, apply metadata filters (severity, source, year) and *then* rank by vector similarity, fusing dense and sparse where sparse is available.

The rule of thumb across the three layers: dependency findings almost always have a CVE (exact filter); code findings usually carry a CWE (mostly exact filter); infra findings often have no CVE, just a rule id — the rule-mapping table resolves the CWE, then semantic search finds related ATT&CK/CAPEC, which is where embeddings carry the load.

### 3A.6 Two hard rules that live in this subsystem

- **Same-model invariant.** The identical embedding model must run at index time and query time. A mismatch puts the vectors in different spaces and makes similarity scores meaningless. Ingest fails loud if the intended model is unavailable rather than silently falling back to a degraded embedder.
- **Source-filtering is mandatory.** Unfiltered semantic retrieval lets the ~200k CVEs drown the ~1.3k ATT&CK technique entries, so the offense and defense paths must query their respective filtered subsets. Retrieving without the source filter buries exactly the attack-technique knowledge the Red agent needs.

### 3A.7 Models and tiers

The agents sit behind Agent Framework's connector abstraction, so the provider is a config change: production is GPT-4o via Azure OpenAI (Azure-native, keeping inference in the same security boundary as embeddings); the demo runs Claude Sonnet for reasoning-heavy turns and Claude Haiku for high-volume routine turns. The Orchestrator routes reasoning-heavy debate turns (chaining, link validation, adjudication) to the high tier and routine turns to the cheap tier; cost per audit by tier is tracked.

| Tier | Model (prod / demo) | Used for |
|------|--------------------|---------|
| High (reasoning) | GPT-4o / Claude Sonnet | Debate turns: chaining, link validation, adjudication |
| Cheap (routine) | GPT-4o-mini / Claude Haiku | High-volume routine turns, formatting, summarization |

### 3A.8 Grounding & honesty guarantees

- Every surviving assertion in the draft audit **cites its retrieved knowledge chunk**.
- DEPRECATED CAPEC/ATT&CK entries are filtered out of results because their thin text scores spuriously high.
- The output is framed as a prioritized **draft audit for human review**, not a verified verdict.
- A future self-improving feedback loop — where recurring debate findings get promoted into the knowledge base — is explicitly deferred; the base build only reserves the fields for it, and **human approval of any promotion is non-negotiable**.

---

## 4. Component responsibilities

### 4.1 Runner-side workflow

The GitHub Actions workflow is a thin trigger and delivery mechanism that also performs observation. It checks out the code, runs the Gitleaks CLI as an early secret filter, runs the five scanners to emit SARIF/JSON, runs `terraform graph` to capture the infra spine, collects the graph-input files, uploads the bundle, then polls for and posts the result. It performs no reasoning — it produces raw material only.

### 4.2 Frontend

**Angular** report UI. Presents each adjudicated exploit chain, its per-hop evidence, severity, and the explicit draft-audit framing. It is a read surface over the report API; it does not run analysis. It enforces the same authentication and tenant scoping as the API.

### 4.3 Backend & orchestration

**ASP.NET Core Web API on .NET**, C# throughout to match the Azure stack. It hosts the scan pipeline and the agent host. The multi-agent debate is orchestrated with **Microsoft Agent Framework 1.0** — the successor to Semantic Kernel, which is now in maintenance mode. Its group-chat workflow, checkpointing, and session state manage the shared memory and state transitions across the debate. This orchestration is the project's top technical risk and is stood up first.

> **Stack corrections carried into Sprint 0.** The original pitch named Semantic Kernel, React, and Azure DevOps. The locked stack is Microsoft Agent Framework 1.0, Angular, and GitHub Actions.

### 4.4 Databases

| Store | Technology | Holds | Embedded? |
|-------|-----------|-------|-----------|
| Relational | SQL Server | Users, tenants, projects, scan jobs, findings, chains, reports, citations | No |
| Rule-mapping table | SQL Server table | Tool check_id → CWE/CVE | No |
| Vector (RAG) | Qdrant (offense/defense collections) | CVE / ATT&CK / CAPEC / OWASP knowledge chunks as vectors + payload | Yes |
| Benchmark corpus | Separate store | TerraGoat, labeled C# fixtures — ground truth | No (never embedded) |

> **The rule-mapping table is a SQL table, not vector data.** It resolves a scanner's `check_id` to a CWE/CVE by **exact lookup** — a deterministic dictionary operation, never a similarity or embedding operation. It lives in SQL Server alongside the relational data, and it never enters Qdrant. Every finding passes through it first to resolve its CWE when the SARIF text lacks one.

> **Never conflate the stores.** The scanned repository is never embedded. Knowledge is embedded once (Pipeline A); findings query that knowledge (Pipeline B). The rule-mapping table and benchmark ground truth stay out of the vector store.

### 4.5 External services

- **Azure OpenAI** — serves GPT-4o for the agents in production; the demo runs Claude Sonnet/Haiku behind the same connector abstraction. Azure-native inference keeps the LLM inside the same security boundary as embeddings.
- **Langfuse** — traces LLM behaviour: per-agent prompts/responses, token and cost per audit, per-turn latency, full debate replay.
- **Azure Application Insights** — system health: API latency, pipeline-step duration, error rates. Agent Framework emits OpenTelemetry natively, so both are configuration rather than custom build.
- **Azure Key Vault** — holds all secrets; nothing sensitive is stored in code or config.

### 4.6 AI components

The AI subsystem is the load-bearing part of the project and is described in full in **Section 3A (The AI subsystem)** above. In summary it comprises four agents (Orchestrator, Red Team, Blue Team, Reporter), the Qdrant-backed RAG knowledge corpus split into offense and defense collections, and the retrieval decision tree that matches each finding to grounded knowledge.

---

## 5. How components communicate

### 5.1 Communication map

| From → To | Mechanism | Notes |
|-----------|-----------|-------|
| Workflow → API | HTTPS/TLS REST, scoped machine token | POST /v1/scans at PR time with the findings + graph-input bundle; returns 202 + poll URL |
| Workflow → PR | GitHub API | Posts acknowledgment, then the adjudicated audit as PR comments |
| API → scan pipeline | In-process, canonical data contracts | Finding, Node, Edge, Chain, Report records |
| Pipeline → rule-mapping table | SQL query | Exact check_id → CWE/CVE lookup |
| Pipeline → Qdrant | Qdrant client (gRPC/HTTP) | Filter-before-rank queries; offense/defense collections |
| Orchestrator → agents | Agent Framework group-chat + session state | Checkpointed; turn-cap; per-turn model-tier routing |
| Agents → Azure OpenAI | HTTPS via connector abstraction | Provider switch is a config change |
| Pipeline → SQL Server | Entity Framework / ADO.NET | Job status, findings, chains, reports, retention state |
| API → Angular UI | HTTPS/TLS REST + JWT | Read-only report surface, tenant-scoped |
| Runtime → Langfuse / App Insights | OpenTelemetry export | Traces and health metrics, native emission |

### 5.2 Contracts hold the seams

Every stage boundary hands off via shared canonical data contracts — **Finding, Node, Edge, Chain, Report** — using a single canonical node-ID scheme (`type:identifier`). If two extractors disagree on IDs, the graph silently splits into disconnected per-layer islands, so contract tests fail loudly on schema or ID drift.

---

## 6. PR comment surfaces

Two comment moments, deliberately distinct:

- **Acknowledgment (on receipt).** Posted once the bundle is accepted: job ID, the commit/PR it is tied to, per-layer finding counts, and a running status. Ideally this same comment is later edited in place to become the result, avoiding a second comment.
- **Draft audit (on completion).** Leads with a one-line *draft-audit-for-human-review* framing, then the prioritized exploit chains highest-severity first. Each chain shows its ordered cross-layer path, per-hop evidence with the cited knowledge chunk, which hops the Blue agent validated, and the impact at the end of the chain. Suggested fixes note that breaking one link breaks the chain. Findings that did not chain are demoted to a short collapsed section or a count, so chains — the differentiator — lead. A link points to the full audit in the Angular UI.

One consolidated comment is posted (not one per finding) to avoid the noise that makes developers mute security tools.

---

## 7. Cross-cutting concerns

### 7.1 Security posture

- Uploaded artifacts are treated as hostile-sensitive across their lifecycle; application source never leaves the runner.
- Secret pre-scan runs on the runner (Gitleaks CLI); a guaranteed backend ingress scan redacts findings text before any content reaches an LLM. Hardcoded secrets are reported as high-severity findings.
- The received bundle is deleted immediately after the audit by default; only opt-in reports are retained; encryption in transit and at rest; secrets in Azure Key Vault.
- Authentication and role-based permissions on all endpoints; per-tenant isolation so no tenant can reach another's data.
- Scanned material is never used to train any model.

### 7.2 Model-tier routing

Reasoning-heavy debate turns use the high tier (Claude Sonnet / GPT-4o); high-volume routine turns use the cheap tier (Claude Haiku / GPT-4o-mini). Cost per audit by tier is a tracked metric.

---

## 8. Sprint 0 exit criteria for this document

1. Layered architecture agreed, including the runner/backend split of the D2 model.
2. D2 bundle contents fixed: findings plus the graph-input artifacts required to reconstruct the cross-layer chain.
3. Component responsibilities fixed for runner-side workflow, frontend, backend, databases, external services, and AI components.
4. Communication mechanisms and the contract-driven seams documented.
5. Store boundary confirmed: relational + rule-mapping table (SQL Server) vs. vector knowledge (Qdrant) vs. benchmark corpus (separate, never embedded).
6. Consistency with the locked stack confirmed (Agent Framework 1.0, Angular, GitHub Actions, Qdrant, SQL Server, Azure OpenAI).
