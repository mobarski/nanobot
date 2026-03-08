# Nanobot Base Design (GPT-5.4)

## Stored Prompt

```text
read CORE-FINAL-{model-name-and-version}.md;
try to DESIGN the BASE for all the functionalities from following nanobot subdirectories: agent, bus, config, cron, heartbeat, skills, templates, utls;
store the results in BASE-{model-name-and-version}.md;
also - store this promp in the resulting file
```

## Source Anchor

This BASE design is derived from `CORE-FINAL-gpt-5.4.md` and then expanded to cover the currently implemented functionality in:

- `nanobot/agent`
- `nanobot/bus`
- `nanobot/config`
- `nanobot/cron`
- `nanobot/heartbeat`
- `nanobot/skills`
- `nanobot/templates`
- `nanobot/utils` (interpreting `utls` as `utils`, because `utls` does not exist)

## Design Thesis

The CORE defines the lean kernel:

- one canonical request shape
- one queue/runner
- one turn orchestration path
- one store contract
- one tools gateway

The BASE should be the smallest layer above that kernel that can host all existing nanobot features without creating a second architecture.

So the BASE is not another engine. It is the stable set of contracts, policies, and adapters that let the kernel support:

- chat channels through the bus
- persistent sessions and memory
- config-driven startup and provider selection
- cron jobs
- heartbeat autonomy
- progressive skill loading
- workspace bootstrap templates
- shared utility behavior
- subagents and MCP-backed tools

## BASE Boundary

### Inside the BASE

- canonical transport and turn contracts
- runtime wiring and lifecycle
- per-session scheduling/cancellation policy
- prompt/context assembly policy
- session persistence and memory policy
- tool registration/execution policy
- scheduler adapters for cron and heartbeat
- workspace asset rules for templates and skills
- config loading and runtime path resolution

### Outside the BASE

- concrete chat channel adapters
- provider SDK internals
- filesystem layout details beyond defined contracts
- specific tool implementations beyond the tool gateway contract
- packaging/CLI/bootstrap scripts

The rule is:

If a feature can be expressed as a normalized event, a store concern, a tools concern, or a scheduler concern, it belongs in the BASE and not in the kernel.

## The BASE Modules

### 1. `base/contracts.py`

Defines the canonical data shapes used across the whole system:

- `TurnRequest`
- `TurnResult`
- `ProgressEvent`
- `ToolContext`
- `ScheduledEvent`
- `RuntimeHandles`

This is the shared language between `agent`, `bus`, `cron`, `heartbeat`, and future adapters.

### 2. `base/router.py`

Normalizes all ingress into one `TurnRequest`.

Input sources:

- `InboundMessage`
- cron triggers
- heartbeat triggers
- subagent completion messages
- direct CLI/API calls

Responsibilities:

- compute canonical `session_key`
- label the `trigger`
- preserve channel/chat routing in metadata
- keep transport metadata opaque to the kernel

### 3. `base/runtime.py`

Owns system startup and shutdown:

- load config
- derive runtime paths
- initialize bus
- initialize provider
- initialize state store
- initialize tool gateway
- start cron
- start heartbeat
- run the turn runner
- manage lazy MCP lifecycle

This is the place where current cross-directory wiring becomes explicit instead of being scattered.

### 4. `base/runner.py`

Owns queueing, concurrency, and cancellation.

Required behavior:

- accept normalized `TurnRequest`
- maintain single-flight per `session_key`
- implement `/stop`
- support background and foreground turns
- emit progress events and final outbound results

Important BASE decision:

The runner should follow the `CORE-FINAL-gpt-5.4.md` invariant of per-session single-flight execution. The current `AgentLoop` uses a global processing lock, which is safe but more serialized than necessary. BASE should preserve safety while allowing independent sessions to run independently.

### 5. `base/engine.py`

This is the only turn engine.

Responsibilities:

- command short-circuit
- load context
- build prompt messages
- call provider
- execute tools
- iterate until final response or max iterations
- produce trace for persistence
- return canonical `TurnResult`

Cron, heartbeat, direct CLI, and subagent follow-ups must all re-enter here.

### 6. `base/context.py`

Owns prompt assembly only.

It preserves the existing nanobot prompt structure:

- identity/runtime section
- workspace bootstrap files: `AGENTS.md`, `SOUL.md`, `USER.md`, `TOOLS.md`
- long-term memory from `memory/MEMORY.md`
- always-on skills
- summarized skills catalog for progressive loading
- runtime metadata envelope injected as non-instructional context
- multimodal message assembly when images are attached

This keeps prompt policy separate from persistence and tool execution.

### 7. `base/state.py`

Unifies session persistence behavior currently spread across agent/session/memory code.

Responsibilities:

- load session history
- sanitize persisted trace
- strip runtime metadata before persistence
- truncate large tool results before saving
- save session state
- support `/new` archival/clear behavior
- coordinate memory consolidation thresholds

This is the correct place for turn commit policy.

### 8. `base/memory.py`

Defines the persistent memory contract:

- `MEMORY.md` is editable long-term memory
- `HISTORY.md` is append-only grep-friendly history
- consolidation is LLM-mediated through a virtual tool call

The current two-layer memory design is already a good BASE primitive and should remain intact.

### 9. `base/tools.py`

Defines one tools gateway above all tool implementations.

Responsibilities:

- register tools
- expose schemas
- cast parameters from schema
- validate parameters
- execute tools
- attach explicit routing/session context
- lazily attach MCP tools
- preserve the current retry-friendly error contract

This should also absorb the duplicated tool setup logic that currently exists in both the main agent and subagent paths.

### 10. `base/schedulers.py`

A unified scheduler facade over:

- `CronService`
- `HeartbeatService`

Responsibilities:

- lifecycle control
- normalized scheduled events
- re-entry into the same turn runner
- optional delivery routing for scheduled outputs

The schedulers are producers of `TurnRequest`, not alternate agent runtimes.

### 11. `base/assets.py`

Defines workspace-owned content rules for:

- built-in templates
- workspace template copies
- built-in skills
- workspace skills override behavior

This is the BASE contract for the markdown-driven part of nanobot.

## Canonical Contracts

### `TurnRequest`

```python
TurnRequest = {
    "session_key": str,              # canonical key, usually "channel:chat_id"
    "trigger": "manual|system|cron|heartbeat|subagent|cli|api",
    "channel": str | None,
    "chat_id": str | None,
    "sender_id": str | None,
    "content": str,
    "media": list[str],
    "metadata": dict,
}
```

Notes:

- `channel` and `chat_id` are promoted fields for routing clarity.
- all source-specific details stay in `metadata`.
- `session_key` remains the single identity used for history and cancellation.

### `TurnResult`

```python
TurnResult = {
    "session_key": str,
    "content": str,
    "emit": bool,
    "outbound": {
        "channel": str,
        "chat_id": str,
        "content": str,
        "media": list[str],
        "metadata": dict,
    } | None,
    "trace": list[dict],
}
```

### `ProgressEvent`

```python
ProgressEvent = {
    "session_key": str,
    "channel": str | None,
    "chat_id": str | None,
    "content": str,
    "tool_hint": bool,
    "metadata": dict,
}
```

### `ToolContext`

```python
ToolContext = {
    "session_key": str,
    "trigger": str,
    "channel": str | None,
    "chat_id": str | None,
    "message_id": str | None,
    "metadata": dict,
}
```

BASE should prefer explicit per-call tool context over mutable hidden tool state. Where mutable state remains for compatibility, it should be treated as an adapter detail.

### `ScheduledEvent`

```python
ScheduledEvent = {
    "id": str,
    "kind": "cron|heartbeat",
    "session_key": str,
    "content": str,
    "deliver": bool,
    "channel": str | None,
    "chat_id": str | None,
    "metadata": dict,
}
```

## How Current Directories Map Into BASE

### `agent` -> cognition and execution

Keep from current code:

- `AgentLoop` as the conceptual turn engine
- `ContextBuilder` for system prompt and message assembly
- `MemoryStore` for two-layer memory
- `SkillsLoader` for progressive markdown skills
- `ToolRegistry` and `Tool` for capability dispatch
- `SubagentManager` for background delegated tasks

BASE adjustments:

- split runtime/runner concerns out of the current monolithic loop
- keep only one execution engine contract
- reduce duplication between main-agent and subagent tool wiring
- make turn persistence and memory policy explicit in `base/state.py`

### `bus` -> transport boundary

Keep from current code:

- `InboundMessage`
- `OutboundMessage`
- `MessageBus` with independent inbound/outbound queues

BASE role:

- the bus remains intentionally small
- router logic converts bus events into canonical `TurnRequest`
- outbound conversion becomes one explicit adapter step

### `config` -> typed runtime policy

Keep from current code:

- Pydantic schema as the single source of truth
- nested channel/provider/tool configuration
- provider auto-matching logic
- migration hook in loader
- path helpers derived from active config

BASE role:

- config remains authoritative for runtime policy
- BASE consumes config but does not duplicate configuration rules
- path helpers become the standard source for cron/log/media/workspace directories

### `cron` -> persistent deterministic scheduling

Keep from current code:

- `CronSchedule`, `CronPayload`, `CronJob`, `CronStore`
- JSON persistence with external modification reload
- one-shot, interval, and cron-expression schedules
- next-run computation and single-timer arming
- callback-based job execution

BASE role:

- cron emits normalized scheduled events
- cron never bypasses the turn runner
- delivery metadata remains attached to the scheduled event

### `heartbeat` -> periodic autonomous wake-up

Keep from current code:

- `HEARTBEAT.md` as the source of truth
- two-phase decision model:
  - phase 1: virtual tool decides `skip` vs `run`
  - phase 2: full agent execution only when needed
- `trigger_now()` manual entry point

BASE role:

- heartbeat is an event producer with policy, not a second agent loop
- heartbeat must route execution through the same turn path as manual messages

### `skills` -> progressive capability layer

Keep from current code:

- built-in skills and workspace skills
- workspace override precedence
- metadata/frontmatter parsing
- requirement gating by CLI tools and env vars
- always-on skills injection
- summary-first, full-read-later loading model

BASE role:

- skills are treated as prompt-time extensions
- the BASE asset contract defines discovery, precedence, and availability semantics

### `templates` -> workspace bootstrap contract

Keep from current code:

- `AGENTS.md`
- `SOUL.md`
- `USER.md`
- `TOOLS.md`
- `HEARTBEAT.md`
- `memory/MEMORY.md`
- `memory/HISTORY.md`

BASE rules:

- packaged templates seed the workspace once
- missing files may be created automatically
- existing user files are never overwritten by sync
- runtime reads the workspace copies, not the packaged originals

### `utils` -> cross-cutting primitives

Keep from current code:

- `ensure_dir`
- `detect_image_mime`
- `timestamp`
- `safe_filename`
- `split_message`
- `sync_workspace_templates`

BASE role:

- helpers remain low-level, dependency-light primitives
- business policy should not drift into `utils`
- `sync_workspace_templates()` is important enough to be recognized by the BASE asset contract

## End-to-End Flows

### 1. Normal chat turn

`channel adapter -> InboundMessage -> MessageBus -> router -> TurnRequest -> runner -> engine -> tools/provider -> state.commit -> TurnResult -> outbound adapter -> MessageBus`

### 2. Slash command turn

`InboundMessage(/help|/new|/stop) -> router -> runner/engine short-circuit -> state/subagent control -> outbound response`

Required semantics:

- `/help` returns immediately
- `/new` archives/consolidates before clearing the session
- `/stop` cancels active session tasks and session subagents

### 3. Cron turn

`CronService timer -> ScheduledEvent(kind="cron") -> router -> TurnRequest(trigger="cron") -> runner -> engine -> optional outbound delivery`

### 4. Heartbeat turn

`HeartbeatService tick -> decision tool -> if run: ScheduledEvent(kind="heartbeat") -> router -> TurnRequest(trigger="heartbeat") -> runner -> engine -> optional notify`

### 5. Subagent completion turn

`spawn tool -> SubagentManager -> delegated execution -> synthetic system ingress -> router -> standard turn path`

## BASE Invariants

1. One ingress contract: every trigger becomes a `TurnRequest`.
2. One execution path: all non-trivial work runs through one engine.
3. One session identity: `session_key` is the canonical history and cancellation key.
4. One commit path: session persistence and memory policy happen through `base/state.py`.
5. One tools gateway: local tools, routed tools, and MCP tools share one contract.
6. Cron and heartbeat are schedulers, not alternate engines.
7. Subagents are delegated workers, not a separate architecture.
8. Runtime metadata is informative-only and must be stripped before persistence as user content.
9. Workspace templates are user-owned after initial sync.
10. Workspace skills override built-ins; built-ins remain the fallback.
11. BASE should preserve current markdown-driven behavior instead of replacing it with code-only policy.
12. Hidden mutable tool context should be minimized; explicit tool context is preferred.

## Recommended BASE File Layout

```text
nanobot/
  base/
    contracts.py   # TurnRequest, TurnResult, ProgressEvent, ScheduledEvent
    router.py      # ingress normalization and outbound adaptation
    runtime.py     # startup/shutdown wiring
    runner.py      # queue, per-session tasks, cancellation
    engine.py      # the single turn loop
    context.py     # prompt assembly policy
    state.py       # session load/save/sanitize/clear
    memory.py      # MEMORY.md + HISTORY.md consolidation contract
    tools.py       # registry/gateway/context/MCP wiring
    schedulers.py  # cron + heartbeat facade
    assets.py      # templates + skills discovery/sync rules
```

This file layout is not a demand for immediate refactoring. It is the clean conceptual shape that the current implementation should converge toward.

## Migration Notes From Current Code

1. Keep current directories and treat BASE as an architectural overlay first, not a rewrite.
2. Move the current `AgentLoop` responsibilities into explicit BASE roles gradually: runner, engine, state, context.
3. Preserve `ContextBuilder`, `MemoryStore`, `SkillsLoader`, and `CronService` behavior because they already express good separations.
4. Reduce duplication between `AgentLoop` and `SubagentManager` by sharing one tool/engine wiring strategy.
5. Replace global serialization with per-session single-flight where safe, matching the `CORE-FINAL-gpt-5.4.md` design.
6. Keep all background features re-entering through the same normalized request path.

## Final BASE Statement

The right BASE for nanobot is a thin but explicit layer that sits above the lean kernel and below the full product surface.

It should define:

- one canonical turn contract
- one runtime wiring model
- one runner with per-session control
- one engine for all triggers
- one state/memory policy
- one tools gateway
- one scheduler facade
- one workspace asset contract

With that shape, the existing functionality in `agent`, `bus`, `config`, `cron`, `heartbeat`, `skills`, `templates`, and `utils` can remain intact while becoming part of one coherent architecture instead of a collection of adjacent subsystems.
