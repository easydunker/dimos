---
title: "Robot Action Runtime Specification"
---

# Robot Action Runtime Specification

Status: proposed prototype specification.

Audience: implementers of the DIMOS agent harness, robot skill authors, and reviewers of the robotics agent harness.

The companion [implementation plan](/docs/capabilities/agents/robot_action_runtime_implementation_plan.md) is the authoritative work breakdown and progress checklist.

## Decision

Add a deterministic Robot Action Runtime behind the standard tool-dispatch boundary for physical DIMOS skills. The model continues to emit ordinary tool calls. When trusted skill metadata includes a `PhysicalActionContract`, the dispatcher derives an `ActionIntent` from that tool call and submits it to the runtime. The runtime grounds the proposed physical action in a fresh world snapshot, prepares and admits it under trusted policy, owns its resources while it executes, monitors typed events, verifies the physical outcome, and writes every transition to an append-only mission journal.

DIMOS Modules, Blueprints, typed streams, RPC, controllers, simulation, replay, and skills remain the robotics data plane. MCP remains an external compatibility boundary. The runtime is the agent control plane and uses native skill dispatch for the built-in harness.

The central rule is:

> The model may propose a physical action, but only trusted harness code may authorize, execute, or settle it.

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
- Convert contracted physical tool calls into trusted, fully specified prepared actions without changing the standard agent tool protocol.
- Admit physical actions using freshness, precondition, resource, safety, and approval checks.
- Retain resource ownership for the complete physical lifetime of an action.
- Monitor action progress without sending high-frequency sensor traffic to the model.
- Verify success from physical evidence instead of trusting RPC completion alone.
- Persist model decisions, tool calls, action transitions, safety decisions, snapshots, and outcomes in an append-only journal.
- Build bounded, deterministic LLM context projections from journaled state.
- Persist and surface unresolved physical actions without automatically redispatching them.
- Support deterministic tests without hardware or a live model.

## Non-goals

- Replacing navigation, manipulation, visual servoing, whole-body control, or controller-level safety.
- Sending every sensor frame or controller tick to the LLM.
- Building a general behavior-tree engine, fleet scheduler, or VLA runtime in the prototype.
- Making MCP the internal execution path for the new built-in harness.
- Proving cryptographic non-repudiation. The prototype journal is append-only and tamper-evident, not a signed audit service.
- Replacing standard tool-call schemas, call IDs, dispatch, or result messages with an action-specific agent protocol.
- Requiring all existing skills to migrate at once. Existing uncontracted skills remain ordinary tools on their current path until explicitly migrated; they do not receive the runtime's physical-action guarantees.
- Implementing generic physical-action cancellation, stop verification, or timeout-driven stopping. Those belong to a separate cancellation workstream.
- Implementing restart reconciliation, controller queries after restart, automated recovery, or physical-command replay. Those belong to a separate restart and recovery workstream.

## Safety and execution invariants

The implementation must preserve these invariants:

1. Preparing an action has no physical side effect.
2. A physical action is never dispatched before its dispatch intent is durably appended.
3. A model-provided value cannot lower a developer-authored risk, resource, freshness, timeout, or verification requirement.
4. An action references both the snapshot used for the model decision and the fresh snapshot used for final admission.
5. Only fields declared as relevant by the physical action contract invalidate an action when the world changes.
6. A physical resource lease remains owned until termination is physically verified. An `UNKNOWN` outcome leaves affected resources unavailable for automatic reuse and visible for operator or future recovery handling.
7. An RPC return cannot by itself settle a physical action as `SUCCEEDED` unless the contract explicitly defines the RPC result as sufficient evidence.
8. Cancelling a model turn does not imply that an active action was cancelled.
9. An unresolved or `UNKNOWN` physical action is never automatically dispatched again by this prototype.
10. Hard real-time control and controller-level limits remain outside the harness.
11. Journal events and persisted snapshots are immutable. Corrections are new events.

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
| `PhysicalActionContractRegistry` | Resolve optional trusted physical-execution metadata for discovered tools | Accept policy metadata from the model or require every ordinary tool to adopt action semantics |
| `ActionPreparer` | Combine a schema-validated physical tool-call intent, contract, and decision snapshot into a `PreparedAction` | Invoke a skill or redefine the standard tool schema |
| `AdmissionPipeline` | Validate freshness, preconditions, safety, approval, and resource availability | Mutate the prepared action to weaken policy |
| `LeaseManager` | Atomically own resources by action and make expired or unresolved ownership unavailable for reuse | Infer physical completion from lease release or implement emergency-stop policy |
| `SkillExecutor` | Dispatch through native `SkillsProxy` and return a correlated execution handle | Decide whether an action is safe or successful |
| `ActionRunner` | Own the action state machine, timeouts, monitoring, verification, settlement, and cleanup | Run high-rate control loops or implement generic cancellation and restart recovery |
| `SafetySupervisor` | Grant, constrain, or reject actions using deterministic policy and observe independent controller safety state | Depend on the model for emergency response or replace controller-level emergency stop |
| `MissionJournal` | Atomically append ordered events and immutable snapshots | Store large sensor payloads inline |
| `ProjectionBuilder` | Derive current mission, active action, unresolved-action, operator, and LLM views | Become a second source of truth |
| `ArtifactStore` | Store content-addressed images, maps, point clouds, and large model artifacts | Own mission or action state |

The prototype may run several of these as plain Python components in the harness-module
worker. Interfaces remain separate so safety, storage, and execution can later move
across process boundaries.

## Standard tools and physical action contracts

The standard agent tool concept remains the public protocol. Tool names, descriptions, JSON argument schemas, tool-call IDs, dispatcher behavior, and tool-result messages remain compatible with the existing model loop and MCP clients. The runtime does not introduce an action-specific tool format.

A physical DIMOS skill opts into managed execution by attaching a trusted `PhysicalActionContract` to its existing `@skill` metadata. The model sees the ordinary tool definition; it does not author or override the contract. The dispatcher resolves the contract after receiving a standard tool call:

- A tool without a `PhysicalActionContract` follows the existing ordinary tool path and does not create an `ActionIntent`.
- A tool with a valid `PhysicalActionContract` preserves its standard tool-call identity, derives an `ActionIntent`, and enters the preparation, admission, leasing, monitoring, and verification path.
- A tool declared for managed physical execution with an invalid or incomplete contract fails registration or preparation before dispatch.

Existing uncontracted physical skills may remain callable on their current ordinary tool path during migration for backward compatibility, but the runtime must not represent them as admitted or verified actions. New physical skills and migrated physical entrypoints must provide a valid contract.

### Required physical action contract fields

| Field | Meaning |
|---|---|
| `contract_id` and `contract_version` | Stable physical-policy identity and metadata version |
| `resources` | Resource requests and modes |
| `risk_class` | Deterministic policy input; initially `LOW`, `MEDIUM`, or `HIGH` |
| `required_observations` | Snapshot fields and maximum allowed ages |
| `preconditions` | Named deterministic predicates evaluated by the harness |
| `execution_timeout_s` | Maximum time the prototype monitors for a terminal result before surfacing the action as unresolved |
| `completion_source` | Synchronous result, typed action event, controller state, or verifier |
| `verifier` | Named deterministic physical-effect verifier |
| `approval_policy` | Whether deterministic human approval is required |

Contract IDs resolve to registered predicates and verifiers in trusted code. Contracts are data, not arbitrary executable source loaded from a journal or model response.

The contract does not duplicate the standard tool's name, description, argument schema, call ID, result envelope, or lifecycle. Existing `SkillInfo` metadata remains authoritative for those fields. Preparation first applies the ordinary tool schema, then combines the validated call with the physical action contract.

The prototype extends the existing `@skill` metadata and `SkillInfo` with one optional physical action contract without breaking bare `@skill` usage. Contract presence selects the action-runtime route; absence preserves ordinary tool execution.

### Prototype resource model

The first implementation supports exclusive resources with an action owner, priority, lease deadline, and heartbeat. This directly replaces the current ambiguous lifetime of the `movement` capability for migrated skills.

The data model must leave room for shared resources, capacity resources, and spatial zones, but those modes are follow-up work. An expired lease or unresolved action does not make a physical resource available to another model action; it leaves the resource blocked and visible to the operator. Deciding how to stop the underlying controller and clear that blocked state belongs to the cancellation or restart-recovery workstream. Normal model actions cannot preempt each other in the prototype.

## Domain records

All durable records include a schema version. Identifiers are opaque strings generated by trusted runtime code. Immutability is deep: JSON mappings and sequences are recursively normalized to immutable representations at the record boundary, not merely placed inside a frozen outer dataclass.

### `ActionIntent`

An `ActionIntent` is the untrusted physical-action proposal derived from one standard tool call whose registered skill has a `PhysicalActionContract`. Ordinary tool calls do not create this record.

| Field | Source |
|---|---|
| `intent_id` | Harness |
| `mission_id` and `turn_id` | Harness |
| `tool_name` and `arguments` | Standard model tool call |
| `decision_snapshot_version` | Harness context used for the model turn |
| `tool_call_id` | Standard tool-call ID preserved unchanged from the model adapter |
| `requested_at` | Harness clock |

### `PreparedAction`

A `PreparedAction` is immutable and side-effect free. It contains validated arguments plus the resolved contract requirements.

| Field | Meaning |
|---|---|
| `action_id` | Stable identity for all later events |
| `intent` | Original proposal and provenance |
| `physical_action_contract_id` and `contract_version` | Exact trusted physical action contract used |
| `validated_arguments` | Canonical arguments after schema validation |
| `decision_snapshot_version` | Snapshot that grounded the model decision |
| `required_observations` | Relevant fields and freshness bounds |
| `resources` | Requested leases |
| `risk_class` and `approval_policy` | Admission inputs |
| `execution_timeout_s` | Monitoring bound after which the action is surfaced as unresolved |
| `completion_source` and `verifier` | Settlement rules |

### Action states

Allowed states are `PROPOSED`, `PREPARED`, `ADMITTED`, `DISPATCHING`, `EXECUTING`, `VERIFYING`, `SUCCEEDED`, `FAILED`, `REJECTED`, and `UNKNOWN`.

The normal path is `PROPOSED` to `PREPARED` to `ADMITTED` to `DISPATCHING` to `EXECUTING` to `VERIFYING` to `SUCCEEDED` or `FAILED`.

`REJECTED` is terminal and means admission or preparation refused the proposal.
`UNKNOWN` is terminal for this prototype's automation and means command delivery or the
physical outcome could not be proven. The runtime exposes the condition and leaves
affected resources unavailable; it does not attempt generic stop or reconciliation.

Invalid transitions fail closed, append an internal-error event, and do not invoke or automatically re-invoke a skill.

The authoritative prototype transition graph is:

- `PROPOSED -> PREPARED | REJECTED`
- `PREPARED -> ADMITTED | REJECTED`
- `ADMITTED -> DISPATCHING | REJECTED`
- `DISPATCHING -> EXECUTING | FAILED | UNKNOWN`
- `EXECUTING -> VERIFYING | FAILED | UNKNOWN`
- `VERIFYING -> SUCCEEDED | FAILED | UNKNOWN`

The four terminal states are `SUCCEEDED`, `FAILED`, `REJECTED`, and `UNKNOWN`.
Preparation and admission failures use `REJECTED`, because no physical call was made.
`FAILED` is reserved for an action whose lack of effect or verified terminal condition
is known after dispatch was attempted. The prototype never turns `UNKNOWN` into another
state automatically; a future recovery workstream may append separate reconciliation
records without rewriting history.

### `ActionEvent`

Controller and skill updates use a typed envelope rather than a synthetic human message.

| Field | Meaning |
|---|---|
| `event_id` and `action_id` | Correlation |
| `event_type` | `STARTED`, `PROGRESS`, `COMPLETION_REPORTED`, `FAILED`, or domain-specific observation |
| `occurred_at` | Source time |
| `world_version` | Snapshot version when available |
| `code` | Stable machine-readable status or error code |
| `message` | Optional operator-readable summary |
| `progress` and `total` | Optional bounded progress |
| `metadata` | Small typed JSON-compatible details |

The executor adapts existing `ToolStream` notifications during migration. Correlation must preserve the progress token or explicit action ID; tool name alone is insufficient when calls overlap.

### `ActionOutcome`

An outcome contains the terminal state, structured error code, decision and admission snapshot versions, final verification snapshot version, observed effects, timing, and a recommended next disposition: continue, replan, request human help, or stop the mission. The harness projects that outcome into a standard tool-result message associated with the original `tool_call_id`; the model loop does not consume a new action-specific response protocol.

## World snapshots

### Semantics

A `WorldSnapshot` is an immutable logical cut through asynchronously updated robot state. It does not claim that every sensor sampled at the same instant. Instead, every field preserves its source time, age, confidence, and provenance, while the snapshot receives one monotonically increasing version and capture time.

Required snapshot metadata:

| Field | Meaning |
|---|---|
| `version` | Durable monotonically increasing version allocated by the snapshot store |
| `captured_at` | Harness wall-clock capture time |
| `trigger` | Decision, admission, safety event, verification, or explicit observation |
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

Snapshots are persisted at decision boundaries, final admission, safety interventions, verification, and optional low-rate diagnostic checkpoints. Raw high-frequency telemetry remains in existing recorders or replay databases.

### Freshness and revalidation

The model turn receives a decision snapshot version. Immediately before dispatch, the `ActionRunner` captures a new admission snapshot after any approval or resource wait.

The runtime compares only the fields declared by the physical action contract. A newer unrelated camera frame does not invalidate an action that depends only on localization and battery. A changed or stale localization field does.

An invalidated action returns to preparation or asks the model to decide again; it is never silently executed against different arguments or weakened preconditions.

### Persistence

Snapshot metadata and compact structured fields live in the same SQLite database as the mission journal. A snapshot insert and its `WORLD_SNAPSHOT_CAPTURED` journal event commit in one transaction. Images, video, maps, and point clouds live in a content-addressed artifact directory and are referenced by digest, media type, size, and source timestamp.

Persisted snapshots are immutable. A corrected interpretation creates a new snapshot or a correction event; it does not update the prior row.

The journal is not a substitute for physical truth. Reopening it may reveal an unresolved action, but this prototype only exposes that record and prevents automatic redispatch. Querying controllers, deciding whether to stop, and reconciling recorded state against observed reality belong to the restart-recovery workstream.

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
| Resources | `LEASE_ACQUIRED`, `LEASE_RENEWED`, `LEASE_RELEASED`, `LEASE_EXPIRED`, `LEASE_BLOCKED` |
| Dispatch | `ACTION_DISPATCH_REQUESTED`, `ACTION_DISPATCH_ACCEPTED`, `ACTION_DISPATCH_FAILED` |
| Execution | `ACTION_STARTED`, `ACTION_PROGRESS`, `ACTION_COMPLETION_REPORTED`, `ACTION_EXECUTION_FAILED` |
| Verification | `ACTION_VERIFICATION_STARTED`, `ACTION_VERIFIED`, `ACTION_VERIFICATION_FAILED` |
| Settlement | `ACTION_SUCCEEDED`, `ACTION_FAILED`, `ACTION_UNKNOWN` |
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

If the process exits between steps two and three, the journal exposes an unresolved dispatch. This prototype does not assume failure or automatically retry it; controller reconciliation is deferred.

## LLM projections

The chat transcript is not the runtime database. `ProjectionBuilder` deterministically derives a bounded `LLMContextProjection` containing:

- Mission goal, constraints, and current status.
- Decision snapshot version and relevant fresh fields.
- Current short plan when present.
- Active actions, state, elapsed time, and owned resources.
- Recent salient progress, failure, safety, and intervention events.
- Unresolved or unknown actions.
- Available standard tools appropriate to the current state.
- Artifact references selected for the next model request.
- The exact decision question the model must answer.

Routine controller progress is coalesced. Safety events, terminal outcomes, unknown effects, and user steering are never dropped.

Every model turn journals the projection version or digest and the response that resulted from it. This makes evaluations reproducible without placing the full event log in the prompt.

Other projections include current mission state, active action and lease state, unresolved-action visibility, operator timeline, and evaluation metrics. Projection checkpoints are caches; deleting them and replaying the immutable journal must produce the same result.

## Action execution flow

### Model decision and preparation

1. A user command or salient runtime event requests a decision.
2. The harness captures and persists a decision snapshot.
3. The projection builder creates bounded model context from the snapshot and journal.
4. The model returns text and zero or more tool calls.
5. The harness journals the model response and standard tool calls while preserving each tool-call ID.
6. The normal tool dispatcher resolves each tool using existing discovery. Uncontracted tools continue through the ordinary execution path.
7. For a tool with a `PhysicalActionContract`, the dispatcher derives and journals an `ActionIntent` tied to the original tool-call ID.
8. The preparer resolves the trusted contract, validates and canonicalizes arguments, and emits a `PreparedAction` without side effects.

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

The harness may execute independent ordinary tool calls according to the existing dispatcher semantics and may prepare independent contracted physical tool calls concurrently. Contracted physical calls may execute concurrently only after the lease manager proves that their resources are disjoint and the safety supervisor admits the combination. The journal's global sequence provides a deterministic ordering for projection even when execution overlaps.

### Monitoring and verification

Controllers retain high-rate feedback loops. The runner consumes correlated, low-rate `ActionEvent`s and relevant world changes. The contract determines which event requests verification.

On reported completion, the runner captures a fresh snapshot and evaluates the named verifier. A verified expected effect settles success. A failed verification settles failure or unknown and may ask the model to replan; this prototype does not run a generic physical recovery procedure.

### Cancellation and restart boundaries

Cancellation of model inference remains distinct from cancellation of a physical action. The prototype may record a cancellation request or expose a robot-specific stop operation, but it does not define a generic cancellation state machine, invoke stop handlers automatically, verify stopped state, or release physical resources based on a stop request. Those behaviors are the responsibility of a separate cancellation workstream and existing controller-level safety systems.

The journal, stable action IDs, correlated events, and resource ownership established here provide inputs for that future workstream. Until it exists, a timed-out or ambiguous physical action becomes `UNKNOWN`, remains visible, and leaves its affected resources unavailable for automatic reuse.

When the journal is reopened, the prototype validates storage and rebuilds projections. If it finds a nonterminal or `UNKNOWN` physical action, it surfaces the action and prevents automatic redispatch of that recorded command. It does not query controllers, infer current physical state, issue stop commands, reconcile outcomes, or resume actions. Those behaviors belong to a separate restart and recovery workstream.

## Integration with existing DIMOS primitives

- Preserve `Module`, `Blueprint`, typed stream, transport, controller, and robot-specific skill implementations.
- Extend `@skill` and the core `SkillInfo` with an optional `PhysicalActionContract` while retaining existing tool schemas, call IDs, result messages, and call sites.
- Use `Dimos.connect()` and `SkillsProxy` from [`dimos/porcelain/dimos.py`](/dimos/porcelain/dimos.py#L107) and [`skills_proxy.py`](/dimos/porcelain/skills_proxy.py#L47) for built-in native dispatch.
- Keep `McpServer` for external agents. Its `tools/list` definitions remain standard and do not expose or duplicate the physical action contract. On invocation, server-side trusted discovery determines whether the call enters the `ActionRuntimeSpec` hosted by the same coordinator-visible `RobotAgentHarnessModule`. MCP must not directly dispatch contracted physical calls or become a second authority for safety or resource ownership. Uncontracted calls retain their ordinary path during migration.
- Adapt legacy `ToolStream` messages into typed action progress during migration. New physical skills should emit correlated `ActionEvent`s directly.
- Remove `CapabilityRegistry` from the dispatch path for contracted physical actions. It remains available to ordinary calls that already use it; `ActionRuntimeSpec` routes both native and MCP contracted calls through the same `LeaseManager`.
- Reuse `SkillResult` from [`skill_result.py`](/dimos/agents/skill_result.py#L55) for structured synchronous results, but do not equate `success=True` with verified physical success unless the contract permits it.
- Reuse replay and `MockModel` fixtures for deterministic integration and end-to-end tests.

## Failure handling

| Failure | Required behavior |
|---|---|
| Tool declared for managed physical execution has a missing or invalid contract | Reject before dispatch and journal the registration or preparation failure; an ordinary uncontracted tool is not an error |
| Required observation missing or stale | Capture again if bounded; otherwise reject without invoking the skill |
| Resource conflict | Wait within policy or reject; never run conflicting actions concurrently |
| Safety rejection | Journal the policy reason and do not invoke the skill |
| Approval delay | Revalidate the world after approval before dispatch |
| Executor proves the RPC was never invoked | Journal dispatch failure and release leases |
| RPC exception after invocation or with ambiguous delivery | Settle `UNKNOWN`, keep affected resources unavailable, surface the condition, and never retry merely because an exception was raised |
| Lost response after possible dispatch | Settle `UNKNOWN`, keep affected resources unavailable, and never blind-retry |
| Progress stream disconnect | Settle `UNKNOWN` unless another trusted completion source proves the outcome; do not start generic reconciliation |
| Execution timeout | Settle `UNKNOWN`, retain resource ownership, and surface the unresolved action; do not start generic cancellation |
| Verification failure | Settle failure or unknown and optionally request replanning; do not run generic physical recovery |
| Journal write failure before dispatch | Do not dispatch |
| Journal write failure after dispatch | Revoke further dispatch authority, surface a fatal unresolved condition, and rely on independent controller safety rather than an in-scope generic stop workflow |
| Snapshot persistence failure | Do not use the unpersisted snapshot for physical admission |

## Observability and evaluation

The journal provides exact metrics without parsing prose:

- Mission and action success rate.
- Preparation and admission rejection counts by reason.
- Unsafe actions blocked.
- Resource conflicts prevented.
- Decision-to-dispatch latency.
- Verification latency and failure rate.
- Unknown and unresolved outcome rate.
- Automatic redispatches of unresolved physical commands; the target is zero.
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

- An ordinary uncontracted tool preserves the standard tool-call ID and result message and does not create an `ActionIntent`.
- A recorded model fixture proposes a contracted physical action.
- The harness captures a decision snapshot and a fresher admission snapshot.
- A stale required observation prevents skill invocation.
- An admitted action holds its exclusive resource through verification.
- Progress and completion are correlated by action ID and stored as typed events.
- A controller completion report alone does not settle success; a fresh snapshot verifier does.
- An ambiguous or timed-out action becomes visible as `UNKNOWN`, keeps affected resources unavailable, and is not automatically redispatched.
- Journal updates and deletes are rejected, and hash-chain verification detects mutation.
- Rebuilding projections from the journal produces the same mission and action state.
- A replay-based end-to-end test persists snapshots from the repository's `go2_short.db` sample.

## Deferred extensions

- A cancellation workstream covering stop contracts, `STOPPING` and `CANCELLED` states, timeout-driven stopping, controller-specific stop verification, and safe resource release.
- A restart and recovery workstream covering controller reconciliation, recovery snapshots, outcome repair events, operator-assisted clearance, and explicitly authorized replay policies.
- Shared, capacity, and spatial-zone leases.
- Priority scheduling and emergency preemption.
- Behavior-tree or DAG execution compiled from mission steps.
- Fleet-level allocation and robot leases.
- Learned VLA policies as contracted physical skills.
- Remote replicated journals and artifact storage.
- Cryptographic signatures and external audit anchoring.
