# SentinelAI — Product Backlog

**Cross-layer exploit-path reasoning for CI/CD security auditing**

*Aligned to the D2 distributed-scan architecture · SDD-01 (System Design) · AID-01 (AI Design & Workflow) · DAD-01a (Database Design) · DAD-01b (API Design)*

---

## How to read this backlog

Each story carries an ID, title, story points (Fibonacci), priority (P0 blocker → P3 nice-to-have), dependencies, and Given/When/Then acceptance criteria. Stories are grouped by epic. The **critical path** runs E0 → E1 → E2 → E3 → E4 → E5: the agent skeleton and the knowledge corpus, then scanners and the resource graph, then retrieval and the debate.

**Locked stack (all stories assume this):** C#/.NET · ASP.NET Core · Microsoft Agent Framework 1.0 · GPT-4o via Azure OpenAI (prod) / Claude Sonnet + Haiku (demo) · Qdrant (offense/defense collections, dense + reserved sparse) · SQL Server · Semgrep + Roslyn/Security Code Scan (SAST) · OWASP Dependency-Check (SCA) · Trivy + Checkov (IaC) · Angular · GitHub Actions + Docker · Langfuse + Azure Application Insights.

**Delivery surface (D2):** a GitHub Actions workflow runs the scanners and graph-input extraction on the runner at pull-request time, then uploads a **bundle** (findings + infrastructure/manifest artifacts — never application source) to the backend. The backend normalizes, rule-maps, builds the resource graph, retrieves, runs the Red/Blue/Reporter debate, and returns a prioritized **draft** audit posted back as PR comments and rendered in an Angular UI.

**Repository structure (5 repos).** Work is organized across the GitHub org `Sentinel-AI-Sec`. Each story carries a `Repo:` field; epics map to repos as follows:

| Repo | Owns | Epics (primary) |
|------|------|-----------------|
| `sentinelai-action` | GitHub Actions composite: secret pre-scan, five scanners, graph-input collection, bundle packaging + upload, PR comment posting | E2 (runner side), parts of E10 |
| `sentinelai-backend` | ASP.NET Core: API/gateway, SQL schema, normalize, rule-map (SQL), resource graph, retrieval, agent debate, security, observability | E0, E2 (backend side), E3, E4, E5, E6, E7, E8, read API in E10, E11 |
| `sentinelai-knowledge` | Pipeline A offline knowledge ingestion → Qdrant corpus | E1 |
| `sentinelai-frontend` | Angular read UI over the report/graph endpoints | E10 (UI) |
| `sentinelai-fixtures` | Authored vulnerable test repo + benchmark corpus the action runs against | SEC-04, E9 |

Cross-repo stories (shared contracts, end-to-end integration) name every repo they span. The scanner-base container image is no longer needed (scanners now run on the caller's runner via `sentinelai-action`).

**Design invariants carried into acceptance criteria:**
- The scanned application source never reaches the backend; only findings + graph-input artifacts do.
- The same embedding model runs at index time and query time (fail loud, never silent fallback).
- The knowledge corpus, the `rule_mappings` SQL table, and the benchmark corpus are three separate stores; the scan target is never embedded.
- Every extractor uses one canonical node-ID scheme (`type:identifier`) or the graph splits into per-layer islands.
- Cross-layer edges carry a confidence tier (`certain` / `inferred` / `unresolved`); unresolved load-bearing joins are surfaced for human review, never silently dropped.
- The output is a draft audit for human review, not a verified verdict.

---

## Epic summary

| Epic | Theme | Stories | Points | Priority |
|------|-------|---------|--------|----------|
| **E0** | Foundations & agent skeleton | 5 | 26 | P0 |
| **E1** | Knowledge corpus — Pipeline A (offline ingest) | 5 | 26 | P0 |
| **E2** | Runner workflow, scanners & unified findings | 6 | 32 | P0 |
| **E3** | Resource graph & cross-layer chaining | 5 | 34 | P0 |
| **E4** | Retrieval — Pipeline B (findings → RAG) | 4 | 21 | P0 |
| **E5** | The debate (Red / Blue / Reporter) | 4 | 26 | P0 |
| **E6** | Provider abstraction & model-tier routing | 2 | 8 | P1 |
| **E7** | Security & compliance | 4 | 21 | P0 |
| **E8** | Observability | 2 | 8 | P1 |
| **E9** | Benchmarking & false-positive validation | 2 | 13 | P1 |
| **E10** | Delivery — CI/CD workflow, API & Angular UI | 5 | 26 | P1 |
| **E11** | Integration & end-to-end | 5 | 31 | P0 |
| | **Total** | **49** | **272** | |

---

## E0 — Foundations & agent skeleton

*De-risk the top technical risk — multi-agent orchestration and shared state — before anything else.*

### SEC-01 · Repository, solution scaffold & CI bootstrap
**Repo:** sentinelai-backend (+ all repos scaffolded) · **Points:** 3 · **Priority:** P0 · **Depends on:** —

**User story:** As a **developer on the team**, I want each repository scaffolded with CI that builds and tests on every push, so that every contribution starts from a green, reproducible baseline.

Initialize the five repositories in the `Sentinel-AI-Sec` org (`sentinelai-action`, `sentinelai-backend`, `sentinelai-knowledge`, `sentinelai-frontend`, `sentinelai-fixtures`). Scaffold the backend .NET solution (agent host, normalizer, graph, retrieval, API projects), the backend Docker image, and per-repo CI that builds and runs unit tests. No scanner-base image is needed (scanners run on the caller's runner via `sentinelai-action`).

- **Given** a fresh clone of each repo, **when** its CI runs, **then** it builds, unit tests execute, and (for the backend) a container image is produced.
- **Given** the backend solution structure, **when** a developer opens it, **then** projects map to the backend epics (Agents, Graph, Retrieval, Api, Normalizer).

### SEC-02 · Agent orchestration skeleton (Microsoft Agent Framework 1.0)
**Repo:** sentinelai-backend · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-01

**User story:** As a **backend engineer**, I want a working Red→Blue→Reporter agent loop with shared state and checkpointing, so that the project's highest-risk component is proven before any feature is built on top of it.

Stand up a sequential three-agent chain (Red → Blue → Reporter) using Agent Framework's group-chat workflow, with checkpointing and session state. Agents are stubs; the point is the loop, shared memory, and state transitions — the project's top risk.

- **Given** the orchestrator, **when** a run starts, **then** all three agents execute in sequence and share a common session state.
- **Given** a mid-run failure, **when** the run resumes from a checkpoint, **then** state is restored without re-running completed turns.
- **Given** a turn-cap, **when** the debate exceeds it, **then** the orchestrator terminates cleanly and the Reporter still produces output.

### SEC-03 · Canonical data contracts (Finding, Node, Edge, Chain, Report)
**Repo:** sentinelai-backend (shared contracts, consumed by sentinelai-action) · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-01

**User story:** As a **backend engineer**, I want one canonical set of data contracts and a single node-ID scheme shared across components, so that stages hand off cleanly and the graph never splits into per-layer islands.

Define the shared record types every component produces/consumes, with a single canonical node-ID scheme (`type:identifier`) used by every extractor, and the confidence tier on edges.

- **Given** any scanner or graph component, **when** it emits an entity, **then** it uses the canonical ID scheme and the shared Finding/Node/Edge contract.
- **Given** two extractors referencing the same resource, **when** they run, **then** their node IDs match exactly (no per-layer islands).
- **Given** an edge, **when** it is emitted, **then** it carries a `seam` and a `confidence` tier (certain / inferred / unresolved).

### SEC-04 · Demo fixture (authored three-layer vulnerable app)
**Repo:** sentinelai-fixtures · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-03

**User story:** As a **security engineer**, I want an authored three-layer vulnerable app fixture, so that I can demonstrate and regression-test the full dep→code→infra→bucket chain end to end.

Author the load-bearing fixture: a C# service with a vulnerable NuGet dependency (Newtonsoft.Json), an unsafe-deserialization weakness (CWE-502), a Dockerfile, and Terraform with an over-permissioned IAM role and a crown-jewel S3 bucket — with clean image-name and role join points so the full chain is demonstrable. Include `packages.lock.json` so SCA and the dep→code seam both resolve.

- **Given** the fixture, **when** all scanners run, **then** each layer (code/dep/infra) produces at least one real finding.
- **Given** the fixture, **when** the graph is built, **then** the dep→code→infra→bucket chain is reconstructable end-to-end.
- **Given** the fixture, **when** the lock file is present, **then** Dependency-Check produces non-empty SCA output.

### SEC-05 · SQL schema & migrations (relational + rule_mappings)
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-01

**User story:** As a **backend engineer**, I want the SQL schema and migrations for the relational store and the rule-mapping table, so that scan records, the resource graph, and rule lookups have a durable, queryable home.

Implement the SQL Server schema from DAD-01a: tenants, users, projects, scan_jobs, scan_bundles, findings, graph_nodes, graph_edges, chains, chain_hops, reports, citations, and the `rule_mappings` reference table. Enforce the canonical-ID unique constraint on (`scan_job_id`, `node_key`).

- **Given** the migrations, **when** applied, **then** every entity in DAD-01a exists with its columns and foreign keys.
- **Given** two nodes with the same `node_key` in one job, **when** inserted, **then** the unique constraint rejects the duplicate (island guard).
- **Given** `rule_mappings`, **when** queried, **then** it resolves a `(source_tool, check_id)` to a CWE by exact lookup with no vector operation.

---

## E1 — Knowledge corpus (Pipeline A)

*Offline, once. Turns canonical security knowledge into searchable Qdrant points.*

### SEC-06 · Source loaders (NVD, ATT&CK, CAPEC, OWASP, CVE2CAPEC)
**Repo:** sentinelai-knowledge · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-03

**User story:** As a **knowledge engineer**, I want loaders that turn each canonical source into the common RawEntry shape, so that heterogeneous security data can be ingested through one consistent pipeline.

Per-source loaders emitting the common RawEntry (text + payload) shape: NVD via bulk mirror, ATT&CK/CAPEC via STIX 2.1, OWASP via markdown, CVE2CAPEC for cross-links.

- **Given** each source's native format, **when** its loader runs, **then** it yields RawEntry objects with clean prose in `text` and identifiers in `payload`.
- **Given** the canonical sources, **when** loaded, **then** they are used (not third-party mirrors) for any citation-bearing corpus.

### SEC-07 · Preprocess, chunk & enrich
**Repo:** sentinelai-knowledge · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-06

**User story:** As a **knowledge engineer**, I want text cleaned, deduplicated, enriched, and chunked per source, so that the embedded corpus is high-quality prose with rich filterable metadata.

Clean CVSS/URL/markup from `text`, dedupe by natural_id, enrich CVEs with CVE2CAPEC cross-links, and chunk per source (atomic for CVE/technique; OWASP header-split ~500–800 tokens).

- **Given** raw entries, **when** preprocessed, **then** CVSS vectors/URLs are stripped from `text` but the description stays meaningful.
- **Given** a CVE with a CVE2CAPEC link, **when** enriched, **then** its payload carries cwe/capec/technique IDs.

### SEC-08 · Embedding (same-model invariant, dense + reserved sparse)
**Repo:** sentinelai-knowledge · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-07

**User story:** As a **knowledge engineer**, I want embedding to enforce the same-model invariant and fail loud on a missing model, so that index-time and query-time vectors are always comparable.

Embed `text` (never payload). Production: text-embedding-3-large via Azure. Demo: BGE-M3 run locally (dense + native sparse). The same embedder MUST run at index and query time.

- **Given** the ingest, **when** it embeds, **then** it fails loud (not silent fallback) if the intended model is unavailable.
- **Given** BGE-M3, **when** it embeds, **then** both dense and sparse vectors are produced and stored.
- **Given** index and query time, **when** both embed, **then** they use the identical model.

### SEC-09 · Qdrant collections, routing & upsert
**Repo:** sentinelai-knowledge · **Points:** 3 · **Priority:** P0 · **Depends on:** SEC-08

**User story:** As a **knowledge engineer**, I want chunks routed and upserted into the offense/defense Qdrant collections with retry-and-backoff, so that the corpus is correctly organized and reliably written at cloud scale.

Two collections (offense/defense) with dense + reserved sparse configs and payload indexes on filter fields. Route each chunk by content (ATT&CK/CAPEC→offense, OWASP→defense, CVE→both). Batched, checkpointed upsert with retry-and-backoff.

- **Given** a chunk, **when** upserted, **then** it lands in the correct collection(s) per its routing.
- **Given** an interrupted upsert, **when** resumed, **then** it skips already-written points (checkpoint) and uses small batches with backoff.
- **Given** the collections, **when** queried by payload filter, **then** exact CVE/CWE/severity lookups return without vector search.

### SEC-10 · Corpus validation harness
**Repo:** sentinelai-knowledge · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-09

**User story:** As a **knowledge engineer**, I want a validation harness for the corpus, so that I can trust source coverage, routing, and retrieval quality before any agent depends on it.

Validate the corpus before any agent exists: source composition per collection, routing correctness, exact-filter smoke tests, and dense-vs-hybrid recall on known targets.

- **Given** the ingested corpus, **when** validated, **then** every source is present and reachable via source-filtered query.
- **Given** an identifier-heavy query, **when** run, **then** hybrid recall meets or exceeds dense-only.
- **Given** validation failure, **when** reported, **then** it distinguishes an ingest gap (source missing) from a ranking issue (source outnumbered).

---

## E2 — Runner workflow, scanners & unified findings

*The runner observes the code and ships a bundle; the backend normalizes it into one findings set.*

### SEC-11 · GitHub Actions composite workflow (observe + package + upload)
**Repo:** sentinelai-action · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-04

**User story:** As a **developer opening a pull request**, I want a composite GitHub Action that scans my code and uploads only findings and graph inputs, so that my application source never leaves my runner while I still get an audit.

Author the runner-side composite Action: checkout → Gitleaks CLI secret pre-scan → run five scanners → `terraform init` + `terraform graph` → collect graph-input files → package the bundle → `POST /v1/scans` with a scoped machine token → poll → post PR comment. Pinned tool versions.

- **Given** a pull request, **when** the workflow runs, **then** it produces SARIF/JSON from all five scanners plus the `terraform graph` DOT and collects the graph-input files.
- **Given** the bundle, **when** uploaded, **then** it contains findings + infrastructure/manifest artifacts and **no** application source (`.cs`).
- **Given** the job completes, **when** the workflow polls, **then** it posts the adjudicated audit as a PR comment.

### SEC-12 · Scanner runners on the runner (Semgrep, Roslyn, Dependency-Check, Trivy, Checkov)
**Repo:** sentinelai-action · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-11

**User story:** As a **developer opening a pull request**, I want all five scanners to run on the runner and degrade gracefully, so that each layer is analyzed even if one tool is unavailable.

Wire each scanner into the workflow. Roslyn (Security Code Scan) carries the C# code layer since Semgrep's C# coverage is thin; Dependency-Check handles NuGet (needs the lock file); Trivy + Checkov handle IaC/Docker. Each is fault-tolerant.

- **Given** the fixture, **when** the scanners run, **then** Roslyn produces the CWE-502 code finding Semgrep misses.
- **Given** a scanner is unavailable, **when** the workflow runs, **then** it degrades gracefully (fewer findings, no crash) and records which tools ran.

### SEC-13 · Bundle ingest endpoint & provenance
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-05, SEC-11

**User story:** As a **backend engineer**, I want a bundle-ingest endpoint that records provenance and rejects application source, so that the backend accepts work safely and can prove what it received.

Implement `POST /v1/scans` as a multipart bundle upload; persist bundle provenance to `scan_bundles` (runner secret-scan status, artifact manifest, scanner versions). Return `202` + poll URL.

- **Given** a bundle upload, **when** accepted, **then** a scan_job and a scan_bundle row are created and `202` + poll URL returned.
- **Given** an upload containing a `.cs` source file, **when** validated, **then** it is rejected or stripped (source must not enter the backend).
- **Given** the provenance, **when** stored, **then** the artifact manifest and scanner versions are recorded for trust/reproducibility.

### SEC-14 · SARIF/JSON normalization (v1 + v2 tolerant)
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-13, SEC-03

**User story:** As a **backend engineer**, I want SARIF/JSON from every scanner normalized into one Finding shape, so that downstream stages consume a single consistent representation regardless of tool.

Parse each scanner into the common Finding: normalize severity scales, tag source_tool + layer, tolerate SARIF v1 and v2. Parse Dependency-Check native JSON to preserve CVE/CWE keys SARIF export drops.

- **Given** SARIF from any scanner in v1 or v2, **when** normalized, **then** it yields Findings without crashing on shape differences.
- **Given** a dependency finding, **when** normalized, **then** its CVE is preserved from native JSON (not lossy SARIF).
- **Given** a Trivy finding, **when** it carries a CVE/package, **then** it is tagged `dep`, else `infra`.

### SEC-15 · Rule-mapping resolution (SQL exact lookup)
**Repo:** sentinelai-backend · **Points:** 3 · **Priority:** P0 · **Depends on:** SEC-14, SEC-05

**User story:** As a **backend engineer**, I want each finding's check_id resolved to a CWE via an exact SQL lookup, so that findings lacking a CWE can still be matched to knowledge without any similarity guesswork.

Every finding passes through the `rule_mappings` SQL table first, resolving a tool's check_id to a CWE when the SARIF text lacks one (e.g. SCS0028→CWE-502, CKV_AWS_*→CWE-284). This is the mandated first retrieval step.

- **Given** a finding whose SARIF carries no CWE, **when** it passes the table, **then** its CWE is resolved by exact `(source_tool, check_id)` lookup.
- **Given** the table, **when** consulted, **then** it is a dictionary lookup — never a similarity/embedding operation.

### SEC-16 · Unify into one findings set
**Repo:** sentinelai-backend · **Points:** 3 · **Priority:** P0 · **Depends on:** SEC-14, SEC-15

**User story:** As a **backend engineer**, I want all findings merged into one deduplicated, canonically-referenced set, so that the graph and retrieval both build on the same clean input.

Merge all normalized, rule-mapped findings into one flat, deduplicated set tagged by layer and severity, with linking keys extracted — the input to both the graph and retrieval.

- **Given** findings from five scanners, **when** unified, **then** they form one deduplicated set with consistent severity and layer tags.
- **Given** the unified set, **when** produced, **then** each finding carries a `node_ref` in the canonical ID scheme.

---

## E3 — Resource graph & cross-layer chaining

*The connective tissue. Primary technical risk alongside orchestration: the cross-layer seams and their confidence.*

### SEC-17 · Infra spine from `terraform graph` DOT (attack-oriented)
**Repo:** sentinelai-backend · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-16

**User story:** As a **backend engineer**, I want the Terraform infra spine parsed and re-oriented to attack direction, so that exploit chains can actually form instead of collapsing to zero.

Parse the `terraform graph` DOT into nodes + edges, re-orient edges to attack direction, and persist to `graph_nodes` / `graph_edges` (seam = infra-spine, confidence = certain). HCL parse is the fallback.

- **Given** the fixture's DOT, **when** parsed, **then** resources become nodes and references become edges re-oriented to attack direction (`oriented_attack_dir` set).
- **Given** wrong orientation, **when** tested, **then** the regression catches it (attack-direction guard — the zero-chains failure mode).

### SEC-18 · Cross-layer seams (dep→code, role→resource) — certain edges
**Repo:** sentinelai-backend · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-17

**User story:** As a **backend engineer**, I want the deterministic dep→code and role→resource seams built as certain edges, so that the reliable cross-layer links are always present in the graph.

Build the deterministic seams: dep→code from csproj + packages.lock.json (join key = package name), and role→resource by parsing all Terraform IAM policy patterns (inline, `aws_iam_role_policy`, managed-policy attachment, wildcards). Both are `certain`.

- **Given** the lock file, **when** parsed, **then** each package (direct and transitive) becomes a node with a `used-by` edge to the code node (confidence certain).
- **Given** the Terraform IAM statements, **when** parsed, **then** a `can-access` edge connects the role to each resource ARN it grants (confidence certain).
- **Given** a wildcard (`s3:*` / `Resource: "*"`), **when** parsed, **then** it widens the edge rather than dropping it.

### SEC-19 · Code→infra seam with confidence scoring (image-name join)
**Repo:** sentinelai-backend · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-17

**User story:** As a **backend engineer**, I want the fragile code→infra image-name join scored by confidence instead of dropped, so that unconfirmable links are surfaced for review rather than silently lost.

Build the fragile code→infra seam: extract the Dockerfile image and the Terraform image field, resolve variables/locals, normalize (strip registry prefix + tag), and emit a confidence-scored edge (`inferred` on normalized match, `unresolved` when unconfirmable). Optionally raise confidence with a second signal or an annotation file.

- **Given** Dockerfile and Terraform image references, **when** resolved and normalized, **then** a matching repo name yields an `inferred` edge.
- **Given** no confident match but both sides reference an image, **when** evaluated, **then** an `unresolved` edge is recorded (not dropped).
- **Given** the fixture's clean joins, **when** built, **then** the code→infra edge resolves and the full chain connects.

### SEC-20 · Exploit-chain traversal (bounded candidate generation)
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-18, SEC-19

**User story:** As a **security engineer**, I want bounded traversal that seeds from hot nodes and propagates the weakest edge confidence, so that candidate chains are realistic, reproducible, and honestly rated.

Seed from high-severity 'hot' nodes; traverse real edges up to 3–4 hops; keep paths crossing ≥2 layers; propagate the weakest edge confidence to `chains.min_confidence`. Deterministic candidates the Red agent later reasons within.

- **Given** a hot seed node, **when** traversed, **then** candidate chains are bounded at the hop cap and contain only real edges.
- **Given** the fixture, **when** traversed, **then** the dep→code→infra→crown-jewel chain appears among candidates.
- **Given** a chain crossing an inferred/unresolved edge, **when** produced, **then** `min_confidence` reflects the weakest edge.

### SEC-21 · Query-string construction & attack-graph handoff
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P1 · **Depends on:** SEC-16, SEC-20

**User story:** As a **backend engineer**, I want per-finding query strings and a clean attack-graph handoff, so that the Red agent receives well-formed candidates without ever seeing raw code or SARIF.

Build a short natural-language query per finding (message + IDs + resource type — not raw code or SARIF). Define the handoff of ordered candidate paths to the Red agent, each carrying nodes, evidence, and empty technique/evidence slots.

- **Given** a finding, **when** a query is built, **then** it contains the finding's identifiers plus a concise description and excludes scanner boilerplate.
- **Given** candidate chains, **when** handed off, **then** each carries nodes, findings, confidence, and empty slots the Red agent fills.

---

## E4 — Retrieval (Pipeline B)

*Findings → RAG. Three modes in one flow.*

### SEC-22 · Retrieval decision tree (exact filter → semantic → hybrid)
**Repo:** sentinelai-backend · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-09, SEC-15, SEC-21

**User story:** As a **security engineer**, I want retrieval to pick exact, semantic, or hybrid per finding with mandatory source-filtering, so that each finding is grounded in the most relevant knowledge without CVEs drowning techniques.

Three modes in one flow: exact payload filter for findings with a clean CVE/CWE; source-filtered semantic (offense=ATT&CK/CAPEC, defense=OWASP) for infra findings without IDs; hybrid dense+sparse where available. Source-filtering the semantic path is mandatory.

- **Given** a dependency finding with a CVE, **when** retrieved, **then** the exact payload filter returns the CVE record first.
- **Given** an infra finding without an ID, **when** retrieved, **then** the semantic path is source-filtered to techniques/guidance, not flooded by CVEs.
- **Given** identical index and query embedders, **when** hybrid runs, **then** dense+sparse are fused via RRF.

### SEC-23 · Offense/defense split retrieval
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-22

**User story:** As a **security engineer**, I want Red to query offense and Blue to query defense within one shared corpus, so that attacker and defender reasoning are each grounded in the right knowledge regardless of which tool found the issue.

Red queries offense (ATT&CK/CAPEC/exploitation); Blue queries defense (OWASP/mitigations). Each finding resolves into the same shared corpus, split only by attacker/defender role — never by tool.

- **Given** a finding, **when** the Red agent retrieves, **then** it hits the offense collection; the Blue agent hits defense.
- **Given** any tool's finding, **when** retrieved, **then** it resolves into the shared corpus (no per-tool index).

### SEC-24 · DEPRECATED / low-quality filtering
**Repo:** sentinelai-backend · **Points:** 3 · **Priority:** P2 · **Depends on:** SEC-22

**User story:** As a **security engineer**, I want DEPRECATED entries filtered from results, so that thin, obsolete records don't score spuriously high and mislead the debate.

Drop DEPRECATED CAPEC/ATT&CK entries (thin text, spurious high scores) via over-fetch-and-filter.

- **Given** retrieval results, **when** filtered, **then** DEPRECATED entries are removed and k results remain.

### SEC-25 · Retrieval evaluation (grounding coverage)
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P1 · **Depends on:** SEC-22, SEC-23

**User story:** As a **security engineer**, I want grounding coverage and per-mode fire rates measured on the fixture, so that I can prove every finding is backed by retrieved knowledge and all three modes work.

Measure grounding coverage (% findings with a retrieved chunk) and per-mode fire rates on the fixture; assert all three modes exercise.

- **Given** the fixture findings, **when** retrieval runs, **then** grounding coverage and per-mode counts are reported and all three modes fire.

---

## E5 — The debate (Red / Blue / Reporter)

*The adversarial core that produces grounded, low-false-positive findings.*

### SEC-26 · Red Team agent (assert exploit chains)
**Repo:** sentinelai-backend · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-21, SEC-23

**User story:** As a **security engineer**, I want the Red agent to assert ordered, cited exploit chains within bounded candidates, so that attack paths are grounded and reproducible rather than hallucinated.

Given candidate chains + retrieved offense knowledge, the Red agent asserts ordered cross-layer exploit paths (node, technique, evidence), reasoning within bounded candidates rather than over the free graph. It reasons over edge confidence.

- **Given** candidate chains and offense knowledge, **when** the Red agent runs, **then** it outputs ordered paths with a cited technique per hop.
- **Given** an unresolved edge in a candidate, **when** the Red agent asserts, **then** the chain is marked for human confirmation rather than presented as certain.
- **Given** the hop cap and turn-cap, **when** reasoning, **then** it terminates within bounds.

### SEC-27 · Blue Team agent (validate each link)
**Repo:** sentinelai-backend · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-26, SEC-23

**User story:** As a **security engineer**, I want the Blue agent to validate each link against real config, so that false positives are cut and only chains whose every link holds survive.

The Blue agent tests each link of an asserted chain against the real configuration and defense knowledge; breaking one link breaks the chain. Inferred edges get extra scrutiny.

- **Given** an asserted chain, **when** the Blue agent validates, **then** each link is checked against real config and a broken link invalidates the chain.
- **Given** an inferred image-name edge, **when** validated, **then** it receives extra scrutiny before the hop is accepted.
- **Given** a validated chain, **when** reported, **then** each surviving link cites its supporting evidence.

### SEC-28 · Reporter agent (adjudicate → draft audit)
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-27

**User story:** As a **security reviewer**, I want the Reporter to adjudicate the debate into a prioritized, cited draft audit that flags unverified joins, so that I get a trustworthy, honestly-framed result to review rather than a false verdict.

Adjudicate the debate into a structured, cited **draft** audit (not a verdict), prioritized by severity and exploitability, carrying per-chain `min_confidence`, with reserved fields for the future feedback loop.

- **Given** the debate, **when** the Reporter runs, **then** it emits a prioritized draft audit where every assertion cites its retrieved chunk.
- **Given** a chain with an unresolved join, **when** reported, **then** it is surfaced as "potential chain, unverified join," not a confirmed finding.
- **Given** the output schema, **when** produced, **then** it reserves fields for later candidate-detection/promotion without retrofit.

### SEC-29 · Debate orchestration (turn-cap, convergence, tiering)
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-02, SEC-26, SEC-27, SEC-28

**User story:** As a **backend engineer**, I want debate orchestration with a turn-cap, convergence detection, and tier routing, so that every debate terminates cleanly at a controlled cost.

Wire the three agents into the group-chat with a turn-cap, convergence detection, and per-turn model-tier routing.

- **Given** a debate, **when** it converges or hits the cap, **then** it terminates and the Reporter still adjudicates.
- **Given** a reasoning-heavy turn, **when** routed, **then** it uses the high tier; routine turns use the cheap tier.

---

## E6 — Provider abstraction & model-tier routing

### SEC-30 · LLM provider abstraction
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P1 · **Depends on:** SEC-02

**User story:** As a **backend engineer**, I want the LLM behind a connector abstraction, so that switching providers between production and demo is a config change proven by test, not a rewrite.

The LLM sits behind Agent Framework's connector abstraction: production GPT-4o (Azure), demo Claude Sonnet/Haiku, switchable by config. Proven by test.

- **Given** a config change, **when** the provider switches, **then** the debate runs unchanged on the new provider (integration test asserts this).

### SEC-31 · Model-tier routing & cost tracking
**Repo:** sentinelai-backend · **Points:** 3 · **Priority:** P1 · **Depends on:** SEC-30, SEC-29

**User story:** As a **team lead**, I want reasoning and routine turns routed to the right model tier with cost tracked per audit, so that quality and spend are both under control.

Route reasoning turns to the high tier and routine turns to the cheap tier; track cost per audit by tier.

- **Given** a completed audit, **when** measured, **then** cost per audit by model tier is recorded.

---

## E7 — Security & compliance

*Uploaded artifacts are treated as hostile-sensitive; application source never leaves the runner.*

### SEC-32 · Authentication, RBAC & tenant isolation
**Repo:** sentinelai-backend · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-05

**User story:** As a **security engineer**, I want authentication, RBAC, and tenant isolation on every endpoint, so that no tenant can ever reach another tenant's bundle, findings, or reports.

Auth + role-based permissions on all endpoints; per-tenant separation so no tenant can reach another's bundle, findings, or reports. The runner uses a scoped machine token (scan:write).

- **Given** any endpoint, **when** accessed, **then** it requires authentication and enforces role permissions.
- **Given** two tenants, **when** either queries, **then** neither can access the other's data.
- **Given** the runner's machine token, **when** used, **then** it is limited to scan:write and its own tenant.

### SEC-33 · Ingress secret scan & redaction (backend gate)
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-13

**User story:** As a **security engineer**, I want a guaranteed backend redaction gate before any LLM call, so that secrets are stripped from content regardless of whether the runner pre-scan ran.

A guaranteed backend gate that scans and redacts findings text (and infra artifacts) before any content reaches an LLM — independent of the runner-side Gitleaks pre-scan. Hardcoded secrets reported as high-severity findings.

- **Given** received bundle content, **when** it enters the pipeline, **then** secrets are detected and redacted before any LLM sees content.
- **Given** a hardcoded secret in an infra artifact, **when** found, **then** it is reported as a high-severity finding.
- **Given** redaction, **when** applied, **then** `ingress_redaction_applied` and per-finding `redacted` flags are set.

### SEC-34 · Sandboxed backend processing
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-13

**User story:** As a **security engineer**, I want each job processed in a sandbox with restricted egress, so that received artifacts are contained and never used to train any model.

Process each job in an ephemeral backend context with egress restricted to required LLM/RAG endpoints; received artifacts never used to train any model.

- **Given** a scan job, **when** it runs, **then** it executes with egress restricted to allowed endpoints.
- **Given** the D2 model, **when** a job runs, **then** no application source is present to process (bundle only).

### SEC-35 · Retention, encryption & purge
**Repo:** sentinelai-backend · **Points:** 3 · **Priority:** P0 · **Depends on:** SEC-32

**User story:** As a **customer administrator**, I want bundles purged after audit and full purge on request, so that my data is retained only with explicit opt-in and I can prove deletion.

Received bundle deleted immediately after audit by default; only opt-in reports retained; on-demand and on-account-deletion purge; encryption in transit and at rest; secrets in Azure Key Vault.

- **Given** a completed audit, **when** retention runs, **then** the bundle is purged (`bundle_purged` set) and only opt-in reports remain.
- **Given** account deletion, **when** requested, **then** all associated data is purged.

---

## E8 — Observability

### SEC-36 · LLM tracing (Langfuse)
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P1 · **Depends on:** SEC-29

**User story:** As a **backend engineer**, I want per-agent LLM tracing with token, cost, and latency capture, so that I can replay and debug any debate.

Per-agent prompts/responses, token+cost per audit, per-turn latency, full debate replay via Langfuse (Agent Framework emits OpenTelemetry natively).

- **Given** an audit, **when** traced, **then** each agent turn, its tokens/cost, and latency are captured and the debate is replayable.

### SEC-37 · System health (Azure Application Insights)
**Repo:** sentinelai-backend · **Points:** 3 · **Priority:** P1 · **Depends on:** SEC-01

**User story:** As a **backend engineer**, I want API latency, step duration, and error rates visible, so that I can monitor system health and catch regressions.

API latency, pipeline-step duration, error rates via App Insights.

- **Given** the running system, **when** monitored, **then** API latency, step duration, and error rates are visible.

---

## E9 — Benchmarking & false-positive validation

### SEC-38 · Ground-truth benchmark corpus
**Repo:** sentinelai-fixtures (+ sentinelai-backend harness) · **Points:** 8 · **Priority:** P1 · **Depends on:** SEC-04

**User story:** As a **security engineer**, I want a labeled benchmark corpus kept separate from RAG, so that I can measure detection quality on ground truth without polluting the knowledge base.

Assemble deliberately-vulnerable fixtures (TerraGoat, Kubernetes-Goat, CfnGoat) + a hand-labeled C# set (honest gap: no C# OWASP Benchmark exists, so this is small and Roslyn-carried), plus clean samples. Kept separate from RAG — never embedded.

- **Given** the benchmark corpus, **when** assembled, **then** it is a separate store, never embedded into RAG.
- **Given** the C# gap, **when** documented, **then** the limited C# ground truth is stated, not papered over.

### SEC-39 · Precision/recall vs SonarQube & Snyk
**Repo:** sentinelai-backend (+ sentinelai-fixtures corpus) · **Points:** 5 · **Priority:** P1 · **Depends on:** SEC-38, SEC-28

**User story:** As a **team lead**, I want precision and recall reported per category against SonarQube and Snyk, so that the false-positive claim is validated with evidence rather than asserted.

Run all tools over the identical corpus; classify every finding TP/FP/FN; report precision AND recall per category — precision as the false-positive headline, recall as the guardrail.

- **Given** the shared corpus, **when** all tools run, **then** precision and recall are reported per category for SentinelAI vs SonarQube and Snyk.

---

## E10 — Delivery (CI/CD workflow, API & Angular UI)

### SEC-40 · Read API (findings, graph, chains, report, bundle)
**Repo:** sentinelai-backend · **Points:** 5 · **Priority:** P1 · **Depends on:** SEC-28, SEC-32

**User story:** As a **frontend engineer**, I want tenant-scoped read endpoints for findings, graph, chains, and report, so that the UI can render the full audit with its confidence tiers.

Implement the read endpoints from DAD-01b: `/v1/scans/{id}`, `/findings`, `/graph`, `/chains`, `/bundle`, `/v1/reports/{id}` — tenant-scoped, cursor-paginated, RFC 7807 errors.

- **Given** a completed scan, **when** the graph endpoint is called, **then** it returns nodes + edges with confidence tiers.
- **Given** a report, **when** fetched, **then** it carries `framing: draft_audit` and per-chain `min_confidence`.

### SEC-41 · PR comment posting (acknowledgment + draft audit)
**Repo:** sentinelai-action · **Points:** 5 · **Priority:** P1 · **Depends on:** SEC-11, SEC-28

**User story:** As a **developer opening a pull request**, I want one consolidated PR comment that leads with cited chains, so that I can see and act on real exploit paths without noise.

Post an acknowledgment on receipt, then the adjudicated audit — chains first, per-hop cited evidence, Blue-validated flags, unresolved-join marking, un-chained findings demoted. One consolidated comment, edited in place.

- **Given** a bundle accepted, **when** acknowledged, **then** a single comment reports job id and per-layer counts.
- **Given** a completed audit, **when** posted, **then** chains lead with cited evidence and the draft-audit framing; findings that did not chain are demoted.

### SEC-42 · Angular report UI
**Repo:** sentinelai-frontend · **Points:** 5 · **Priority:** P1 · **Depends on:** SEC-40

**User story:** As a **security reviewer**, I want an Angular UI that shows each chain, its hops, cited evidence, and confidence, so that I can review the draft audit in depth beyond the PR comment.

Angular UI presenting the adjudicated audit: chains, per-hop evidence, severity, confidence tier, and the draft-audit framing, reading the graph and report endpoints.

- **Given** a completed audit, **when** viewed, **then** the UI shows each exploit chain, its hops, cited evidence, and its confidence tier.

### SEC-43 · GitHub Actions delivery integration
**Repo:** sentinelai-action (+ sentinelai-backend) · **Points:** 3 · **Priority:** P1 · **Depends on:** SEC-11, SEC-13

**User story:** As a **developer opening a pull request**, I want the action wired end-to-end to the backend with pinned versions, so that a PR reliably triggers a scan and gets annotated.

Wire the composite Action end-to-end against the backend: authenticate with the machine token, upload the bundle, poll, and annotate the PR. Pin versions for reproducibility.

- **Given** a pull request, **when** the workflow runs, **then** it uploads the bundle, polls to completion, and annotates the PR.

### SEC-44 · End-to-end demo run (full three-layer chain)
**Repo:** sentinelai-action + sentinelai-backend + sentinelai-frontend + sentinelai-fixtures · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-43, SEC-42

**User story:** As a **stakeholder**, I want the full three-layer chain demonstrated end-to-end on the fixture, so that the system's core value is proven working in one acceptance run.

The acceptance demo: on the authored fixture, demonstrate the full dep→code→infra→crown-jewel chain end-to-end through the workflow and backend, adjudicated, posted, and displayed.

- **Given** the fixture PR, **when** the workflow runs end-to-end, **then** the full three-layer chain is scanned, graphed, retrieved, debated, adjudicated, posted, and displayed.

---

## E11 — Integration & end-to-end

*The seams between components — where the real risk lives. A thin slice early, then stage-to-stage wiring.*

### SEC-45 · End-to-end thin slice (walking skeleton)
**Repo:** sentinelai-backend · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-02, SEC-09, SEC-16

**User story:** As a **backend engineer**, I want one seeded finding to flow through every stage early, so that the integration seams are proven before any stage is built out in depth.

The earliest end-to-end proof: one seeded finding flows through every stage — normalize → graph node → retrieve one chunk → stub debate → stub report — on the fixture. Stages are minimal stubs; the point is the wiring exists before any stage is deep.

- **Given** a single seeded finding, **when** the thin slice runs, **then** it passes through normalize, graph, retrieval, debate, and report without a broken seam.
- **Given** the slice, **when** a stage is later deepened, **then** its boundary contract is already fixed and tested.

### SEC-46 · Scan-time orchestration (Pipeline B end-to-end)
**Repo:** sentinelai-backend · **Points:** 8 · **Priority:** P0 · **Depends on:** SEC-16, SEC-19, SEC-22, SEC-29

**User story:** As a **backend engineer**, I want the full scan-time pipeline orchestrated end-to-end, so that a bundle produces an adjudicated draft audit while the scan target is never embedded.

Wire the full backend scan-time flow as one orchestrated pipeline: ingress redaction → normalize → rule-map → graph build → candidate chains → retrieval → debate → report. Enforces the Pipeline A/B separation.

- **Given** a bundle at PR time, **when** Pipeline B runs, **then** it executes all stages in order and produces an adjudicated draft audit.
- **Given** the run, **when** it processes the bundle, **then** the scan target is never embedded into the knowledge corpus.

### SEC-47 · Stage contracts & handoff tests
**Repo:** sentinelai-backend (+ sentinelai-action handoff) · **Points:** 5 · **Priority:** P0 · **Depends on:** SEC-03, SEC-45

**User story:** As a **backend engineer**, I want contract tests on every stage handoff, so that schema or canonical-ID drift fails loudly instead of silently splitting the graph.

Integration tests for each stage boundary: findings→graph attachment, graph→retrieval query construction, retrieval→Red, Red→Blue, Blue→Reporter, Reporter→PR. Each handoff fails loudly on schema/ID drift.

- **Given** each stage boundary, **when** a handoff test runs, **then** it verifies the contract (fields present, IDs consistent) and fails on drift.
- **Given** a canonical-ID mismatch, **when** tests run, **then** the per-layer island split is caught.

### SEC-48 · Pipeline A ↔ B integration (corpus freshness & query parity)
**Repo:** sentinelai-backend (+ sentinelai-knowledge) · **Points:** 5 · **Priority:** P1 · **Depends on:** SEC-10, SEC-22

**User story:** As a **backend engineer**, I want the offline corpus wired to scan-time retrieval with the same-model invariant enforced at the boundary, so that every audit retrieves against a known, matching corpus version.

Wire the offline corpus (A) to scan-time retrieval (B): assert the same embedding model at index and query time, corpus availability checks before a scan, and a corpus-version stamp on each audit.

- **Given** a scan, **when** retrieval runs, **then** it uses the identical embedder that built the corpus (same-model invariant enforced at the boundary).
- **Given** an audit, **when** produced, **then** it records the corpus version it retrieved against.

### SEC-49 · Full-flow regression harness (fixture golden path)
**Repo:** sentinelai-backend (+ sentinelai-fixtures) · **Points:** 5 · **Priority:** P1 · **Depends on:** SEC-46, SEC-47

**User story:** As a **security engineer**, I want a repeatable regression harness over the fixture, so that the flagship chain, retrieval modes, and edge orientation stay correct as stages deepen.

A repeatable regression over the authored fixture asserting the whole flow: finding counts per layer, the flagship chain reconstructs, all three retrieval modes fire, edge orientation is correct, and the draft audit contains the expected cited path.

- **Given** the fixture, **when** the harness runs, **then** it asserts the three-layer chain, per-mode retrieval, attack-direction orientation, and the expected cited path.
- **Given** a regression, **when** any assertion fails, **then** the harness pinpoints which stage broke.

---

## Critical path & sequencing

**Foundations first (de-risk):** SEC-01 → SEC-02 (orchestration skeleton, the top risk) → SEC-03 → SEC-04 → SEC-05. Parallel: SEC-06→SEC-09 (corpus ingest). SEC-45 (thin slice) starts as soon as SEC-16 lands.

**Wire the core:** SEC-10 (corpus validation); SEC-11→SEC-16 (runner workflow, scanners, normalize, rule-map, unify); SEC-17→SEC-21 (graph + seams + traversal); SEC-22→SEC-23 (retrieval); SEC-47 (stage contracts).

**The debate:** SEC-26→SEC-29 (Red/Blue/Reporter + orchestration); SEC-46 (scan-time end-to-end); SEC-48 (A↔B integration); SEC-30→SEC-31 (provider/tiering); E7 security in parallel.

**Harden + demo:** SEC-36→SEC-39 (observability, benchmarking); SEC-49 (regression); SEC-40→SEC-44 (API, PR comments, UI, delivery, demo). No new features once hardening begins.

## Honest open questions (carried from design)

- **Cross-layer seam fragility (SEC-19)** is the primary graph risk — image-name and role joins are convention-dependent; the confidence tier surfaces unresolved joins rather than hiding them.
- **Client-produced findings** are trusted under D2; the pinned composite Action and SARIF validation are the mitigations, stated as a limitation.
- **C# ground-truth gap (SEC-38)** limits how much benchmarking can lean on pre-labeled corpora; the C# set is small and hand-labeled.
- **Rule-mapping coverage (SEC-15)** is illustrative for the current build; broader coverage needs generation from full tool rule catalogs.
- **Self-improving RAG feedback loop** is deferred; the base build only reserves the fields (SEC-28), and human approval of any promotion is non-negotiable.
- **Sparse/hybrid retrieval** ships enabled where the local embedder provides it; the reserved sparse field lets production add it without re-indexing.

---

*SentinelAI Product Backlog · 49 stories · 272 points · 12 epics · 5 repositories (`Sentinel-AI-Sec` org) · aligned to the D2 architecture and the four Sprint 0 design documents.*
