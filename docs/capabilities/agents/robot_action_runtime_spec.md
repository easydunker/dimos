---
title: "Robot Action Runtime Specification"
---

# Robot Action Runtime Specification

Status: proposed prototype specification.

Audience: implementers of the DIMOS agent harness, robot skill authors, and reviewers of the robotics agent harness.

The companion [implementation plan](/docs/capabilities/agents/robot_action_runtime_implementation_plan.md) is the authoritative work breakdown and progress checklist.

## Decision

Add a deterministic Robot Action Runtime between model-generated tool calls and physical DIMOS skills. The model proposes an `ActionIntent`; the runtime grounds it in a fresh world snapshot, prepares and admits it under trusted policy, owns its resources while it executes, monitors typed events, verifies the physical outcome, and writes every transition to an append-only mission journal.

DIMOS Modules, Blueprints, typed streams, RPC, controllers, simulation, replay, and skills remain the robotics data plane. MCP remains an external compatibility boundary. The runtime is the agent control plane and uses native skill dispatch for the built-in harness.

The central rule is:

> The model may propose a physical action, but only trusted harness code may authorize, execute, settle, or replay it.

## Motivation

The current `McpClient` is a conventional LangGraph tool loop with an in-memory message history. It converts explicit tool-stream updates into synthetic human messages and does not consume general robot streams. See `McpClient` in [`mcp_client.py`](/dimos/agents/mcp/mcp_client.py#L67).

Current skills already expose argument schemas, capability names, and an `instant` or `background` lifecycle through `SkillInfo` in [`dimos/core/module.py`](/dimos/core/module.py#L69). The `CapabilityRegistry` in [`capabilities.py`](/dimos/agents/capabilities.py#L52) provides process-local exclusive locks, and `ToolStream` in [`tool_stream.py`](/dimos/agents/mcp/tool_stream.py#L117) provides manually authored text updates.

These are useful foundations, but a physical action differs from a normal software tool call:

- A controller may continue moving after the initiating RPC returns.
- A successful RPC proves command acceptance, not physical success.
- Relevant observations may become stale while the model reasons or waits for approval.
- Concurrent actions can conflict through motors, arms, zones, power, or compute.
- Cancelling model inference does not stop hardware.
- A process can fail after a command is sent but before its outcome is recorded.
- Blindly repeating a command with an unknown outcome can duplicate a physical effect.
- High-rate control and safety responses cannot depend on an LLM round trip.

The runtime makes those conditions explicit instead of encoding them in prompts or free-form skill results.

## Goals

- Ground each model decision in an immutable, versioned view of relevant robot and environment state.
- Convert model tool calls into trusted, fully specified prepared actions without side effects.
- Admit physical actions using freshness, precondition, resource, safety, and approval checks.
- Retain resource ownership for the complete physical lifetime of an action.
- Monitor action progress without sending high-frequency sensor traffic to the model.
- Verify success from physical evidence instead of trusting RPC completion alone.
- Implement bounded cancellation that ends with observed safe state or an explicit unknown outcome.
- Persist model decisions, tool calls, action transitions, safety decisions, snapshots, and outcomes in an append-only journal.
- Build bounded, deterministic LLM context projections from journaled state.
- Recover after restart without blindly replaying unresolved physical actions.
- Support deterministic tests without hardware or a live model.

## Non-goals

- Replacing navigation, manipulation, visual servoing, whole-body control, or controller-level safety.
- Sending every sensor frame or controller tick to the LLM.
- Building a general behavior-tree engine, fleet scheduler, or VLA runtime in the prototype.
- Making MCP the internal execution path for the new built-in harness.
- Proving cryptographic non-repudiation. The prototype journal is append-only and tamper-evident, not a signed audit service.
- Requiring all existing skills to migrate at once. Uncontracted skills remain callable through the legacy path until explicitly classified.

## Safety and execution invariants

The implementation must preserve these invariants:

1. Preparing an action has no physical side effect.
2. A physical action is never dispatched before its dispatch intent is durably appended.
3. A model-provided value cannot lower a developer-authored risk, resource, freshness, timeout, stop, verification, or replay requirement.
4. An action references both the snapshot used for the model decision and the fresh snapshot used for final admission.
5. Only fields declared as relevant by the skill contract invalidate an action when the world changes.
6. A physical resource lease remains owned until termination is physically verified. An `UNKNOWN` outcome transfers affected resources to a quarantined safety state; it does not make them available to another action.
7. An RPC return cannot by itself settle a physical action as `SUCCEEDED` unless the contract explicitly defines the RPC result as sufficient evidence.
8. Cancelling a model turn does not imply that an active action was cancelled.
9. A stop request is not successful until its stop condition is verified.
10. An unresolved action with a non-replayable or unknown replay policy is never automatically dispatched again after restart.
11. Hard real-time control and controller-level limits remain outside the harness.
12. Journal events and persisted snapshots are immutable. Corrections are new events.

## Runtime boundary

The runtime consists of one lifecycle-owning root and several injected components. A
`RobotAgentHarnessModule` hosts that root in the coordinator so both native callers and
`McpServer` resolve the same `ActionRuntimeSpec` RPC service. The harness itself remains
a plain, directly testable Python object. The host module starts and stops its stores,
subscriptions, and executors; no component relies on a global mutable registry. A CLI
client may attach to the host module, but must not construct a second runtime in its own
process.

| Component | Responsibility | Must not do |
|---|---|---|
| `RobotAgentHarness` | Own mission lifecycle, model turns, projections, and runtime component lifecycle | Issue motor commands directly |
| `WorldSnapshotProvider` | Obtain a detached observation draft from robot-specific adapters, merge runtime projections, and persist an immutable logical cut | Interpret mission goals |
| `SkillContractRegistry` | Resolve trusted execution metadata for discovered skills | Accept policy metadata from the model |
| `ActionPreparer` | Combine an intent, contract, and decision snapshot into a `PreparedAction` | Invoke a skill |
| `AdmissionPipeline` | Validate freshness, preconditions, safety, approval, and resource availability | Mutate the prepared action to weaken policy |
| `LeaseManager` | Atomically own resources by action, enforce expiry, and support emergency revocation | Infer physical completion from lease release |
| `SkillExecutor` | Dispatch through native `SkillsProxy` and return a correlated execution handle | Decide whether an action is safe or successful |
| `ActionRunner` | Own the action state machine, timeouts, monitoring, cancellation, verification, settlement, and cleanup | Run high-rate control loops |
| `SafetySupervisor` | Grant, constrain, reject, pause, or stop actions using deterministic policy | Depend on the model for emergency response |
| `MissionJournal` | Atomically append ordered events and immutable snapshots | Store large sensor payloads inline |
| `ProjectionBuilder` | Derive current mission, active action, operator, recovery, and LLM views | Become a second source of truth |
| `ArtifactStore` | Store content-addressed images, maps, point clouds, and large model artifacts | Own mission or action state |

The prototype may run several of these as plain Python components in the harness-module
worker. Interfaces remain separate so safety, storage, and execution can later move
across process boundaries.

## Skill classification and contracts

Every migrated skill has a trusted `SkillContract`. The model sees only the skill name, description, and argument schema; it does not author the contract.

### Skill kinds

| Kind | Meaning | Execution treatment |
|---|---|---|
| `QUERY` | Reads state without changing the physical world, such as battery or observation | Validate and journal the call; no physical transaction required |
| `COMPUTE` | Produces a plan or inference without commanding hardware | Apply timeout and artifact handling; no physical lease unless declared |
| `PHYSICAL` | Commands or maintains a physical effect | Full preparation, admission, leasing, monitoring, cancellation, and verification |

### Required contract fields

| Field | Meaning |
|---|---|
| `name` and `contract_version` | Stable skill identity and metadata version |
| `kind` | `QUERY`, `COMPUTE`, or `PHYSICAL` |
| `argument_schema` | Existing generated JSON schema |
| `execution_mode` | Synchronous return or background handle |
| `resources` | Resource requests and modes |
| `risk_class` | Deterministic policy input; initially `LOW`, `MEDIUM`, or `HIGH` |
| `required_observations` | Snapshot fields and maximum allowed ages |
| `preconditions` | Named deterministic predicates evaluated by the harness |
| `timeout_s` | Maximum execution time before cancellation begins |
| `stop_handler` | Named skill or executor operation used to stop an active effect |
| `stop_timeout_s` | Deadline for proving the stop condition |
| `completion_source` | Synchronous result, typed action event, controller state, or verifier |
| `verifier` | Named deterministic physical-effect verifier |
| `replay_policy` | `SAFE_QUERY`, `IDEMPOTENT`, `COMPENSATABLE`, or `NEVER_IF_UNKNOWN` |
| `approval_policy` | Whether deterministic human approval is required |

Contract names resolve to registered predicates and verifiers in trusted code. They are data, not arbitrary executable source loaded from a journal or model response.

The prototype extends the existing `@skill` metadata and `SkillInfo` without breaking bare `@skill` usage. A skill without a physical contract is treated as legacy and cannot enter the physical `ActionRunner` path until it is classified.

### Prototype resource model

The first implementation supports exclusive resources with an action owner, priority, lease deadline, and heartbeat. This directly replaces the current ambiguous lifetime of the `movement` capability for migrated skills.

The data model must leave room for shared resources, capacity resources, and spatial zones, but those modes are follow-up work. Emergency stop may revoke an action's dispatch authority regardless of priority, but revocation or heartbeat expiry quarantines a physical resource rather than making it available. Only verified safe termination releases it. Normal model actions cannot preempt each other in the prototype.

## Domain records

All durable records include a schema version. Identifiers are opaque strings generated by trusted runtime code. Immutability is deep: JSON mappings and sequences are recursively normalized to immutable representations at the record boundary, not merely placed inside a frozen outer dataclass.

### `ActionIntent`

An `ActionIntent` is the untrusted proposal derived from one model tool call.

| Field | Source |
|---|---|
| `intent_id` | Harness |
| `mission_id` and `turn_id` | Harness |
| `skill_name` and `arguments` | Model tool call |
| `decision_snapshot_version` | Harness context used for the model turn |
| `model_call_id` | Model adapter |
| `requested_at` | Harness clock |

### `PreparedAction`

A `PreparedAction` is immutable and side-effect free. It contains validated arguments plus the resolved contract requirements.

| Field | Meaning |
|---|---|
| `action_id` | Stable identity for all later events |
| `intent` | Original proposal and provenance |
| `contract_name` and `contract_version` | Exact trusted contract used |
| `validated_arguments` | Canonical arguments after schema validation |
| `decision_snapshot_version` | Snapshot that grounded the model decision |
| `required_observations` | Relevant fields and freshness bounds |
| `resources` | Requested leases |
| `risk_class` and `approval_policy` | Admission inputs |
| `timeout_s`, `stop_handler`, `stop_timeout_s` | Runtime bounds |
| `completion_source` and `verifier` | Settlement rules |
| `replay_policy` | Restart behavior |

### Action states

Allowed states are `PROPOSED`, `PREPARED`, `ADMITTED`, `DISPATCHING`, `EXECUTING`, `VERIFYING`, `STOPPING`, `SUCCEEDED`, `FAILED`, `CANCELLED`, `REJECTED`, and `UNKNOWN`.

The normal path is `PROPOSED` to `PREPARED` to `ADMITTED` to `DISPATCHING` to `EXECUTING` to `VERIFYING` to `SUCCEEDED` or `FAILED`.

`REJECTED` is terminal and means admission or preparation refused the proposal. A
pre-dispatch `CANCELLED` action needs proof that the executor was never invoked; after
possible dispatch, `CANCELLED` is terminal only after the stop condition is verified.
`UNKNOWN` is terminal for the action's automation but requires operator visibility or
reconciliation because the physical effect could not be proven. Resources affected by
an unknown physical effect remain quarantined.

Invalid transitions fail closed, append an internal-error event, and do not invoke or replay a skill.

The authoritative prototype transition graph is:

- `PROPOSED -> PREPARED | REJECTED`
- `PREPARED -> ADMITTED | REJECTED | CANCELLED`
- `ADMITTED -> DISPATCHING | CANCELLED`
- `DISPATCHING -> EXECUTING | FAILED | CANCELLED | STOPPING | UNKNOWN`
- `EXECUTING -> VERIFYING | FAILED | STOPPING | UNKNOWN`
- `VERIFYING -> SUCCEEDED | FAILED | STOPPING | UNKNOWN`
- `STOPPING -> CANCELLED | UNKNOWN`

The five terminal states are `SUCCEEDED`, `FAILED`, `CANCELLED`, `REJECTED`, and
`UNKNOWN`. Preparation and admission failures use `REJECTED`, because no physical call
was made. `FAILED` is reserved for an action whose lack of effect or safe termination is
known after dispatch was attempted. Cancellation may settle directly as `CANCELLED`
only while the runtime can prove the executor was not invoked; after invocation or
ambiguous delivery it must pass through `STOPPING`. Recovery does not mutate a terminal
`UNKNOWN` action into another state: later evidence is appended as a
reconciliation event, and a quarantined resource is cleared only by an explicit,
verified safety-reconciliation event.

### `ActionEvent`

Controller and skill updates use a typed envelope rather than a synthetic human message.

| Field | Meaning |
|---|---|
| `event_id` and `action_id` | Correlation |
| `event_type` | `STARTED`, `PROGRESS`, `COMPLETION_REPORTED`, `FAILED`, `STOPPED`, or domain-specific observation |
| `occurred_at` | Source time |
| `world_version` | Snapshot version when available |
| `code` | Stable machine-readable status or error code |
| `message` | Optional operator-readable summary |
| `progress` and `total` | Optional bounded progress |
| `metadata` | Small typed JSON-compatible details |

The executor adapts existing `ToolStream` notifications during migration. Correlation must preserve the progress token or explicit action ID; tool name alone is insufficient when calls overlap.

### `ActionOutcome`

An outcome contains the terminal state, structured error code, decision and admission snapshot versions, final verification snapshot version, observed effects, timing, and a recommended next disposition: continue, retry deterministically, replan, request human help, or stop the mission.

## World snapshots

### Semantics

A `WorldSnapshot` is an immutable logical cut through asynchronously updated robot state. It does not claim that every sensor sampled at the same instant. Instead, every field preserves its source time, age, confidence, and provenance, while the snapshot receives one monotonically increasing version and capture time.

Required snapshot metadata:

| Field | Meaning |
|---|---|
| `version` | Durable monotonically increasing version allocated by the snapshot store |
| `captured_at` | Harness wall-clock capture time |
| `trigger` | Decision, admission, safety event, verification, recovery, or explicit observation |
| `fields` | Mapping from stable field name to `TimedObservation` |
| `active_actions` | Current action IDs and states from the journal projection |
| `resource_leases` | Current ownership projection |
| `safety_state` | Current supervisor state and reason codes |
| `artifact_refs` | Content-addressed references to large payloads |
| `schema_version` | Snapshot encoding version |

Each `TimedObservation` contains the value or artifact reference, source stream, source timestamp, capture-time age, optional confidence, and optional frame ID.

### Capture model

Robot-specific adapter modules subscribe to existing typed streams and update a lock-protected or serially owned latest-value cache. Their capture RPC returns a detached `WorldSnapshotDraft` containing source values and timestamps but no durable version. The harness-owned `WorldSnapshotProvider` derives ages from one capture time, merges action, lease, and safety projections, writes large payloads to the artifact store, allocates the version, and persists the immutable snapshot through the journal's single writer.

The first Go2 adapter captures at least odometry, latest camera reference, navigation state when available, battery when available, active actions, movement ownership, and safety state. Missing fields remain explicitly unavailable; they are never replaced with placeholder values.

Snapshots are persisted at decision boundaries, final admission, safety interventions, cancellation boundaries, verification, recovery, and optional low-rate diagnostic checkpoints. Raw high-frequency telemetry remains in existing recorders or replay databases.

### Freshness and revalidation

The model turn receives a decision snapshot version. Immediately before dispatch, the `ActionRunner` captures a new admission snapshot after any approval or resource wait.

The runtime compares only the fields declared by the skill contract. A newer unrelated camera frame does not invalidate an action that depends only on localization and battery. A changed or stale localization field does.

An invalidated action returns to preparation or asks the model to decide again; it is never silently executed against different arguments or weakened preconditions.

### Persistence

Snapshot metadata and compact structured fields live in the same SQLite database as the mission journal. A snapshot insert and its `WORLD_SNAPSHOT_CAPTURED` journal event commit in one transaction. Images, video, maps, and point clouds live in a content-addressed artifact directory and are referenced by digest, media type, size, and source timestamp.

Persisted snapshots are immutable. A corrected interpretation creates a new snapshot or a correction event; it does not update the prior row.

The journal is not a substitute for physical truth. On restart, the runtime restores recorded state, captures a fresh recovery snapshot, queries relevant controllers, and reconciles unresolved actions against observed reality.

## Mission journal

### Event envelope

Every journal event contains a global sequence, event ID, schema version, event type, mission ID, optional turn and action IDs, occurrence and recording timestamps, optional world version, causation and correlation IDs, canonical JSON payload, previous-event hash, and event hash.

The hash chain makes accidental or offline mutation detectable. It is not a security
boundary: the prototype has no signing key, and an administrator who can rewrite the
database can recompute the unkeyed hashes.

### Required event families

| Family | Minimum events |
|---|---|
| Mission | `MISSION_CREATED`, `MISSION_STATUS_CHANGED`, `MISSION_FINISHED` |
| Model | `MODEL_CONTEXT_BUILT`, `MODEL_RESPONSE_RECORDED`, `TOOL_CALL_PROPOSED` |
| Snapshot | `WORLD_SNAPSHOT_CAPTURED` |
| Preparation | `ACTION_PREPARED`, `ACTION_PREPARATION_FAILED` |
| Admission | `ADMISSION_GRANTED`, `ADMISSION_REJECTED`, `APPROVAL_REQUESTED`, `APPROVAL_RECORDED` |
| Resources | `LEASE_ACQUIRED`, `LEASE_RENEWED`, `LEASE_RELEASED`, `LEASE_REVOKED` |
| Dispatch | `ACTION_DISPATCH_REQUESTED`, `ACTION_DISPATCH_ACCEPTED`, `ACTION_DISPATCH_FAILED` |
| Execution | `ACTION_STARTED`, `ACTION_PROGRESS`, `ACTION_COMPLETION_REPORTED`, `ACTION_EXECUTION_FAILED` |
| Stop | `ACTION_CANCEL_REQUESTED`, `STOP_COMMAND_SENT`, `STOP_COMMAND_ACCEPTED`, `STOP_COMMAND_FAILED`, `STOP_COMMAND_UNKNOWN`, `STOP_VERIFIED`, `STOP_VERIFICATION_FAILED` |
| Verification | `ACTION_VERIFICATION_STARTED`, `ACTION_VERIFIED`, `ACTION_VERIFICATION_FAILED` |
| Settlement | `ACTION_SUCCEEDED`, `ACTION_FAILED`, `ACTION_CANCELLED`, `ACTION_UNKNOWN` |
| Recovery | `RECOVERY_STARTED`, `ACTION_RECONCILED`, `RECOVERY_FINISHED` |
| Safety | `SAFETY_ACTION_CONSTRAINED`, `SAFETY_ACTION_REJECTED`, `SAFETY_INTERVENTION` |

Model prompts, raw responses, and tool arguments pass through a redaction policy before persistence. Large model artifacts use the artifact store.

### SQLite prototype

The prototype uses the standard-library SQLite driver in WAL mode with full synchronous durability. The configurable default location is under `STATE_DIR / "agent_runtime"`; tests always use `tmp_path`.

The database contains:

| Table | Purpose | Mutation policy |
|---|---|---|
| `journal_events` | Ordered canonical event envelopes and hash chain | Insert only |
| `world_snapshots` | Versioned compact snapshot payload and digest | Insert only |
| `artifacts` | Content-addressed artifact metadata | Insert or idempotent insert |
| `projection_checkpoints` | Disposable acceleration for derived projections | Replaceable and rebuildable |

Database triggers reject updates and deletes from `journal_events` and `world_snapshots`. A single writer owned by the runtime allocates sequences inside `BEGIN IMMEDIATE` transactions. Readers may run concurrently.

On open, the store validates schema compatibility and the event hash chain. An invalid chain fails startup rather than silently constructing a projection from corrupted history.

### Effect sandwich

For every external physical effect, the runner performs three durable steps:

1. Append `ACTION_DISPATCH_REQUESTED` with the exact skill, validated arguments, contract version, and admission snapshot.
2. Invoke the uncertain external effect through `SkillExecutor`.
3. Append `ACTION_DISPATCH_ACCEPTED`, `ACTION_DISPATCH_FAILED`, or `ACTION_UNKNOWN`.

If the process exits between steps two and three, recovery sees an unresolved dispatch and must query the controller and current world. It does not assume failure and does not automatically retry.

## LLM projections

The chat transcript is not the runtime database. `ProjectionBuilder` deterministically derives a bounded `LLMContextProjection` containing:

- Mission goal, constraints, and current status.
- Decision snapshot version and relevant fresh fields.
- Current short plan when present.
- Active actions, state, elapsed time, and owned resources.
- Recent salient progress, failure, safety, and intervention events.
- Unresolved or unknown actions.
- Available contracted skills appropriate to the current state.
- Artifact references selected for the next model request.
- The exact decision question the model must answer.

Routine controller progress is coalesced. Safety events, terminal outcomes, unknown effects, user steering, and failed deterministic recovery are never dropped.

Every model turn journals the projection version or digest and the response that resulted from it. This makes evaluations reproducible without placing the full event log in the prompt.

Other projections include current mission state, active action and lease state, operator timeline, recovery state, and evaluation metrics. Projection checkpoints are caches; deleting them and replaying the immutable journal must produce the same result.

## Action execution flow

### Model decision and preparation

1. A user command or salient runtime event requests a decision.
2. The harness captures and persists a decision snapshot.
3. The projection builder creates bounded model context from the snapshot and journal.
4. The model returns text and zero or more tool calls.
5. Each tool call is journaled as an `ActionIntent`.
6. The preparer resolves the trusted contract, validates and canonicalizes arguments, and emits a `PreparedAction` without side effects.

### Admission and dispatch

1. Perform schema and static contract validation.
2. Capture current state and evaluate required observations and preconditions.
3. Apply deterministic safety policy and request human approval if required.
4. Acquire required leases with a bounded wait.
5. Capture a final admission snapshot and repeat relevant freshness, precondition, and safety checks after the wait.
6. Append the dispatch intent.
7. Invoke the skill through native `SkillsProxy` with the action ID as correlation context.
8. Append the dispatch settlement and enter `EXECUTING` only when command acceptance is known.

Any failed gate releases provisional resources, journals a structured rejection, and returns a structured outcome. Later gates may make a decision stricter but never weaken an earlier policy requirement.

The harness may prepare independent tool calls concurrently. Query and compute calls may execute concurrently when their contracts permit it. Physical calls may execute concurrently only after the lease manager proves that their resources are disjoint and the safety supervisor admits the combination. The journal's global sequence provides a deterministic ordering for projection even when execution overlaps.

### Monitoring and verification

Controllers retain high-rate feedback loops. The runner consumes correlated, low-rate `ActionEvent`s and relevant world changes. The contract determines which event requests verification.

On reported completion, the runner captures a fresh snapshot and evaluates the named verifier. A verified expected effect settles success. A failed verification chooses a contract-defined deterministic recovery, requests replanning, or settles failure or unknown.

### Cancellation

Cancellation revokes the action's authority and durably appends
`ACTION_CANCEL_REQUESTED` followed by `STOP_COMMAND_SENT` before invoking the trusted
stop handler. It then records the call settlement and monitors the stop condition until
its deadline. The stop command is itself an external physical effect and follows the
same intent-before-effect rule as initial dispatch. Only observed safe state produces
`STOP_VERIFIED`, `ACTION_CANCELLED`, and lease release.

If stopping cannot be proven, the runner appends `ACTION_UNKNOWN`, quarantines the affected resources, retains an explicit unsafe or unknown runtime state, invokes the independent safety escalation path, and exposes the condition to the operator.

### Restart recovery

At startup, the runtime validates the journal, rebuilds projections, and finds every nonterminal action. For each action it:

1. Prevents new conflicting dispatches.
2. Captures a recovery snapshot.
3. Queries the controller or skill-specific reconciler.
4. Stops motion when ownership or state is uncertain and stopping is safe.
5. Appends the reconciled terminal state or `ACTION_UNKNOWN`.
6. Releases leases only after a reconciled, physically verified non-active condition. An unknown condition keeps the affected resources quarantined.
7. Asks the model to replan only after deterministic reconciliation finishes.

Replay follows the contract's policy. `SAFE_QUERY` may run again, `IDEMPOTENT` requires the same idempotency key, `COMPENSATABLE` requires an explicit recovery path, and `NEVER_IF_UNKNOWN` is never automatically replayed.

## Integration with existing DIMOS primitives

- Preserve `Module`, `Blueprint`, typed stream, transport, controller, and robot-specific skill implementations.
- Extend `@skill` and the core `SkillInfo` with optional contract metadata while retaining existing call sites.
- Use `Dimos.connect()` and `SkillsProxy` from [`dimos/porcelain/dimos.py`](/dimos/porcelain/dimos.py#L107) and [`skills_proxy.py`](/dimos/porcelain/skills_proxy.py#L47) for built-in native dispatch.
- Keep `McpServer` for external agents. Its tool metadata should expose non-sensitive contract fields for discovery. Contracted physical calls from MCP and the native model loop must enter the `ActionRuntimeSpec` hosted by the same coordinator-visible `RobotAgentHarnessModule`; MCP must not directly dispatch them or become a second authority for safety or resource ownership. Uncontracted legacy calls retain their current path during migration.
- Adapt legacy `ToolStream` messages into typed action progress during migration. New physical skills should emit correlated `ActionEvent`s directly.
- Remove `CapabilityRegistry` from the dispatch path for contracted physical actions. It remains only for uncontracted legacy calls; `ActionRuntimeSpec` routes both native and MCP contracted calls through the same `LeaseManager`.
- Reuse `SkillResult` from [`skill_result.py`](/dimos/agents/skill_result.py#L55) for structured synchronous results, but do not equate `success=True` with verified physical success unless the contract permits it.
- Reuse replay and `MockModel` fixtures for deterministic integration and end-to-end tests.

## Failure handling

| Failure | Required behavior |
|---|---|
| Missing or invalid contract | Reject before dispatch and journal the preparation failure |
| Required observation missing or stale | Capture again if bounded; otherwise reject without invoking the skill |
| Resource conflict | Wait within policy or reject; never run conflicting actions concurrently |
| Safety rejection | Journal the policy reason and do not invoke the skill |
| Approval delay | Revalidate the world after approval before dispatch |
| Executor proves the RPC was never invoked | Journal dispatch failure and release leases |
| RPC exception after invocation or with ambiguous delivery | Settle `UNKNOWN`, reconcile, and never release or retry merely because an exception was raised |
| Lost response after possible dispatch | Settle `UNKNOWN`, reconcile, and never blind-retry |
| Progress stream disconnect | Query controller state, then continue, stop, or settle unknown |
| Execution timeout | Begin cancellation; timeout alone does not release physical resources |
| Verification failure | Run bounded deterministic recovery or request replanning |
| Journal write failure before dispatch | Do not dispatch |
| Journal write failure after dispatch | Revoke further authority, stop if safe, and surface a fatal runtime condition |
| Snapshot persistence failure | Do not use the unpersisted snapshot for physical admission |

## Observability and evaluation

The journal provides exact metrics without parsing prose:

- Mission and action success rate.
- Preparation and admission rejection counts by reason.
- Unsafe actions blocked.
- Resource conflicts prevented.
- Decision-to-dispatch latency.
- Cancellation request-to-zero-motion latency.
- Verification latency and failure rate.
- Recovery success and unknown-outcome rate.
- Duplicate physical effects after recovery; the target is zero.
- Model calls and replans per mission.
- Snapshot age distributions for each required observation.

Operational logs remain useful for process debugging, but they do not replace domain events in the mission journal.

## Security and privacy

- Redact secrets and configured sensitive fields before journal or artifact persistence.
- Do not embed API keys, authentication headers, or raw credentials in action arguments or model records.
- Make artifact retention configurable independently from event retention.
- Record safety and approval decisions with policy identifiers and versions.
- Treat journal and snapshot payloads as untrusted when reading old or externally supplied databases; validate schema and bounds before projection.

## Prototype acceptance criteria

The prototype is complete when all of the following are demonstrated without hardware or a live model:

- A recorded model fixture proposes a contracted physical action.
- The harness captures a decision snapshot and a fresher admission snapshot.
- A stale required observation prevents skill invocation.
- An admitted action holds its exclusive resource through verification.
- Progress and completion are correlated by action ID and stored as typed events.
- A controller completion report alone does not settle success; a fresh snapshot verifier does.
- Cancellation invokes a stop handler and releases the lease only after stop verification.
- Restart recovery detects an unresolved dispatch and does not execute it twice.
- Journal updates and deletes are rejected, and hash-chain verification detects mutation.
- Rebuilding projections from the journal produces the same mission and action state.
- A replay-based end-to-end test persists snapshots from the repository's `go2_short.db` sample.

## Deferred extensions

- Shared, capacity, and spatial-zone leases.
- Priority scheduling beyond emergency preemption.
- Behavior-tree or DAG execution compiled from mission steps.
- Fleet-level allocation and robot leases.
- Learned VLA policies as contracted physical skills.
- Remote replicated journals and artifact storage.
- Cryptographic signatures and external audit anchoring.
