## Super Loom Implementation Plan

Based on my analysis of the specification and Loom v3 codebase patterns, here is a comprehensive 5-phase implementation plan for Super Loom:

---

## Executive Summary

Super Loom adds three architectural layers to Loom v3:
1. **Multi-principal isolation** via Row-Level Security and principal identity tracking
2. **Stage 0 Confidence Gate** that evaluates trust before retrieval proceeds
3. **Principal Trust Ledger** that empirically calibrates confidence over time through corroboration

The spec intentionally defers identity verification (handled by external IAM) while building an evidence-based trust system inside Super Loom. All five phases have been designed for incremental delivery with clear gating criteria.

---

## Phase Architecture & Critical Path

```
Phase 1: Schema + Stage 0 Skeleton
  ↓ (must complete before Phase 2)
Phase 2: Confidence Receipt + Gate Integration ← GATE GATING MILESTONE
  ↓ (must complete before Phase 3)
Phase 3: Agent Observation + Trust Calibration
  ↓ (can run in parallel with Phase 4)
Phase 4: Inference Routing
  ↓ (can run in parallel with Phase 5)
Phase 5: Declaration Tooling + Dashboard
```

**Critical dependencies:**
- Phase 1 schema must exist before any online pipeline changes (Phase 2)
- Stage 0 gate must be a real working gate by end of Phase 2 (not a pass-through)
- Principal trust ledger must exist before agent observation ingestion can work (Phase 3)
- Inference routing requires completed online pipeline (Phase 2 prerequisite)

---

## Phase 1: Schema + Stage 0 Skeleton

**Timeline estimate:** 2-3 weeks  
**Deliverable:** Working database schema + non-functional gate stub

### Milestones

#### 1.1 PostgreSQL Schema Extensions
Create migration set that builds on Loom v3 schema:

**New tables:**
- `superloom_principal_trust` — Principal trust ledger tracking corroboration/contradiction
- `superloom_namespaces` — First-class namespace entity with sensitivity declarations
- `superloom_role_classes` — Role class registry (governance per predicate pack)

**Schema modifications:**
- Extend `superloom_episodes` with `principal_id` (nullable, required for `agent_observation` mode)
- Extend `superloom_episodes` with `ingestion_mode` enum to include `agent_observation`
- Add Row-Level Security policies on all sensitive tables to enforce namespace + principal boundaries

**Database-layer isolation:**
- RLS policies must prevent a principal from reading episodes/facts in namespaces they're not authorized for
- Audit must be tamper-evident (pgAudit configured to log all access to sensitive tables)

**Key implementation notes:**
- Copy Loom's migrations verbatim for inherited tables (episodes, entities, facts, predicates)
- Create 6-8 new migration files:
  - `001_superloom_principal_trust.sql`
  - `002_superloom_namespaces.sql`
  - `003_superloom_role_classes.sql`
  - `004_episodes_principal_fields.sql`
  - `005_rls_policies.sql`
  - `006_audit_enhancements.sql`
- RLS policies must use `current_setting('superloom.principal_id')` context variable (set per connection)

**Completion criteria:**
- All 6-8 migrations apply cleanly in order
- `psql` can query `superloom_principal_trust`, `superloom_namespaces`, `superloom_role_classes`
- Test RLS by creating two principals, inserting namespace with one, verifying other cannot read
- `pgAudit` logs access events to `pgaudit.log`

---

#### 1.2 Stage 0 Gate: Skeleton + Hardcoded Threshold Table

Create `superloom-engine/src/gate/` module with the gate evaluation logic:

**Module structure:**
```
src/gate/
  ├── mod.rs              (public API + error types)
  ├── evaluator.rs        (core gate evaluation logic)
  ├── thresholds.rs       (hardcoded threshold table)
  └── types.rs            (CallerContext, GateOutcome, etc.)
```

**Core types (in `src/gate/types.rs`):**
```rust
pub struct CallerContext {
    pub principal_type: PrincipalType,  // human / orchestrator / agent
    pub principal_id: String,            // opaque ID
    pub role_class: String,              // validated against pack
    pub declared_task: String,           // task class name
}

pub struct GateInput {
    pub caller_context: CallerContext,
    pub requested_depth: CompilationDepth,  // episodic / semantic / procedural
    pub namespace: String,
}

pub enum GateOutcome {
    Pass { depth_authorized: CompilationDepth },
    Narrow { depth_authorized: CompilationDepth },
    Reject { reason: String },
}
```

**Threshold table (in `src/gate/thresholds.rs`):**
Hard-code a 3x3 matrix of (namespace_sensitivity, principal_trust_tier) → authorized_depth:

```rust
// Example (actual values will be in the spec or config)
Sensitivity::HIGH + TrustTier::UNVERIFIED → Depth::NONE (reject)
Sensitivity::HIGH + TrustTier::CORROBORATED → Depth::SEMANTIC
Sensitivity::HIGH + TrustTier::ESTABLISHED → Depth::EPISODIC
Sensitivity::MEDIUM + TrustTier::UNVERIFIED → Depth::PROCEDURAL
// ... etc
```

**Gate evaluator (in `src/gate/evaluator.rs`):**
```rust
pub async fn evaluate_stage_0(
    input: GateInput,
    pools: &DbPools,
) -> Result<GateOutcome, GateError> {
    // 1. Look up principal in superloom_principal_trust, default to UNVERIFIED
    // 2. Validate role_class against role_classes table for namespace's pack
    // 3. Read namespace sensitivity_class and min_confidence_threshold
    // 4. Look up in threshold matrix
    // 5. Return GateOutcome with full audit context
}
```

**No frontier routing yet.** The gate is purely pass/narrow/reject; no fallback to frontier models.

**Completion criteria:**
- `gate::evaluate_stage_0()` takes a `GateInput` and returns `Result<GateOutcome>`
- Hardcoded threshold table in code (not in database)
- Unit tests cover all 9 cells of the matrix
- Gate evaluation runs in <1ms (critical for online pipeline latency)
- Compiles cleanly, no unused warnings

---

### Test Strategy for Phase 1

**Unit tests:**
- Schema: migration ordering, idempotency, RLS policy correctness
- Gate: threshold matrix coverage, error handling, principal not found (defaults to UNVERIFIED)

**Integration tests:**
- Create test database, apply migrations, verify schema exists
- Insert test principals + namespaces, verify RLS blocks unauthorized reads
- Call gate with various (sensitivity, trust_tier) combos, verify outcome matches matrix

**Manual validation:**
- Deploy to local Docker Compose, seed test data via psql
- Call gate via Rust test harness
- Query audit log to confirm policy decisions are logged

---

## Phase 2: Confidence Receipt + Online Pipeline Integration

**Timeline estimate:** 3-4 weeks  
**Gating milestone:** Stage 0 must demonstrably narrow/reject requests against real declarations

### Milestones

#### 2.1 Confidence Receipt Type + Serialization

Create `src/types/confidence_receipt.rs`:

```rust
pub struct ConfidenceReceipt {
    pub compilation_depth_authorized: CompilationDepth,
    pub gate_outcome: GateOutcome,
    pub principal_trust_tier: TrustTier,
    pub facts: Vec<FactCorroborationInfo>,
    pub namespace_sensitivity_class: String,
    pub namespace_confidence_floor_for_depth: f32,
    pub overall_compilation_note: String,
}

pub struct FactCorroborationInfo {
    pub fact_id: Uuid,
    pub corroboration_count: i32,
    pub independent_source_count: i32,
    pub most_recent_corroboration_age_days: i32,
    pub active_contradiction: bool,
    pub sole_source_flag: bool,
}
```

This receipt is the only artifact returned alongside the compiled context package on `loom_think`.

**Completion criteria:**
- `ConfidenceReceipt` serializes to JSON per the spec example
- Unit tests verify all fields are present and properly typed
- Integrates with existing `ThinkResponse` type from Loom

---

#### 2.2 Stage 0 Gate Integration into Online Pipeline

Modify `src/api/mcp.rs::handle_loom_think()` to call Stage 0 gate before retrieval:

**New flow:**
```
handle_loom_think
  ↓
Extract caller_context from request headers or body
  ↓
Call gate::evaluate_stage_0(GateInput) ← NEW
  ↓
  If Reject: log audit, return confidence receipt with Reject outcome, no context package
  If Narrow: proceed to online pipeline with narrow depth
  If Pass: proceed to online pipeline with full depth
  ↓
[Existing online pipeline: classify → retrieve → weight → rank → compile]
  ↓
[NEW] Generate confidence receipt with per-fact corroboration data
  ↓
Return context package + confidence receipt + compilation_id
```

**Key implementation:**
- Extract `CallerContext` from request. Options:
  - JSON body field (if sent by agent or orchestrator)
  - Header (if sent by gateway/IAM layer)
  - Default to anonymous human principal if absent (for backward compatibility with Loom)
- Call gate synchronously (must complete in <1ms)
- If gate rejects, return empty context package with Reject receipt (not an HTTP error)
- Log full gate decision to audit log: inputs, outcome, threshold comparison

**Schema for per-fact corroboration:**
The ranking stage already has access to `source_episodes`. Extend this to count:
- How many independent principals have contributed to this fact's provenance
- How many times this fact has been corroborated vs. contradicted
- Store this in a new `superloom_fact_corroboration` table or as a view

**Completion criteria:**
- `handle_loom_think` calls gate before retrieval
- Gate rejection returns 200 OK with Reject receipt (not HTTP error)
- Confidence receipt includes per-fact corroboration counts
- Audit log captures full gate decision
- End-to-end latency impact <10ms (gate is fast, RLS check adds latency)
- Integration test: send request with low-trust principal for high-sensitivity namespace, verify Reject

---

#### 2.3 Per-Fact Corroboration Tracking

Add corroboration metadata to the ranking stage:

**New database table:**
```sql
CREATE TABLE superloom_fact_corroboration (
  fact_id UUID PRIMARY KEY REFERENCES loom_facts(id),
  corroboration_count INT DEFAULT 0,
  independent_source_count INT DEFAULT 0,
  most_recent_corroboration_at TIMESTAMPTZ,
  active_contradiction BOOLEAN DEFAULT false,
  sole_source_flag BOOLEAN DEFAULT false,
  last_updated_at TIMESTAMPTZ DEFAULT now()
);
```

**Ranking stage changes (in `src/pipeline/online/rank.rs`):**
- When ranking candidates, query `superloom_fact_corroboration` for each fact
- Integrate corroboration count into scoring formula (new dimension)
  - More corroborations = higher score (independent verification)
  - Active contradiction = lower score
  - Sole source = slightly lower score (less reliable)
- Return corroboration data in the `selected_items` JSONB for audit log

**Completion criteria:**
- Per-fact corroboration query runs in parallel with existing retrieval
- Corroboration count affects final ranking (verifiable in test)
- Corroboration data flows through to audit log and confidence receipt
- Unit test: mock 2 independent sources for fact A, 1 source for fact B, verify A ranks higher

---

#### 2.4 Gate Decision Logging + Audit Trail

Extend `superloom_audit_log` with gate-specific columns:

```sql
ALTER TABLE superloom_audit_log ADD COLUMN (
  principal_id TEXT,
  principal_type TEXT,
  role_class TEXT,
  declared_task TEXT,
  gate_outcome TEXT,
  gate_reason TEXT,
  compilation_depth_authorized TEXT,
  confidence_receipt JSONB
);
```

Every `loom_think` call logs:
- Stage 0 inputs (principal_type, principal_id, role_class, declared_task, requested_depth)
- Stage 0 outcome (pass/narrow/reject, specific threshold comparison)
- Full confidence receipt as JSONB
- Existing pipeline telemetry (classification, retrieval, ranking, compilation)

**Completion criteria:**
- Audit entries include gate decision + confidence receipt
- Dashboard can query gate decision history per principal or namespace
- Test verifies audit log captures both Accept and Reject outcomes

---

### Test Strategy for Phase 2

**Unit tests:**
- Confidence receipt JSON serialization
- Gate → online pipeline routing logic
- Per-fact corroboration scoring impact

**Integration tests:**
- End-to-end `loom_think` with gate enabled
- Gate rejects low-trust principal for high-sensitivity namespace
- Confidence receipt properly populated with per-fact corroboration data
- Audit log captures full decision trail

**Manual acceptance criteria (GATING MILESTONE):**
- Deploy Phase 1+2 to local Docker Compose
- Create two namespaces: one high-sensitivity, one low
- Create two principals: one unverified, one corroborated
- Seed data: high-confidence facts via human seed, low-confidence via agent observation
- Run loom_think queries with various (namespace, principal) combos
- **Verify in dashboard that gate demonstrably narrows or rejects** (not just passing everything through)
- Gate must actually change behavior based on configuration, or Phase 3 does not start

---

## Phase 3: Agent Observation Ingestion + Trust Calibration

**Timeline estimate:** 3-4 weeks  
**Prerequisite:** Phase 2 gate is working and actually filtering

### Milestones

#### 3.1 Agent Observation Ingestion Mode

Extend `src/types/ingestion.rs`:

```rust
pub enum IngestionMode {
    UserAuthoredSeed,
    VendorImport,
    LiveMcpCapture,
    AgentObservation,  // ← NEW
}
```

Modify `src/api/mcp.rs::handle_loom_learn()` to accept agent-formatted payloads:

**Agent observation payload (new request variant):**
```json
{
  "content": "{\"tool_call\": \"query_database\", \"result\": {...}}",
  "source": "agent-id-xyz",
  "namespace": "production_incidents",
  "ingestion_mode": "agent_observation",
  "principal_id": "agent-id-xyz",
  "occurred_at": "2025-06-24T10:15:00Z",
  "metadata": {
    "agent_instance": "incident-responder-v2",
    "confidence": 0.95
  }
}
```

**Validation:**
- `principal_id` is required for agent_observation (reject if missing)
- `content` must be non-empty (cannot be LLM summarization, but this is trust-based)
- Payload shape validated against agent-specific schema if available

**Completion criteria:**
- `loom_learn` accepts agent_observation mode
- Principal_id is required and validated
- Ingestion mode is enforced in database constraint
- Integration test: POST agent observation, verify episode enters table with `ingestion_mode='agent_observation'` and `principal_id` set

---

#### 3.2 Corroboration/Contradiction Detection

Create `src/pipeline/offline/trust_calibration.rs`:

**New module functions:**
```rust
pub async fn detect_corroboration_or_contradiction(
    newly_ingested_episode_id: Uuid,
    pools: &DbPools,
) -> Result<TrustDelta, TrustCalibrationError> {
    // 1. Extract facts from the new episode
    // 2. Search for existing facts that match these entities/relationships
    // 3. For each match:
    //    - Same fact (entities + predicate + object) = CORROBORATION
    //    - Contradicting fact (superseded_by set) = CONTRADICTION
    // 4. Return TrustDelta { corroboration_count, contradiction_count }
}

pub struct TrustDelta {
    pub corroboration_count: i32,
    pub contradiction_count: i32,
}
```

**Integration into offline pipeline (in `src/pipeline/offline/ingest.rs`):**
- After extraction, if episode is agent_observation mode, call `detect_corroboration_or_contradiction`
- Do NOT update trust tier synchronously; queue for batch recalculation

**Completion criteria:**
- Unit test: ingest episode corroborating existing fact, verify corroboration_count increments
- Unit test: ingest episode contradicting existing fact, verify contradiction_count increments
- Integration test: agent observation corroborates human-seeded fact, trust delta captured in audit

---

#### 3.3 Scheduled Trust Tier Recalculation

Create scheduled offline task in `src/worker/scheduler.rs`:

**Scheduled task (runs every 6 hours, configurable):**
```rust
pub async fn recalculate_trust_tiers(pools: &DbPools) -> Result<TrustRecalcResult, WorkerError> {
    // For each principal in superloom_principal_trust:
    //   1. Sum corroboration_count and contradiction_count from all episodes
    //   2. Compute trust_tier:
    //      - unverified: 0 corroborations
    //      - corroborated: 1+ corroborations, no contradictions (or ratio > 5:1)
    //      - established: 5+ corroborations, multiple independent sources, ratio > 10:1
    //   3. Update superloom_principal_trust with new tier and last_recalculated_at
    //   4. Log to superloom_audit_log
}

pub struct TrustRecalcResult {
    pub principals_processed: i32,
    pub tier_promotions: i32,
    pub tier_demotions: i32,
}
```

**Recalculation logic:**
- Trust tier is NOT a single function of counts. Thresholds are:
  - `unverified`: 0 total corroborations (default for new principals)
  - `corroborated`: ≥1 corroboration, corroboration:contradiction ratio ≥ 5:1
  - `established`: ≥5 corroborations, ≥2 independent sources, ratio ≥ 10:1
- Recalculation is offline, not synchronous, so gate uses trust tier as of last sweep
- New tier is available at next gate evaluation after sweep completes

**Dormancy & Drift Sweep (bonus):**
- Scan namespaces for fact staleness (no updates in 90 days)
- Scan principals for activity staleness (no new observations in 180 days)
- Log dormancy alerts to dashboard
- (Optional for v1, but mentioned in spec; flag as "defer if time-constrained")

**Completion criteria:**
- Scheduled task runs every 6 hours by default
- Test: create 6 agent observations from same principal, alternating corroboration/contradiction, verify trust tier progression
- Audit log captures recalculation results per principal
- Dashboard can view trust tier timeline for a given principal

---

### Test Strategy for Phase 3

**Unit tests:**
- Corroboration detection: same fact, contradicting fact, unrelated fact
- Trust tier recalculation: all three tier transitions
- Dormancy sweep: namespace staleness threshold

**Integration tests:**
- Agent observation ingestion through full offline pipeline
- Corroboration increments trust tier over time
- Contradiction prevents tier advancement
- Dormancy sweep flags stale namespaces

**Manual acceptance criteria:**
- Deploy Phases 1-3 to Docker Compose
- Create agent principal, submit 10 agent observations
- Half corroborate human-seeded facts, half contradict them
- Wait for scheduled trust recalculation (or trigger manually)
- Verify trust tier reflects corroboration/contradiction ratio
- Dashboard shows trust tier history with decision rationale

---

## Phase 4: Inference Routing (Local-First, Frontier Escalation)

**Timeline estimate:** 2-3 weeks  
**Prerequisite:** Phase 2 (must have working online pipeline)

### Milestones

#### 4.1 Local-First Inference Default

Modify `src/llm/client.rs` and extraction/classification logic:

**Current state (from Loom):**
- All classification and extraction runs via Ollama (local)
- Only frontier fallback is hardcoded for specific models

**Super Loom change:**
- Explicitly route all classification and extraction requests to Ollama
- Log every classification/extraction attempt with model name and outcome
- Capture confidence scores from local model

**Implementation:**
- In `src/llm/client.rs`, add routing decision enum:
  ```rust
  enum InferenceRoute {
      Local { model: String },
      Frontier { model: String, reason: EscalationReason },
  }
  
  enum EscalationReason {
      ConfidenceBelowThreshold { confidence: f32, threshold: f32 },
      PolicyDecision,
      // others as needed
  }
  ```
- Classification and extraction always route to Local first
- If local confidence is below a namespace-configured threshold, then evaluate frontier escalation

**Completion criteria:**
- All classification/extraction routes to Ollama by default
- Logging captures route decision for every inference
- No changes to local model behavior, just explicit routing

---

#### 4.2 Frontier Escalation Policy (Compiled Package Only)

Create `src/inference/escalation_policy.rs`:

**Escalation decision:**
```rust
pub async fn should_escalate_to_frontier(
    namespace: &str,
    task_type: TaskType,  // classification or extraction
    local_confidence: f32,
    policy_db: &PgPool,
) -> Result<EscalationDecision, PolicyError> {
    // 1. Query namespace's escalation_threshold for this task_type
    // 2. Compare local_confidence against threshold
    // 3. Return Allow (with compiled package) or Deny (route to human review)
}

pub enum EscalationDecision {
    Allow { model: String, payload: CompiledPackage },
    Deny { reason: String },
}
```

**Payload restriction:**
- If escalation is approved, ONLY send the compiled context package (never raw episodes)
- Compiled package has already been filtered by Stage 0 gate and depth narrowing
- Log the escalation with package size, namespace sensitivity, and principal trust tier

**Policy storage (new database table):**
```sql
CREATE TABLE superloom_inference_escalation_policy (
  namespace TEXT,
  task_type TEXT CHECK (task_type IN ('classification', 'extraction')),
  escalation_threshold_confidence FLOAT,
  frontier_model_allowed TEXT[],  -- e.g. ['gpt-4-turbo', 'claude-opus']
  created_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (namespace, task_type)
);
```

**Hard boundary enforcement:**
- Code inspection: verify no call path sends raw `episodes.content` to frontier models
- Every escalation request must include `compiled_package` parameter, never raw fields
- Test fails if raw content is detected in any frontier request

**Completion criteria:**
- Escalation policy queryable from database
- Escalation decision logged to audit trail
- Test: low local confidence + high threshold = Deny escalation, flag for human review
- Test: low local confidence + low threshold = Allow escalation with compiled package only
- Code review verifies no raw episode content can reach frontier model

---

#### 4.3 Ollama Embedding Consistency

Ensure embeddings remain local-only:

**Current behavior:**
- Embeddings via Ollama's nomic-embed-text (or similar local model)
- Used for cosine similarity search in retrieval

**Super Loom requirement:**
- No embedding content ever crosses to frontier model
- Document this as an immutable invariant (not changeable per policy)
- Test: verify embedding route is always local, audit logs confirm

**Completion criteria:**
- Embeddings always route to Ollama
- Audit log captures embedding model name and latency
- Test covers the embedding path (cannot be overridden by config)

---

### Test Strategy for Phase 4

**Unit tests:**
- Escalation policy evaluation: all threshold scenarios
- Inference route selection: local vs. frontier logic

**Integration tests:**
- Classification with various confidence scores, verify escalation decision
- Escalation payload contains compiled package only, never raw content
- Audit log captures escalation decision + reason + payload model

**Manual acceptance criteria:**
- Deploy Phases 1-4 to Docker Compose
- Configure escalation threshold for a namespace
- Run classification against boundary condition (confidence just below threshold)
- Verify escalation decision appears in audit and is logged correctly
- Inspect network requests (via tcpdump or proxy): no raw episode content reaches frontier model

---

## Phase 5: Declaration Tooling + Dashboard

**Timeline estimate:** 3-4 weeks  
**Prerequisite:** Phases 1-4 complete

### Milestones

#### 5.1 Namespace & Role Class Declaration CLI

Create `superloom-cli` (Rust binary or Python wrapper):

**CLI commands:**

```bash
superloom-cli namespace declare \
  --namespace production_incidents \
  --sensitivity HIGH \
  --reason "Contains customer incident details" \
  --governing-pack core \
  --confidence-episodic 0.85 \
  --confidence-semantic 0.65 \
  --confidence-procedural 0.40

superloom-cli role-class declare \
  --role-class incident_responder \
  --pack core \
  --description "On-call responder with access to raw incidents" \
  --default-min-trust-tier corroborated
```

**Declaration record generation:**
- Creates `declaration_artifact` JSONB in database
- Includes CLI invocation timestamp, operator identity (from env var), full arguments
- Artifact is immutable; updates create new records
- Dashboard shows full audit trail of declaration changes

**Validation:**
- Namespace must not already exist (no overwrites, only new declarations)
- Role class must belong to declared pack
- Sensitivity class must be from vocabulary (or check against hardcoded enum)
- Confidence thresholds must be 0.0-1.0 and episodic ≥ semantic ≥ procedural

**Completion criteria:**
- CLI commands produce correct database entries
- Declaration artifacts capture full context
- Test: declare namespace, verify artifact in database with all fields
- Test: reject invalid sensitivity class

---

#### 5.2 Dashboard Views (Minimal Web UI)

Create `superloom-dashboard` (React + Vite, reusing Loom patterns):

**Views to implement:**

1. **Gate Decision Log**
   - Timeline of pass/narrow/reject decisions
   - Filter by principal, namespace, outcome
   - Show gate inputs (principal_type, role_class, declared_task)
   - Show gate outcome + specific threshold that was applied

2. **Principal Trust History**
   - Per principal_id, show trust tier progression
   - Corroboration/contradiction count timeline
   - Last seen, first seen, activity pattern
   - Confidence receipt inspector (drill into specific compilation)

3. **Namespace Registry Browser**
   - List all declared namespaces
   - Show sensitivity_class, declared by, declared at
   - Show confidence thresholds per depth
   - Show governing pack
   - List role classes associated with pack

4. **Confidence Receipt Inspector**
   - View full receipt for a given compilation_id
   - Show per-fact corroboration data
   - Highlight sole sources, active contradictions
   - Show gate decision that led to this receipt

5. **Policy Viewer** (stretch, optional)
   - Current inference escalation policies
   - Threshold matrix (sensitivity × trust_tier → depth)
   - Declaration artifacts (namespace & role class history)

**Technical approach:**
- Reuse Loom's dashboard stack (React, Vite, TypeScript)
- API endpoints feed data from `superloom_audit_log` and related tables
- No real-time SSE yet (Phase 1 can do this; not critical for v1)
- Static file serving via caddy

**Completion criteria:**
- Gate Decision Log shows real decisions from Phase 2 testing
- Principal Trust History shows trust tier progression from Phase 3
- Namespace Registry Browser lists declared namespaces
- All views render without errors
- Basic filtering/sorting works

---

#### 5.3 Dashboard API Endpoints

Create REST endpoints in `src/api/dashboard.rs` (extending Loom pattern):

```rust
GET /dashboard/gate-decisions?principal_id=&namespace=&outcome=&limit=100
GET /dashboard/principal/:principal_id/trust-history
GET /dashboard/namespaces
GET /dashboard/namespace/:namespace
GET /dashboard/role-classes
GET /dashboard/compilation/:compilation_id/receipt
GET /dashboard/inference-policies
```

**Each endpoint:**
- Queries audit log or related tables
- Returns JSON
- Includes pagination/sorting
- Requires bearer token (same as loom_think)

**Completion criteria:**
- All endpoints return 200 OK with expected schema
- Integration test: POST gate decision, query via endpoint, verify data
- Performance: queries complete in <500ms for reasonable data sizes

---

### Test Strategy for Phase 5

**Unit tests:**
- CLI argument parsing and validation
- Declaration artifact generation

**Integration tests:**
- CLI declares namespace, query dashboard API, verify data appears
- Gate decision appears in dashboard within 1 second
- Dashboard queries return correct data for various filters

**Manual acceptance criteria:**
- Deploy full Phases 1-5 to Docker Compose
- CLI declare namespace, verify in dashboard
- Run loom_think queries, gate decisions appear in timeline
- Create agent principal, submit observations, trust tier increments, appears in dashboard
- Confidence Receipt Inspector shows per-fact corroboration
- Navigate through all dashboard views, all data renders correctly

---

## Rust Codebase Module Structure

```
superloom-engine/
├── Cargo.toml                    (add casbin, governor crates)
├── migrations/
│   ├── 001_superloom_principal_trust.sql
│   ├── 002_superloom_namespaces.sql
│   ├── 003_superloom_role_classes.sql
│   ├── 004_episodes_principal_fields.sql
│   ├── 005_rls_policies.sql
│   └── 006_audit_enhancements.sql
├── src/
│   ├── main.rs                  (unmodified, spawns new worker tasks)
│   ├── lib.rs                   (re-exports gate, inference modules)
│   ├── gate/                    (NEW — Phase 1-2)
│   │   ├── mod.rs
│   │   ├── evaluator.rs         (stage 0 gate logic)
│   │   ├── thresholds.rs        (hardcoded matrix)
│   │   └── types.rs             (CallerContext, GateOutcome)
│   ├── inference/               (NEW — Phase 4)
│   │   ├── mod.rs
│   │   ├── escalation_policy.rs (frontier escalation decision)
│   │   └── routing.rs           (local vs. frontier routing)
│   ├── pipeline/
│   │   ├── offline/
│   │   │   ├── mod.rs           (unmodified)
│   │   │   └── trust_calibration.rs  (NEW — Phase 3)
│   │   └── online/
│   │       ├── mod.rs           (add gate call)
│   │       └── corroboration.rs (NEW — Phase 2, per-fact tracking)
│   ├── api/
│   │   ├── mcp.rs               (integrate gate into loom_think)
│   │   ├── dashboard.rs         (extend with Phase 5 endpoints)
│   │   └── rest.rs              (no changes)
│   ├── types/
│   │   ├── confidence_receipt.rs (NEW — Phase 2)
│   │   └── mcp.rs               (extend ThinkRequest with CallerContext)
│   ├── worker/
│   │   ├── scheduler.rs         (add trust recalculation task — Phase 3)
│   │   └── processor.rs         (integrate trust calibration — Phase 3)
│   └── db/
│       ├── pool.rs              (runs migrations, no changes to logic)
│       └── audit.rs             (extend with gate columns — Phase 2)
└── tests/
    ├── gate_evaluation.rs       (Phase 1-2 tests)
    ├── agent_observation.rs     (Phase 3 tests)
    ├── inference_routing.rs     (Phase 4 tests)
    └── e2e_scenarios.rs         (integration tests across phases)
```

**New Cargo dependencies (Phase 4):**
```toml
casbin = "2.0"              # Policy evaluation engine (or equivalent)
governor = "0.10"           # Rate limiting for per-principal request shaping
```

---

## Architectural Decision Points Requiring Early Validation

### 1. **Policy Engine Selection (Phase 1, Week 1)**
- **Decision:** Use `casbin` crate for gate policy evaluation vs. hand-coded threshold matrix
- **Trade-off:** Casbin is more flexible (load policies from file), hand-coded is simpler to verify and audit
- **Recommendation:** Start with hand-coded matrix (Phase 1), migrate to casbin later if needed
- **Validation:** Implement gate with hardcoded matrix, measure evaluation latency, confirm <1ms

### 2. **RLS Performance Overhead (Phase 1, Week 2)**
- **Decision:** Test row-level security performance on PostgreSQL before full rollout
- **Risk:** RLS can add 5-15% query latency per policy; cumulative impact unknown
- **Validation:** Create test database with 100K episodes + 3 RLS policies, benchmark query latency
- **Exit criteria:** RLS overhead <5% on queries, OR redesign to application-layer filtering

### 3. **Trust Tier Thresholds (Phase 3, Week 1)**
- **Decision:** Hardcode thresholds (1 corroboration → corroborated, 5 → established) vs. configurable
- **Risk:** Wrong thresholds make gate useless (too permissive) or paralyzing (too strict)
- **Validation:** During Phase 3, collect metrics on actual corroboration/contradiction rates in test data
- **Exit criteria:** Adjust thresholds based on observed distributions before Phase 4

### 4. **Frontier Escalation Scope (Phase 4, Week 1)**
- **Decision:** Allow escalation only for classification/extraction, never for compilation or retrieval
- **Risk:** Pressure to escalate retrieval or compilation to frontier models
- **Validation:** Code review checklist: every frontier callsite must be reviewed for raw content
- **Exit criteria:** Documented enforcement point, automated test that fails if raw content reaches frontier

### 5. **Audit Log Volume (All phases)**
- **Decision:** How much detail to log for every gate decision, every corroboration check, every escalation
- **Risk:** Audit table can grow to 100M+ rows (1 row per loom_think call)
- **Validation:** Measure table size after 1 week of moderate load (100 loom_think/min)
- **Exit criteria:** Implement table partitioning if >10GB, or add retention policy

---

## Testing Strategy by Phase

### Phase 1: Schema + Gate Skeleton
- **Unit:** Schema idempotency, gate matrix coverage
- **Integration:** RLS blocks unauthorized reads, gate latency <1ms
- **Manual:** Local Docker Compose deployment, seed data, verify gate outputs
- **Acceptance:** Gate evaluates correctly against hardcoded matrix

### Phase 2: Confidence Receipt + Online Pipeline
- **Unit:** Receipt JSON structure, gate → pipeline routing
- **Integration:** End-to-end loom_think with gate enabled
- **Manual:** Dashboard shows gate decisions, confidence receipt visible
- **GATING CRITERION:** Gate demonstrably narrows or rejects requests (not pass-through)

### Phase 3: Agent Observation + Trust Calibration
- **Unit:** Corroboration/contradiction detection, trust tier recalculation
- **Integration:** Agent observation ingestion, trust tier progression over time
- **Manual:** Dashboard shows trust tier history, trust ledger increments match expectations
- **Acceptance:** Trust system works end-to-end, tiers advance based on evidence

### Phase 4: Inference Routing
- **Unit:** Escalation policy evaluation, route selection
- **Integration:** Classification with various confidence levels, verify escalation decision
- **Manual:** Network inspection confirms no raw content to frontier model
- **Acceptance:** Escalation policy enforced, boundary is hard

### Phase 5: Declaration Tooling + Dashboard
- **Unit:** CLI argument parsing, declaration artifact generation
- **Integration:** CLI → database → dashboard API → UI
- **Manual:** Declare namespace, view in dashboard, data consistent
- **Acceptance:** All dashboard views render, data is accurate and auditable

---

## Completion Criteria & Gating

| Phase | Gating Criterion | What Must Be True |
|-------|------------------|------------------|
| **1** | Schema passes migrations, gate evaluates in <1ms | All tables exist, RLS policies in place, gate function callable |
| **2** | **Gate actually filters requests** | Real namespace sensitivity declarations, gate narrows/rejects low-trust principals for high-sensitivity namespaces, audit log confirms decisions |
| **3** | Trust system advances tiers based on evidence | Agent principals' trust tiers increment with corroboration, decrement with contradiction, recalculation runs on schedule |
| **4** | No raw content crosses to frontier | Code inspection + automated test prove only compiled packages escalate, embeddings remain local |
| **5** | Dashboard shows operational data | Gate timeline, trust history, namespace registry all visible and accurate |

**If gate is a pass-through (Phase 2 criterion fails):**
- Do NOT proceed to Phase 3 until gate is a real gate
- Investigate why gate is not filtering (common causes: threshold too low, test data missing, principal_id not flowing through)
- Phase 2 is the critical inflection point; fix before moving forward

---

## Prototyping vs. Incremental Build

| Component | Strategy |
|-----------|----------|
| **Stage 0 Gate** | Spike/prototype first (1-2 days) with hardcoded matrix, validate latency <1ms, then productionize with proper error handling |
| **RLS Policies** | Prototype on test database before applying to production schema; measure performance overhead |
| **Corroboration Detection** | Prototype with simple string matching first (same subject/predicate/object), then add entity resolution |
| **Trust Tier Recalculation** | Prototype with manual trigger (CLI command), then add scheduler |
| **Inference Escalation** | Prototype by adding log statements first (no actual frontier calls), then route to real frontier model |
| **Dashboard** | Start with basic table views (no filtering), add filters/drill-down incrementally |

---

## Risk & Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **RLS adds unacceptable latency** | Online pipeline latency SLA breached | Benchmark Phase 1; redesign to app-layer if needed |
| **Gate thresholds are wrong** | System is either useless (passes everything) or paralyzing (rejects everything) | Validate thresholds during Phase 3 with real data; keep configurable |
| **Raw episode content leaks to frontier** | Data sovereignty compromise, catastrophic | Enforce at code-review gate; automated test in CI; code inspection checklist |
| **Audit table explodes** | Operational crisis (disk full, slow queries) | Implement table partitioning or retention policy from start |
| **Trust system is gamed by collusion** | Multiple agents claim same principal_id, earn trust together | Document as known limitation (IAM responsibility); log anomalies |
| **Gate decision latency is >10ms** | Online pipeline latency impact unacceptable | Profile gate evaluation; optimize or defer to background |

---

## Deferred Capabilities (Not in v1)

Per spec, explicitly deferred to later phases:
1. **CRG integration** — Super Loom produces signals (receipts), CRG produces routing decisions
2. **Cross-namespace synthesis** — Intentionally hard in v1; revisit after gate usage patterns emerge
3. **IAM integration** — Identity verification belongs to infrastructure outside Super Loom
4. **Variance monitoring** — Gate decision anomaly detection (adapted from CRG's variance mechanics)
5. **Multi-region deployment** — v1 targets single deployment per organization

These are designed as future extensions, not as gaps to be filled in v1.

---

## Summary: 5-Phase Roadmap

```
Week 1-3:   Phase 1  → Schema + Gate Skeleton
Week 4-7:   Phase 2  → Confidence Receipt + Online Pipeline [GATING: Gate must filter]
Week 8-11:  Phase 3  → Agent Observation + Trust Calibration
Week 12-14: Phase 4  → Inference Routing [GATING: No raw content to frontier]
Week 15-18: Phase 5  → Declaration Tooling + Dashboard [SHIP v1]
```

**Total estimate:** 18 weeks (~4.5 months) for a team of 1 full-time engineer.

---

## Critical Files for Implementation Reference

- `/Users/jason/Projects/project-loom/loom-engine/migrations/001_episodes.sql` — Reference schema structure
- `/Users/jason/Projects/project-loom/loom-engine/migrations/004_predicates.sql` — Pack-based governance pattern
- `/Users/jason/Projects/project-loom/loom-engine/src/api/mcp.rs` — MCP handler pattern for integration point
- `/Users/jason/Projects/project-loom/loom-engine/src/pipeline/online/mod.rs` — Online pipeline architecture
- `/Users/jason/Projects/project-loom/loom-engine/src/pipeline/offline/ingest.rs` — Episode ingestion pattern
- `/Users/jason/Projects/project-loom/loom-engine/src/types/mcp.rs` — Request/response type definitions
- `/Users/jason/Projects/project-loom/loom-engine/src/db/pool.rs` — Database initialization and migrations
- `/Users/jason/Projects/project-loom/loom-engine/Cargo.toml` — Dependency baseline
