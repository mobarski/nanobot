# Nanobot Base Design - gpt-5.3-codex

## Stored Prompt

`read CORE-FINAL-{model-name-and-version}.md;
try to DESIGN the BASE for all the functionalities from following nanobot subdirectories: agent, bus, config, cron, heartbeat, skills, templates, utls;
store the results in BASE-{model-name-and-version}.md;
also - store this promp in the resulting file`

## Scope and Intent

Design a practical BASE layer that keeps one canonical turn pipeline while preserving all functionality currently implemented in:

- `nanobot/agent`
- `nanobot/bus`
- `nanobot/config`
- `nanobot/cron`
- `nanobot/heartbeat`
- `nanobot/skills`
- `nanobot/templates`
- `nanobot/utils` (interpreting `utls` as `utils`)

The BASE should be small, explicit, and adapter-first.

## BASE Modules (Minimal but Complete)

1. `base/contracts.py`  
   Canonical event/turn/tool/schedule dataclasses and enums.

2. `base/router.py`  
   Ingress normalization from channel/system/cron/heartbeat/subagent into one `TurnRequest`.

3. `base/runtime.py`  
   Global orchestrator: starts/stops agent loop, cron service, heartbeat service, MCP lifecycle.

4. `base/runner.py`  
   Async queue + per-session task control + `/stop` semantics.

5. `base/engine.py`  
   Single message->LLM->tools iterative loop with bounded iterations and progress events.

6. `base/context.py`  
   System prompt builder: identity + templates + memory + skills summary + runtime metadata envelope.

7. `base/state.py`  
   Session persistence and turn commit policy (including tool-output truncation and runtime-context stripping).

8. `base/memory.py`  
   Two-layer memory (`MEMORY.md`, `HISTORY.md`) and consolidation policy/tool-call interface.

9. `base/tools.py`  
   Tool registration/execution, schema casting/validation, contextual tool routing (message/spawn/cron), MCP wrapping.

10. `base/schedulers.py`  
    Unified scheduler facade exposing cron timers and heartbeat ticks as normal BASE events.

## Canonical Contracts

`TurnRequest`

```python
{
  "session_key": str,                 # "channel:chat_id"
  "origin": "channel|system|cron|heartbeat|subagent|cli",
  "channel": str,
  "chat_id": str,
  "sender_id": str,
  "content": str,
  "media": list[str],
  "metadata": dict
}
```

`TurnResult`

```python
{
  "content": str,
  "emit": bool,
  "outbound": {
    "channel": str,
    "chat_id": str,
    "content": str,
    "media": list[str],
    "metadata": dict
  } | None
}
```

`ProgressEvent`

```python
{
  "session_key": str,
  "text": str,
  "is_tool_hint": bool
}
```

`ScheduledTrigger`

```python
{
  "id": str,
  "kind": "cron|heartbeat",
  "message": str,
  "deliver": bool,
  "channel": str | None,
  "to": str | None
}
```

## How Existing Directories Map into BASE

### 1) `agent` -> cognition, iteration, tools, subagents

- Keep one `AgentLoop`-style engine with max tool iterations, model settings, and reasoning mode.
- Preserve `ContextBuilder` behavior:
  - identity/runtime section,
  - bootstrap templates: `AGENTS.md`, `SOUL.md`, `USER.md`, `TOOLS.md`,
  - long-term memory and always-on skills,
  - runtime metadata block prefixed as non-instructional context.
- Preserve `MemoryStore` consolidation path:
  - LLM function call `save_memory(history_entry, memory_update)`,
  - update `memory/MEMORY.md`,
  - append to `memory/HISTORY.md`,
  - update `last_consolidated`.
- Preserve subagent manager semantics:
  - spawn background jobs with limited tools,
  - report completion via injected system inbound message,
  - cancel by session via `/stop`.
- Preserve tool framework:
  - schema-driven type casting and validation,
  - registry-based dynamic execution,
  - structured hint on tool errors to encourage retry strategy.

### 2) `bus` -> decoupled transport layer

- Keep `InboundMessage` and `OutboundMessage` as transport envelopes.
- Keep async inbound/outbound queues (`MessageBus`) as BASE IO boundary.
- Enforce normalized session key: `session_key_override` else `"{channel}:{chat_id}"`.

### 3) `config` -> typed configuration and runtime paths

- Keep Pydantic schema as single source of truth for:
  - agents defaults (model, iteration, memory window, reasoning effort),
  - channels policies and credentials,
  - providers and provider matching logic,
  - gateway heartbeat controls,
  - tools (`exec`, web search, MCP servers, workspace restrictions).
- Keep provider resolution algorithm:
  - explicit forced provider overrides auto,
  - model-prefix exact match wins,
  - keyword match second,
  - non-OAuth fallback with available key.
- Keep config migration hook (legacy `tools.exec.restrictToWorkspace`).
- Keep runtime path helpers:
  - instance data dir from current config path,
  - runtime subdirs (`media`, `cron`, `logs`),
  - workspace and bridge/history/session compatibility paths.

### 4) `cron` -> persisted deterministic scheduling

- Keep JSON-backed cron store with external mtime reload support.
- Keep schedule types:
  - `at` (one-shot timestamp),
  - `every` (interval),
  - `cron` (expr with optional IANA timezone).
- Keep lifecycle:
  - compute next wake,
  - arm single timer task,
  - execute due jobs sequentially,
  - save state and re-arm.
- Keep job state fields (`next_run`, `last_run`, status, error).
- Keep one-shot behavior: disable or delete-after-run.

### 5) `heartbeat` -> periodic autonomous check-and-run

- Keep two-phase heartbeat:
  1. decision phase via virtual `heartbeat` function tool (`skip|run`, optional tasks),
  2. execution phase via callback only when decision is `run`.
- Keep source of truth file: workspace `HEARTBEAT.md`.
- Keep optional notify callback for delivery.
- Keep manual `trigger_now` entrypoint.

### 6) `skills` -> progressive capability loading

- Keep dual-source skill discovery:
  - workspace skills override built-ins,
  - built-ins used as fallback.
- Keep frontmatter-based metadata extraction.
- Keep nanobot/openclaw metadata parsing for compatibility.
- Keep requirement gating (`bins`, env vars) and unavailability reporting.
- Keep progressive loading:
  - always-in-context summary list,
  - full `SKILL.md` loaded only on demand,
  - always skills auto-injected.
- Built-in skill domains to preserve:
  - scheduling (`cron`),
  - GitHub workflows,
  - weather lookup,
  - summarization/transcription,
  - tmux automation (with scripts),
  - external skill registry (`clawhub`),
  - skill authoring (`skill-creator`),
  - memory operations (`memory`).

### 7) `templates` -> bootstrap identity and operating rules

- Preserve template seed files:
  - `AGENTS.md` (behavior and reminder/heartbeat operational rules),
  - `SOUL.md` (persona/values),
  - `USER.md` (user profile/preferences),
  - `TOOLS.md` (non-obvious tool constraints),
  - `memory/MEMORY.md`,
  - `HEARTBEAT.md`.
- Preserve startup sync behavior:
  - create missing template files only,
  - do not overwrite user-edited files.

### 8) `utils` -> cross-cutting helper primitives

- Keep file-system primitive `ensure_dir`.
- Keep `safe_filename`, `split_message`, timestamp helper.
- Keep media MIME sniffing from magic bytes.
- Keep workspace template synchronization utility.

## End-to-End BASE Flows

### A) Normal chat turn

`channel adapter -> bus.inbound -> router.normalize -> runner.submit -> engine.run -> tool loop -> state.commit -> bus.outbound`

### B) Slash command turn (`/help`, `/new`, `/stop`)

- `/help`: immediate response from runner/engine short-circuit.
- `/new`: archive/consolidate session snapshot first, then clear session.
- `/stop`: cancel active session tasks + session subagents.

### C) Cron turn

`cron timer due -> scheduler emits ScheduledTrigger -> router creates TurnRequest(origin=cron) -> engine.run -> optional outbound delivery`

### D) Heartbeat turn

`heartbeat tick -> heartbeat decision(tool) -> if run: scheduler emits TurnRequest(origin=heartbeat) -> engine.run -> optional notify`

### E) Subagent completion turn

`subagent completes -> synthetic inbound system message -> router parses origin chat -> standard turn pipeline`

## BASE Invariants (Non-Negotiable)

1. One ingress contract: every trigger becomes `TurnRequest`.
2. One processing path: all non-command turns execute via one engine loop.
3. One persistence policy: turn saving + memory policy always go through `state`/`memory`.
4. One session identity rule: canonical `channel:chat_id` keying.
5. One cancellation authority: runner controls in-flight task and subagent cancellation.
6. Tool execution is schema-cast, schema-validated, and explicit-context only.
7. Runtime metadata is informative-only and stripped before persistence as user content.
8. Cron and heartbeat are schedulers that emit events, not alternate agent engines.
9. Skills are discoverable metadata-first and loaded progressively.
10. Templates are seeded once and user-owned afterward.

## Implementation Skeleton (BASE-first)

```python
class BaseRuntime:
    async def start(self) -> None: ...
    async def stop(self) -> None: ...

class TurnRouter:
    def normalize(self, inbound) -> dict: ...

class TurnRunner:
    async def submit(self, req: dict) -> None: ...
    async def handle_stop(self, req: dict) -> dict: ...

class TurnEngine:
    async def run(self, req: dict, on_progress=None) -> dict: ...

class StateStore:
    def load_session(self, session_key: str): ...
    def commit_turn(self, session_key: str, messages: list[dict]) -> None: ...

class MemoryConsolidator:
    async def maybe_consolidate(self, session) -> bool: ...

class SchedulerFacade:
    async def start(self) -> None: ...
    def stop(self) -> None: ...
```

## Migration Notes from Current Code

- Keep existing modules as adapters behind BASE interfaces first; avoid full rewrites.
- Move cross-cutting policies (context strip/truncate/consolidation thresholds) into BASE policy objects.
- Keep MCP lazy-connect behavior and retry-on-next-message semantics.
- Keep channel-specific delivery constraints out of engine core; enforce at channel adapters.

## Why This BASE Works

- It is lean: one turn engine, one runner, one event normalization point.
- It is complete: retains cron, heartbeat, skills, subagents, MCP, memory, and templates.
- It is maintainable: modules align with current subdirectory boundaries.
- It is extensible: new channels/tools/schedulers plug into contracts, not control-flow forks.
