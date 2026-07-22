
## `tenants`
**Why:** SentinelAI is multi-tenant. Every piece of operational data must belong to exactly one organization, or one customer can read another's vulnerabilities — the worst possible leak for a security product.

| Column | Why it exists |
|---|---|
| `id` | The tenant boundary key. Every query in the system ultimately filters on it. |
| `name` | Display name of the organization. |
| `plan_tier` | Entitlement — controls scan quotas, model-tier access, retention length. |
| `created_at` | Audit trail; billing anchor. |

---

## `users`
**Why:** RBAC needs an identity to attach a role to, and audits need to record *who* triggered a scan.

| Column | Why it exists |
|---|---|
| `id` | Identity key. |
| `tenant_id` | Scopes the user; a user exists only inside one org. |
| `email` | Login identity, unique per tenant (same email could exist in two orgs). |
| `role` | admin / analyst / viewer — the RBAC enforcement point on every endpoint. |
| `created_at` | Audit trail. |

---

## `projects`
**Why:** A scan is always *of a repository*. Without this table, scan jobs would have no stable parent and you couldn't show "history for this repo" or hold per-repo config.

| Column | Why it exists |
|---|---|
| `id` | Project key. |
| `tenant_id` | Ownership. |
| `repo_url` | Which repository this is. |
| `default_branch` | Baseline for comparison / what a PR is merging into. |
| `github_installation_id` | The GitHub App install token needed to post PR comments back. Nullable because a project may be registered before the App is installed. |

---

## `scan_jobs`
**Why:** The central operational record. Everything produced by one run — findings, nodes, edges, chains, report — hangs off a single job so the whole audit is retrievable, replayable, and purgeable as one unit.

| Column | Why it exists |
|---|---|
| `id` | The spine of the whole schema. |
| `project_id` | Which repo. |
| `triggered_by` | Which human. Nullable because CI runs on a machine token with no user behind them. |
| `pr_ref` | Which pull request — the delivery surface is PR-time. |
| `commit_sha` | Exact code state scanned. Without it a finding can't be reproduced. |
| `status` | queued/running/completed/failed — drives the async 202 + poll pattern. |
| `corpus_version` | Which knowledge snapshot the retrieval ran against. This is the same-model / corpus-parity invariant made auditable: a finding cited last month may cite a chunk that no longer exists. |
| `bundle_purged` | Proof the retention guarantee actually executed. Renamed from `code_purged` because under D2 no source ever arrives — only the bundle. |
| `started_at` / `completed_at` | Duration metrics, SLA, ordering. |

---

## `scan_bundles`
**Why this table exists at all:** D2 flipped the trust model. The scanners run on the customer's GitHub Actions runner, and the backend receives a *bundle* of outputs — not the source. That means the backend is now trusting artifacts it did not produce. This table records exactly what arrived and under what conditions, so the trust assumption is documented rather than implicit.

| Column | Why it exists |
|---|---|
| `id` | Key. |
| `scan_job_id` | 1:1 with the job. |
| `runner_secret_scan` | Did Gitleaks run on the runner, before anything left the customer's environment? `skipped`/`unknown` is itself important — it tells you the ingress redaction is the only line of defense. |
| `ingress_redaction_applied` | Proof the backend redacted before any LLM call. This is the auditable form of the "no secret reaches the model" promise. |
| `artifact_manifest` | JSON list of what was actually received (dot graph, tf source, dockerfile, manifests, SARIF). Explains missing findings: if no Dockerfile arrived, the code→infra seam couldn't be built. |
| `scanner_versions` | Tool → version map. A finding count changes when Trivy upgrades; without this the change is inexplicable. |
| `received_at` | Ingress timestamp. |

---

## `findings`
**Why:** The normalized output of five heterogeneous scanners. Everything downstream (graph decoration, RAG query, chain hops) consumes this one shape, so the tool differences must be flattened exactly once, here.

| Column | Why it exists |
|---|---|
| `id` | Key. |
| `scan_job_id` | Scope + purge unit. |
| `source_tool` | Provenance — needed for benchmarking and to explain disagreement between tools. |
| `layer` | code / dep / infra. The whole thesis is cross-*layer* chaining; a chain is only interesting if it spans ≥2 layers, so this must be a first-class column, not inferred. |
| `severity` | Normalized 0–4. Each tool uses a different scale (CRITICAL/HIGH vs ERROR/WARNING vs CVSS float); comparison is impossible without one shared scale. |
| `cwe_id` | Linking key → RAG exact filter and graph. Nullable — many infra findings arrive without one, which is exactly why `rule_mappings` exists. |
| `cve_id` | Linking key, mostly dep findings. Nullable. |
| `node_ref` | Canonical `type:identifier` of the graph node this finding decorates. This is the join that turns a flat finding list into graph annotations. |
| `message` | Human text; also the raw material for the constructed RAG query string. |
| `redacted` | Per-finding proof that redaction touched this text before it reached a model. |

---

## `graph_nodes`
**Why this table exists:** The resource graph used to live only in memory during a scan. Persisting it means (a) the UI can render the chain visually, (b) the audit is replayable months later, (c) a regression can be diagnosed by diffing graphs between runs.

| Column | Why it exists |
|---|---|
| `id` | Surrogate key for FK use. |
| `scan_job_id` | Graphs are per-scan, not global. |
| `node_key` | The canonical `type:identifier`. The single most fragile thing in the system — if the Terraform extractor writes `iam_role:order` and the finding decorator looks for `role:order`, the graph silently splits into per-layer islands and zero chains form. |
| `node_type` | pkg / code / image / task / role / resource. Drives traversal rules and rendering. |
| `layer` | Lets traversal cheaply assert "this path crosses ≥2 layers". |
| `is_hot` | Marks chain seeds (node carries a high-severity finding). Traversal starts only from hot nodes — this is the bound that keeps chaining tractable. |
| `attrs` | JSON escape hatch for type-specific detail (package version, TF resource address) without one column per node type. |

**Unique (`scan_job_id`, `node_key`)** — this constraint is the ID-consistency invariant enforced by the database rather than by hope. A duplicate key means the island bug is surfacing.

---

## `graph_edges`
**Why:** Findings are nodes; the *edges* are where the value is. No competitor tool has them. An exploit chain is literally a path over these rows.

| Column | Why it exists |
|---|---|
| `id` | Key, and referenced by hops. |
| `scan_job_id` | Scope. |
| `from_node_id` / `to_node_id` | The relation itself. |
| `relation` | used-by / built-into / deployed-as / assumes / can-access — semantic type, used both in reasoning and in the rendered narrative. |
| `seam` | Which of the cross-layer joins produced this edge (infra-spine / dep-code / code-infra / role-resource / transitive). Lets you measure *which seam* is failing when chains don't form — the POC's zero-chain incident was a missing role→resource seam. |
| `confidence` | `certain` (explicit reference), `inferred` (normalized image-name match), `unresolved` (load-bearing join that couldn't be confirmed). The honest-framing principle encoded in a column: a shaky join is recorded and flagged, never silently dropped and never silently trusted. |
| `oriented_attack_dir` | Terraform emits depends-on edges; attacks flow the opposite way. Wrong orientation = zero chains. This bit asserts the re-orientation actually ran. |

---

## `chains`
**Why:** The unit the user actually cares about and acts on. Findings are noise; a chain is the product.

| Column | Why it exists |
|---|---|
| `id` | Key. |
| `scan_job_id` | Scope. |
| `hop_count` | Bounded 3–4; makes the cap auditable. |
| `priority` | Rank by severity + exploitability — the "prioritized" in "prioritized draft audit". |
| `status` | asserted / validated / rejected. Rejected chains are kept, not deleted: they're the evidence that the Blue agent is doing false-positive reduction, and the raw material for precision/recall benchmarking. |
| `min_confidence` | Weakest-link propagation from `graph_edges.confidence`. A chain crossing one `unresolved` edge is itself `unresolved`, and the report marks it "potential chain, unverified join" for human confirmation. |

---

## `chain_hops`
**Why:** A chain is ordered and heterogeneous — each step has its own finding, its own traversed edge, and its own ATT&CK technique. That's a child table, not a JSON blob, because citations attach per-hop.

| Column | Why it exists |
|---|---|
| `id` | Key; citations point here. |
| `chain_id` | Parent. |
| `finding_id` | The weakness at this step. Many-to-one: one finding can appear in several chains. |
| `edge_id` | *Which* edge was traversed to get here. Nullable — the first hop is a seed node with no incoming edge. This is what makes a chain provably a real path rather than an agent-invented sequence. |
| `hop_order` | Sequence; ATT&CK tactic ordering constrains what may follow what. |
| `technique_id` | The ATT&CK technique explaining the transition. Graph gives connectivity; ATT&CK gives sequence. |
| `blue_validated` | Per-link Blue verdict. One false = chain broken. This column *is* the false-positive reducer, recorded at the granularity where it operates. |

---

## `reports`
**Why:** One durable, adjudicated artifact per scan, separate from the chains so retention and framing are managed at the document level.

| Column | Why it exists |
|---|---|
| `id` | Key. |
| `scan_job_id` | 1:1. |
| `summary` | The Reporter's adjudicated narrative. |
| `framing` | Always `draft_audit`. Storing the literal value makes it structurally impossible for a UI or export to present the output as a verified verdict. |
| `retained` | Opt-in retention; default is purge. |
| `feedback_slot` | Reserved for the Phase 2 self-improving loop. Present now specifically so enabling it later needs no migration. |
| `created_at` | Audit trail. |

---

## `citations`
**Why:** Grounding is the anti-hallucination guarantee. "Every surviving assertion cites its retrieved chunk" is only enforceable if citations are rows you can count, not prose in a summary.

| Column | Why it exists |
|---|---|
| `id` | Key. |
| `chain_hop_id` | Grounds a specific hop. Nullable. |
| `report_id` | Grounds a report-level statement. Nullable. Exactly one of the two is set. |
| `knowledge_id` | The Qdrant point id — the pointer across the SQL↔vector store boundary. |
| `source` | NVD / ATTACK / CAPEC / OWASP — provenance shown to the user. |
| `collection` | offense / defense. Reveals which agent retrieved it: Red cites offense, Blue cites defense. |

---

## `rule_mappings`
**Why it was promoted from a static dict to a table:** Coverage is the POC's known weak spot (it was "illustrative"). Production needs it generated from full tool rule catalogs, versioned, and editable without redeploying the application. It's also the mandated *first* retrieval step for every finding, so it can't be an afterthought.

| Column | Why it exists |
|---|---|
| `id` | Key. |
| `source_tool` | Same `check_id` string can mean different things across tools, so the tool is part of identity. |
| `check_id` | The tool's rule identifier (SCS0028, CKV_AWS_20, AVD-AWS-0086). |
| `cwe_id` | The resolved CWE — this is the whole point: it gives an ID-less infra finding a linking key into RAG and the graph. Nullable. |
| `cve_id` | Where the tool supplies one directly. Nullable. |
| `notes` | Provenance of the mapping — hand-authored vs. scraped from the tool's catalog, which matters when a mapping is disputed. |

**Unique (`source_tool`, `check_id`)** — one mapping per rule; ambiguity here would make CWE resolution nondeterministic.

**Not tenant-scoped** — it's global reference data. CKV_AWS_20 means the same thing for every customer, and per-tenant copies would drift.

---
