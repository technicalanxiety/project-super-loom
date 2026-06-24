# Super Loom
## A Multi-Principal Memory Infrastructure for Agent, Orchestrator, and Human Operation
### Confidence-Gated · Provenance-Calibrated · Signal-Only at the Trust Boundary

---

## What This Document Is

This is the implementation specification for Super Loom. It is a standalone system, not a fork, branch, or configuration mode of Project Loom. It shares lineage with Loom's memory taxonomy and pipeline architecture but diverges deliberately at the points where Loom's personal, single-tenant, human-trust-anchored design assumptions stop holding.

**Super Loom and Loom will never merge.** Where a concept is inherited from Loom, it is restated here in full, not referenced, because every inherited concept must be re-justified against Super Loom's actual constraints rather than assumed to transfer.

The central principle:

> **Loom assumed a human in the loop who could compensate for imprecise retrieval. Super Loom cannot make that assumption, because its principals include software that cannot supplement a compiled context package with judgment. Every architectural difference from Loom follows from this one fact.**

---

## What Super Loom Is Not

Stated up front because the boundary is load-bearing and gets eroded by drift if left implicit.

Super Loom is not a routing or action-authorization system. It does not decide whether a unit of agent work ships without human review. That decision belongs to Confidence Routing Gates (CRG), a separate system governed by Confidence Engineering. Super Loom's only output to that decision is a signal: a structured confidence receipt describing what was compiled and how much it can be trusted. **Super Loom produces signals. It never produces routing decisions.** If a future integration need ever pressures this boundary, the answer is a cleaner interface contract between Super Loom and CRG, not a routing function grown inside Super Loom.

Super Loom is not an identity or access management system. It does not authenticate principals, issue credentials, or maintain a registry of who is allowed to claim a given identity. It accepts identity claims at face value and builds empirical trust in those claims over time through provenance, exactly as it does for every other kind of evidence in the system.

Super Loom is not a classifier of sensitivity or risk. It does not infer how sensitive a namespace is by reading its contents. Sensitivity is declared by whoever defines the namespace's governing skill or solutioning artifact, and Super Loom only consumes that declaration.

---

## Technology Stack

Super Loom forks Loom's stack choices because the underlying engineering rationale (compile-time SQL safety, true concurrency, single system of record, local-first inference) holds independently of who the caller is. The stack is restated, not assumed, because a future divergence here should be a deliberate decision, not an accident of copy-paste.

### Application Platform

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Engine** | Rust (tokio async runtime) | Same rationale as Loom: compile-time checked SQL via sqlx, true concurrency, predictable latency. Multi-principal load makes predictable latency more important, not less, since software callers have less tolerance for variance than human callers. |
| **Dashboard UI** | TypeScript + React (Vite) | Operational visibility for Decision Authority and solutioning teams, not for agents. Agents never consume the dashboard; it exists for the humans governing the system. |
| **Database** | PostgreSQL 16 + pgvector + pgAudit | Single system of record, identical to Loom. Row-level security (RLS) is used for tenant and namespace boundary enforcement, a requirement Loom never had because Loom has no multi-principal isolation problem. |
| **LLM Inference** | Ollama (local), frontier fallback under explicit policy | Local inference is default for cost, latency, and data locality. Frontier fallback is a routed escalation, never a default path, and operates only on the compiled context package, never on raw episodic content. See Section on Inference Routing. |
| **Bootstrap / Declaration Tooling** | Python | Run-once transformation and declaration-artifact tooling, same role as Loom's bootstrap scripts. |
| **Build Tooling** | Claude Code | Implementation. No Kiro dependency assumed; not load-bearing to this spec. |

### Key Rust Crates

Identical core set to Loom (`axum`, `tokio`, `sqlx`, `pgvector`, `serde`/`serde_json`, `reqwest`, `sha2`, `uuid`, `chrono`, `tracing`, `tower`), plus:

| Crate | Purpose |
|-------|---------|
| `casbin` or equivalent policy-evaluation crate | Stage 0 gate policy evaluation (identity class, task confidence, namespace sensitivity against compilation depth). Specific crate selection deferred to implementation; the requirement is a policy engine separate from ad hoc conditional logic, so gate rules are inspectable and testable independently of the pipeline code that calls them. |
| `governor` or equivalent rate-limiting crate | Per-principal request shaping. A misbehaving or compromised agent principal should not be able to flood the gate the way a single human never could by hand. |

### Deployment Topology

```
Docker Compose (or equivalent orchestration, see note below)
├── superloom-engine (Rust binary)
│   ├── MCP endpoint (:8080/mcp)
│   ├── REST endpoint (:8080/api)
│   ├── Dashboard API (:8080/dashboard)
│   ├── Stage 0 Gate (in-process, evaluated before classification)
│   ├── Background worker (tokio spawned tasks)
│   └── Scheduled tasks (decay sweep, dormancy check, snapshot)
│
├── superloom-dashboard (Vite + React, static files)
│   ├── Pipeline health
│   ├── Gate decision log / audit trail viewer
│   ├── Namespace registry browser (sensitivity, ownership, declared scope)
│   ├── Principal trust history viewer
│   ├── Predicate / role-class pack browser
│   └── Confidence receipt inspector
│
├── postgres (PostgreSQL 16)
│   ├── pgvector, pgAudit, row-level security policies
│   └── superloom_* tables
│
├── ollama (local LLM inference)
│
└── caddy (reverse proxy + TLS + static file serving)
```

**Deployment note:** Unlike Loom, Super Loom's deployment target is not assumed to be a single self-hosted machine. Docker Compose is the default for development and small-scale operation. Production deployment into a customer-controlled VPC or on-prem environment is the expected enterprise path; this spec does not assume SaaS hosting by a third party, consistent with the data-sovereignty premise carried over from the original framing of this project ("your data is your own").

---

## Scope: What Ships in v1

| Ships | Does Not Ship |
|-------|--------------|
| PostgreSQL schema: episodes, entities, facts, predicates, role classes, namespaces with declared sensitivity, principal trust ledger | Cross-namespace synthesis beyond the confidence-gated read path defined in Stage 0 |
| Stage 0 Confidence Gate (identity class, task confidence, compilation depth, namespace sensitivity) | A second, competing routing/action-authorization system (this is CRG's job, not Super Loom's) |
| Three principal types as first-class callers: human, orchestrator, agent | Agent identity verification or credentialing (explicitly out of scope, owned by infrastructure outside Super Loom) |
| Three ingestion modes inherited from Loom (user-authored seed, vendor import, live capture) plus a fourth: agent observation | LLM reconstruction as an ingestion path (inherited prohibition from Loom, holds without exception) |
| Confidence receipt as a structured output on every `loom_think` call | A single-float confidence score (explicitly rejected, see Confidence Receipt section) |
| Role-class registry, pack-governed, sibling to predicate registry | Derived/inferred namespace sensitivity (explicitly rejected, sensitivity is always declared) |
| Principal trust ledger with empirical, evidence-based calibration per `principal_id` | A registry of verified agent identities (this is IAM's job, not Super Loom's) |
| Frontier inference routing with data-boundary enforcement at the compiled-package level | Raw episodic content ever crossing to a frontier model |
| MCP interface (`loom_think`, `loom_learn`, `loom_recall`) extended with `caller_context` | A routing decision returned from any of these calls; they return compiled context and a confidence receipt, nothing else |
| Declaration artifacts for namespace and role-class creation | Informal, undocumented namespace creation |

---

## Architecture

### System of Record

Identical premise to Loom: PostgreSQL is the single system of record, no second database, no external vector store, no graph database. Graph traversal via recursive CTEs, embeddings via pgvector, audit via pgAudit.

**Divergence from Loom:** Row-level security is mandatory, not optional, and is enforced at the namespace and principal level. Loom never needed this because Loom has exactly one trust anchor: the person running it. Super Loom has many principals whose access to a namespace must be enforced at the database layer, not just the application layer, because application-layer-only enforcement has a long history of privilege escalation failures and a multi-principal system is exactly the context where that history repeats.

### Three Pipelines (Loom Had Two)

Super Loom adds a pipeline stage Loom never needed: the gate evaluation, which runs before the online pipeline's classification step and is itself fed by the same offline trust-calibration process that updates the principal trust ledger.

**Stage 0: Gate Evaluation** (runs before the online pipeline, on every request):
1. Parse `caller_context`: `principal_type`, `principal_id`, `role_class`, `declared_task`
2. Look up `principal_id` in the trust ledger; unknown principals default to lowest trust tier
3. Validate `role_class` against the role-class registry for the requested namespace's governing pack; unrecognized role class is rejected outright, not defaulted
4. Read the namespace's declared `sensitivity_class` and `min_confidence_threshold` per requested compilation depth
5. Evaluate principal trust tier against namespace sensitivity and requested depth
6. Return one of: **pass** (proceed to online pipeline at requested depth), **narrow** (proceed at a reduced depth, e.g., semantic only, no episodic), or **reject** (no compilation, reason logged)

**Online pipeline** (serves queries that passed Stage 0, latency-sensitive):
1. Intent classification (primary + secondary class with confidence) — identical mechanism to Loom
2. Namespace resolution
3. Determine retrieval profiles, bounded by whatever depth Stage 0 authorized
4. Run profiles in parallel via `tokio::join!`
5. Apply memory weight modifiers per task class
6. Rank and trim on scored dimensions, **plus a new dimension: principal-sourced corroboration weight** (Section on Schema)
7. Compile package
8. **Generate confidence receipt** (new terminal step Loom does not have)
9. Audit log, including the Stage 0 decision and its rationale, not just the online pipeline's trace

**Offline pipeline** (learns from episodes, runs asynchronously):
1. Ingest episode (now from four source types, not three: user-authored seed, vendor import, live capture, **agent observation**)
2. Enforce idempotency / deduplicate
3. Extract entities, resolve, extract facts — identical mechanism to Loom
4. **Update principal trust ledger** if the episode's source is an agent observation: corroboration against existing facts raises trust for that `principal_id`; contradiction lowers it
5. Resolve supersession / currentness
6. Compute derived ranking state
7. **Run dormancy and drift sweep** against namespace and principal trust state (new scheduled task, not present in Loom, borrowed conceptually from CRG's decay mechanics but applied here to memory freshness rather than task-type calibration)
8. Snapshot hot-tier state periodically
9. Log extraction and trust-calibration metrics per episode

---

## The Confidence Gate (Stage 0), In Detail

### Why This Exists and Where the Boundary Sits

Loom never needed a pre-retrieval gate because Loom has one principal: the person who installed it. That person is the implicit trust anchor for every query. Super Loom replaces that implicit anchor with an explicit evaluation, because a namespace shared across human, orchestrator, and agent callers has no single party whose presence vouches for the request.

**The gate decides whether and how deeply to compile. It does not decide whether the compiled output may be acted upon.** That second decision, gating action rather than gating access to memory, belongs to CRG. A request that passes Stage 0 and receives a fully compiled context package with a low confidence receipt has still only received memory. What happens next, whether an agent acts on it autonomously or routes to human review, is CRG's evaluation, fed in part by the receipt Super Loom produced, but decided elsewhere.

### Gate Inputs

| Field | Source | Purpose |
|---|---|---|
| `principal_type` | `caller_context` | human / orchestrator / agent. Drives default confidence floor and output format. |
| `principal_id` | `caller_context` | Opaque, stable identifier. Never verified by Super Loom. Used to look up accumulated trust in the ledger. |
| `role_class` | `caller_context` | Validated against the pack-governed role-class registry for the target namespace. |
| `declared_task` | `caller_context` | Consumed by the same classification mechanism Loom already uses for human queries; for software callers, this field is populated directly rather than inferred from conversational text. |
| `namespace.sensitivity_class` | Namespace registry, declared at creation | Set by whoever defines the namespace's governing skill or solutioning artifact. Never inferred by Super Loom. |
| `namespace.min_confidence_threshold[depth]` | Namespace registry, declared at creation | A threshold per compilation depth (episodic / semantic / procedural), since episodic access is higher trust than semantic, which is higher trust than procedural. |
| `requested_depth` | Derived from the query and `declared_task` | What tier of memory the request is asking to read. |

### Gate Outputs

`pass`, `narrow`, or `reject`, each logged with the full input set and the specific threshold comparison that produced the outcome. A `narrow` result must specify exactly which depth was authorized, since "narrow" without a stated depth is not auditable.

### Identity Is a Claim, Trust Is Earned

This is the resolution of the identity question worked through at length before this document existed, restated here as binding spec language rather than left as conversational consensus:

Super Loom never verifies that a `principal_id` is who it claims to be. That verification, if it happens at all, happens in infrastructure outside Super Loom, an orchestration layer, an IAM system, a credential broker. Super Loom's only obligation is to **never let an unverified claim borrow more trust than it has earned**.

Mechanically: the first time a `principal_id` is seen, its observations and its declared tasks are treated at the lowest trust tier, identical treatment to an unverified vendor import. As that `principal_id` accumulates observations that get corroborated by independent sources, other principals' observations, human confirmation, semantic facts derived from multiple episodes, its trust tier rises. Trust never rises from self-consistency alone; a principal that only ever corroborates itself does not earn trust, because that is indistinguishable from a principal fabricating a consistent but false narrative. Trust rises only from independent corroboration.

`principal_id` collision or spoofing, where infrastructure outside Super Loom allows two distinct actual agents to share one identifier, is explicitly named as a caller-side integrity requirement. Super Loom cannot detect this and does not attempt to. The trust ledger will silently merge their histories, and that is a documented limitation, not a hidden one.

---

## Schema

Restated from Loom in full where inherited, with every addition marked.

### Episodes (Inherited, Two Fields Added)

```sql
CREATE TABLE superloom_episodes (
  -- CANONICAL (identical to Loom)
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source          TEXT NOT NULL,
  source_id       TEXT,
  source_event_id TEXT,
  content         TEXT NOT NULL,
  content_hash    TEXT NOT NULL,
  occurred_at     TIMESTAMPTZ NOT NULL,
  ingested_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  namespace       TEXT NOT NULL,
  metadata        JSONB DEFAULT '{}',
  participants    TEXT[],

  -- NEW: principal attribution, distinct from source
  principal_id    TEXT,                  -- opaque caller identity, null for pre-v1 ingestion modes
  ingestion_mode  TEXT NOT NULL CHECK (ingestion_mode IN (
                    'user_authored_seed', 'vendor_import',
                    'live_mcp_capture', 'agent_observation'
                  )),

  -- EXTRACTION LINEAGE (identical to Loom)
  extraction_model TEXT,
  classification_model TEXT,
  extraction_metrics JSONB,

  -- DERIVED (identical to Loom)
  embedding       vector(768),
  tags            TEXT[],
  processed       BOOLEAN DEFAULT false,

  -- DELETION SEMANTICS (identical to Loom)
  deleted_at      TIMESTAMPTZ,
  deletion_reason TEXT,

  UNIQUE(source, source_event_id)
);
```

`agent_observation` is the fourth ingestion mode, absent from Loom. It carries `principal_id` as a required field; the other three modes leave it nullable because they predate the multi-principal model and may originate from a human with no stable opaque identifier in play.

### Principal Trust Ledger (New, No Loom Analog)

```sql
CREATE TABLE superloom_principal_trust (
  principal_id          TEXT PRIMARY KEY,
  principal_type        TEXT NOT NULL CHECK (principal_type IN ('human', 'orchestrator', 'agent')),
  first_seen_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_seen_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
  trust_tier             TEXT NOT NULL DEFAULT 'unverified' CHECK (trust_tier IN (
                            'unverified', 'corroborated', 'established'
                          )),
  corroboration_count    INT DEFAULT 0,
  contradiction_count    INT DEFAULT 0,
  last_recalculated_at   TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_principal_trust_tier ON superloom_principal_trust (trust_tier);
```

Trust tier recalculation is a scheduled offline task, not a synchronous part of the online pipeline. A principal's tier at query time reflects its state as of the last sweep, not a live recompute, for the same latency-predictability reason Loom keeps its offline pipeline asynchronous.

### Namespaces (New as an Explicit Table; Implicit in Loom)

Loom treats namespace as a string field scattered across other tables. Super Loom requires namespace to be a first-class entity with declared properties, because sensitivity and confidence thresholds must live somewhere durable and auditable.

```sql
CREATE TABLE superloom_namespaces (
  namespace                  TEXT PRIMARY KEY,
  sensitivity_class           TEXT NOT NULL,        -- declared, not inferred; vocabulary owned by governing pack
  min_confidence_episodic     FLOAT NOT NULL,
  min_confidence_semantic     FLOAT NOT NULL,
  min_confidence_procedural   FLOAT NOT NULL,
  declared_by                  TEXT NOT NULL,         -- identity of the human or process declaring this namespace
  declared_at                   TIMESTAMPTZ NOT NULL DEFAULT now(),
  declaration_artifact          JSONB NOT NULL,        -- full declaration record, see Declaration Artifacts section
  governing_pack                TEXT REFERENCES superloom_predicate_packs(pack)
);
```

### Role Classes (Sibling Table to Predicates, Not a Field Within)

This corrects an earlier informal proposal to overload the predicate table. Role classes are principal classifications, not relationships between entities, and require their own table sharing the same pack-governance parent.

```sql
CREATE TABLE superloom_role_classes (
  role_class               TEXT PRIMARY KEY,
  pack                      TEXT NOT NULL REFERENCES superloom_predicate_packs(pack),
  description               TEXT,
  default_min_trust_tier    TEXT NOT NULL DEFAULT 'corroborated',
  created_at                 TIMESTAMPTZ DEFAULT now()
);
```

### Predicates, Predicate Packs, Entities, Facts

Inherited from Loom without structural change. Restated in the full implementation document but not duplicated here since no divergence exists; see Loom v3 spec Schema section for the canonical definition, which Super Loom's migration scripts copy directly rather than import as a dependency, consistent with the no-merge, no-shared-crate boundary established for this project.

---

## Confidence Receipt

### Why Not a Single Float

A single float collapses several independent kinds of uncertainty into one number, and the action-class gate downstream (CRG, not Super Loom) needs to know which dimension is weak in order to decide what to do about it. "Confidence is low because corroboration is thin" and "confidence is low because the namespace itself is barely past cold start" require different responses, and a single number cannot distinguish them.

### Receipt Structure

Returned as a structured object alongside the compiled context package on every `loom_think` call:

```json
{
  "compilation_depth_authorized": "semantic",
  "gate_outcome": "narrow",
  "principal_trust_tier": "corroborated",
  "facts": [
    {
      "fact_id": "...",
      "corroboration_count": 3,
      "independent_source_count": 2,
      "most_recent_corroboration_age_days": 4,
      "active_contradiction": false,
      "sole_source_flag": false
    }
  ],
  "namespace_sensitivity_class": "...",
  "namespace_confidence_floor_for_depth": 0.75,
  "overall_compilation_note": "Episodic access was requested and denied; package compiled at semantic depth only."
}
```

Every fact in the package carries its own corroboration data. The package as a whole carries the gate's decision and the namespace context that decision was evaluated against. **This receipt is the only artifact Super Loom hands to a downstream routing decision.** It is evidence, not a verdict.

---

## Ingestion: Four Modes

Three inherited from Loom (user-authored seed, vendor import, live MCP capture), restated without change since the underlying provenance logic holds regardless of caller. The fourth is new.

### Agent Observation (New)

Structured output from an agent operating in the environment: tool call results, decision rationale, action outcomes. Distinct from a conversational episode because it is typically structured data, not prose, and its evidentiary value depends entirely on the issuing `principal_id`'s trust tier.

**Treatment at ingestion:**

- Carries `principal_id` as a required field.
- Enters the system at the issuing principal's current trust tier. A first-time principal's observations enter at `unverified`, identical treatment to an unverified vendor import, and do not gain elevated authority merely by being machine-structured.
- Corroboration against this observation, by another independent principal, by a human, or by an existing well-corroborated semantic fact, raises the issuing principal's trust tier over time (see Principal Trust Ledger above). Contradiction lowers it.
- The `loom_learn` MCP contract accepts agent-formatted payloads (structured tool-call and outcome records) in addition to conversational payloads. Payload shape is validated against a schema specific to `agent_observation`, distinct from the freer-form validation applied to human-authored content.

**The LLM reconstruction prohibition, inherited from Loom without exception:** An agent observation must be a record of what actually happened: an actual tool call result, an actual logged outcome. It must never be an LLM's generated summary of what it believes happened. This is the same trap Loom's bootstrap work named for human conversational summarization, applied to agents: a confident-sounding generated account of an agent's own actions is not evidence, and treating it as such poisons the system the same way LLM reconstruction would have poisoned Loom's bootstrap. There is no `ingestion_mode` value for this. There is no official door for it.

---

## Inference Routing and the Data Boundary

### The Boundary Statement

Local inference via Ollama is the default path for every classification, extraction, and compilation operation. Frontier model escalation is a routed exception, evaluated under explicit policy, never a default.

**What may cross the boundary to a frontier model:** the compiled context package, after Stage 0 and the online pipeline have already run, after sensitivity-aware depth narrowing has already been applied. This package has already been filtered through Super Loom's provenance and confidence system before it is eligible to leave the local boundary.

**What may never cross the boundary:** raw episodic content. An episode's `content` field, however sensitive its namespace, is local-inference-only. If a task genuinely requires frontier-level reasoning over episodic detail, the policy decision is to deny the escalation and route to mandatory human review, not to relax the boundary.

This is the literal mechanism behind "your data is your own": data sovereignty is enforced as a routing rule with a hard boundary, not as a promise about vendor behavior.

### Escalation Policy

A request escalates to a frontier model only when: local inference confidence on a classification or extraction task falls below a declared threshold, the escalation is logged with the reason, and the payload sent is the compiled package, never raw content. This is a policy enforced in code, at the inference-routing layer, not a convention left to implementer discipline.

---

## Declaration Artifacts

Every namespace and role-class creation event produces a structured, reviewable record. This is not optional documentation; it is the evidentiary basis for any later question about why a boundary was drawn where it was.

### Namespace Declaration Record

- **Sensitivity class** and the stated reasoning for that classification (declared by the namespace's governing skill or solutioning process, never inferred by Super Loom)
- **Confidence floor per depth** (episodic, semantic, procedural) and the stated reasoning for each
- **Governing pack**, if applicable
- **Declared by** and **declared at**

### Role Class Declaration Record

- **The authority or function this role class represents**, stated specifically enough to be recognized again
- **Default minimum trust tier** required for a principal claiming this role class to receive standard compilation depth
- **Pack** this role class belongs to
- **Declared by** and **declared at**

These records are read, not on a standing cadence, but when something downstream, a gate rejection pattern, a trust ledger anomaly, an audit request, suggests the original declaration needs revisiting. This mirrors the same exception-based review discipline established for Confidence Routing Gates: instrumentation watches continuously, humans are pulled in by evidence, not by patrol.

---

## Observability

All observability data is captured via the `tracing` crate and stored in `superloom_audit_log`, extending Loom's pattern with gate-specific and trust-specific fields.

### Per-Request (Online, Gate + Pipeline)
- Stage 0 inputs in full: `principal_type`, `principal_id`, `role_class`, `declared_task`, `requested_depth`
- Stage 0 outcome: pass / narrow / reject, with the specific threshold comparison
- Everything Loom already logs for the online pipeline (classification, retrieval, ranking, compilation, latency per stage)
- The full confidence receipt returned with the compiled package

### Per-Ingestion (Offline)
- Everything Loom already logs (episode metrics, extraction quality, supersession)
- `ingestion_mode`, including `agent_observation`
- `principal_id` and the trust-tier delta this episode's corroboration or contradiction produced, if any

### Per-Trust-Recalculation (New, Scheduled)
- `principal_id`, prior tier, new tier, corroboration/contradiction counts driving the change
- Dormancy and drift sweep results: namespaces or principals whose trust state has decayed past a threshold

---

## Implementation Timeline

| Phase | Milestone | Deliverable |
|---|---|---|
| 1 | **Schema + Stage 0 Skeleton** | Migrations for episodes (with `principal_id`, `ingestion_mode` including `agent_observation`), principal trust ledger, namespace registry, role-class registry. Stage 0 gate implemented as a real function: takes `caller_context`, returns pass/narrow/reject against a hardcoded threshold table. No frontier routing yet. |
| 2 | **Confidence Receipt + Online Pipeline Integration** | Online pipeline emits a structured confidence receipt on every `loom_think` call. Per-fact corroboration tracking wired into the ranking stage. Gate decisions logged to audit. |
| 3 | **Agent Observation Ingestion** | `loom_learn` accepts agent-formatted payloads. Trust ledger recalculation as a scheduled offline task. Corroboration/contradiction detection against existing facts. |
| 4 | **Inference Routing** | Local-default classification and extraction. Frontier escalation policy implemented as an explicit, logged routing decision operating only on compiled packages, never raw episodic content. |
| 5 | **Declaration Tooling + Dashboard** | CLI or minimal UI for namespace and role-class declaration, producing the structured artifacts specified above. Dashboard views for gate decisions, trust ledger, and confidence receipts. |
| **Gate** | **End of Phase 2** | Stage 0 must demonstrably narrow or reject requests against real namespace sensitivity declarations, not just pass everything through. If the gate is a no-op in practice, do not proceed to Phase 3 until it is exercised by real policy. |

---

## What Is Explicitly Deferred

| Capability | Why Deferred | Trigger to Revisit |
|---|---|---|
| Direct interoperability contract with CRG | The boundary (Super Loom produces signals, CRG produces routing decisions) is specified; the wire-level integration is not, since CRG itself is still being specified independently | When CRG and Super Loom have a concrete shared deployment to integrate against |
| Cross-namespace synthesis for high-trust orchestrator principals | Namespace isolation is intentionally hard in v1, mirroring Loom's own caution here | After gate logs show a consistent, justified pattern of orchestrators needing synthesis across namespace boundaries they're otherwise entitled to read |
| Formal IAM integration pattern (SSO/OIDC claim mapping into `caller_context`) | Out of scope per this project's own stance: identity verification belongs to infrastructure outside Super Loom | When a concrete enterprise deployment defines its actual IAM stack |
| Variance-style monitoring of gate outcomes (a misconfigured namespace threshold producing inconsistent pass/reject patterns) | Conceptually borrowed from CRG's variance-within-bucket mechanism but not yet adapted to Super Loom's gate; needs its own design pass rather than a hasty transplant | After Phase 2 produces enough gate decision volume to evaluate whether this is needed |
| Multi-region or multi-instance deployment | v1 targets a single deployment per organization, consistent with the customer-controlled VPC or on-prem model | If enterprise scale requires geographic distribution |
