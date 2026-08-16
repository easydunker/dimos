---
title: "Robot Action Runtime Prototype Implementation Plan"
---

# Robot Action Runtime Prototype Implementation Plan

Status: not started.

This document is both the implementation handoff and the progress tracker for the [Robot Action Runtime specification](/docs/capabilities/agents/robot_action_runtime_spec.md). A downstream implementer should be able to execute it without additional design decisions.

## How to use this plan

- Check an item only after its stated tests and acceptance criteria pass.
- Work phases in order unless a phase explicitly says it may run in parallel.
- Every behavior change follows red-green-refactor: add a failing test, implement the smallest passing change, then refactor with the suite green.
- Do not weaken, delete, or skip an existing test to make a phase pass.
- Tests must use deterministic clocks, IDs, fixtures, and event synchronization. Do not add fixed sleeps.
- Tests own cleanup through fixtures or context managers.
- Keep imports at module scope and do not add `__init__.py` re-exports.
- Use `tmp_path` for journals and artifacts; runtime defaults use `STATE_DIR`.
- Run the focused test listed for each phase before running broader regression tests.

Progress legend: `[ ]` not started, `[x]` complete. If work is partially complete, leave the parent unchecked and check only finished child items.

## Definition of done

- [ ] The specification's prototype acceptance criteria all pass.
- [ ] The built-in prototype dispatch path is native and does not make localhost MCP HTTP calls.
- [ ] Existing MCP clients and ordinary uncontracted tools retain their standard schemas, call IDs, dispatch behavior, and result messages.
- [ ] Unit, component, process-integration, recovery, and replay end-to-end tests pass without hardware or a live model.
- [ ] No physical action is dispatched before its journal intent commit.
- [ ] No physical resource is released before verified termination or explicit unknown-state escalation.
- [ ] The replayable journal rebuilds identical projections after restart.
- [ ] Ruff, mypy, documentation-link checks, and the relevant pytest suites pass.

## Prototype scope

The prototype implements:

- Standard tool-call compatibility with optional `PhysicalActionContract` routing for managed physical skills.
- Immutable action records and the complete action state machine.
- SQLite mission journal, snapshot storage, hash-chain validation, and immutable-table triggers.
- Content-addressed local artifact storage.
- Go2 world snapshots containing odometry, camera artifact reference, active actions, movement lease, and safety state; battery is optional because `go2_short.db` has no low-state stream.
- Exclusive resource leases with action ownership, deadlines, heartbeats, and emergency revocation.
- Freshness, precondition, resource, safety, and approval admission hooks.
- Native skill dispatch with action correlation.
- Typed progress, completion, cancellation, and verification.
- Journal-derived mission, active-action, recovery, and LLM context projections.
- Recorded-model and `go2_short.db` end-to-end tests.
- One migrated production physical skill: `navigate_with_text`.

The prototype does not implement shared, capacity, or zone leases; fleet allocation; behavior trees; a remote journal; or cryptographic signatures.

## Planned file layout

The target layout is listed here so implementers do not invent competing locations. Split a file further if it becomes difficult to review, but preserve these public boundaries.

| Path | Purpose |
|---|---|
| `dimos/agents/runtime/models.py` | Frozen enums and domain records |
| `dimos/agents/runtime/transitions.py` | Action transition validation |
| `dimos/agents/runtime/journal.py` | Journal protocol and SQLite implementation |
| `dimos/agents/runtime/artifacts.py` | Content-addressed artifact protocol and local implementation |
| `dimos/agents/runtime/snapshots.py` | Harness-owned snapshot provider, persistence, freshness comparison, and repository API |
| `dimos/agents/runtime/snapshot_module.py` | Go2 typed-stream cache and detached observation-draft RPC |
| `dimos/agents/runtime/physical_action_contracts.py` | Physical action contract records, resolution, and predicate/verifier registries |
| `dimos/agents/runtime/leases.py` | Exclusive lease manager |
| `dimos/agents/runtime/admission.py` | Ordered admission gates and decisions |
| `dimos/agents/runtime/executor.py` | Native `SkillsProxy` executor and execution handle |
| `dimos/agents/runtime/action_events.py` | Typed event publisher/subscriber and legacy ToolStream adapter |
| `dimos/agents/runtime/action_runner.py` | Action state machine orchestration |
| `dimos/agents/runtime/action_runtime_spec.py` | RPC protocol used by native and MCP callers |
| `dimos/agents/runtime/projections.py` | Deterministic journal projections |
| `dimos/agents/runtime/harness.py` | Lifecycle-owning root and model-turn integration |
| `dimos/agents/runtime/harness_module.py` | Coordinator module that owns one harness and exposes `ActionRuntimeSpec` |
| `dimos/agents/runtime/testing.py` | Reusable deterministic fakes for runtime tests |
| `dimos/agents/runtime/fixtures/` | Small recorded model responses only |
| `dimos/cli/agent_harness.py` | Experimental native harness CLI |
| `dimos/e2e_tests/test_agent_runtime_replay.py` | Replay and restart acceptance tests |

Tests for each source file live beside it as `test_<name>.py`. Do not place implementation-only test helpers in `conftest.py`; reusable fakes belong in `dimos/agents/runtime/testing.py`.

## Test layers

| Layer | Dependencies | Purpose |
|---|---|---|
| Unit | Fake clock, deterministic ID source, in-memory collaborators | Models, transitions, freshness, leases, policies, projections |
| Storage component | SQLite and local artifact directory under `tmp_path` | Transactions, immutability, hash chain, restart |
| Runtime component | Scripted executor, snapshot provider, verifier, safety policy | Full `ActionRunner` behavior and fault injection |
| Process integration | DIMOS coordinator, modules, LCM, recorded `MockModel` | Native discovery, RPC, correlation, cleanup |
| Replay end-to-end | `go2_short.db`, recorded `MockModel`, no hardware | Real camera/odometry streams, snapshots, journal, recovery |

## Phase 0: baseline and scaffolding

Goal: establish a clean baseline and package boundaries before behavior changes.

- [ ] Record `git status --short` and preserve unrelated user changes.
- [ ] Run `uv run pytest dimos/agents/mcp/test_mcp_client_unit.py dimos/agents/test_capabilities.py dimos/agents/mcp/test_tool_stream.py -v`.
- [ ] Run `uv run mypy dimos/agents dimos/porcelain` and record any pre-existing failures separately.
- [ ] Create `dimos/agents/runtime/` without `__init__.py` re-exports.
- [ ] Add a test-only fake clock, deterministic ID source, snapshot provider, safety policy, executor, verifier, and event source in `testing.py`.
- [ ] Ensure every fake records exact calls and supports event-based synchronization without sleeping.

Phase acceptance:

- [ ] Existing focused tests still pass.
- [ ] Importing any new runtime module performs no I/O, starts no thread, and opens no database.

## Phase 1: domain models and state transitions

Goal: define immutable records and reject illegal state changes before adding storage or execution.

### Red

- [ ] Add tests that all persistent records are frozen and compare by value.
- [ ] Add tests that nested argument, metadata, observation, and payload mappings cannot mutate a record after construction.
- [ ] Add table-driven tests for every permitted action transition.
- [ ] Add table-driven tests proving every unlisted transition is rejected.
- [ ] Add tests for terminal-state detection and `UNKNOWN` classification.
- [ ] Add tests that model arguments cannot populate trusted contract fields.
- [ ] Add canonical JSON round-trip tests for every persisted enum and record.

### Green

- [ ] Implement string enums for risk, physical-action replay policy, action state, event type, lease status, and next disposition; reuse existing `SkillInfo` lifecycle metadata rather than defining a competing execution-mode enum.
- [ ] Implement frozen records for `ActionIntent`, `RequiredObservation`, `ResourceRequest`, `PhysicalActionContract`, `PreparedAction`, `ActionEvent`, `ActionOutcome`, `TimedObservation`, `ArtifactRef`, `WorldSnapshotDraft`, `WorldSnapshot`, and `JournalEvent`.
- [ ] Implement explicit codec functions with schema versions. Do not persist `repr()` output or pickle.
- [ ] Recursively freeze JSON mappings and sequences at record construction and thaw them only inside explicit codecs.
- [ ] Implement one authoritative transition table and transition validator.
- [ ] Reject non-JSON-compatible metadata at record construction or encoding time.

### Refactor and acceptance

- [ ] Remove duplicate string literals in favor of shared enums and constants.
- [ ] Run `uv run pytest dimos/agents/runtime/test_models.py dimos/agents/runtime/test_transitions.py -v`.
- [ ] Run `uv run mypy dimos/agents/runtime/models.py dimos/agents/runtime/transitions.py`.

## Phase 2: append-only mission journal

Goal: provide durable ordering, immutable snapshots, effect-intent persistence, and restart reads.

### Red

- [ ] Test that appends receive strictly increasing sequence numbers.
- [ ] Test that events survive closing and reopening the SQLite store.
- [ ] Test that canonical payloads produce a deterministic event hash chain.
- [ ] In a copied test database, temporarily drop the immutability trigger, mutate an event with a raw SQLite connection, restore the trigger, and test that hash-chain verification fails on reopen.
- [ ] Test that SQL `UPDATE` and `DELETE` against journal events are rejected by database triggers.
- [ ] Test atomic insertion of a world snapshot and its `WORLD_SNAPSHOT_CAPTURED` event.
- [ ] Test rollback leaves neither row when either half of snapshot persistence fails.
- [ ] Test concurrent append callers produce a gap-free, unique order.
- [ ] Test schema-version rejection for a database newer than the reader.
- [ ] Test that a journal write failure is surfaced synchronously to the caller.

### Green

- [ ] Define a narrow `MissionJournal` protocol used by the runner and projection code.
- [ ] Implement `SqliteMissionJournal` with explicit `start()` and `stop()` lifecycle.
- [ ] Enable WAL and full synchronous durability on the runtime connection.
- [ ] Allocate the next sequence under `BEGIN IMMEDIATE`; do not use a process-global counter.
- [ ] Compute SHA-256 hashes over canonical envelope data plus the previous event hash.
- [ ] Create insert-only triggers for `journal_events` and `world_snapshots`.
- [ ] Store compact snapshot JSON and its digest in `world_snapshots`.
- [ ] Add read APIs by mission, action, sequence range, and event type.
- [ ] Add `verify_chain()` and call it during recovery startup.
- [ ] Default runtime storage beneath `STATE_DIR / "agent_runtime"`; accept an explicit path in configuration.

### Refactor and acceptance

- [ ] Keep SQL migrations and canonical encoding centralized.
- [ ] Ensure all connections and cursors are fixture-owned and closed on assertion failure.
- [ ] Run `uv run pytest dimos/agents/runtime/test_journal.py -v`.
- [ ] Run `uv run mypy dimos/agents/runtime/journal.py`.

## Phase 3: artifact store and world snapshots

Goal: assemble versioned snapshots from asynchronous inputs and persist large payloads by reference.

### Red

- [ ] Test that one capture time is used to calculate every observation age.
- [ ] Test that capture returns a detached immutable value while new stream updates continue.
- [ ] Test explicit missing observations; no placeholder pose, battery, or confidence is allowed.
- [ ] Test that stale and fresh classifications use an injected monotonic clock.
- [ ] Test that comparison examines only fields declared by a physical action contract.
- [ ] Test that equal artifact bytes deduplicate to the same SHA-256 identifier.
- [ ] Test that artifact metadata preserves media type, byte size, source time, and digest.
- [ ] Test that snapshot persistence stores image references rather than image bytes.
- [ ] Test snapshot version monotonicity across journal restart.
- [ ] Test that persistence failure prevents the snapshot from being returned as admissible.

### Green

- [ ] Implement a thread-safe snapshot-source assembler whose update and draft-capture paths use one lock, or an async single-owner equivalent.
- [ ] Implement `LocalArtifactStore` with atomic temporary-file rename and content-addressed paths.
- [ ] Implement the snapshot repository on top of the journal transaction API.
- [ ] Implement required-field freshness evaluation and relevant-field diffing.
- [ ] Implement the harness-owned `SnapshotProvider`: request a detached draft, merge action, lease, and safety projections, persist artifacts, then atomically persist the versioned snapshot and journal event.
- [ ] Implement `SnapshotSource` and `ArtifactStore` protocols for test injection.
- [ ] Implement `WorldSnapshotModule` with typed `odom` and `color_image` inputs and registered disposable subscriptions.
- [ ] Add optional battery and navigation-state inputs without fabricating values when a replay lacks them.
- [ ] Expose an RPC that returns a detached observation draft with one capture time and a supplied trigger. The module must not open or write the mission journal.
- [ ] Keep version allocation and the merge of active actions, resource ownership, and safety state in the harness-owned provider so the journal retains one writer.

### Refactor and acceptance

- [ ] Keep image serialization outside message types and snapshot records.
- [ ] Verify `start()` owns subscriptions and `stop()` calls `super().stop()` and releases them.
- [ ] Run `uv run pytest dimos/agents/runtime/test_artifacts.py dimos/agents/runtime/test_snapshots.py dimos/agents/runtime/test_snapshot_module.py -v`.
- [ ] Run `uv run mypy dimos/agents/runtime/artifacts.py dimos/agents/runtime/snapshots.py dimos/agents/runtime/snapshot_module.py`.

## Phase 4: physical-action-contract discovery

Goal: add optional managed-physical-execution metadata without changing standard tool behavior, current decorators, MCP, or native discovery.

### Red

- [ ] Test bare `@skill` retains its current schema, RPC behavior, and default metadata.
- [ ] Test existing `@skill(uses=[...], lifecycle=...)` remains compatible.
- [ ] Test an uncontracted tool uses the ordinary dispatcher, preserves its standard tool-call ID and result message, and does not create an `ActionIntent`.
- [ ] Test a contracted physical skill exposes every required `PhysicalActionContract` field through core `SkillInfo` without changing its model-visible tool schema.
- [ ] Test invalid physical action contracts fail during class or module setup, before the robot can run.
- [ ] Test a physical action contract without resources, timeout, stop handler, or verifier is rejected.
- [ ] Test the physical action contract cannot redefine the standard tool name, description, argument schema, call ID, result envelope, or lifecycle.
- [ ] Test the model-visible schema cannot override trusted contract metadata.
- [ ] Test native and MCP dispatch resolve the same server-side physical action contract ID and version without exposing that contract in the standard model-facing schema.
- [ ] Test only tools with a valid physical action contract can enter the physical runner.

### Green

- [ ] Extend `skill()` with one optional typed `PhysicalActionContract` argument while preserving both existing decorator forms and standard tool-call behavior.
- [ ] Attach the immutable contract to the RPC wrapper and include it in core `SkillInfo`.
- [ ] Update module skill discovery and serialization for the new optional fields.
- [ ] Update `SkillsProxy` to expose skill info to `PhysicalActionContractRegistry` without reaching into private fields.
- [ ] Keep MCP `tools/list` output backward compatible; resolve physical action contracts only in trusted server-side dispatch.
- [ ] Implement `PhysicalActionContractRegistry` with live refresh and ambiguity detection matching `SkillsProxy` behavior.
- [ ] Implement named predicate, verifier, and stop-handler registries owned by the harness root.

### Refactor and acceptance

- [ ] Do not duplicate `SkillInfo` definitions; reconcile or clearly separate the core RPC record from UI introspection metadata.
- [ ] Add core skill-discovery cases in `dimos/core/test_module_skills.py` and run `uv run pytest dimos/agents/test_annotation.py dimos/core/test_module_skills.py dimos/porcelain/test_skills_proxy.py dimos/agents/mcp/test_mcp_server.py -v`.
- [ ] Run `uv run mypy dimos/agents/annotation.py dimos/core/module.py dimos/porcelain/skills_proxy.py`.

## Phase 5: leases and admission

Goal: guarantee one owner for physical resources and make all pre-dispatch gates deterministic.

### Red

- [ ] Test atomic acquisition of multiple exclusive resources.
- [ ] Test that a conflict acquires none of the requested resources.
- [ ] Test renew and release require the owning action ID and lease token.
- [ ] Test an expired lease becomes unsafe and invokes the configured deadman callback; it is not silently reused.
- [ ] Test emergency revocation works regardless of normal priority.
- [ ] Test expiry and emergency revocation remove dispatch authority but quarantine physical resources until verified safe; neither operation makes a resource reusable.
- [ ] Test model actions cannot preempt another normal action in the prototype.
- [ ] Test admission rejects missing and stale required observations without invoking the executor.
- [ ] Test unrelated snapshot changes do not reject the action.
- [ ] Test relevant changes force revalidation after approval or lease wait.
- [ ] Test safety constraints can reduce an action but never expand its contract limits.
- [ ] Test every rejection contains a stable code and journal event.
- [ ] Test provisional leases are released when final revalidation fails.

### Green

- [ ] Implement a harness-owned `LeaseManager` using a lock and injected monotonic clock.
- [ ] Represent ownership with action ID, opaque token, deadline, heartbeat time, priority, and status; distinguish active, revoked, and quarantined from available.
- [ ] Implement acquire, renew, release, snapshot, conflict, and emergency-revoke operations.
- [ ] Implement ordered admission gates: schema, initial snapshot, preconditions, safety, approval, lease acquisition, final snapshot revalidation, final safety.
- [ ] Return immutable `AdmissionDecision` values with stable reason codes.
- [ ] Journal all approval, safety, and lease transitions.
- [ ] Add an `ActionRuntimeSpec` RPC boundary owned by the harness root. Native and MCP callers use this same boundary for contracted physical actions.
- [ ] Update `McpServer` so contracted physical calls route through `ActionRuntimeSpec` and skip direct RPC dispatch and `CapabilityRegistry`; uncontracted calls retain their ordinary path and standard MCP responses.
- [ ] Test that a native call and an MCP call for `base.motion` conflict through the same `LeaseManager`, proving there is no split ownership.

### Refactor and acceptance

- [ ] Keep lease state grouped in one record and lock every read and write.
- [ ] Run `uv run pytest dimos/agents/runtime/test_leases.py dimos/agents/runtime/test_admission.py -v`.
- [ ] Run `uv run mypy dimos/agents/runtime/leases.py dimos/agents/runtime/admission.py`.

## Phase 6: native execution and typed action events

Goal: invoke skills without MCP HTTP and preserve action correlation through progress and completion.

### Red

- [ ] Test `SkillsProxyExecutor` resolves exactly one skill and passes canonical arguments.
- [ ] Test the executor injects the harness action ID into trusted call context, not the model arguments.
- [ ] Test synchronous success, synchronous structured failure, RPC exception, timeout, and lost-response outcomes.
- [ ] Test background invocation returns a handle before physical completion.
- [ ] Test two concurrent invocations of the same skill are distinguished by action ID.
- [ ] Test typed action events preserve event type, progress, source time, status code, and action ID.
- [ ] Test the legacy ToolStream adapter preserves `progressToken` correlation instead of reducing it to tool name.
- [ ] Test uncorrelated legacy messages are operator logs and cannot settle an action.
- [ ] Test terminal and stopped events are delivered before stream teardown.

### Green

- [ ] Define `SkillExecutor` and `ActionEventSource` protocols.
- [ ] Implement `SkillsProxyExecutor` using `Dimos.connect()` and public `SkillsProxy` APIs.
- [ ] Run blocking RPC calls in a bounded worker owned and stopped by the executor.
- [ ] Extend skill call context with `action_id` while retaining MCP progress-token behavior.
- [ ] Implement typed action event publication over the configured DIMOS transport.
- [ ] Implement a migration adapter from ToolStream progress notifications to `ActionEvent`.
- [ ] Include action correlation on the stopped notification used for lifecycle completion.
- [ ] Ensure executor shutdown cancels waits, closes the remote connection, and joins owned workers.

### Refactor and acceptance

- [ ] Keep transport encoding separate from action state-machine logic.
- [ ] Run `uv run pytest dimos/agents/runtime/test_executor.py dimos/agents/runtime/test_action_events.py dimos/agents/mcp/test_tool_stream.py -v`.
- [ ] Run `uv run mypy dimos/agents/runtime/executor.py dimos/agents/runtime/action_events.py`.

## Phase 7: `ActionRunner`

Goal: implement the complete prepared-action transaction and failure semantics.

### Red

- [ ] Happy path: assert the exact event subsequence from proposal through verified success.
- [ ] Assert preparation performs no executor call, lease acquisition, or physical side effect.
- [ ] Assert dispatch does not occur when the pre-dispatch journal append fails.
- [ ] Assert dispatch occurs only after `ACTION_DISPATCH_REQUESTED` is durable.
- [ ] Assert a synchronous RPC success enters verification rather than directly settling a physical action.
- [ ] Assert completion events with the wrong action ID are ignored and journaled as diagnostics.
- [ ] Assert a failed verifier produces failure or recovery according to contract.
- [ ] Assert timeout starts cancellation and does not directly release resources.
- [ ] Assert cancellation before executor invocation settles directly and releases provisional leases; cancellation after possible invocation enters `STOPPING`.
- [ ] Assert successful cancellation requires stop verification before `ACTION_CANCELLED` and lease release.
- [ ] Assert failed stop verification produces `ACTION_UNKNOWN`, safety escalation, and a quarantined resource that rejects new normal actions.
- [ ] Assert a lost dispatch response produces `ACTION_UNKNOWN` and no automatic retry.
- [ ] Assert two admitted actions with disjoint resources may execute concurrently while conflicting actions cannot.
- [ ] Assert all exit paths release nonphysical resources and close subscriptions.
- [ ] Assert user cancellation and safety cancellation are distinguishable in the journal.
- [ ] Assert `STOP_COMMAND_SENT` commits before the stop handler is invoked, including the journal-failure path.
- [ ] Assert every invoked stop command records accepted, failed-before-invocation, or unknown-delivery settlement before stop verification.
- [ ] Assert only a proven pre-invocation failure settles `FAILED`; any ambiguous RPC delivery settles `UNKNOWN` and keeps resources quarantined.

### Green

- [ ] Implement `ActionPreparer` as a pure dependency of the runner.
- [ ] Implement `ActionRunner` with one transition function and one cleanup path.
- [ ] Persist every state transition before publishing it to projections or UI.
- [ ] Capture decision, admission, verification, and cancellation snapshots at the specified boundaries.
- [ ] Monitor event source and timeout concurrently with explicit cancellation.
- [ ] Invoke verifiers through the trusted registry using a fresh snapshot.
- [ ] Return a structured `ActionOutcome` for every terminal path.
- [ ] Add deterministic recovery hooks, initially limited to stop-and-reconcile.
- [ ] Ensure fatal post-dispatch journal failure revokes action authority and triggers safe-stop escalation.

### Refactor and acceptance

- [ ] Keep policy, persistence, execution, and verification behind injected protocols.
- [ ] Ensure there is no broad `except Exception` around the complete run method.
- [ ] Run `uv run pytest dimos/agents/runtime/test_action_runner.py -v`.
- [ ] Run `uv run mypy dimos/agents/runtime/action_runner.py`.

## Phase 8: projections and restart recovery

Goal: rebuild all agent-facing state from immutable history and safely reconcile interrupted actions.

### Red

- [ ] Test mission-state projection from an empty journal and from a complete successful mission.
- [ ] Test active-action and lease projections for every transition.
- [ ] Test progress coalescing keeps the latest routine update while retaining safety and terminal events.
- [ ] Test the LLM projection includes goal, constraints, snapshot version, relevant observations, active actions, resources, salient events, unknown effects, and decision question.
- [ ] Test deterministic event replay produces byte-equivalent canonical projection output.
- [ ] Delete projection checkpoints and test that replay rebuilds the same state.
- [ ] Test restart identifies `DISPATCHING`, `EXECUTING`, `VERIFYING`, and `STOPPING` actions as unresolved.
- [ ] Test unresolved `NEVER_IF_UNKNOWN` actions are not dispatched again.
- [ ] Test reconciliation queries current state, requests stop where required, captures a recovery snapshot, and appends a terminal or unknown outcome.
- [ ] Test corrupted journal startup fails before any executor is available.

### Green

- [ ] Implement pure fold functions for mission, active-action, lease, recovery, and LLM projections.
- [ ] Implement deterministic salience and progress-coalescing rules.
- [ ] Add optional rebuildable projection checkpoints only after replay behavior passes.
- [ ] Implement `RecoveryManager` owned by the harness lifecycle root.
- [ ] Block conflicting new actions until recovery reconciliation finishes.
- [ ] Expose unresolved and unknown actions in both operator and model projections.

### Refactor and acceptance

- [ ] Ensure projections never perform robot I/O or append journal events.
- [ ] Run `uv run pytest dimos/agents/runtime/test_projections.py dimos/agents/runtime/test_recovery.py -v`.
- [ ] Run `uv run mypy dimos/agents/runtime/projections.py dimos/agents/runtime/recovery.py`.

## Phase 9: model loop and harness lifecycle

Goal: integrate the runtime with a bounded model decision loop while keeping durable state outside chat history.

### Red

- [ ] Test a recorded `MockModel` response containing an ordinary tool call is journaled with its standard tool-call ID.
- [ ] Test an uncontracted query tool uses the ordinary dispatcher, returns its normal structured tool result, and creates neither an `ActionIntent` nor a physical lease.
- [ ] Test a contracted physical tool call becomes one `ActionIntent` tied to the decision snapshot and the unchanged standard tool-call ID.
- [ ] Test physical tool wrappers call `ActionRunner`, never `SkillsProxy` directly.
- [ ] Test a physical `ActionOutcome` is encoded into a standard tool-result message associated with the original tool-call ID rather than returned as raw controller prose.
- [ ] Test a safety or unknown event wakes the model only after deterministic handling completes.
- [ ] Test routine progress does not trigger an LLM turn.
- [ ] Test user steering can cancel a model turn without falsely marking a robot action stopped.
- [ ] Test harness startup performs recovery before accepting new mission input.
- [ ] Test harness shutdown cancels model work, stops or hands off active actions according to policy, then closes subscriptions and stores.

### Green

- [ ] Implement `RobotAgentHarness` as the root owner of model adapter, standard tool dispatcher, journal, snapshot provider, physical action contracts, leases, executor, runner, projections, and recovery.
- [ ] Implement `RobotAgentHarnessModule` as the sole coordinator-visible lifecycle owner of one `RobotAgentHarness` and the `ActionRuntimeSpec` RPC implementation.
- [ ] Reuse LangGraph or the existing model abstraction only as the bounded decision loop; do not use chat history as mission state.
- [ ] Generate standard model tools from native skill discovery without changing their schemas; wrap only tools with a `PhysicalActionContract` using `ActionRunner`.
- [ ] Expose the same contracted physical action entrypoint to `McpServer` through `ActionRuntimeSpec`; do not maintain a separate MCP action runner.
- [ ] Record projection digest, model response, and tool calls after redaction.
- [ ] Implement bounded context construction from projections and artifact references.
- [ ] Add explicit start and stop lifecycle and no import-time work.
- [ ] Add the experimental `dimos agent-harness` CLI that connects to the `RobotAgentHarnessModule` in an already running DIMOS coordinator; fail clearly if the running blueprint does not contain the host module, and never create a second local runtime.
- [ ] Register the command in `dimos/cli/dimos.py` and cover command discovery in the existing CLI startup tests.
- [ ] Put model-fixture and journal-path configuration on the host module or blueprint. Support noninteractive one-message execution in the CLI without letting a CLI client replace those runtime-owned settings after startup.

### Refactor and acceptance

- [ ] Verify the native path performs no POST to the MCP server.
- [ ] Run `uv run pytest dimos/agents/runtime/test_harness.py dimos/cli/test_agent_harness.py dimos/cli/test_cli_startup.py -v`.
- [ ] Run `uv run mypy dimos/agents/runtime/harness.py dimos/cli/agent_harness.py`.

## Phase 10: migrate navigation as the vertical slice

Goal: demonstrate a real DIMOS physical skill whose resource lifetime matches controller lifetime.

### Red

- [ ] Reproduce the existing issue where `navigate_with_text` releases movement after `set_goal()` returns.
- [ ] Test the migrated `PhysicalActionContract` declares exclusive base movement, freshness bounds, timeout, stop handler, verifier, and `NEVER_IF_UNKNOWN` replay.
- [ ] Test tagged-location and semantic-map navigation return a correlated background action rather than claiming completion.
- [ ] Test goal-reached produces a correlated completion report.
- [ ] Test navigation success is verified from a fresh pose and zero-motion or idle-controller evidence.
- [ ] Test `stop_navigation` produces a stop event and only releases movement after stopped state is observed.
- [ ] Test follow, patrol, or exploration cannot acquire movement while navigation is executing.
- [ ] Test stale odometry rejects navigation before `set_goal()`.

### Green

- [ ] Add action ID propagation from the runner context into `NavigationSkillContainer`.
- [ ] Refactor `navigate_with_text` so starting a goal has a background lifecycle for every asynchronous path.
- [ ] Emit typed navigation started, progress where meaningful, completion, failure, and stopped events.
- [ ] Preserve the existing user-visible tool name, schema, call ID, and MCP result behavior.
- [ ] Add a trusted navigation verifier that checks controller state and target tolerance using a fresh snapshot.
- [ ] Route migrated movement ownership through `LeaseManager` and disable duplicate `CapabilityRegistry` acquisition for both native and MCP contracted action paths.
- [ ] Keep path planning, obstacle avoidance, and velocity control inside existing navigation modules.

### Refactor and acceptance

- [ ] Run `uv run pytest dimos/agents/skills/test_navigation.py dimos/navigation/replanning_a_star/test_global_planner.py dimos/navigation/basic_path_follower/test_module.py -v`.
- [ ] Run `uv run pytest dimos/agents/runtime/test_navigation_action.py -v`.
- [ ] Run `uv run mypy dimos/agents/skills/navigation.py dimos/agents/runtime`.

## Phase 11: deterministic process-integration test

Goal: prove the complete model-to-action-to-projection flow without LFS, hardware, or a live model.

- [ ] Add `dimos/agents/runtime/fixtures/test_scripted_navigation_mission.json` containing recorded `MockModel` responses.
- [ ] Add an ordinary uncontracted query tool and assert it preserves its standard call/result correlation, creates no `ActionIntent`, and acquires no physical lease.
- [ ] Add a scripted physical skill module that emits typed progress and changes a fake world state using event synchronization.
- [ ] Compose the scripted module, snapshot module, `RobotAgentHarnessModule`, and recorded model through a real `ModuleCoordinator`.
- [ ] Send one mission input: navigate to the scripted target.
- [ ] Assert exact journal ordering: decision snapshot, tool proposal, preparation, provisional lease, admission snapshot, admission grant, dispatch intent, acceptance, progress, completion report, verification snapshot, success, release.
- [ ] Assert the final LLM projection reports success and contains no raw high-rate state.
- [ ] Assert no MCP HTTP server is required.
- [ ] Assert fixture teardown stops the coordinator, transports, worker pool, executor, journal, and artifact store even on failure.
- [ ] Run `uv run pytest dimos/agents/runtime/test_harness_integration.py -v`.

## Phase 12: repository-sample replay end-to-end tests

Goal: validate real recorded camera and odometry flow using repository sample data.

Create `dimos/e2e_tests/test_agent_runtime_replay.py`. Mark tests that require LFS data with the repository's appropriate slow or self-hosted marker so the default suite remains fast. Resolve data through `get_data("go2_short.db")`; never hardcode a downloaded path.

### E2E A: replay snapshot persistence

- [ ] Start a replay connection over `go2_short.db` with a short bounded duration and no viewer.
- [ ] Subscribe the snapshot module to real replayed odometry and camera streams.
- [ ] Wait with events until both fields have arrived.
- [ ] Capture decision and admission snapshots at two different replay points.
- [ ] Assert increasing world versions, increasing source times, non-placeholder poses, bounded calculated ages, and content-addressed camera references.
- [ ] Reopen the SQLite journal and assert both snapshots and their capture events are intact.

### E2E B: stale-world admission

- [ ] Prepare a navigation action from a valid replay snapshot.
- [ ] Advance the injected admission clock beyond the odometry freshness bound without injecting a new odometry message.
- [ ] Run admission and assert a stable `STALE_OBSERVATION` rejection.
- [ ] Assert no navigation RPC was called, no movement lease remains, and the rejection references both snapshot versions.

### E2E C: replay-driven monitored action

- [ ] Use a test controller adapter that treats a later recorded odometry pose as its target and emits completion when replay reaches it.
- [ ] Use a recorded `MockModel` fixture to propose the navigation tool call.
- [ ] Assert movement ownership spans dispatch through fresh-snapshot verification.
- [ ] Assert progress is coalesced for model context but fully available in the operator timeline.
- [ ] Assert the final outcome is physically verified from the replayed pose, not from the controller report alone.

### E2E D: crash recovery without duplicate dispatch

- [ ] Start the harness with a persistent test journal and dispatch a scripted background action.
- [ ] Terminate only the harness after `ACTION_DISPATCH_REQUESTED` and controller acceptance, leaving an unresolved journal state.
- [ ] Restart against the same database and a controller adapter reporting that the action may still be active.
- [ ] Assert recovery runs before new input, requests stop, verifies stopped state, and appends reconciliation events.
- [ ] Assert the original skill invocation count remains exactly one.
- [ ] Assert the outcome is `CANCELLED` when stop is proven and `UNKNOWN` with a quarantined movement resource in the fault-injected no-acknowledgement variant.

### E2E E: projection rebuild

- [ ] Delete only `projection_checkpoints` after the replay mission.
- [ ] Reopen the journal and rebuild all projections.
- [ ] Assert canonical mission, action, lease, recovery, and LLM projections equal those captured before shutdown.

Replay acceptance commands:

- [ ] Run `uv run pytest -m self_hosted dimos/e2e_tests/test_agent_runtime_replay.py -v` after the LFS fixture is available.
- [ ] Run the same file twice to prove journal and test cleanup isolation.
- [ ] Confirm the tests make no OpenAI request and use no robot or simulator process.

## Phase 13: regression, documentation, and handoff

Goal: leave a reviewable prototype with reproducible evidence.

- [ ] Run all new runtime tests: `uv run pytest dimos/agents/runtime -v`.
- [ ] Run MCP regressions: `uv run pytest dimos/agents/mcp dimos/agents/test_capabilities.py -v`.
- [ ] Run native proxy regressions: `uv run pytest dimos/porcelain -v`.
- [ ] Run navigation regressions selected in Phase 10.
- [ ] Run the replay end-to-end suite from Phase 12.
- [ ] Run `uv run mypy dimos/agents/runtime dimos/agents/annotation.py dimos/core/module.py dimos/porcelain/skills_proxy.py dimos/agents/skills/navigation.py`.
- [ ] Run `uv run ruff format --check dimos/agents/runtime dimos/agents/annotation.py dimos/core/module.py dimos/porcelain/skills_proxy.py dimos/agents/skills/navigation.py`.
- [ ] Run `uv run ruff check dimos/agents/runtime dimos/agents/annotation.py dimos/core/module.py dimos/porcelain/skills_proxy.py dimos/agents/skills/navigation.py`.
- [ ] Run `doclinks docs/capabilities/agents/robot_action_runtime_spec.md docs/capabilities/agents/robot_action_runtime_implementation_plan.md`.
- [ ] Run `md-babel-py run docs/capabilities/agents/robot_action_runtime_spec.md --dry-run` and the same command for this plan.
- [ ] Regenerate `dimos/robot/all_blueprints.py` with `pytest dimos/robot/test_all_blueprints_generation.py` if a runnable built-in blueprint was added.
- [ ] Document the exact passing commands and any excluded platform-specific tests in the final handoff.
- [ ] Capture one example mission journal and render its action timeline for the final demonstration.

## Required fault-injection matrix

The following cases must be covered before declaring the prototype complete:

| Fault | Injection point | Required assertion |
|---|---|---|
| Stale odometry | Before final admission | Rejected; executor call count zero |
| Relevant world change | During approval or lease wait | Revalidated or rejected against new snapshot |
| Resource conflict | Lease acquisition | Only one action owns movement |
| Journal failure | Before dispatch | No physical call occurs |
| Lost RPC response | Immediately after executor receives call | Outcome unknown; no automatic retry |
| Controller failure | During execution | Structured failure and bounded recovery |
| Progress disconnect | While action runs | Controller reconciliation begins |
| Execution timeout | No terminal event | Stop protocol begins; lease remains held |
| Stop timeout | No zero-motion evidence | Safety escalation and unknown outcome |
| Verification mismatch | Controller claims success at wrong pose | Action does not succeed |
| Harness crash | After dispatch, before settlement | Restart reconciles; invocation count remains one |
| Corrupt journal | Before startup | Startup fails before skill dispatch is available |
| Artifact write failure | During snapshot capture | Snapshot is not admissible or journaled as complete |

## Review checklist

Architecture:

- [ ] One root object owns component lifecycle.
- [ ] No global mutable action, lease, snapshot, or journal registry exists.
- [ ] Model, policy, persistence, and execution responsibilities are separated.
- [ ] High-rate control remains inside robot controllers.
- [ ] MCP compatibility is preserved without granting MCP safety authority.

Correctness:

- [ ] Every physical effect has a durable pre-effect intent.
- [ ] Every action and event is correlated by action ID.
- [ ] Every success has verifier evidence and a verification snapshot.
- [ ] Every cancellation has stop evidence or an unknown-state escalation.
- [ ] Relevant observation freshness is checked after waits.
- [ ] Recovery never blind-replays an uncertain physical command.

Persistence:

- [ ] Journal and snapshots reject update and delete.
- [ ] Hash-chain validation runs on recovery startup.
- [ ] Snapshot and capture event are atomic.
- [ ] Large artifacts are referenced, not embedded.
- [ ] Projection checkpoints are disposable and rebuildable.

Testing:

- [ ] Tests use injected clocks and IDs.
- [ ] Tests contain assertions and no prints.
- [ ] Async tests use events or bounded polling, not fixed sleeps.
- [ ] Fixtures clean up databases, transports, modules, processes, and threads.
- [ ] Recorded model fixtures remove live-model nondeterminism.
- [ ] Replay tests use `get_data("go2_short.db")`.

## Recommended delivery order

For a time-bounded implementation, the first reviewable milestone is Phases 0 through 8, Phase 11, and E2E A through D. That milestone demonstrates the design with a scripted physical skill but is not the final definition of done. The completed prototype also includes the production navigation migration in Phase 10; if it cannot be completed safely within the time box, leave its checklist open and describe the submission as a vertical runtime prototype rather than a finished DIMOS integration.

Do not cut journal durability, stale-state rejection, verified cancellation, or crash recovery to add UI polish. Those behaviors are the robotics-specific value of the prototype.
