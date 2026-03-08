# Nanobot BASE Design — claude-4.6-opus-high-thinking

## Stored Prompt

```
read CORE-FINAL-{model-name-and-version}.md;
try to DESIGN the BASE for all the functionalities from following nanobot subdirectories: agent, bus, config, cron, heartbeat, skills, templates, utls;
store the results in BASE-{model-name-and-version}.md;
also - store this promp in the resulting file
```

## Source Documents

- CORE-FINAL-claude-4.6-opus.md (the kernel reference)
- nanobot/agent/ (loop, context, memory, skills, subagent, tools/)
- nanobot/bus/ (events, queue)
- nanobot/config/ (schema, loader, paths)
- nanobot/cron/ (types, service)
- nanobot/heartbeat/ (service)
- nanobot/templates/ (AGENTS.md, SOUL.md, USER.md, TOOLS.md, HEARTBEAT.md, memory/)
- nanobot/utils/ (helpers)

---

## Design Thesis

CORE-FINAL proved that the agentic kernel is one function — `run_turn(req, deps)`. But the actual codebase has ~2,500 lines of real infrastructure (sessions, memory consolidation, MCP, cron timers, heartbeat decisions, skill loading, template syncing, config parsing) that must exist *somewhere*. The BASE defines where.

**The BASE is the set of contracts and adapters that sit between the one-function kernel and the outside world.** It answers one question: *given `run_turn`, what do you need to build around it so that all eight subsystems (agent, bus, config, cron, heartbeat, skills, templates, utils) actually work?*

Three principles govern the design:

1. **The kernel never changes.** New features add adapters, not kernel code.
2. **Every subsystem is a plug factory.** It constructs one or more `deps` callables. It never touches `run_turn` directly.
3. **The seams are explicit.** Each connection point between subsystems has a named contract — not an implicit coupling buried in constructor args.

---

## Architecture Overview

```
                    ┌──────────────┐
                    │   Channels   │  telegram, discord, cli, ...
                    └──────┬───────┘
                           │ InboundMessage
                    ┌──────▼───────┐
                    │  MessageBus  │  async queues
                    └──────┬───────┘
                           │ InboundMessage
              ┌────────────▼────────────┐
              │       TurnRunner        │  queue + per-sk cancellation
              │  make_request(msg)→req  │
              └────────────┬────────────┘
                           │ req
                    ┌──────▼───────┐
                    │  run_turn()  │  THE KERNEL
                    │  req + deps  │
                    └──────┬───────┘
                           │ result
              ┌────────────▼────────────┐
              │    to_outbound(result)   │
              └────────────┬────────────┘
                           │ OutboundMessage
                    ┌──────▼───────┐
                    │  MessageBus  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Channels   │
                    └──────────────┘

   Side-entries (all produce req, all call run_turn):
   ┌──────────┐  ┌───────────┐  ┌────────────┐
   │   Cron   │  │ Heartbeat │  │  Subagent  │
   └────┬─────┘  └─────┬─────┘  └─────┬──────┘
        └───────────────┼──────────────┘
                        ▼
                   MessageBus.publish_inbound(msg)
```

---

## 1. The Kernel (Unchanged from CORE-FINAL)

```python
async def run_turn(req: dict, deps) -> dict:
    cmd = deps.handle_command(req)
    if cmd is not None:
        await deps.commit(req["sk"], cmd.get("trace", []))
        return cmd

    ctx = await deps.load(req["sk"])
    msgs = deps.build(ctx, req)
    trace = []

    for _ in range(deps.max_iter):
        llm = await deps.chat(msgs, deps.schemas(req))
        trace.append({"role": "assistant", "content": llm["text"], "tool_calls": llm["tc"]})
        if not llm["tc"]:
            break
        for tc in llm["tc"]:
            out = await deps.exec(tc, req)
            trace.append({"role": "tool", "call_id": tc["id"], "content": out})
        msgs = deps.build(ctx, req) + trace
        if deps.on_progress:
            await deps.on_progress(llm, trace)

    await deps.commit(req["sk"], trace)
    return {"emit": True, "content": llm.get("text", ""), "trace": trace}
```

Nine deps, zero classes, zero globals. Everything below exists to *construct these nine callables*.

---

## 2. The Request and Result Shapes

### 2.1 Canonical Request

```python
req = {
    "sk":      str,    # session key — the only identity the kernel sees
    "content": str,    # user / system / cron message text
    "media":   list,   # optional attachments (paths or URLs)
    "meta":    dict,   # opaque bag: channel, chat_id, trigger, message_id, sender_id, ...
}
```

The kernel reads `sk` and `content`. It passes `meta` through to tools and back in the result. It never inspects `meta`.

**Trigger taxonomy** (carried in `meta["trigger"]`):

| Trigger | Source | How it enters |
|---------|--------|---------------|
| `manual` | User message via channel | `MessageBus.publish_inbound` |
| `cron` | CronService timer fires | `CronService.on_job` → bus |
| `heartbeat` | HeartbeatService tick | `HeartbeatService.on_execute` → bus |
| `subagent` | SubagentManager background task | `SubagentManager._announce_result` → bus |
| `system` | Internal system message | direct `bus.publish_inbound` |
| `api` | External API call | gateway → bus |

### 2.2 Canonical Result

```python
result = {
    "emit":    bool,   # True = send to channel; False = silent (e.g. subagent consumed output)
    "content": str,    # final text response
    "trace":   list,   # full assistant/tool message history for this turn
    "meta":    dict,   # passed through from req for routing
}
```

For command short-circuits (`/help`, `/new`), `emit` is True and `trace` may be empty.

---

## 3. Deps Contracts — The Nine Plugs

Each plug is defined by its signature and its responsibilities. The BASE maps each plug to the subsystem(s) that implement it.

### 3.1 `deps.load(sk) → ctx`

**Implementor:** Store (backed by SessionManager + MemoryStore + ContextBuilder + SkillsLoader)

```python
ctx = {
    "history":  list[dict],  # recent messages (windowed by memory_window)
    "identity": str,         # core identity block (runtime, workspace, guidelines)
    "memory":   str,         # MEMORY.md content
    "bootstrap": str,        # AGENTS.md + SOUL.md + USER.md + TOOLS.md
    "skills":   str,         # always-on skills content + summary of available skills
    "max_iter": int,         # from config (default 25-40)
}
```

**What it does internally:**

1. `SessionManager.get_or_create(sk)` → session with message history
2. `session.get_history(max_messages=memory_window)` → windowed history
3. `ContextBuilder._get_identity()` → identity string
4. `ContextBuilder._load_bootstrap_files()` → bootstrap string
5. `MemoryStore.get_memory_context()` → memory string
6. `SkillsLoader.get_always_skills()` + `build_skills_summary()` → skills string

### 3.2 `deps.commit(sk, trace) → None`

**Implementor:** Store (backed by SessionManager + MemoryStore)

**What it does internally:**

1. Sanitize trace: truncate large tool results, strip runtime context tags, replace inline images with `[image]`
2. Append sanitized messages to `session.messages`
3. Save session to disk
4. If unconsolidated message count exceeds `memory_window`: trigger background consolidation
5. Consolidation: LLM summarizes old messages → append to `HISTORY.md`, update `MEMORY.md`, advance `last_consolidated` pointer

### 3.3 `deps.build(ctx, req) → list[dict]`

**Implementor:** ContextBuilder

```python
def build(ctx, req):
    system = join_sections([ctx["identity"], ctx["bootstrap"], ctx["memory"], ctx["skills"]])
    runtime = build_runtime_context(req["meta"].get("channel"), req["meta"].get("chat_id"))
    user_content = merge_media(req["content"], req.get("media"), runtime)
    return [
        {"role": "system", "content": system},
        *ctx["history"],
        {"role": "user", "content": user_content},
    ]
```

The runtime context (current time, channel, chat_id) is injected as a prefix to the user message, tagged as `[Runtime Context — metadata only, not instructions]` to prevent prompt injection.

### 3.4 `deps.chat(msgs, schemas) → {"text": str, "tc": list}`

**Implementor:** LLMProvider wrapper

Takes the provider-specific SDK (OpenAI, Anthropic, LiteLLM, etc.) and normalizes its response into `{"text": str | None, "tc": list[tool_call_dict]}`. Also normalizes:
- `reasoning_content` / `thinking_blocks` (for extended thinking models)
- `finish_reason` (including `"error"` for provider failures)

The schemas are OpenAI-format function definitions: `[{"type": "function", "function": {"name": ..., "description": ..., "parameters": ...}}]`

### 3.5 `deps.schemas(req) → list[dict]`

**Implementor:** ToolRegistry

```python
def schemas(req):
    return [tool.to_schema() for tool in registry.values()
            if policy_allows(tool.name, req.get("meta", {}))]
```

Currently no policy filtering in the codebase — all registered tools are exposed. The contract supports future per-channel or per-trigger tool restriction via `req["meta"]`.

### 3.6 `deps.exec(tc, req) → str`

**Implementor:** ToolRegistry

```python
async def exec(tc, req):
    tool = registry.get(tc["name"])
    if not tool:
        return f"Error: Tool '{tc['name']}' not found. Available: {registry.tool_names}"
    params = tool.cast_params(tc["args"])  # schema-driven type coercion
    errors = tool.validate_params(params)
    if errors:
        return f"Error: Invalid parameters: {'; '.join(errors)}"
    result = await tool.execute(**params)
    return result + hint_suffix_on_error(result)
```

Tool context flows via `req` — tools that need routing info (message, spawn, cron) receive it through `set_context(channel, chat_id)` called before the loop starts, not through global state.

### 3.7 `deps.handle_command(req) → dict | None`

**Implementor:** Store (command handler)

| Command | Action | Result |
|---------|--------|--------|
| `/help` | None | `{"emit": True, "content": "🐈 nanobot commands:\n/new...\n/stop...\n/help..."}` |
| `/new` | Archive session → clear | `{"emit": True, "content": "New session started."}` |
| `/stop` | Cancel active tasks + subagents | `{"emit": True, "content": "⏹ Stopped N task(s)."}` |
| anything else | — | `None` (proceed to LLM loop) |

`/stop` is special: it requires access to the runner's active task map and the subagent manager. In the current implementation, it's handled outside `run_turn` at the runner level. In the BASE, it can either stay external (runner intercepts before calling `run_turn`) or be injected as a dep.

### 3.8 `deps.on_progress(llm, trace) → None`

**Implementor:** Streaming callback (optional)

Emits partial results to the channel while the agent is still working:
- Stripped `<think>...</think>` blocks as interim text
- Tool call hints like `web_search("query")` as status indicators
- Configurable via `channels.send_progress` and `channels.send_tool_hints`

### 3.9 `deps.max_iter → int`

**Source:** Config (`agents.defaults.max_tool_iterations`, default 40)

---

## 4. Subsystem Contracts

### 4.1 Bus (nanobot/bus/)

The bus is the *only* way messages enter and leave the system. No shortcutting.

```python
@dataclass
class InboundMessage:
    channel: str                          # telegram, discord, slack, cli, system
    sender_id: str                        # user identifier
    chat_id: str                          # chat/channel identifier
    content: str                          # message text
    timestamp: datetime = now()
    media: list[str] = []                 # file paths or URLs
    metadata: dict[str, Any] = {}         # channel-specific opaque data
    session_key_override: str | None = None

    @property
    def session_key(self) -> str:
        return self.session_key_override or f"{self.channel}:{self.chat_id}"

@dataclass
class OutboundMessage:
    channel: str
    chat_id: str
    content: str
    reply_to: str | None = None
    media: list[str] = []
    metadata: dict[str, Any] = {}

class MessageBus:
    inbound:  asyncio.Queue[InboundMessage]
    outbound: asyncio.Queue[OutboundMessage]

    async def publish_inbound(msg)  → None
    async def consume_inbound()     → InboundMessage
    async def publish_outbound(msg) → None
    async def consume_outbound()    → OutboundMessage
```

**Adapter: InboundMessage → req**

```python
def make_request(msg: InboundMessage, trigger: str = "manual") -> dict:
    return {
        "sk": msg.session_key,
        "content": msg.content,
        "media": msg.media or [],
        "meta": {
            "channel": msg.channel,
            "chat_id": msg.chat_id,
            "sender_id": msg.sender_id,
            "trigger": trigger,
            "message_id": (msg.metadata or {}).get("message_id"),
            **(msg.metadata or {}),
        },
    }
```

**Adapter: result → OutboundMessage**

```python
def to_outbound(result: dict) -> OutboundMessage | None:
    if not result.get("emit"):
        return None
    meta = result.get("meta", {})
    return OutboundMessage(
        channel=meta.get("channel", "cli"),
        chat_id=meta.get("chat_id", "direct"),
        content=result.get("content", ""),
        metadata=meta,
    )
```

### 4.2 Config (nanobot/config/)

Config is the **static wiring specification**. It determines what gets plugged into deps at bootstrap time.

**Schema hierarchy (Pydantic):**

```
Config (root)
├── agents: AgentsConfig
│   └── defaults: AgentDefaults
│       ├── workspace, model, provider, max_tokens, temperature
│       ├── max_tool_iterations, memory_window, reasoning_effort
├── channels: ChannelsConfig
│   ├── send_progress, send_tool_hints
│   └── telegram, discord, slack, whatsapp, feishu, dingtalk,
│       email, mochat, qq, matrix  (each: enabled, token/key, allow_from, ...)
├── providers: ProvidersConfig
│   └── openai, anthropic, openrouter, deepseek, groq, gemini,
│       custom, azure_openai, ...  (each: api_key, api_base, extra_headers)
├── gateway: GatewayConfig
│   ├── host, port
│   └── heartbeat: HeartbeatConfig (enabled, interval_s)
├── tools: ToolsConfig
│   ├── web: WebToolsConfig (proxy, search.api_key)
│   ├── exec: ExecToolConfig (timeout, path_append)
│   ├── restrict_to_workspace: bool
│   └── mcp_servers: dict[str, MCPServerConfig]
```

**Loader:**

```python
def load_config(path: Path | None) -> Config    # JSON → Pydantic, with migration
def save_config(config: Config, path: Path | None) -> None
def get_config_path() -> Path                    # global state for multi-instance
def set_config_path(path: Path) -> None
```

**Paths (derived from config location):**

```python
def get_data_dir() -> Path                       # config_path.parent
def get_runtime_subdir(name: str) -> Path        # data_dir / name
def get_media_dir(channel: str | None) -> Path   # data_dir / media / [channel]
def get_cron_dir() -> Path                       # data_dir / cron
def get_logs_dir() -> Path                       # data_dir / logs
def get_workspace_path(ws: str | None) -> Path   # expand ~ + ensure_dir
```

**Provider matching:** `Config._match_provider(model)` resolves which provider config to use based on model name keywords, explicit prefix, or fallback — returning `(ProviderConfig, spec_name)`.

### 4.3 Tools (nanobot/agent/tools/)

**Tool ABC:**

```python
class Tool(ABC):
    @property
    def name(self) -> str: ...
    @property
    def description(self) -> str: ...
    @property
    def parameters(self) -> dict: ...        # JSON Schema

    async def execute(self, **kwargs) -> str
    def cast_params(self, params) -> dict    # schema-driven type coercion
    def validate_params(self, params) -> list[str]  # returns error list
    def to_schema(self) -> dict              # OpenAI function format
```

**ToolRegistry:**

```python
class ToolRegistry:
    def register(tool: Tool) → None
    def unregister(name: str) → None
    def get(name: str) → Tool | None
    def get_definitions() → list[dict]       # all schemas
    async def execute(name, params) → str    # cast → validate → execute → error hint
```

**Built-in tools and their concerns:**

| Tool | Module | Key concern |
|------|--------|-------------|
| `read_file` | filesystem.py | 128KB cap, workspace-relative resolution, allowed_dir enforcement |
| `write_file` | filesystem.py | auto-create parents, allowed_dir enforcement |
| `edit_file` | filesystem.py | exact-match replace, fuzzy diff on mismatch, uniqueness check |
| `list_dir` | filesystem.py | allowed_dir enforcement |
| `exec` | shell.py | deny patterns (rm -rf, dd, shutdown…), timeout, workspace restriction, path-traversal guard |
| `web_search` | web.py | Brave Search API, proxy support, lazy API key resolution |
| `web_fetch` | web.py | Readability extraction, URL validation, proxy, redirect limit |
| `message` | message.py | per-turn send tracking (`_sent_in_turn`), routing via `set_context` |
| `spawn` | spawn.py | delegates to SubagentManager, carries origin channel/chat_id |
| `cron` | cron.py | add/list/remove jobs, prevents recursive scheduling from cron context |
| `mcp_*` | mcp.py | MCPToolWrapper wraps MCP server tools as native Tool instances |

**Tool context pattern:** Tools that need routing info (message, spawn, cron) expose `set_context(channel, chat_id)`. The runner calls this once before each turn. This avoids global mutable state — context is scoped to the current turn.

### 4.4 Memory (nanobot/agent/memory.py)

Two-layer persistent memory, independent of session history:

```
workspace/
  memory/
    MEMORY.md     # long-term facts (full rewrite on each consolidation)
    HISTORY.md    # grep-searchable log (append-only)
```

**MemoryStore:**

```python
class MemoryStore:
    def __init__(workspace: Path)
    def read_long_term() → str
    def write_long_term(content: str) → None
    def append_history(entry: str) → None
    def get_memory_context() → str

    async def consolidate(session, provider, model,
                          archive_all=False, memory_window=50) → bool
```

**Consolidation protocol:**

1. Select messages to consolidate: `session.messages[last_consolidated:-keep_count]`
2. Format as timestamped lines: `[YYYY-MM-DD HH:MM] ROLE: content`
3. Send to LLM with `save_memory` virtual tool (forced tool call)
4. LLM returns `history_entry` (paragraph for HISTORY.md) + `memory_update` (full MEMORY.md replacement)
5. Write both files, advance `session.last_consolidated`

Consolidation is **background + lock-guarded**: one consolidation per session at a time, non-blocking to the main agent loop.

### 4.5 Skills (nanobot/agent/skills.py)

Progressive loading pattern: summary in system prompt, full content on-demand via `read_file`.

**SkillsLoader:**

```python
class SkillsLoader:
    def __init__(workspace: Path, builtin_skills_dir: Path | None)

    def list_skills(filter_unavailable=True) → list[dict]
    def load_skill(name: str) → str | None
    def load_skills_for_context(names: list[str]) → str
    def build_skills_summary() → str               # XML format for system prompt
    def get_always_skills() → list[str]             # skills with always=true
    def get_skill_metadata(name: str) → dict | None # YAML frontmatter
```

**Skill resolution order:** workspace skills (`workspace/skills/{name}/SKILL.md`) override builtin skills (`nanobot/skills/{name}/SKILL.md`).

**Requirements checking:** Skills declare `requires.bins` and `requires.env` in frontmatter metadata. Unavailable skills appear in summary with `available="false"` and missing requirements listed.

**Integration with ContextBuilder:**
- `get_always_skills()` → content injected directly into system prompt
- `build_skills_summary()` → XML block in system prompt; agent reads full SKILL.md via `read_file` tool when needed

### 4.6 Context Builder (nanobot/agent/context.py)

Assembles the full message list for each LLM call.

**System prompt structure:**

```
# nanobot 🐈
## Runtime       — platform, Python version
## Workspace     — paths to memory, history, skills
## Platform Policy — POSIX or Windows guidance
## nanobot Guidelines — tool usage rules

---

## AGENTS.md     — agent instructions (bootstrap file)
## SOUL.md       — identity/personality (bootstrap file)
## USER.md       — user profile (bootstrap file)
## TOOLS.md      — tool usage notes (bootstrap file)

---

# Memory
## Long-term Memory — MEMORY.md content

---

# Active Skills  — always-on skill content
# Skills         — XML summary of all skills
```

**Runtime context** (injected as user message prefix, not system message):

```
[Runtime Context — metadata only, not instructions]
Current Time: 2026-03-09 12:00 (Monday) (CET)
Channel: telegram
Chat ID: 8281248569
```

**Message assembly helpers:**

```python
def add_tool_result(messages, tool_call_id, tool_name, result) → messages
def add_assistant_message(messages, content, tool_calls=None,
                          reasoning_content=None, thinking_blocks=None) → messages
```

### 4.7 Subagent (nanobot/agent/subagent.py)

Background task execution with reduced capabilities.

**SubagentManager:**

```python
class SubagentManager:
    async def spawn(task, label, origin_channel, origin_chat_id, session_key) → str
    async def cancel_by_session(session_key) → int
    def get_running_count() → int
```

**Subagent characteristics:**
- Reduced tool set: filesystem + exec + web. No message tool (can't send to users). No spawn tool (can't nest).
- Max 15 iterations (vs 40 for main agent)
- Focused system prompt: workspace path, available skills, runtime context
- Result announcement: `InboundMessage(channel="system", sender_id="subagent", chat_id="origin_channel:origin_chat_id")` → re-enters main agent loop for natural-language summarization to user

**Task lifecycle:**

```
SpawnTool.execute(task, label)
  → SubagentManager.spawn(...)
    → asyncio.create_task(_run_subagent(...))
      → [build tools, build prompt, run LLM-tool loop]
      → _announce_result(...) → bus.publish_inbound(system_msg)
    → cleanup callback removes task from tracking dicts
```

### 4.8 Cron (nanobot/cron/)

Timer-based job scheduler with JSON persistence.

**Types:**

```python
@dataclass
class CronSchedule:
    kind: Literal["at", "every", "cron"]   # one-shot, interval, cron expression
    at_ms: int | None        # for "at"
    every_ms: int | None     # for "every"
    expr: str | None         # for "cron" (e.g. "0 9 * * *")
    tz: str | None           # IANA timezone for cron expressions

@dataclass
class CronPayload:
    kind: Literal["system_event", "agent_turn"] = "agent_turn"
    message: str = ""
    deliver: bool = False
    channel: str | None = None
    to: str | None = None

@dataclass
class CronJob:
    id: str                  # uuid[:8]
    name: str
    enabled: bool
    schedule: CronSchedule
    payload: CronPayload
    state: CronJobState      # next_run_at_ms, last_run_at_ms, last_status, last_error
    created_at_ms: int
    updated_at_ms: int
    delete_after_run: bool   # True for one-shot "at" jobs
```

**CronService:**

```python
class CronService:
    def __init__(store_path: Path, on_job: Callable[[CronJob], Coroutine[..., str | None]])

    async def start() → None     # load store, compute next runs, arm timer
    def stop() → None
    def list_jobs(include_disabled=False) → list[CronJob]
    def add_job(name, schedule, message, deliver, channel, to, delete_after_run) → CronJob
    def remove_job(job_id) → bool
    def enable_job(job_id, enabled) → CronJob | None
    async def run_job(job_id, force=False) → bool
```

**Timer mechanism:** Single `asyncio.Task` sleeps until nearest `next_run_at_ms`. On wake: execute all due jobs, recompute next runs, re-arm. External file modification detected via mtime check on next load.

**Integration:** `on_job(job)` callback builds `req` from `job.payload.message` and submits to runner. If `job.payload.deliver`, result routes to `job.payload.channel`/`job.payload.to`.

### 4.9 Heartbeat (nanobot/heartbeat/)

Periodic wake-up with LLM-based decision gate.

**HeartbeatService:**

```python
class HeartbeatService:
    def __init__(workspace, provider, model, on_execute, on_notify, interval_s, enabled)

    async def start() → None
    def stop() → None
    async def trigger_now() → str | None
```

**Two-phase protocol:**

| Phase | Input | Output | Cost |
|-------|-------|--------|------|
| 1. Decide | HEARTBEAT.md content | `skip` or `run` (via virtual `heartbeat` tool call) | Cheap LLM call |
| 2. Execute | tasks string | full agent response | Full agent turn |

Phase 1 uses a forced tool call (`heartbeat` with `action: skip|run, tasks: str`) to avoid free-text parsing. Only when `action == "run"` does Phase 2 fire, calling `on_execute(tasks)` which runs a full agent turn through the main loop.

**HEARTBEAT.md:** User-editable file in workspace. If empty or only headers/comments, Phase 1 returns `skip`. Tasks are managed via file tools (edit_file, write_file) by the agent itself.

### 4.10 Templates (nanobot/templates/)

Bootstrap files that define the agent's initial workspace.

| Template | Purpose | Consumed by |
|----------|---------|-------------|
| `AGENTS.md` | Agent instructions (reminders, heartbeat guidance) | ContextBuilder (bootstrap) |
| `SOUL.md` | Identity, personality, values | ContextBuilder (bootstrap) |
| `USER.md` | User profile, preferences | ContextBuilder (bootstrap) |
| `TOOLS.md` | Tool usage notes, safety limits | ContextBuilder (bootstrap) |
| `HEARTBEAT.md` | Active/completed task list template | HeartbeatService |
| `memory/MEMORY.md` | Long-term memory template | MemoryStore |

**Sync function:**

```python
def sync_workspace_templates(workspace: Path, silent=False) → list[str]:
    """Create missing template files from bundled package resources.
    Never overwrites existing files. Returns list of created paths."""
```

Called at startup. Only creates files that don't exist yet — user customizations are preserved.

### 4.11 Utils (nanobot/utils/)

Stateless utility functions used across subsystems.

```python
def ensure_dir(path: Path) → Path           # mkdir -p, return path
def detect_image_mime(data: bytes) → str | None  # magic bytes → MIME
def timestamp() → str                       # ISO format now()
def safe_filename(name: str) → str          # replace unsafe chars with _
def split_message(content: str, max_len=2000) → list[str]  # chunk for Discord etc.
def sync_workspace_templates(workspace, silent=False) → list[str]  # see Templates
```

---

## 5. The Runner (Adapter Between Bus and Kernel)

The runner is **not** part of the kernel. It owns:
- The consume loop (blocking on `bus.consume_inbound`)
- Per-session task management (one active task per `sk`, new request cancels prior)
- `/stop` interception (before `run_turn`)
- Tool context setup (calling `set_context` on routing-aware tools)
- MCP connection lifecycle (lazy, one-time)
- Progress streaming callback construction

```python
class TurnRunner:
    def __init__(self, bus, deps, subagents):
        self.bus = bus
        self.deps = deps
        self.subagents = subagents
        self.active_tasks: dict[str, list[asyncio.Task]] = {}
        self.processing_lock = asyncio.Lock()

    async def run(self):
        while self._running:
            msg = await bus.consume_inbound()
            if msg.content.strip().lower() == "/stop":
                await self._handle_stop(msg)
            else:
                task = asyncio.create_task(self._dispatch(msg))
                self.active_tasks.setdefault(msg.session_key, []).append(task)

    async def _dispatch(self, msg):
        async with self.processing_lock:
            req = make_request(msg)
            self._set_tool_context(req["meta"])
            result = await run_turn(req, self.deps)
            if out := to_outbound(result):
                await self.bus.publish_outbound(out)
```

---

## 6. Bootstrap Wiring

```python
from types import SimpleNamespace

def bootstrap(config: Config) -> TurnRunner:
    bus = MessageBus()
    workspace = config.workspace_path

    # Subsystem construction
    sessions = SessionManager(workspace)
    memory = MemoryStore(workspace)
    skills = SkillsLoader(workspace)
    context = ContextBuilder(workspace)  # uses memory + skills internally
    provider = make_provider(config)     # LLMProvider from config
    model = config.agents.defaults.model

    # Tool registry
    tools = ToolRegistry()
    tools.register(ReadFileTool(workspace=workspace))
    tools.register(WriteFileTool(workspace=workspace))
    tools.register(EditFileTool(workspace=workspace))
    tools.register(ListDirTool(workspace=workspace))
    tools.register(ExecTool(working_dir=str(workspace), timeout=config.tools.exec.timeout))
    tools.register(WebSearchTool(api_key=config.tools.web.search.api_key))
    tools.register(WebFetchTool(proxy=config.tools.web.proxy))
    tools.register(MessageTool(send_callback=bus.publish_outbound))
    subagents = SubagentManager(provider, workspace, bus, model)
    tools.register(SpawnTool(manager=subagents))
    # + optional: CronTool, MCP tools (lazy)

    # Store (deps.load, deps.commit, deps.handle_command)
    store = make_store(sessions, memory, context, provider, model, config)

    # Deps namespace
    deps = SimpleNamespace(
        load            = store.load,
        commit          = store.commit,
        handle_command  = store.handle_command,
        build           = context.build_messages_from_ctx,
        chat            = lambda msgs, schemas: provider_chat(msgs, schemas, model, provider),
        schemas         = lambda req: tools.get_definitions(),
        exec            = lambda tc, req: tools.execute(tc["name"], tc["args"]),
        on_progress     = None,  # set per-turn by runner
        max_iter        = config.agents.defaults.max_tool_iterations,
    )

    runner = TurnRunner(bus, deps, subagents)

    # Side-entry services
    if config.gateway.heartbeat.enabled:
        heartbeat = HeartbeatService(workspace, provider, model,
                                     on_execute=runner.process_direct,
                                     interval_s=config.gateway.heartbeat.interval_s)
    cron = CronService(get_cron_dir() / "jobs.json",
                       on_job=lambda job: runner.submit_cron(job))

    return runner
```

---

## 7. Seam Diagram

Every arrow is a named contract. No implicit coupling.

```
  Config ──────────────────────────────────────────────────┐
    │                                                      │
    │ workspace, model, provider, timeouts, keys           │
    ▼                                                      ▼
  ┌─────────────┐  ┌──────────┐  ┌────────┐  ┌──────────────────┐
  │ SessionMgr  │  │MemoryStr │  │ Skills │  │  ToolRegistry    │
  │  .get(sk)   │  │ .read()  │  │ .list()│  │  .register(tool) │
  │  .save(s)   │  │ .write() │  │ .load()│  │  .execute(n,p)   │
  └──────┬──────┘  └────┬─────┘  └───┬────┘  └────────┬─────────┘
         │              │            │                 │
         └──────────────┼────────────┘                 │
                        │                              │
                   ┌────▼─────────────┐                │
                   │  ContextBuilder   │                │
                   │  .build_messages()│                │
                   └────┬─────────────┘                │
                        │                              │
              ┌─────────▼──────────────────────────────▼──┐
              │              Store                         │
              │  .load(sk) → ctx                           │
              │  .commit(sk, trace)                        │
              │  .handle_command(req) → dict | None        │
              └─────────────────┬─────────────────────────┘
                                │
                                │  deps.load / deps.commit / deps.handle_command
                                │
         ┌──────────────────────▼──────────────────────────┐
         │                  run_turn(req, deps)            │
         │                  THE KERNEL                      │
         └──────────────────────┬──────────────────────────┘
                                │
                         ┌──────▼──────┐
                         │  TurnRunner │
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                         │  MessageBus │
                         └─────────────┘
```

---

## 8. Invariants

1. **One turn function.** All triggers (manual, cron, heartbeat, subagent, system, api) produce `req` and call `run_turn`. No second entry point into the LLM loop.

2. **One commit path.** All traces persist through `deps.commit`. No tool or adapter writes directly to session history.

3. **One ingress shape.** Every external event becomes `req = {sk, content, media, meta}`. The kernel has one input type.

4. **Kernel ignores `meta`.** Channel, chat_id, trigger type, message_id — all opaque. The kernel passes `meta` through to tools and result, never inspects it.

5. **Tools are stateless per-call.** Context flows via `req` and `set_context` (called once before the turn). No mutable shared state between tool calls within a turn. No tool reads global variables.

6. **Runner owns lifecycle.** Task cancellation, MCP connection management, processing lock, `/stop` handling — all runner concerns, not kernel concerns.

7. **Side-entries re-enter through the bus.** Cron fires → builds `InboundMessage` → `bus.publish_inbound`. Heartbeat executes → calls `run_turn` directly (or submits to runner). Subagent result → `bus.publish_inbound(system_msg)`. No backdoor paths.

8. **Config is read-once at bootstrap.** After `bootstrap()`, subsystems hold their computed values. No runtime config reloading (except cron store mtime watch and lazy MCP connection).

9. **Templates never overwrite.** `sync_workspace_templates` only creates missing files. User customizations to AGENTS.md, SOUL.md, etc. are preserved across upgrades.

10. **Memory consolidation is non-blocking.** Background task with per-session lock. The agent loop never waits for consolidation to complete before responding.

---

## 9. What Lives Where (File Mapping)

```
nanobot/
  agent/
    loop.py         → TurnRunner + run_turn (or runner wrapping kernel)
    context.py      → ContextBuilder (deps.build)
    memory.py       → MemoryStore (inside deps.load/commit)
    skills.py       → SkillsLoader (inside deps.load via ContextBuilder)
    subagent.py     → SubagentManager (inside deps.exec via SpawnTool)
    tools/
      base.py       → Tool ABC
      registry.py   → ToolRegistry (deps.schemas, deps.exec)
      filesystem.py → read_file, write_file, edit_file, list_dir
      shell.py      → exec
      web.py        → web_search, web_fetch
      message.py    → message (routing-aware tool)
      spawn.py      → spawn (delegates to SubagentManager)
      cron.py       → cron (add/list/remove scheduled jobs)
      mcp.py        → MCPToolWrapper + connect_mcp_servers
  bus/
    events.py       → InboundMessage, OutboundMessage
    queue.py        → MessageBus
  config/
    schema.py       → Config + all nested Pydantic models
    loader.py       → load_config, save_config, config path management
    paths.py        → Runtime path helpers
  cron/
    types.py        → CronJob, CronSchedule, CronPayload, CronJobState, CronStore
    service.py      → CronService (timer loop, job execution, JSON persistence)
  heartbeat/
    service.py      → HeartbeatService (two-phase: decide via tool call, then execute)
  templates/        → Bootstrap markdown files (synced to workspace on first run)
  utils/
    helpers.py      → ensure_dir, detect_image_mime, split_message, sync_workspace_templates
```

---

## 10. Comparison: Kernel vs BASE vs Full Implementation

| Aspect | Kernel (CORE-FINAL) | BASE (this document) | Full implementation |
|--------|---------------------|----------------------|---------------------|
| Entry point | `run_turn(req, deps)` | `run_turn(req, deps)` | `AgentLoop._process_message` |
| Classes | 0 | 0 in kernel; contracts describe interfaces | AgentLoop, ContextBuilder, MemoryStore, SkillsLoader, SubagentManager, ToolRegistry, Tool, ... |
| LoC (kernel) | ~20 | ~20 | ~80 (`_run_agent_loop`) |
| LoC (total) | ~20 | ~20 kernel + contract specs | ~2,500 across all subsystems |
| Config | part of deps | full Pydantic schema + loader + paths | same |
| Memory | `deps.commit` (opaque) | MemoryStore with consolidation protocol | same |
| Tools | `deps.exec` + `deps.schemas` | Tool ABC + ToolRegistry + 10 built-in tools + MCP | same |
| Scheduling | "caller of run_turn" | CronService + HeartbeatService with full type system | same |
| Subagents | "spawn tool starts background run_turn" | SubagentManager with lifecycle tracking, result announcement via bus | same |

---

## 11. What the BASE Trades Away vs Gains

| Trade-off | Rationale |
|-----------|-----------|
| Store is the thickest adapter | It wraps 4 subsystems (sessions, memory, context, skills) behind 3 callables. Complex internally, simple externally. |
| No formal Protocol/ABC for deps | At 9 plugs, duck typing is sufficient. Adding ABCs would add ceremony without safety. |
| Config is Pydantic, not plain dicts | Config has 50+ fields across nested models. Pydantic validation prevents silent misconfiguration. Worth the dependency. |
| MCP is lazy-loaded | MCP connections are expensive and may fail. Lazy initialization on first message avoids blocking startup. |
| `/stop` handled outside kernel | `/stop` needs to cancel `asyncio.Task` objects that the kernel doesn't own. Runner intercepts it before `run_turn`. |
| Consolidation is async fire-and-forget | Blocking the response to consolidate memory would add user-visible latency. Background consolidation trades strict ordering for responsiveness. |

---

## One-Sentence Summary

The BASE defines nine deps contracts plus five adapter boundaries (bus→req, result→bus, cron→bus, heartbeat→run_turn, subagent→bus) that let the 20-line `run_turn` kernel drive 2,500 lines of real infrastructure across sessions, memory, tools, scheduling, skills, and multi-channel communication.
