# SentinelAI — AI Design & Workflow (AID-01)

*Cross-layer exploit-path reasoning for CI/CD security auditing*

**Sprint 0 Deliverable** · SentinelAI-Team · Graduate Capstone Project
Version 2.0 · Sprint 0 baseline (D2 architecture) · July 2026

---

## Document control

| Field | Detail |
|-------|--------|
| Project | SentinelAI — Automated Threat Modeling & SecOps Engine |
| Team | SentinelAI-Team (6 members) |
| Members | Mohamed Fathi Ibrahim, Mohamed Yasser Elsherbiny, Seif Eldin Medhat Farouk, George Mariey Demyan Beshara, Hana Mohamed Elsayeh, Mostafa Fouad Hendy |
| Sprint | Sprint 0 — Foundations, contracts & de-risk (Week 1) |
| Status | Baseline design, reflects the D2 distributed-scan architecture |
| Locked stack | C#/.NET · ASP.NET Core · Microsoft Agent Framework 1.0 · GPT-4o (Azure) / Claude (demo) · Qdrant · SQL Server · Angular · GitHub Actions + Docker |

> **What changed for D2.** The AI subsystem is unchanged in principle but is now fed by a **bundle** (findings + infrastructure/manifest artifacts) uploaded from a GitHub Actions runner — never application source. Pipeline A (knowledge ingestion) is untouched. In Pipeline B (scan-time), the scanners and graph-input extraction run on the runner; normalization, rule-mapping, graph construction, retrieval, and the agent debate run in the backend. The rule-mapping step is now a SQL table lookup, and cross-layer edges carry a confidence tier that the agents reason over.

---

## 1. Purpose & scope

This document specifies the AI subsystem of SentinelAI as fixed in Sprint 0 under the D2 architecture: the LLM, the agent framework, the vector database, the embedding model, and the end-to-end RAG / agent workflow. It is the load-bearing part of the project — the whole thesis is cross-layer exploit-path reasoning, which rests entirely on the pieces defined here.

> **The load-bearing idea.** There are two pipelines. Knowledge is embedded once, offline (Pipeline A). The scanned project is turned into findings + a graph, and those findings query the knowledge (Pipeline B). The repository's own code is never vectorized — and under D2 it never even reaches the backend.

---

## 2. LLM

Production uses **GPT-4o via Azure OpenAI** — Azure-native, so inference stays inside the same security boundary as the embeddings. The demo runs **Claude Sonnet** for reasoning-heavy debate turns and **Claude Haiku** for high-volume routine turns. The model sits behind Microsoft Agent Framework's connector abstraction, so switching provider is a config change, proven by an integration test.

### 2.1 Model-tier routing

| Tier | Model (prod / demo) | Used for |
|------|--------------------|---------|
| High (reasoning) | GPT-4o / Claude Sonnet | Debate turns: chaining, link validation, adjudication |
| Cheap (routine) | GPT-4o-mini / Claude Haiku | High-volume routine turns, formatting, summarization |

Cost per audit by model tier is a tracked metric.

---

## 3. Agent framework

Orchestration uses **Microsoft Agent Framework 1.0**, the successor to Semantic Kernel (now in maintenance mode). Its group-chat workflow, checkpointing, and session state manage the shared memory and state transitions across the debate. Building this is the project's top technical risk, so it is stood up and validated first.

### 3.1 The four agents

| Agent | Role | Reads from |
|-------|------|-----------|
| Orchestrator | Sequences the debate; enforces the turn-cap; detects convergence; routes model tier | Session state |
| Red Team | Asserts ordered cross-layer exploit paths (node, technique, evidence) within bounded candidates | Offense collection (ATT&CK / CAPEC) |
| Blue Team | Validates each link against the real config; breaking one link breaks the chain (the false-positive reducer) | Defense collection (OWASP / mitigations) |
| Reporter | Adjudicates the debate into a prioritized, cited draft audit | The debate transcript |

> **Why multi-agent.** The adversarial structure is the source of grounded, low-false-positive findings. A claim survives only if it withstands Blue's rebuttal **and** validates against the real configuration. A single model asked to "find attack paths" tends to hallucinate plausible chains; forcing every assertion through an adversary that can kill it on one broken link is what makes the output trustworthy.

### 3.2 Bounded chaining

Chaining is bounded: candidate chains are capped at **3–4 hops**, seeded from the highest-severity "hot" findings, with the turn-cap enforcing termination. The Red agent reasons within deterministically-generated candidates (real edges only) rather than over the free graph — the middle path that avoids hallucinated edges, cost blow-up, and non-reproducible benchmarks.

### 3.3 Reasoning over join confidence

Under D2, cross-layer edges carry a confidence tier — `certain`, `inferred`, or `unresolved`. The agents treat these explicitly:

- A `certain` edge (infra spine, dependency manifest, IAM parse) is taken as a real link.
- An `inferred` edge (normalized image-name match) is usable but flagged; the Blue agent gives it extra scrutiny against the real config.
- An `unresolved` edge (a load-bearing join that could not be confirmed) does not kill the chain silently. The Red agent may still assert the chain, and the Reporter surfaces it as a **"potential chain, unverified join"** for human confirmation — never as a confirmed verdict.

A chain inherits the weakest edge's confidence, so a single unresolved hop marks the whole chain for human review.

---

## 4. Vector database

The vector store is **Qdrant** (cloud, with a self-host option). It holds two collections — **offense** (ATT&CK / CAPEC / exploitation) and **defense** (OWASP / mitigations). Each point is a dense vector plus a rich payload.

### 4.1 Stored point shape

```
Point {
  id:      uuid
  vector:  float[3072]        // embedding of the chunk TEXT
  payload: {
    source:     'NVD' | 'ATTACK' | 'CAPEC' | 'OWASP',
    layer_hint: 'code' | 'dep' | 'infra' | null,
    cve_id?, cwe_id?, capec_id?, technique_id?,
    cvss?, severity?, published_year?,
    tactic?, mitigation_type?, fix_available?,
    routing:    'offense' | 'defense' | 'both'
  }
}
```

### 4.2 Why Qdrant, and design decisions

- **Filter-before-rank** — Qdrant restricts to payload matches (e.g. severity=HIGH, source=NVD) and then ranks by vector similarity in one query. This is why a reranker was cut: metadata filtering already narrows the candidate set (Cohere Rerank 3.5 was evaluated and removed).
- **Sparse vector slot reserved** — the schema reserves a sparse field so hybrid dense+sparse retrieval can be enabled later without re-indexing.
- **Batch + retry** — cloud upserts use small batches with retry-and-backoff; a POC incident (upserts crawling on a tiny blocking batch) made this an explicit requirement.

---

## 5. Embedding model

Production embeds with **text-embedding-3-large via Azure OpenAI** (3072-dim). The POC / local demo embeds with **BGE-M3 run locally** (dense 1024 + native sparse; MIT-licensed). Only the *text* field is embedded — never the payload.

> **The same-model invariant.** The identical embedding model MUST run at index time and query time. A mismatch puts vectors in different spaces and makes similarity scores meaningless. Ingest fails loud if the intended model is unavailable rather than silently falling back — a silent 512-dim bag-of-words fallback during the POC is exactly the failure this guard now prevents.

This choice is deliberate: the corpus is security prose (CVE / ATT&CK / CAPEC / OWASP descriptions), not source code, so code-specialized embedders are ruled out. But the data is also identifier-heavy (CVE-2021-44228, CWE-502), where exact-token matching beats dense similarity — which is why the most suitable production setup is hybrid dense + sparse rather than any single dense model.

---

## 6. RAG / AI workflow

### 6.1 Pipeline A — knowledge ingestion (offline, once)

Unaffected by D2. Runs in the backend environment, offline.

- Load canonical sources: NVD CVE/CWE (JSON), ATT&CK & CAPEC (STIX 2.1), OWASP (markdown), CVE2CAPEC (cross-links).
- Convert each to a RawEntry: a clean-prose `text` field (the only thing embedded) and a `payload` of all identifiers (never embedded).
- Preprocess: strip CVSS/URLs/markup from text, dedupe by natural_id, enrich CVEs with CVE2CAPEC cross-links.
- Chunk per source: one per CVE, one per technique (atomic); OWASP header-split ~500–800 tokens.
- Embed `text` with the same model that will run at query time.
- Route each chunk to offense, defense, or both by content (ATT&CK/CAPEC→offense, OWASP→defense, CVE→both).
- Batched, checkpointed upsert into Qdrant with payload indexes on cve_id, cwe_id, severity, source.

### 6.2 Pipeline B — scan-time (per pull request), split runner/backend

**On the runner (observe):**

- Secret pre-scan (Gitleaks CLI) → early filter before anything is uploaded.
- Run five scanners in parallel: Semgrep + Roslyn (code), OWASP Dependency-Check (dep), Trivy + Checkov (infra) → emit SARIF/JSON.
- Run `terraform init` + `terraform graph` → capture the infra spine DOT.
- Collect graph-input files (Terraform source, Dockerfile, csproj + packages.lock.json).
- Package findings + graph inputs into the **bundle** and `POST /v1/scans`.

**In the backend (reason):**

- Ingress secret-scan and redact findings text before any content reaches an LLM — a guaranteed gate, independent of the runner-side pre-scan.
- Normalize to one unified Finding set (native JSON for the dep layer to preserve CVE/CWE keys SARIF drops).
- Every finding passes the **SQL rule-mapping table** first (check_id → CWE), an exact lookup, never embedded.
- Build the resource graph from the shipped inputs (Terraform spine + three cross-layer seams) with confidence-tiered edges; generate bounded candidate chains.
- For each finding, run the retrieval decision tree (below) against the offense/defense collections.
- Red asserts chains grounded in offense knowledge; Blue validates each link against the graph + defense knowledge; Reporter adjudicates.
- Emit a prioritized draft audit where every assertion cites its retrieved chunk; the runner posts PR comments; the Angular UI presents the full audit.

> **The scan target is never embedded.** Under D2 the backend receives even less of the target than before — findings plus infra/manifest artifacts, no application source. The Pipeline A/B separation therefore holds by construction.

### 6.3 The retrieval decision tree

Matching a finding to the right knowledge chunk is a decision tree, not a single search. The presence or absence of a clean identifier decides the path:

```
finding arrives
   │
[1] Rule-mapping table lookup (SQL, exact, always first)
      tool check_id -> CWE (and CVE if provided)
   │
   Does the finding now have a clean CVE or CWE id?
   │                                   │
  YES                                  NO
   │                                   │
[2] Exact payload filter in       [3] Build a text query from the
    Qdrant:                            finding (message + resource
    payload.cve_id = X                 type), embed it, semantic
    OR payload.cwe_id = Y              search the corpus
   │                                   │
   '----------------+------------------'
                    │
[4] HYBRID refine: within the offense (Red) or defense (Blue)
    collection, apply metadata filters (severity, source, year)
    THEN rank by vector similarity (dense + sparse, RRF)
                    │
      top-k chunks -> injected into agent prompt + cited
```

### 6.4 The three modes, and which layer uses which

| Mode | Question it answers | When it fires |
|------|--------------------|--------------|
| Exact lookup | What does this rule id mean / which CWE? | Every finding, first step (SQL rule-mapping table) |
| Exact filter | Is there a chunk for this exact CVE/CWE? | Finding has a clean id (dep, most code) |
| Semantic | What attacks/patterns relate to this? | Finding has no clean id (many infra misconfigs) |
| Hybrid | Retrieve, narrowed then ranked | Normal knowledge retrieval |

> **Source-filtering is mandatory.** Unfiltered semantic retrieval lets ~200k CVEs drown the ~1.3k ATT&CK technique entries. Offense and defense paths must query their respective filtered subsets — a POC incident (CVE dominance burying ATT&CK/CAPEC) made this an explicit rule.

---

## 7. Grounding & honesty guarantees

- Every surviving assertion in the draft audit cites its retrieved knowledge chunk; the POC measured 100% grounding coverage on the fixture.
- DEPRECATED CAPEC/ATT&CK entries are filtered from results (they score spuriously high on thin text) via over-fetch-and-filter.
- Cross-layer chains that depend on an `unresolved` join are surfaced as potential-chain-unverified, not as confirmed findings.
- The output is framed as a prioritized draft audit for human review, not a verified verdict.
- A future self-improving RAG feedback loop is Phase 2 only; the base build reserves the metadata/output fields, and human approval of any promotion is non-negotiable.

---

## 8. Sprint 0 exit criteria for this document

1. LLM, agent framework, vector DB, and embedding model all named and locked.
2. The two-pipeline separation and same-model invariant documented as guardrails.
3. Pipeline B split across runner (observe) and backend (reason) per the D2 model.
4. Rule-mapping step fixed as a SQL exact lookup, first in the retrieval decision tree.
5. Join-confidence reasoning specified so unresolved joins are surfaced, not dropped.
6. Retrieval decision tree and offense/defense split specified.
7. Feeds the knowledge-ingestion, retrieval, and debate implementation stories.
