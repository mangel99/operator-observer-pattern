# Operator--Observer Pattern for AI Systems

**Category:** AI Orchestration & Reliability Pattern\
**Status:** Draft (v0.9)\
**Audience:** AI engineers, platform architects, MLOps/LLMOps

## 1) Purpose

Define an architectural pattern for **AI agent-operated systems**
that: 1) **Use** tools/platforms ("the plant") to produce a result
(e.g., building an app).\
2) **Observe** the process and its signals (metrics, logs,
validations).\
3) **Decide** whether a problem is at **app-level** or
**motor/plant-level**.\
4) **Act**: fix locally or **self-improve the motor**, and **resume**.

## 2) Scope

-   Applies to *generative pipelines* (apps, data, documentation,
    code).\
-   Multi-provider and multi-agent (e.g., GPT/Claude/Gemini +
    subagents).\
-   Compatible with external orchestrators (CLI, bus of agents, CI
    runners).

## 3) Terms & Roles

-   **App‑Context**: State, artifacts, and signals of the *specific
    task*.\
-   **Motor‑Context**: State of the *plant/motor*: prompts, templates,
    rules, validators.\
-   **Operator (AI‑Operator)**: Supervisory agent that arbitrates:
    classifies incidents, decides plan of action, coordinates
    pause‑patch‑resume.\
-   **Observers**: Subagents/sensors that collect signals (metrics,
    validation, traces, costs) and deliver them to the Operator.\
-   **Tooling/Plant**: The executing tool/platform (e.g., app factory).

## 4) Problem Statement

Failures mix **app** and **motor** symptoms. Without arbitration, the
system retries blindly: fixing the app when rules are missing in the
motor, or touching the motor when the problem is app-specific. Result:
**high cost, latency, instability**, and no learning.

## 5) Solution Overview

Insert an **Operator--Observer Layer** with **dual context**: -
**Observers** instrument both app‑context and motor‑context.\
- **Operator** consumes signals, **classifies** (app vs. motor
vs. mixed), **decides** and **executes**:\
- *App‑Fix*: local patch (prompt, spec, mapping, dependency).\
- *Motor‑Fix*: patch of rules/templates/validators + documented
feedback.\
- *Mixed*: safe sequence (e.g., motor then app).\
- **Resumability**: pause, patch, **resume** without starting over.\
- **Learning Loop**: motor fixes are consolidated (versioned,
changelogged) to benefit future runs.

## 6) Non‑Goals

-   Not tied to one AI vendor or tool.\
-   Does not prescribe prompt/template formats, only **minimal
    interfaces**.

------------------------------------------------------------------------

## 7) Normative Requirements (RFC‑style)

### 7.1 Context Separation

-   MUST persist **App‑Context** and **Motor‑Context** separately.\
-   MUST version both contexts (semantic ID + artifact hash).\
-   SHOULD allow diffs and checkpoints for resumption.

### 7.2 Observation

-   Observers MUST emit structured events:\
    `{ timestamp, scope: "app"|"motor", signal_type, severity, payload }`.\
-   Minimal signal_types: `validation`, `error`, `latency`, `cost`,
    `coverage`, `quality_score`.\
-   SHOULD attach trace_id and artifact_ids.

### 7.3 Classification & Arbitration

-   Operator MUST use a **failure taxonomy** (see §10).\
-   Operator MUST produce a **decision record**.\
-   For mixed, Operator SHOULD apply deterministic order (motor→app).

### 7.4 Actuation

-   For `app`: MUST apply local patches and retry with same motor
    version.\
-   For `motor`: MUST create a change request, run tests, bump version
    if needed, and **resume** app if validation passes.\
-   Pipeline MUST be resumable.

### 7.5 Learning Loop

-   Motor fixes MUST be logged and linked to metrics of impact.\
-   System SHOULD deduplicate incidents and promote learned patterns.

### 7.6 Safety & Governance

-   Motor changes MUST pass gates.\
-   MUST maintain rollback policy.

------------------------------------------------------------------------

## 8) Reference Interfaces

### 8.1 Event Envelope (Observers → Operator)

``` json
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

``` json
{
  "decision_id": "uuid",
  "classification": "app | motor | mixed",
  "rationale": "short explanation",
  "action_plan": [
    {"step": "pause", "target": "pipeline"},
    {"step": "apply_patch", "target": "motor|app", "patch_ref": "PR#123"},
    {"step": "validate", "target": "motor|app"},
    {"step": "resume", "target": "pipeline"}
  ],
  "safety_checks": ["tests:all", "validators:strict"]
}
```

### 8.3 SPI (Plant Interface)

``` ts
export interface PlantRunRequest {
  appSpecRef: string;
  profileTargets: string[];
  motorVersion: string;
  runMode?: "fresh"|"resume";
  checkpoint?: string;
}

export interface PlantRunResult {
  traceId: string;
  status: "success"|"failed"|"partial";
  artifacts: ArtifactRef[];
  signals: EventEnvelope[];
  nextCheckpoint?: string;
}
```

------------------------------------------------------------------------

## 9) Lifecycle & State Machine

States:
`IDLE → RUNNING → PAUSED → PATCHING(MOTOR|APP) → VALIDATING → RESUMING → RUNNING → (SUCCESS|FAILED)`

------------------------------------------------------------------------

## 10) Error Taxonomy

-   **APP‑SPEC**: missing fields, invalid mappings.\
-   **APP‑BUILD**: broken dependency, compilation, integration test.\
-   **MOTOR‑RULES**: wrong validator, incomplete template.\
-   **MOTOR‑PERF**: latency/cost budget exceeded.\
-   **MIXED‑DRIFT**: drift between motor version and app spec.

------------------------------------------------------------------------

## 11) Decision Heuristics

-   If reproduced in ≥2 apps → MOTOR‑RULES.\
-   If disappears when changing profile with same motor → APP.\
-   If quality drops with new motor version → MIXED‑DRIFT.

------------------------------------------------------------------------

## 12) Safety, Rollback & Versioning

-   MUST use semantic versioning.\
-   MUST maintain rollback plan.\
-   Prefer roll‑forward safe fixes.

------------------------------------------------------------------------

## 13) Metrics & SLOs

-   **App SLOs**: build success, quality, latency, cost.\
-   **Motor SLOs**: incident categories, MTTR, impact.\
-   **Operator KPIs**: classification accuracy, resumption rate.

------------------------------------------------------------------------

## 14) Security & Compliance

-   MUST isolate app vs. motor data.\
-   SHOULD redact logs.\
-   MUST enforce gated motor patches.

------------------------------------------------------------------------

## 15) Implementation Notes

-   Context store for transcripts, checkpoints, artifacts.\
-   Operator as policy engine.\
-   Observers as wrappers.\
-   Orchestration integrated with agent bus.

------------------------------------------------------------------------

## 16) Reference Topologies

-   Mono‑plant, Multi‑plant, Federated.

------------------------------------------------------------------------

## 17) Testing Strategy

-   Golden Incidents dataset.\
-   Replay harness.\
-   Chaos drills.\
-   Canary motor upgrades.\
-   Contract tests.

------------------------------------------------------------------------

## 18) Anti‑Patterns

-   Mixing contexts.\
-   Observer without operator.\
-   Ungoverned motor fixes.\
-   Blind retries.

------------------------------------------------------------------------

## 19) Compliance Checklist

-   [ ] Separate contexts.\
-   [ ] Structured events.\
-   [ ] Operator decision records.\
-   [ ] Resumable pipeline.\
-   [ ] Changelog of motor fixes.\
-   [ ] Rollback tested.

------------------------------------------------------------------------

## 20) Example Flow

1)  Observer emits validation error.\
2)  Operator classifies as MOTOR‑RULES.\
3)  Motor patched, validated, version bumped.\
4)  App resumes from checkpoint → success.\
5)  Decision record + impact logged.

------------------------------------------------------------------------
