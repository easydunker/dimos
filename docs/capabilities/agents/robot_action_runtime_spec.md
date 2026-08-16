---
title: "Closed-Loop Robot Agent Harness Specification"
---

# Closed-Loop Robot Agent Harness Specification

Status: proposed prototype specification.

Audience: agent-runtime, robotics, navigation, observability, and evaluation contributors.

Companion plan: [Robot Action Runtime Prototype Implementation Plan](robot_action_runtime_implementation_plan.md).

## Decision

DIMOS should add a persistent, closed-loop robot agent harness that:

1. records standard model tool calls and their results;
2. tracks physical tool calls beyond command acceptance to an observed robot outcome; and
3. builds and injects world snapshots at harness-controlled decision boundaries.

The standard agent tool call remains the canonical invocation. The proposal does not replace tools with a new action API. It adds recording and, for skills that initiate physical work, a small amount of metadata and lifecycle tracking around the existing tool call.

The intended loop is:

```text
capture world -> decide -> call standard tool -> observe execution -> verify outcome
      ^                                                               |
      +------------------------ next decision -------------------------+
```

## Current gap

The current `McpClient` is a conventional LangGraph tool-using agent. It keeps conversational history in memory, invokes discovered MCP tools, and turns tool-stream updates into messages for the model. This is appropriate for general tool use, but it does not provide a durable robot execution record or a harness-owned perception loop.

In particular:

- model messages, tool calls, and tool results are not persisted as a mission record;
- robot state is observed when the model chooses an observation tool, rather than automatically at every decision boundary;
- some physical skills return after a controller accepts a command, before the robot reaches the requested outcome; and
- later controller state is not consistently correlated with the tool call that initiated the motion.

For example, navigation currently calls `set_goal()` and can return “Started navigating”. That is useful command acknowledgement, but it is not evidence that navigation succeeded.

## Goals

- Preserve DIMOS `@skill`, MCP discovery, standard tool schemas, tool-call IDs, and tool results.
- Persist every tool call observed by the harness and its corresponding result.
- Distinguish controller command acceptance from physical outcome.
- Correlate physical action progress and outcomes with the originating tool call.
- Automatically capture fresh robot state before model decisions and after physical execution.
- Inject a bounded, useful projection of robot state into each model turn.
- Support deterministic tests and replay without requiring a live model or robot.
- Deliver one navigation vertical slice before generalizing to other robot actions.

## Non-goals

- Replacing standard agent tools with a new universal action abstraction.
- Redesigning the existing capability coordination mechanism.
- General resource arbitration or cross-client exclusivity.
- Defining approval, risk-classification, or authorization policy.
- Sending raw high-rate sensor streams to the model.
- Solving cancellation, restart recovery, or distributed failover in this prototype.
- Claiming physical success from an RPC return value alone.

## Runtime boundary

The harness sits around the existing model/tool loop and uses existing DIMOS modules, streams, RPCs, and skills.

```text
typed robot streams --------> WorldSnapshotBuilder ----+
                                                     |
                                                     v
user/model events -> RobotAgentHarness -> ProjectionBuilder -> model
                           |                              |
                           |                              v
                           |                  standard tool calls
                           |                              |
                           v                              v
                     MissionJournal <--- RecordedToolDispatcher
                           ^                              |
                           |                              v
typed action events -> RobotActionTracker <--- existing @skill / RPC
                           |
                           +---- fresh snapshot + verified outcome
```

### `RobotAgentHarness`

Owns decision boundaries and the closed-loop sequence. It requests snapshots, builds model context, records model output, dispatches standard tools, and waits for a managed robot action outcome before opening the next decision turn.

### `WorldSnapshotBuilder`

Subscribes to selected typed robot streams and maintains a latest-value cache. A capture freezes that cache into a timestamped `WorldSnapshot`. Missing or stale values remain explicit rather than being silently reused as current facts.

### `ProjectionBuilder`

Converts durable records and the latest snapshot into bounded model context. It controls size, freshness labels, and artifact references without changing the underlying records.

### `RecordedToolDispatcher`

Wraps the existing tool dispatch path. It records the call before execution and records exactly one standard tool result after execution. Ordinary tools otherwise keep their existing behavior.

### `RobotActionTracker`

Tracks only tool calls whose skill metadata declares that they start physical robot work. It correlates controller events, maintains the action lifecycle, requests verification, and produces a durable outcome.

### `MissionJournal`

Persists ordered model, snapshot, tool, and robot-action records in SQLite. The prototype requires reliable reopen and query behavior, not tamper-evident storage.

## Standard tool recording

All tool calls that pass through the harness are recorded, whether they are queries, observation tools, computation, speech, or physical robot commands. A call is not converted into another invocation type.

At minimum, a `ToolCallRecord` contains:

- journal sequence and timestamp;
- mission and model-turn IDs;
- standard tool-call ID;
- tool name and arguments, subject to redaction policy;
- whether the discovered skill has robot-action metadata; and
- dispatch status.

A `ToolResultRecord` contains:

- the same mission, turn, and tool-call IDs;
- completion timestamp and duration;
- returned content or structured error;
- result status; and
- an optional robot action ID when the result acknowledges physical work.

Recording happens at the dispatch boundary so it does not depend on the model provider retaining history. Calls made outside this harness, such as direct external MCP calls, are outside the prototype's recording guarantee unless they are routed through the same recorder later.

## World snapshots

### Snapshot contents

A snapshot is a compact state observation, not a copy of every sensor payload. Depending on the active blueprint, it may contain:

- robot pose, velocity, and motion state;
- navigation and controller state;
- battery and safety state when available;
- active robot actions and recent outcomes;
- detected-object or map summaries;
- references to camera, point-cloud, map, or other large artifacts; and
- source timestamp, capture timestamp, age, confidence, and provenance for each field.

The schema permits partial snapshots. Consumers must be able to distinguish unavailable, stale, and current values.

### Capture boundaries

The harness captures a snapshot:

1. immediately before every model turn;
2. after a managed physical action reports a terminal condition and before verification;
3. after a salient robot event that should wake the agent; and
4. optionally at a bounded diagnostic interval configured by the blueprint.

High-rate sensor updates refresh the builder's cache. They do not each create a model turn or durable snapshot.

### Model injection

Before a model turn, `ProjectionBuilder` injects a compact snapshot projection containing the state relevant to the mission, active action, and recent outcome. Large data stays in the artifact store and is referenced by ID.

The model may still call observation tools such as `observe()` when it needs information that is absent, stale, more detailed, or viewpoint-dependent. Automatic snapshots establish a reliable baseline; they do not remove agent-directed perception.

## Robot action handling

### Optional skill metadata

An existing `@skill` may declare optional `RobotActionMetadata`. Skills without this metadata remain ordinary standard tools.

The metadata is intentionally small:

- whether the tool starts physical execution;
- whether its immediate result means command acceptance or physical completion;
- the typed event source used to correlate progress and completion;
- an optional verifier name;
- optional snapshot fields required by that verifier; and
- an execution timeout after which the outcome becomes `UNKNOWN`.

This metadata must not alter the generated tool schema or require external MCP clients to understand a DIMOS-specific action protocol.

### Lifecycle

A managed robot action uses the following lifecycle:

```text
REQUESTED -> ACCEPTED -> RUNNING -> VERIFYING -> SUCCEEDED
                                              -> FAILED
                         \--------------------> UNKNOWN
```

- `REQUESTED`: the standard tool call was recorded and dispatch began.
- `ACCEPTED`: the underlying skill or controller accepted the command.
- `RUNNING`: correlated feedback indicates ongoing physical execution.
- `VERIFYING`: a terminal report caused the harness to capture fresh state and run the verifier.
- `SUCCEEDED`: terminal feedback and verification support the requested outcome.
- `FAILED`: the controller rejected or failed the command, or verification disproved the outcome.
- `UNKNOWN`: execution timed out or available evidence cannot establish success or failure.

Every transition is journaled with the originating tool-call ID and a harness-generated robot action ID.

### Acceptance is not completion

A successful RPC can mean only that a controller accepted or started work. In that case, the standard tool result records `accepted` and includes the robot action ID. It does not claim that the robot has succeeded.

The later physical outcome is recorded as a correlated robot-action event and included in the next model turn. It is not emitted as a second result for the same standard tool call.

After a managed action is accepted, the harness waits for a terminal or timeout outcome before requesting another model decision. The prototype supports at most one newly accepted managed robot action in a decision step. If a model response requests additional physical actions in the same step, the dispatcher returns structured standard tool errors asking the model to reconsider after the current outcome. This is sequencing inside one closed loop, not a general resource arbitration system.

### Verification

Controller feedback supplies execution status; a fresh snapshot supplies observed robot state. A verifier combines the two to determine the outcome.

For navigation, success should normally require both:

- a correlated navigation terminal event; and
- a fresh pose/controller snapshot consistent with arrival within configured tolerance.

When evidence conflicts, the verifier returns `FAILED` or `UNKNOWN` with reasons. It must not convert uncertainty into success.

## Closed-loop execution flow

For each decision step, the harness:

1. captures and persists a world snapshot;
2. builds model context from the mission, recent journal records, active action state, and snapshot projection;
3. invokes the model and records its response;
4. records and dispatches each standard tool call;
5. records exactly one standard result for each dispatched tool call;
6. completes the normal tool loop immediately for ordinary tools;
7. when a physical command is accepted, records its robot action and waits for correlated execution feedback;
8. on terminal feedback or timeout, captures a fresh snapshot and runs the verifier;
9. persists the resulting action outcome; and
10. starts the next model turn with that outcome and fresh world state in context.

Salient external events can also open a decision step. The harness must coalesce noisy updates and serialize model turns so a burst of sensor events does not create overlapping decisions.

## Durable records

The journal uses ordered, typed event records such as:

- `MODEL_TURN_STARTED`;
- `WORLD_SNAPSHOT_CAPTURED`;
- `MODEL_RESPONSE_RECORDED`;
- `TOOL_CALL_RECORDED`;
- `TOOL_RESULT_RECORDED`;
- `ROBOT_ACTION_STARTED`;
- `ROBOT_ACTION_PROGRESS`; and
- `ROBOT_ACTION_OUTCOME`.

Each record includes schema version, sequence number, timestamp, mission ID, and relevant correlation IDs. The payload is JSON-compatible. Large or binary values are stored separately and referenced by content ID.

SQLite is the prototype store. A transaction must make each record durable before the harness relies on it as prior state. Tests must cover reopen, ordering, partial/corrupt tail handling, and schema migration failure messages.

## Compatibility with existing DIMOS

- `@skill` remains the definition of an agent-callable robot capability.
- MCP remains the interoperable tool-discovery and invocation boundary.
- Existing tool names, descriptions, JSON schemas, call IDs, and result messages remain valid.
- The built-in harness may call native skill proxies to receive typed events while the same skills remain available through MCP.
- Existing capability coordination continues unchanged and is not extended by this proposal.
- `Module`, `In`/`Out`, typed transports, blueprints, and RPC injection remain the composition mechanism.

## Failure handling

- Tool dispatch exceptions produce a recorded standard tool error.
- A rejected physical command produces both a standard tool result and a failed robot-action outcome.
- Missing completion feedback produces `UNKNOWN` after the configured timeout.
- Missing snapshot fields are visible to the verifier and model projection.
- A snapshot capture failure prevents a new physical decision unless the blueprint explicitly allows a degraded mode.
- Journal write failure stops the harness from initiating new physical work because the execution would otherwise be unrecorded.
- Conflicting or uncorrelated controller events are recorded for diagnosis but cannot complete an action.

Detailed cancellation semantics and reconstruction of in-flight work after restart are separate workstreams.

## Observability and evaluation

The journal should make it possible to answer:

- What did the model know before it acted?
- Which standard tool call initiated the robot action?
- Did the command merely start, or did the robot reach the outcome?
- Which controller event and snapshot fields supported verification?
- How long did decision, dispatch, execution, and verification take?

The same records should drive deterministic evaluation using a scripted model, fake skills, and replayed robot streams.

## Prototype acceptance criteria

The prototype is complete when:

- every harness-dispatched tool call and result is durably recorded;
- every model turn receives a harness-captured snapshot projection;
- ordinary tools retain standard behavior and require no robot-action metadata;
- a navigation tool result distinguishes accepted from completed;
- navigation progress and terminal events correlate to the originating tool call;
- a fresh post-action snapshot participates in outcome verification;
- the verified outcome triggers the next model decision;
- deterministic and replay tests exercise the full closed loop; and
- existing MCP clients and non-harness agent paths continue to work.

## Deferred workstreams

The design can support later work on cancellation, restart recovery, resource coordination, policy approval, multi-agent scheduling, richer safety monitors, and stronger audit storage. Those topics should be specified independently and should not expand this prototype's runtime contract.
