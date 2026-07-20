# SentinelAI — API Design (DAD-01b)

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

> **What changed for D2.** The scan-submission endpoint no longer accepts a repository or a code tarball. It accepts a **bundle**: the five scanners' findings (SARIF/JSON) plus the graph-input artifacts (`terraform graph` DOT, Terraform source, Dockerfile, dependency manifests). The GitHub Actions runner is the caller. New read endpoints expose the persisted resource graph (nodes/edges with confidence) and bundle provenance. The report explicitly carries the `draft_audit` framing and per-chain join-confidence.

---

## 1. Purpose & scope

This document defines the REST API surface for SentinelAI under the D2 architecture. The API is the boundary between the GitHub Actions runner (which observes code and uploads a bundle) and the backend (which reasons over that bundle). It also serves the Angular read UI. Application source never crosses this API — only findings and infrastructure/manifest artifacts do.

---

## 2. Conventions

- JSON request/response bodies; UTF-8; versioned under `/v1`.
- The scan-submission body is a **multipart upload**: a JSON metadata part plus the bundle artifacts as file parts (SARIF/JSON, DOT, Terraform source, Dockerfile, manifests).
- Cursor pagination on list endpoints: `?cursor=&limit=`.
- Errors return RFC 7807 `problem+json` with a stable `type` and `detail` — never raw exception strings.
- Per-tenant rate limiting; `429` with `Retry-After` when exceeded.
- Scanning is asynchronous: `POST /v1/scans` returns `202` with a poll URL.

---

## 3. Authentication

All endpoints require a bearer JWT (`Authorization: Bearer <token>`), obtained from `/v1/auth/token`. Every request is scoped to the caller's tenant and RBAC role. Tokens are short-lived; HTTPS/TLS is enforced; secrets sit in Azure Key Vault.

The **GitHub Actions runner authenticates with a scoped machine token** carrying the `scan:write` scope. This token is the runner's only credential — it cannot read another tenant's data, and it is the identity recorded in `scan_bundles` provenance.

---

## 4. Endpoints

| Method | Endpoint | Purpose | Scope |
|--------|----------|---------|-------|
| POST | /v1/auth/token | Exchange credentials for a JWT | public |
| POST | /v1/projects | Register a repo / project | project:write |
| GET | /v1/projects/{id} | Fetch project config | project:read |
| POST | /v1/scans | Submit a scan **bundle** at PR time (multipart) | scan:write |
| GET | /v1/scans/{id} | Poll job status + stage progress | scan:read |
| GET | /v1/scans/{id}/bundle | Bundle provenance (what the runner sent) | scan:read |
| GET | /v1/scans/{id}/findings | Normalized findings, filter by layer/severity | scan:read |
| GET | /v1/scans/{id}/graph | Resource graph: nodes + edges with confidence | scan:read |
| GET | /v1/scans/{id}/chains | Adjudicated exploit chains + hops | scan:read |
| GET | /v1/reports/{id} | Draft audit + citations | report:read |
| POST | /v1/scans/{id}/purge | On-demand bundle/data purge | scan:write |
| DELETE | /v1/account | Account deletion + full data purge | account:admin |

> Two endpoints are new under D2: `GET /v1/scans/{id}/bundle` (provenance of what the runner uploaded) and `GET /v1/scans/{id}/graph` (the persisted resource graph the UI renders). `POST /v1/scans` changes from a repo/tarball submission to a bundle multipart upload.

---

## 5. Request / response formats

### 5.1 POST /v1/scans — request (multipart)

The runner uploads a metadata part plus the bundle artifacts. The metadata part:

```json
{
  "project_id": "b2c1...",
  "pr_ref": "refs/pull/482/head",
  "commit_sha": "9f3a...",
  "model_tier_hint": "auto",
  "retain_report": true,
  "runner_secret_scan": "passed",
  "scanner_versions": {
    "semgrep": "1.90.0",
    "roslyn-security": "8.0.0",
    "dependency-check": "10.0.0",
    "trivy": "0.55.0",
    "checkov": "3.2.0"
  },
  "artifacts": [
    { "kind": "sarif", "tool": "semgrep",           "filename": "semgrep.sarif" },
    { "kind": "sarif", "tool": "checkov",            "filename": "checkov.sarif" },
    { "kind": "sarif", "tool": "trivy",              "filename": "trivy.sarif" },
    { "kind": "json",  "tool": "dependency-check",   "filename": "dependency-check.json" },
    { "kind": "sarif", "tool": "roslyn-security",    "filename": "roslyn.sarif" },
    { "kind": "dot",   "tool": "terraform",          "filename": "terraform-graph.dot" },
    { "kind": "tf",    "tool": "terraform",          "filename": "main.tf" },
    { "kind": "dockerfile", "tool": "docker",        "filename": "Dockerfile" },
    { "kind": "manifest",   "tool": "nuget",         "filename": "OrderApp.csproj" },
    { "kind": "manifest",   "tool": "nuget",         "filename": "packages.lock.json" }
  ]
}
```

The corresponding file parts carry each artifact's bytes. **No application source (`.cs` files) is part of this bundle** — only findings and graph-input artifacts.

### 5.2 POST /v1/scans — response (202 Accepted)

```json
{
  "scan_job_id": "7d4e...",
  "status": "queued",
  "corpus_version": "2026-07-15",
  "poll_url": "/v1/scans/7d4e...",
  "created_at": "2026-07-18T09:14:22Z"
}
```

### 5.3 GET /v1/scans/{id}/bundle — response

```json
{
  "scan_job_id": "7d4e...",
  "runner_secret_scan": "passed",
  "ingress_redaction_applied": true,
  "artifact_manifest": [
    "semgrep.sarif", "checkov.sarif", "trivy.sarif",
    "dependency-check.json", "roslyn.sarif",
    "terraform-graph.dot", "main.tf", "Dockerfile",
    "OrderApp.csproj", "packages.lock.json"
  ],
  "scanner_versions": { "semgrep": "1.90.0", "checkov": "3.2.0" },
  "received_at": "2026-07-18T09:14:23Z"
}
```

### 5.4 GET /v1/scans/{id}/graph — response (abridged)

The persisted resource graph, edges carrying their confidence tier.

```json
{
  "scan_job_id": "7d4e...",
  "nodes": [
    { "node_key": "pkg:Newtonsoft.Json", "type": "pkg",  "layer": "dep",   "is_hot": true },
    { "node_key": "code:OrderService",   "type": "code", "layer": "code",  "is_hot": true },
    { "node_key": "image:tinyapp/order", "type": "image","layer": "infra", "is_hot": false },
    { "node_key": "task:order",          "type": "task", "layer": "infra", "is_hot": false },
    { "node_key": "iam_role:order",      "type": "role", "layer": "infra", "is_hot": true },
    { "node_key": "s3:customer-data",    "type": "resource","layer":"infra","is_hot": false }
  ],
  "edges": [
    { "from": "pkg:Newtonsoft.Json", "to": "code:OrderService",   "relation": "used-by",     "seam": "dep-code",       "confidence": "certain" },
    { "from": "code:OrderService",   "to": "image:tinyapp/order", "relation": "built-into",  "seam": "code-infra",     "confidence": "inferred" },
    { "from": "image:tinyapp/order", "to": "task:order",          "relation": "deployed-as", "seam": "infra-spine",    "confidence": "certain" },
    { "from": "task:order",          "to": "iam_role:order",      "relation": "assumes",     "seam": "infra-spine",    "confidence": "certain" },
    { "from": "iam_role:order",      "to": "s3:customer-data",    "relation": "can-access",  "seam": "role-resource",  "confidence": "certain" }
  ]
}
```

### 5.5 GET /v1/reports/{id} — response (abridged)

```json
{
  "report_id": "a91c...",
  "scan_job_id": "7d4e...",
  "framing": "draft_audit",
  "corpus_version": "2026-07-15",
  "chains": [
    {
      "id": "c1", "priority": 1, "severity": "critical",
      "min_confidence": "inferred",
      "hops": [
        { "order": 1, "layer": "dep",
          "finding": "CVE-2023-1234 (Newtonsoft.Json)",
          "technique_id": "T1190", "blue_validated": true,
          "edge_confidence": "certain",
          "citations": [
            { "knowledge_id": "cve-2023-1234", "source": "NVD", "collection": "offense" }
          ] },
        { "order": 2, "layer": "code",
          "finding": "CWE-502 unsafe deserialization",
          "technique_id": "T1203", "blue_validated": true,
          "edge_confidence": "inferred",
          "citations": [
            { "knowledge_id": "capec-586", "source": "CAPEC", "collection": "offense" }
          ] },
        { "order": 3, "layer": "infra",
          "finding": "over-permissioned IAM role (s3:*)",
          "technique_id": "T1078", "blue_validated": true,
          "edge_confidence": "certain",
          "citations": [
            { "knowledge_id": "owasp-a01", "source": "OWASP", "collection": "defense" }
          ] }
      ],
      "impact": "attacker reads the customer-data bucket"
    }
  ]
}
```

> `min_confidence: "inferred"` signals a chain that traverses an inferred (image-name) join — surfaced to the reviewer as confirmed-but-check-the-link, per the join-confidence model. A chain traversing an `unresolved` join would carry `min_confidence: "unresolved"` and read as "potential chain, unverified join."

### 5.6 Error format (RFC 7807)

```json
{
  "type": "https://sentinelai/errors/scan-not-found",
  "title": "Scan not found",
  "status": 404,
  "detail": "No scan job with id 7d4e... for this tenant"
}
```

---

## 6. How the API maps to the D2 flow

| Step | Actor | API interaction |
|------|-------|-----------------|
| Trigger + observe | Runner | Runs scanners + `terraform graph`, packages the bundle |
| Submit | Runner | `POST /v1/scans` (multipart bundle) → `202` + poll URL |
| Gate | Backend | Auth/RBAC/tenant-scope; ingress redaction before any LLM |
| Process | Backend | Normalize → rule-map (`rule_mappings`) → build graph → traverse → retrieve → debate |
| Poll | Runner | `GET /v1/scans/{id}` until `completed` |
| Post back | Runner | Reads report, posts the PR comment via the GitHub API |
| Present | UI | `GET /v1/scans/{id}/graph`, `/chains`, `/v1/reports/{id}` |
| Retain/purge | Backend | Bundle purged after audit; `bundle_purged` set; opt-in report retained |

---

## 7. Sprint 0 exit criteria for this document

1. `POST /v1/scans` redefined as a bundle multipart upload (findings + graph-inputs), not a repo/tarball.
2. New read endpoints defined: `/v1/scans/{id}/bundle` (provenance) and `/v1/scans/{id}/graph` (persisted graph with confidence).
3. Report response carries `framing: draft_audit` and per-chain / per-hop confidence.
4. Runner machine-token auth (scan:write) and tenant scoping fixed.
5. API mapped to the D2 runner/backend split end to end.
6. Feeds the canonical data contracts, the CI/CD workflow, and the Angular report UI.
