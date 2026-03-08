# Nanobot BASE — claude-4.6-sonnet-medium-thinking

## Stored Prompt

```
read CORE-FINAL-{model-name-and-version}.md;
try to DESIGN the BASE for all the functionalities from following nanobot subdirectories: agent, bus, config, cron, heartbeat, skills, templates, utls;
store the results in BASE-{model-name-and-version}.md;
also - store this promp in the resulting file
```

## Foundation: The CORE-FINAL Kernel

This BASE document takes the irreducible kernel from `CORE-FINAL-claude-4.6-sonnet-medium-thinking.md`
as its anchor point — one async function, five dependency slots — and maps every feature found
in the actual codebase onto that structure. Nothing in this design requires modifying `run_turn`.

```
run_turn(req, deps)
  deps.store    ←──── memory + session + commands
  deps.llm      ←──── provider-normalized chat
  deps.tools    ←──── tool registry + policy
  deps.build    ←──── context assembly
  deps.on_progress ←─ streaming / observability
```

---

## Module → Slot Mapping

| Nanobot module | CORE-FINAL slot | Role |
|---|---|---|
| `bus/` | **caller of `run_turn`** | normalizes inbound events into `req`; routes `emit` results outward |
| `config/` | **all slots (wiring)** | supplies all parameters at bootstrap time |
| `agent/context.py` | **`deps.build`** | assembles system prompt + history into `list[dict]` |
| `agent/memory.py` | **`deps.store`** | backs `store.load` (memory) and `store.commit` (consolidation) |
| `agent/loop.py` | **runner + dispatcher** | `TurnRunner` equivalent; owns the processing lock |
| `agent/skills.py` | **`deps.build`** | injected via `ContextBuilder`; always-on skills inline, others as XML index |
| `agent/subagent.py` | **`deps.tools.exec`** | `spawn` tool delegates here; result re-enters bus as `channel="system"` req |
| `agent/tools/` | **`deps.tools`** | `ToolRegistry` implements `schemas(req)` and `exec(tc, req)` |
| `cron/` | **caller of `run_turn`** | timer fires → builds `req` with `meta.trigger="cron"` → bus / `process_direct` |
| `heartbeat/` | **caller of `run_turn`** | interval fires → two-phase LLM decide → builds `req` → `process_direct` |
| `skills/` | **`deps.build`** (data) | markdown files loaded by `SkillsLoader`, injected by `ContextBuilder` |
| `templates/` | **`deps.store.load`** (data) | workspace bootstrap files, loaded once per workspace |
| `utils/` | **no slot** | pure helpers used across all adapters |

---

## BASE Architecture

### 1. Request Shape (canonical ingress)

Every trigger — chat message, cron job, heartbeat tick, subagent result — becomes one `req` dict:

```python
req = {
    "sk":      str,    # session key: "{channel}:{chat_id}" or override
    "content": str,    # user/system message text
    "media":   list,   # base64-encoded file paths
    "meta": {
        "channel":   str,          # "telegram" | "discord" | "cli" | "system" | ...
        "chat_id":   str,
        "sender_id": str,
        "trigger":   str,          # "message" | "cron" | "heartbeat" | "subagent"
        "timestamp": str,          # ISO datetime (for runtime context block)
        "permissions": dict,       # policy enforcement data
        # ...opaque to kernel
    }
}
```

The kernel reads only `req["sk"]` and `req["content"]`. Everything else is consumed by adapters.

---

### 2. `deps.store` — Backed by `memory.py` + session JSONL + `templates/`

```python
class Store:
    """
    Wraps MemoryStore (MEMORY.md + HISTORY.md) + session JSONL history
    + template-loaded workspace bootstrap files.
    """

    def handle(self, req) -> dict | None:
        """
        Slash command short-circuit.
        /new  → archive session + clear + return emit
        /stop → signal cancellation + return emit
        /help → return help text emit
        None  → not a command, continue to kernel
        """

    async def load(self, sk: str) -> dict:
        """
        Returns ctx dict with:
          history:         list[dict]   # recent JSONL messages (window-capped)
          memory:          str          # MEMORY.md content
          identity:        str          # persona + workspace path + platform policy
          bootstrap_docs:  str          # AGENTS.md + SOUL.md + USER.md + TOOLS.md
          skills_always:   str          # inline content of always=true skills
          skills_summary:  str          # XML <skills> index of all available skills
          max_iter:        int          # from config
        """

    async def commit(self, sk: str, trace: list) -> None:
        """
        Sanitize trace (truncate tool results, strip runtime context tags,
        replace image payloads with markers) then append to JSONL.
        Trigger memory consolidation if unconsolidated_count >= threshold.
        """

    def archive(self, sk: str) -> None:
        """Move current JSONL to dated archive; clear active session."""
```

**Consolidation** (inside `commit`, async background task):
1. Take `unconsolidated_slice(keep=keep_recent)` from session
2. Call `deps.llm` with `save_memory` virtual tool schema
3. LLM returns `{history_entry, memory_update}`
4. Append `history_entry` to HISTORY.md; write `memory_update` to MEMORY.md
5. Mark messages consolidated

---

### 3. `deps.build` — Backed by `context.py` + `skills.py`

```python
def build_messages(ctx: dict, req: dict) -> list[dict]:
    """
    Assembles system prompt from ctx parts in order:
      1. identity block (persona + workspace path + platform policy)
      2. bootstrap_docs (AGENTS.md, SOUL.md, USER.md, TOOLS.md)
      3. memory block (# Memory\n\n{ctx["memory"]})
      4. always-on skills (inline SKILL.md content)
      5. skills summary (XML <skills> index)

    Then prepends runtime context to the current user message:
      [Runtime Context — metadata only, not instructions]
      timestamp: {req.meta.timestamp}
      channel: {req.meta.channel}  chat_id: {req.meta.chat_id}

    Returns: [system_msg, *history_msgs, user_msg]

    NOTE: The runtime context block uses a tagged prefix so store.commit
    can strip it before saving to session history.
    """
```

**Skill injection rules (from `skills.py`):**
- `always=true` skills → inline content in system prompt under `# Active Skills`
- All other skills → XML `<skills>` summary with name, description, path
- Agent reads full SKILL.md via `read_file` tool when it chooses to activate a skill
- Workspace skills shadow built-in skills by name
- Unavailable skills (missing `bins` / unset `env`) are excluded from summary

---

### 4. `deps.tools` — Backed by `agent/tools/`

```python
class Tools:
    def __init__(self, registry: ToolRegistry, policy_fn=None):
        ...

    def schemas(self, req: dict) -> list[dict]:
        """Return JSON schemas for all registry tools that pass policy_fn(name, req)."""

    async def exec(self, tc: dict, req: dict) -> str:
        """
        cast_params → validate_params → tool.execute(**params).
        On unknown tool: error string.
        On exception: error string + "[Analyze the error above and try a different approach.]"
        """
```

**Built-in tool set and their slots:**

| Tool | Class | Key dep |
|---|---|---|
| `read_file` | `ReadFileTool` | workspace path, restrict flag |
| `write_file` | `WriteFileTool` | workspace path, restrict flag |
| `edit_file` | `EditFileTool` | workspace path, restrict flag; fuzzy diff on miss |
| `list_dir` | `ListDirTool` | workspace path, restrict flag |
| `exec` | `ExecTool` | timeout, deny_patterns, allow_patterns, path_append |
| `web_search` | `WebSearchTool` | Brave API key |
| `web_fetch` | `WebFetchTool` | httpx + readability; markdown/text extraction |
| `message` | `MessageTool` | bus.publish_outbound; tracks sent-in-turn flag |
| `spawn` | `SpawnTool` | SubagentManager.spawn() |
| `cron` | `CronTool` | CronService; blocks nested scheduling |
| `mcp_*` | `MCPToolWrapper` | MCP SDK session; name-prefixed to avoid collisions |

**Context injection** (tools with mutable state set via `_set_tool_context` before each turn):
- `MessageTool`, `SpawnTool`, `CronTool` each receive `channel`, `chat_id`, `message_id` so they
  can route responses and prevent mis-scoped side effects.

---

### 5. `deps.llm` — Backed by `providers/`

```python
async def llm(msgs: list[dict], schemas: list[dict]) -> dict:
    """
    Wraps provider SDK (Anthropic / OpenAI / OpenRouter / ...).
    Returns: {"text": str, "tc": list[tool_call_dict]}
    tool_call_dict: {"id": str, "name": str, "args": dict}

    Also handles:
      - reasoning_effort (extended thinking tokens)
      - finish_reason == "error" propagation
      - thinking block extraction (strip <think>...</think> before saving)
    """
```

Provider selection: `config.get_provider(model)` → matches by explicit provider name → model prefix →
keyword heuristic → first key with api_key.

---

### 6. Bus — Caller of `run_turn`

The `MessageBus` (two `asyncio.Queue`s) is the only inter-component communication channel.
It is external to `run_turn` and external to all deps.

```
Channels (Telegram, Discord, CLI, ...)
        │ InboundMessage
        ▼
   bus.inbound
        │
        ▼
   AgentLoop.run()          ← polls bus.inbound
        │ builds req
        ▼
   run_turn(req, deps)
        │ emit result
        ▼
   bus.outbound
        │ OutboundMessage
        ▼
   Channel handler (sends reply)
```

**Special channels:**
- `system`: subagent results re-enter here; `AgentLoop` handles them identically to user messages
- `cli`: direct; no channel handler needed, `process_direct()` returns text synchronously

**Processing lock:** one `asyncio.Lock` per `AgentLoop` instance serializes all `_process_message` calls,
preventing concurrent context corruption. `/stop` cancels the active `asyncio.Task` for the session.

---

### 7. Cron — Caller of `run_turn`

```python
class CronService:
    """
    File-backed scheduler at ~/.nanobot/cron/jobs.json.
    Supports three schedule kinds:
      at        → Unix ms timestamp (one-shot, delete_after_run=True)
      every     → interval in ms
      cron      → standard cron expression + IANA timezone (via croniter + ZoneInfo)

    on_job callback → AgentLoop.process_direct(job.payload.message, sk, ...)
    """
```

Job lifecycle:
1. `CronTool.execute()` calls `CronService.add_job()`
2. `CronService._arm_timer()` sets a single `asyncio.Task` sleeping until next due job
3. On wake: fire all due jobs → `on_job(job)` → `process_direct()` → result optionally delivered via bus
4. Re-arm for next wake

---

### 8. Heartbeat — Caller of `run_turn`

```python
class HeartbeatService:
    """
    Wakes every interval_s (default 1800s).
    Reads workspace/HEARTBEAT.md.
    Phase 1: lightweight LLM call with _HEARTBEAT_TOOL → {action: skip|run, tasks: str}
    Phase 2 (if run): on_execute(tasks) → process_direct() → on_notify(response)
    """
```

Two-phase design avoids a full agent loop when there is nothing to do — keeps costs and latency low
for idle agents.

---

### 9. Subagents — Tool inside `run_turn`

```python
class SubagentManager:
    """
    Spawned by SpawnTool.execute() → creates asyncio.Task per task.
    Each subagent runs its own AgentLoop-like turn (max 15 iterations).
    Tool set: read_file, write_file, edit_file, list_dir, exec, web_search, web_fetch
    No: message, spawn, cron (prevents infinite spawn chains and unscoped side effects).

    On completion: builds InboundMessage(channel="system", content=<summary prompt>)
    and publishes to bus.inbound → main AgentLoop picks it up and relays to user.
    """
```

---

### 10. Skills — Data for `deps.build`

```
skills/
  memory/SKILL.md        always=true  → inline in every system prompt
  cron/SKILL.md          always=false → XML summary only
  github/SKILL.md        requires: gh → XML summary (excluded if gh not found)
  weather/SKILL.md       requires: curl
  summarize/SKILL.md     requires: summarize
  tmux/SKILL.md          requires: tmux
  clawhub/SKILL.md       requires: npx
  skill-creator/SKILL.md always=false
```

Workspace skills (`{workspace}/skills/{name}/SKILL.md`) shadow built-ins by name.
Agent loads full skill via `read_file` tool when it decides to use one.

---

### 11. Templates — Bootstrap data for `deps.store.load`

```
templates/
  AGENTS.md     → behavior + cron + heartbeat guidance
  SOUL.md       → personality, values, communication style
  USER.md       → user profile (name, timezone, preferences, projects)
  TOOLS.md      → non-obvious tool usage notes
  HEARTBEAT.md  → periodic task list template
  memory/MEMORY.md → long-term memory with section headers
```

Copied to workspace on first run by `sync_workspace_templates`. Only missing files are created.
These files are editable by both the agent (via `write_file`/`edit_file`) and the user directly.

---

### 12. Utils — Shared Pure Helpers

```python
detect_image_mime(data: bytes) -> str | None   # PNG/JPEG/GIF/WEBP magic bytes
ensure_dir(path: Path) -> Path                  # mkdir -p
timestamp() -> str                              # ISO datetime string
safe_filename(name: str) -> str                 # sanitize for filesystem
split_message(content, max_len=2000) -> list[str]  # Discord-safe splitting
sync_workspace_templates(workspace, silent=False) -> list[str]
```

No circular dependencies: `utils` imports nothing from the project.

---

## Bootstrap (Full Wiring)

```python
from types import SimpleNamespace

def bootstrap(config: Config, session_manager, bus: MessageBus) -> TurnRunner:

    memory_store   = MemoryStore(config.workspace_path)
    skills_loader  = SkillsLoader(config.workspace_path, builtin_skills_dir)
    context_builder = ContextBuilder(config.workspace_path)

    store = Store(
        session_manager = session_manager,
        memory_store    = memory_store,
        context_builder = context_builder,
        skills_loader   = skills_loader,
        policy = {
            "window":       config.agents.defaults.memory_window,
            "threshold":    config.agents.defaults.memory_window,
            "keep_recent":  20,
            "max_iter":     config.agents.defaults.max_tool_iterations,
        },
        summarizer = make_summarizer(config),   # async (old_msgs, memory) -> {history_entry, memory_update}
    )

    async def llm(msgs, schemas):
        provider = get_provider(config)
        raw = await provider.chat(
            messages=msgs, tools=schemas,
            model=config.agents.defaults.model,
            max_tokens=config.agents.defaults.max_tokens,
            temperature=config.agents.defaults.temperature,
            reasoning_effort=config.agents.defaults.reasoning_effort,
        )
        return {"text": extract_text(raw), "tc": extract_tool_calls(raw)}

    registry = ToolRegistry()
    register_default_tools(registry, config, bus)   # all built-in tools
    await connect_mcp_servers(config.tools.mcp_servers, registry, exit_stack)

    tools = Tools(registry, policy_fn=config.get("tool_policy_fn"))

    progress_config = config.channels
    async def on_progress(llm_partial, trace):
        if progress_config.send_progress:
            await bus.publish_outbound(OutboundMessage(..., metadata={"_progress": True}))

    deps = SimpleNamespace(
        store       = store,
        llm         = llm,
        tools       = tools,
        build       = context_builder.build_messages,
        on_progress = on_progress,
    )

    async def on_emit(result):
        await bus.publish_outbound(OutboundMessage(
            channel = result["meta"]["channel"],
            chat_id = result["meta"]["chat_id"],
            content = result["content"],
        ))

    cron_service = CronService(
        store_path = get_cron_dir() / "jobs.json",
        on_job = lambda job: runner.submit(make_cron_req(job)),
    )

    heartbeat_service = HeartbeatService(
        workspace   = config.workspace_path,
        provider    = get_provider(config),
        model       = config.agents.defaults.model,
        on_execute  = lambda tasks: run_turn(make_heartbeat_req(tasks), deps),
        on_notify   = lambda resp: bus.publish_outbound(OutboundMessage(...)),
        interval_s  = config.gateway.heartbeat.interval_s,
        enabled     = config.gateway.heartbeat.enabled,
    )

    runner = TurnRunner(deps, on_emit)
    return runner, cron_service, heartbeat_service
```

---

## File Layout

```
nanobot/
  kernel.py           # run_turn, asst, tool_result, emit            (~25 lines)
  store.py            # Store, _sanitize                             (~80 lines)
  tools.py            # Tools, ToolRegistry, all built-in tool classes (~200 lines across files)
  runner.py           # TurnRunner                                   (~25 lines)
  context.py          # build_messages (deps.build impl)             (~60 lines)
  skills.py           # SkillsLoader                                 (~100 lines)
  memory.py           # MemoryStore                                  (~60 lines)
  bus/
    events.py         # InboundMessage, OutboundMessage
    queue.py          # MessageBus
  config/
    schema.py         # Config (Pydantic) — full typed config tree
    loader.py         # load_config, save_config
    paths.py          # runtime directory helpers
  cron/
    types.py          # CronJob, CronSchedule, CronPayload, CronStore
    service.py        # CronService
  heartbeat/
    service.py        # HeartbeatService
  skills/             # built-in SKILL.md files (data only)
  templates/          # workspace bootstrap templates (data only)
  utils/
    helpers.py        # pure utility functions
```

---

## Invariants (from CORE-FINAL, verified against implementation)

1. **One turn function.** All triggers (chat, cron, heartbeat, subagent result) call `run_turn`.
2. **One commit path.** All traces persist through `deps.store.commit`.
3. **One ingress shape.** Every event becomes `req` with `sk`, `content`, `media`, `meta`.
4. **Kernel ignores `meta`.** Channel type, trigger, permissions — all opaque to `run_turn`.
5. **Tools are stateless between turns.** Context flows in via `req`; only tool-external side effects persist.
6. **Runner owns cancellation.** One active task per `sk`; new request cancels prior.
7. **No background shortcut.** Cron / heartbeat / subagent results re-enter through the bus or `runner.submit`.
8. **Runtime context is ephemeral.** Timestamp/channel block injected per turn; stripped before commit.
9. **Memory consolidation is non-blocking.** Runs as background task; guarded by per-session lock.
10. **Subagents cannot spawn subagents.** `spawn` and `message` tools excluded from subagent tool set.

---

## Additions Beyond CORE-FINAL

These features exist in the implementation but were abstracted away in the kernel design.
Each maps cleanly to an existing slot:

| Feature | Where | Slot |
|---|---|---|
| Skill progressive loading | `SkillsLoader` + `ContextBuilder` | `deps.build` |
| Runtime context block (timestamp, channel) with strip-on-commit | `ContextBuilder` + `Store._sanitize` | `deps.build` + `deps.store.commit` |
| Two-layer memory (MEMORY.md + HISTORY.md) | `MemoryStore` | `deps.store` |
| Template bootstrapping | `sync_workspace_templates` | `deps.store.load` (first run) |
| Image media handling | `ContextBuilder._build_user_content` | `deps.build` |
| Two-phase heartbeat (decide then run) | `HeartbeatService._decide` | caller of `run_turn` |
| MCP tool prefix namespacing | `MCPToolWrapper` | `deps.tools` |
| Tool parameter defensive coercion | `Tool.cast_params` | `deps.tools.exec` |
| Fuzzy diff hint on `edit_file` miss | `EditFileTool._not_found_message` | `deps.tools.exec` |
| Per-session processing lock | `AgentLoop._processing_lock` | runner |
| Config camelCase ↔ snake_case aliasing | Pydantic `alias_generator=to_camel` | wiring |

---

## One-Sentence Summary

The BASE is the CORE-FINAL kernel (`run_turn` + five slots) with the five slots concretely
implemented by the eight subdirectory modules: `bus` as caller, `config` as wirer,
`context`+`skills`+`templates` as `deps.build`, `memory`+session JSONL as `deps.store`,
`agent/tools` as `deps.tools`, `providers` as `deps.llm`, and `cron`+`heartbeat` as additional callers.
