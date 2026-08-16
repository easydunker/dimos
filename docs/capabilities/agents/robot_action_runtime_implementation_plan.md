---
title: "Closed-Loop Robot Agent Harness Implementation Plan"
---

# Closed-Loop Robot Agent Harness Implementation Plan

Status: proposed prototype plan.

Specification: [Closed-Loop Robot Agent Harness Specification](robot_action_runtime_spec.md).

## How to use this plan

Implement the phases in order. Each phase should land with focused tests and leave ordinary DIMOS skills and MCP clients working. Navigation is the first physical vertical slice; do not generalize the metadata or lifecycle until that slice proves the interfaces.

The plan deliberately concentrates on three outcomes:

1. durable recording of standard tool calls and results;
2. physical action tracking from command acceptance through observed outcome; and
3. harness-managed world snapshots and model-context injection.

## Definition of done

The prototype is done when a deterministic mission can:

1. capture and inject current robot state before a model decision;
2. receive an unchanged standard navigation tool call from the model;
3. record the call and its immediate accepted result;
4. correlate navigation progress and completion with that call;
5. capture fresh state and verify the physical outcome;
6. record the outcome and expose it to the next model turn; and
7. reproduce the same ordered record in an integration test and a repository replay test.

## Scope

In scope:

- a SQLite mission journal;
- standard tool-call and tool-result recording;
- snapshot cache, capture, persistence, and bounded projection;
- optional robot-action metadata on existing skills;
- typed robot action progress and terminal events;
- physical action lifecycle and verification;
- a serialized harness decision loop;
- navigation migration as the initial vertical slice;
- deterministic and replay-based tests.

Out of scope:

- a replacement for standard tools or MCP;
- changes to existing capability coordination;
- general resource arbitration or cross-client concurrency control;
- approval and risk-policy systems;
- cancellation and in-flight restart recovery;
- multi-agent scheduling;
- tamper-evident storage; and
- migration of every physical skill in the prototype.

## Proposed file layout

Use a focused package rather than adding more responsibilities to `McpClient`:

```text
dimos/agents/harness/
├── __init__.py
├── models.py                 # journal, snapshot, and action record types
├── journal.py                # SQLite MissionJournal
├── snapshots.py              # WorldSnapshotBuilder and snapshot schemas
├── projections.py            # bounded model context
├── tool_recording.py         # RecordedToolDispatcher
├── robot_actions.py          # lifecycle, correlation, verification
├── harness.py                # closed-loop coordinator
├── harness_module.py         # DIMOS Module/blueprint integration
└── testing.py                # scripted model and fake event helpers
```

Tests should live beside the implementation using repository conventions. Existing files likely to receive narrow changes are:

- `dimos/agents/annotation.py` for optional skill metadata;
- `dimos/agents/mcp/mcp_client.py` for a clean integration seam, not a second implementation of the loop;
- the navigation skill and its controller event source;
- agentic blueprints that opt into the prototype; and
- agent documentation describing the selected blueprint.

Do not create all files before their phase needs them.

## Test strategy

Use three layers:

- Unit tests for record schemas, state transitions, projection bounds, correlation, timeouts, and verifiers.
- Process integration tests with a scripted model, fake skill proxy, real DIMOS streams/RPC wiring, and a temporary SQLite database.
- Replay end-to-end tests using a small checked-in DIMOS replay and deterministic model outputs.

Tests must never require a paid model call or physical robot.

## Phase 0: baseline and integration seam

### Work

- Trace the current `McpClient` model/tool loop, tool-stream handling, and in-memory history.
- Trace one navigation command from `@skill` through `set_goal()` and controller feedback.
- Identify the typed streams needed for pose, navigation state, and terminal status.
- Record current MCP schemas and representative results for regression fixtures.
- Define how an agentic blueprint selects the new harness while the existing blueprint remains usable.

### Tests and evidence

- Add a regression fixture for an ordinary tool schema and navigation tool schema.
- Add or identify a test showing that navigation currently returns after command start.
- Document the event or adapter that can report navigation completion.

### Exit criteria

- The implementation seam is explicit.
- No production behavior has changed.
- The existing test suite is green.

## Phase 1: persistent tool-call journal

### Work

- Define versioned journal envelopes with sequence, timestamp, mission ID, model-turn ID, event type, and correlation IDs.
- Implement `MissionJournal` on SQLite with transactional append and ordered reads.
- Implement records for model turn start, model response, standard tool call, and standard tool result.
- Implement `RecordedToolDispatcher` around the existing dispatch interface.
- Record the call before invoking the tool and exactly one result after it returns or raises.
- Preserve the original tool name, arguments, call ID, schema, and result semantics.
- Add configurable argument/result redaction without changing execution inputs.

### Tests

- append and ordered read;
- close and reopen without record loss;
- concurrent append serialization;
- schema-version rejection with a useful error;
- recorded successful ordinary tool call;
- recorded tool exception as a standard error result;
- original call ID and result content preserved; and
- redaction affects storage only, not tool invocation.

### Exit criteria

- Every harness-dispatched standard tool call has a durable call and result record.
- Ordinary tools behave exactly as before apart from recording.

## Phase 2: harness-managed world snapshots

### Work

- Define `WorldSnapshot` and per-field freshness/provenance metadata.
- Implement a latest-value cache fed by selected typed robot streams.
- Support partial snapshots with explicit missing and stale values.
- Store large sensor values as artifact references rather than inline journal payloads.
- Persist a compact snapshot record before every model turn.
- Implement `ProjectionBuilder` with field selection, age labels, artifact references, and a strict size budget.
- Add capture triggers for post-action terminal events and selected salient robot events.
- Coalesce high-rate updates so they update the cache without creating model turns.

Start with the fields required by the navigation vertical slice:

- current pose and velocity;
- navigation/controller state;
- active robot action and recent outcome; and
- source timestamps and ages.

Add battery, safety, perception, and artifact summaries only when the active blueprint exposes reliable sources.

### Tests

- latest source value wins;
- capture freezes a consistent point-in-time view;
- missing, stale, and current values remain distinguishable;
- high-rate source events are coalesced;
- artifact payloads are referenced rather than inlined;
- projection remains within its configured budget; and
- every model invocation receives a newly captured snapshot record.

### Exit criteria

- The harness, not the model, establishes baseline robot context for every decision.
- The model can still invoke observation tools for additional information.

## Phase 3: robot action tracking

### Work

- Extend `@skill` with optional `RobotActionMetadata` that does not affect generated MCP schemas.
- Initially support metadata for:
  - immediate result meaning: acceptance or completion;
  - typed event/correlation source;
  - verifier name;
  - snapshot fields needed by the verifier; and
  - execution timeout.
- Define the lifecycle `REQUESTED`, `ACCEPTED`, `RUNNING`, `VERIFYING`, `SUCCEEDED`, `FAILED`, and `UNKNOWN`.
- Implement `RobotActionTracker` with tool-call ID and generated robot-action ID correlation.
- Persist start, progress, transition, and outcome records.
- Adapt controller feedback into typed events carrying enough identity to reject unrelated completions.
- Treat timeout or ambiguous evidence as `UNKNOWN`, never success.

The immediate standard tool result remains unique. If the result means acceptance, include the robot-action ID and accepted status. Record the later physical outcome separately and inject it into the next model turn.

### Tests

- an ordinary skill creates no robot action;
- action metadata does not alter the standard tool schema;
- an accepted command does not become `SUCCEEDED`;
- correlated progress advances the correct action;
- an unrelated terminal event cannot complete an action;
- controller rejection becomes `FAILED`;
- timeout becomes `UNKNOWN`;
- duplicate events are idempotent; and
- every transition retains the originating tool-call ID.

### Exit criteria

- The runtime can state separately whether a command was accepted and whether its physical outcome was verified.
- Existing `@skill` declarations without metadata are unchanged.

## Phase 4: closed-loop harness

### Work

- Implement `RobotAgentHarness` as a serialized event loop.
- At each decision boundary:
  1. capture and journal a snapshot;
  2. construct bounded context from the snapshot and recent records;
  3. invoke and record the model response;
  4. record and dispatch standard tool calls; and
  5. record each standard tool result.
- Continue the conventional model/tool loop for ordinary tools.
- When a managed physical command is accepted, close the current decision step and wait for its terminal event or timeout.
- On terminal feedback, capture a fresh snapshot, run the verifier, record the outcome, and wake the next model decision.
- Permit only one newly accepted managed physical action per decision step. Return structured standard tool errors for additional physical calls in the same model response.
- Serialize model turns and debounce salient events.
- Stop initiating physical work if the journal cannot persist the call or action record.

### Tests

- snapshot is recorded before model invocation;
- ordinary tools can complete multiple normal tool-loop rounds;
- accepted physical action pauses the next decision;
- terminal event causes fresh snapshot capture and verification;
- verified outcome appears in the next model context;
- bursty events create only one serialized wakeup;
- a second physical call in one step receives a standard structured error; and
- journal failure prevents new physical dispatch.

### Exit criteria

- The observe-decide-act-observe/verify-decide loop works without the model explicitly polling for completion.
- Standard tool semantics remain intact.

## Phase 5: navigation vertical slice

### Work

- Add robot-action metadata to the selected navigation skill without changing its public tool schema.
- Make its immediate result unambiguously report command rejection or acceptance and include the action ID through the dispatcher result envelope.
- Emit or adapt typed navigation progress and terminal events with correlation data.
- Implement a navigation verifier using:
  - the correlated terminal event;
  - fresh pose and navigation/controller state; and
  - configured position/orientation tolerance.
- Include goal, current pose, controller status, and reason in failed or unknown outcomes.
- Add an opt-in agentic blueprint wiring the harness, snapshot sources, navigation event source, and native skill proxy.

Do not change the repository's existing capability coordination behavior in this phase.

### Tests

- goal rejection records a failed action without claiming execution;
- accepted goal enters `RUNNING` when feedback arrives;
- goal-reached plus in-tolerance pose verifies `SUCCEEDED`;
- goal-reached plus out-of-tolerance pose becomes `FAILED` or `UNKNOWN` according to verifier policy;
- controller abort records `FAILED` with its reason;
- missing terminal feedback times out to `UNKNOWN`; and
- the same navigation skill remains callable through MCP.

### Exit criteria

- Navigation demonstrates the complete proposal using a normal model tool call.
- No other physical skill is migrated merely to increase coverage.

## Phase 6: deterministic and replay end-to-end tests

### Deterministic process test

Build a test blueprint containing:

- a scripted model;
- the real harness and SQLite journal;
- a fake navigation skill/controller;
- typed pose and navigation streams; and
- controlled progress, completion, failure, and timeout events.

The test should assert the exact causal order:

```text
snapshot
model response
tool call
tool accepted result
robot action progress
terminal event
fresh snapshot
verified outcome
next model turn
```

It should also reopen the journal and derive the same sequence without process memory.

### Repository replay test

Use the smallest suitable checked-in replay. If existing replay data lacks navigation terminal events, add a compact deterministic event fixture rather than depending on timing or a live controller.

Assert that:

- snapshot projections are deterministic for the replay;
- the scripted model emits the expected standard tool call;
- all calls and results are recorded;
- the physical outcome is based on replayed event and snapshot evidence; and
- the next decision receives the verified result.

### Exit criteria

- Both tests pass repeatedly without network access, a paid model, or hardware.
- Failures show the journal sequence and correlation IDs.

## Phase 7: regression and handoff

### Work

- Run focused harness, MCP, navigation, blueprint, and replay tests.
- Run the repository's required fast test suite and static checks for changed packages.
- Verify generated blueprint registry files if a built-in blueprint is added or renamed.
- Document how to start the opt-in harness blueprint and inspect its mission journal.
- Document snapshot freshness, accepted-versus-succeeded semantics, and verifier output.
- Add a short guide for annotating the next physical skill only after navigation is stable.

### Exit criteria

- Existing non-harness agentic blueprints remain functional.
- Existing external MCP clients need no protocol changes.
- The new closed-loop behavior is opt-in and documented.
- Deferred work is clearly separated from prototype requirements.

## Required fault matrix

Before completion, automated tests must cover:

| Fault | Required result |
|---|---|
| Tool raises before command acceptance | one recorded standard error result; action `FAILED` if created |
| Controller rejects command | accepted is false; action `FAILED` |
| Progress arrives for another action | event retained for diagnosis; target action unchanged |
| Terminal event never arrives | action `UNKNOWN` after timeout |
| Terminal success conflicts with fresh pose | verifier returns `FAILED` or `UNKNOWN` with evidence |
| Snapshot field is missing or stale | explicit in projection and verifier input |
| Journal append fails before physical dispatch | physical tool is not invoked |
| Duplicate progress or terminal event | no duplicate transition or second model turn |
| Model requests two physical actions in one step | first may proceed; later call gets a standard structured error |

## Review checklist

- Standard tool calls remain the canonical invocation.
- Every dispatched call has exactly one standard result.
- Recording covers ordinary as well as physical tools.
- `@skill` metadata is optional and does not alter MCP schemas.
- Command acceptance is never labeled physical success.
- Robot action events carry usable correlation IDs.
- Verification uses a fresh snapshot.
- Snapshot capture is harness-managed at every model boundary.
- Raw high-rate sensor payloads are not injected into model context.
- Missing and stale state are explicit.
- Model turns are serialized and event wakeups are bounded.
- Navigation proves the vertical slice before broader migration.
- Resource coordination, cancellation, and restart recovery remain outside this plan.

## Recommended delivery order

Land the work as small, reviewable changes:

1. journal records and recorded standard tool dispatch;
2. snapshot cache, capture, persistence, and projection;
3. optional action metadata and tracker;
4. serialized harness loop;
5. navigation vertical slice;
6. deterministic process integration test;
7. repository replay test and documentation.

Each change should preserve existing agent behavior until an agentic blueprint explicitly selects the new harness.
