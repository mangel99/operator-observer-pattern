# Operator–Observer Pattern for AI Systems

**Category:** AI Orchestration & Reliability Pattern  
**Status:** Draft (v0.9) — Updated August 19, 2025  
**Audience:** AI engineers, platform architects, MLOps/LLMOps

## 1) Purpose

Define an architectural pattern for **AI agent-operated systems** that:
1) **Use** tools/platforms (“the plant”) to produce a result (e.g., building an app).  
2) **Observe** the process and its signals (metrics, logs, validations).  
3) **Decide** whether a problem is at **app level** or **motor/plant level**.  
4) **Act**: fix locally or **self‑improve the motor**, and **resume** without starting over.

## 2) Scope

- Applies to *generative pipelines* (apps, data, documentation, code).  
- Multi‑provider and multi‑agent (e.g., GPT/Claude/Gemini + sub‑agents).  
- Compatible with external orchestrators (CLI, agent buses, CI runners).

## 3) Terms & Roles

- **App‑Context**: state, artifacts, and signals of the *specific task*.  
- **Motor‑Context**: state of the *plant/motor*: prompts, templates, rules, validators.  
- **Operator (AI‑Operator)**: supervisory agent that arbitrates—classifies incidents, decides plan of action, and coordinates pause‑patch‑resume.  
- **Observers**: sub‑agents/sensors that collect signals (metrics, validation, traces, costs) and deliver them to the Operator.  
- **Tooling/Plant**: the executing tool/platform (e.g., an app factory).

## 4) Problem Statement

Failures mix **app** and **motor** symptoms. Without arbitration, the system retries blindly: fixing the app when rules are missing in the motor, or touching the motor when the problem is app‑specific. Result: **high cost, latency, instability**, and no learning.

## 5) Solution Overview

Insert an **Operator–Observer Layer** with **dual context**:
- **Observers** instrument both app‑context and motor‑context.  
- **Operator** consumes signals, **classifies** (app vs. motor vs. mixed), **decides** and **executes**:  
  - *App‑Fix*: local patch (prompt, spec, mapping, dependency).  
  - *Motor‑Fix*: patch of rules/templates/validators with documented feedback.  
  - *Mixed*: safe sequence (e.g., motor then app).  
- **Resumability**: pause, patch, validate, **resume** from checkpoints.  
- **Learning Loop**: motor fixes are consolidated (versioned, changelogged) to benefit future runs.

## 6) Non‑Goals

- Not tied to one AI vendor or tool.  
- Does not prescribe prompt/template formats—only **minimal interfaces**.

---

## 7) Normative Requirements (RFC‑style)

### 7.1 Context Separation
- The system **MUST** persist **App‑Context** and **Motor‑Context** separately.  
- The system **MUST** version both contexts (semantic ID + artifact hash).  
- The system **SHOULD** allow diffs and checkpoints for resumption.

### 7.2 Observation
- Observers **MUST** emit structured events:  
  `{ timestamp, scope: "app"|"motor", signal_type, severity, payload }`.  
- Minimal `signal_type` values: `validation`, `error`, `latency`, `cost`, `coverage`, `quality_score`.  
- Observers **SHOULD** attach `trace_id` and `artifact_ids`.

### 7.3 Classification & Arbitration
- Operator **MUST** use a **failure taxonomy** (see §10).  
- Operator **MUST** produce a **decision record** for each incident.  
- For `mixed` cases, Operator **SHOULD** apply a deterministic order (e.g., motor → app).

### 7.4 Actuation
- For **app** classification: **MUST** apply local patches and retry with the **same motor version**.  
- For **motor** classification: **MUST** create a change request, run tests/validators, bump version if needed, and **resume** the app if validation passes.  
- The pipeline **MUST** be **resumable** post‑patch.

### 7.5 Learning Loop
- Motor fixes **MUST** be logged and linked to impact metrics.  
- The system **SHOULD** deduplicate incidents and promote learned patterns.

### 7.6 Safety & Governance
- **Motor changes MUST pass gates** (tests, validators, linting).  
- The system **MUST** maintain a rollback policy with restore points and retention windows.

---

## 8) Reference Interfaces

### 8.1 Event Envelope (Observers → Operator)
```json
{
  "trace_id": "uuid",
  "timestamp": "2025-08-19T11:35:12Z",
  "scope": "app | motor",
  "signal_type": "validation | error | latency | cost | coverage | quality_score",
  "severity": "info | warn | error | critical",
  "payload": { "details": "..." },
  "artifact_ids": ["..."],
  "context_ref": { "app_ctx": "id@ver", "motor_ctx": "id@ver" }
}
```

### 8.2 Operator Decision Record
```json
{
  "decision_id": "uuid",
  "classification": "app | motor | mixed",
  "rationale": "short explanation",
  "action_plan": [
    { "step": "pause", "target": "pipeline" },
    { "step": "apply_patch", "target": "motor|app", "patch_ref": "PR#123" },
    { "step": "validate", "target": "motor|app" },
    { "step": "resume", "target": "pipeline" }
  ],
  "safety_checks": ["tests:all", "validators:strict"]
}
```

### 8.3 SPI (Plant Interface)
```ts
export interface PlantRunRequest {
  appSpecRef: string;
  profileTargets: string[];
  motorVersion: string;
  runMode?: "fresh" | "resume";
  checkpoint?: string;
}

export interface PlantRunResult {
  traceId: string;
  status: "success" | "failed" | "partial";
  artifacts: ArtifactRef[];
  signals: EventEnvelope[];
  nextCheckpoint?: string;
}
```

---

## 9) Lifecycle & State Machine

**States:** `IDLE → RUNNING → PAUSED → PATCHING(MOTOR|APP) → VALIDATING → RESUMING → RUNNING → (SUCCESS|FAILED)`

---

## 10) Error Taxonomy

- **APP‑SPEC**: missing fields, invalid mappings.  
- **APP‑BUILD**: broken dependency, compilation failure, integration test error.  
- **MOTOR‑RULES**: wrong validator, incomplete template.  
- **MOTOR‑PERF**: latency/cost budget exceeded.  
- **MIXED‑DRIFT**: drift between motor version and app spec.

---

## 11) Decision Heuristics

- If reproduced in **≥ 2 apps** → classify as **MOTOR‑RULES**.  
- If the issue disappears when changing **profile/target** with the same motor → classify as **APP**.  
- If quality drops with a **new motor version** on the same app → classify as **MIXED‑DRIFT** and prioritize a motor patch.

---

## 12) Safety, Rollback & Versioning

- **MUST** use semantic versioning for motor changes.  
- **MUST** maintain a rollback plan (snapshot of motor + app checkpoints).  
- Prefer **safe roll‑forward**; if validation fails, perform **immediate rollback**.

---

## 13) Metrics & SLOs

- **App SLOs**: build success, quality score, end‑to‑end time, cost per build.  
- **Motor SLOs**: incident distribution by category, MTTR (motor fix), impact of changes.  
- **Operator KPIs**: classification accuracy (vs. ground truth), successful resumption rate, false motor‑fix rate.

---

## 14) Security & Compliance

- **MUST** isolate app vs. motor data; limit visibility as needed.  
- **SHOULD** redact/obfuscate sensitive fields in logs and events.  
- **MUST** enforce gated motor patches with optional human review (break‑glass controls).

---

## 15) Implementation Notes

- **Context Store**: transcripts, checkpoints, artifacts index, decision records.  
- **Operator**: implement as a policy engine with idempotent action runners.  
- **Observers**: wrappers around validators, compilers, test runners, cost/latency meters.  
- **Orchestration**: integrate with your agent bus; each Operator action is a job with retries.

---

## 16) Reference Topologies

- **Mono‑plant**: single motor, multiple profiles.  
- **Multi‑plant**: multiple motors (e.g., data‑plant, code‑plant) with a global Operator.  
- **Federated**: multiple teams/motors coordinated by a central Operator.

---

## 17) Testing Strategy

- **Golden Incidents**: curated dataset of labeled failures (APP/MOTOR/MIXED).  
- **Replay Harness**: re‑execute historical traces to validate new Operator versions.  
- **Chaos Drills**: inject controlled failures; measure classification and MTTR.  
- **Canary Motor Upgrades**: promote motor version to a subset and monitor.  
- **Contract Tests**: for the Plant SPI.

---

## 18) Anti‑Patterns

- Mixing contexts (app artifacts inside motor context and vice versa).  
- “Mute observer”: collecting signals without an Operator to arbitrate.  
- Ungoverned motor fixes (no tests, no versioning, no resumability).  
- Blind retries when the problem is systemic.

---

## 19) Compliance Checklist

- [ ] App/Motor contexts separated and versioned.  
- [ ] Observers emit structured events with `trace_id`.  
- [ ] Operator produces decision records with rationale and plan.  
- [ ] Pipeline is resumable after patches.  
- [ ] Motor changelog with impact metrics.  
- [ ] Rollback is tested (table‑top exercise).

---

## 20) Example Flow

1) An observer emits a validation error during an app run.  
2) The operator finds similar incidents across multiple apps and classifies it as **MOTOR‑RULES**.  
3) The motor is patched, validated, and the version is bumped.  
4) The app resumes from its checkpoint and completes successfully.  
5) The decision record and impact metrics are logged.
