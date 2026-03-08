# Nanobot BASE Design — cursor

## Stored Prompt

```
read CORE-FINAL-{model-name-and-version}.md;
try to DESIGN the BASE for all the functionalities from following nanobot subdirectories: agent, bus, config, cron, heartbeat, skills, templates, utls;
store the results in BASE-{model-name-and-version}.md;
also - store this promp in the resulting file
```

---

## Design Thesis

The BASE layer sits between the lean kernel (from CORE-FINAL) and the full nanobot implementation. It defines the minimal contracts and structures needed to support all functionalities in agent, bus, config, cron, heartbeat, skills, templates, and utils — without re-implementing them.

**One kernel (`run_turn`), one BASE shape, many adapters.**

---

## Source Documents

- CORE-FINAL-gpt-5.4.md
- CORE-FINAL-claude-4.6-opus.md
- CORE-FINAL-claude-4.6-sonnet-medium-thinking.md
- CORE-FINAL-cursor.md
- CORE-FINAL-gpt-5.3-codex.md
- nanobot/agent/, bus/, config/, cron/, heartbeat/, skills (agent/skills), templates/, utils/

---

## Subdirectory → BASE Mapping

| Subdirectory | Responsibility | BASE Contract |
|--------------|----------------|---------------|
| **agent** | Loop, context, memory, subagent, tools | `run_turn`, Store, Tools, ContextBuilder |
| **bus** | Message queue, events | `InboundMessage` → `req`, `OutboundMessage` ← result |
| **config** | Schema, loader, paths | `Config`, `load_config`, path helpers |
| **cron** | Scheduled jobs | `CronService`, `CronJob` → `TurnRequest` |
| **heartbeat** | Periodic task check | `HeartbeatService` → `TurnRequest` |
| **skills** | Agent capabilities (SKILL.md) | `SkillsLoader` → context/bootstrap |
| **templates** | Workspace bootstrap files | `sync_workspace_templates`, template paths |
| **utils** | Helpers | `ensure_dir`, `detect_image_mime`, `split_message`, etc. |

---

## 1. Agent BASE

### 1.1 Canonical Request (from bus)

```python
req = {
    "sk":       str,   # session_key — from InboundMessage.session_key
    "content":  str,   # message text
    "media":    list,  # optional attachments (paths or URLs)
    "meta":     dict,  # opaque: channel, chat_id, trigger, message_id, ...
}
```

Triggers: `manual`, `cron`, `heartbeat`, `subagent`, `api`, `system`.

### 1.2 Canonical Result

```python
result = {
    "emit":    bool,
    "content": str,
    "sk":      str,
    "meta":    dict,   # channel, chat_id for routing
    "trace":   list,   # optional, for debugging
}
```

### 1.3 Store Contract

```python
class Store:
    def handle(self, req: dict) -> dict | None:
        """Command short-circuit (/new, /help, /stop). Returns result or None."""
        ...

    async def load(self, sk: str) -> dict:
        """Load context: history, memory, identity, bootstrap_docs, max_iter."""
        ...

    async def commit(self, sk: str, trace: list) -> None:
        """Persist trace, sanitize, roll memory if threshold exceeded."""
        ...

    def archive(self, sk: str) -> None:
        """Archive and clear session (/new support)."""
        ...
```

### 1.4 Tools Contract

```python
class Tools:
    def schemas(self, req: dict) -> list[dict]:
        """Policy-filtered tool schemas (OpenAI format)."""
        ...

    async def exec(self, tc: dict, req: dict) -> str:
        """Execute tool call. Context flows via req (sk, meta)."""
        ...
```

### 1.5 Context Builder

```python
def build_messages(ctx: dict, req: dict, trace: list | None = None) -> list[dict]:
    """Assemble LLM message list: system + history + user + optional trace."""
    ...
```

### 1.6 Agent Loop (kernel)

```python
async def run_turn(req: dict, deps) -> dict:
    """The entire agentic kernel. ~20 lines."""
    early = deps.store.handle(req)
    if early is not None:
        return early

    ctx = await deps.store.load(req["sk"])
    msgs = deps.build(ctx, req)
    trace = []

    for _ in range(ctx.get("max_iter", 25)):
        llm = await deps.llm(msgs, deps.tools.schemas(req))
        trace.append(asst(llm))
        if not llm["tc"]:
            break
        for tc in llm["tc"]:
            trace.append(tool_result(tc, await deps.tools.exec(tc, req)))
        msgs = deps.build(ctx, req) + trace
        await deps.on_progress(llm, trace)

    await deps.store.commit(req["sk"], trace)
    return emit(req, llm.get("text") or "[no response]")
```

### 1.7 Memory

- **MEMORY.md**: Long-term facts (read by `load`, written by `commit` consolidation).
- **HISTORY.md**: Grep-searchable log (append-only, from consolidation).
- Consolidation: threshold-based, inside `Store.commit()`.

### 1.8 Subagent

- Spawn tool → background `run_turn` with reduced tools (no message, no spawn).
- Result re-enters via bus as `InboundMessage(channel="system", ...)`.

---

## 2. Bus BASE

### 2.1 Events

```python
@dataclass
class InboundMessage:
    channel: str
    sender_id: str
    chat_id: str
    content: str
    timestamp: datetime = ...
    media: list[str] = ...
    metadata: dict = ...
    session_key_override: str | None = None

    @property
    def session_key(self) -> str:
        return self.session_key_override or f"{channel}:{chat_id}"

@dataclass
class OutboundMessage:
    channel: str
    chat_id: str
    content: str
    reply_to: str | None = None
    media: list = ...
    metadata: dict = ...
```

### 2.2 Message Bus

```python
class MessageBus:
    inbound: asyncio.Queue[InboundMessage]
    outbound: asyncio.Queue[OutboundMessage]

    async def publish_inbound(self, msg: InboundMessage) -> None: ...
    async def consume_inbound(self) -> InboundMessage: ...
    async def publish_outbound(self, msg: OutboundMessage) -> None: ...
    async def consume_outbound(self) -> OutboundMessage: ...
```

### 2.3 Adapter: InboundMessage → req

```python
def make_request(msg: InboundMessage, trigger: str = "manual") -> dict:
    return {
        "sk": msg.session_key,
        "content": msg.content,
        "media": msg.media or [],
        "meta": {
            "channel": msg.channel,
            "chat_id": msg.chat_id,
            "trigger": trigger,
            **(msg.metadata or {}),
        },
    }
```

### 2.4 Adapter: result → OutboundMessage

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

---

## 3. Config BASE

### 3.1 Schema (Pydantic)

- **Config**: Root (agents, channels, providers, gateway, tools).
- **AgentDefaults**: workspace, model, provider, max_tokens, temperature, max_tool_iterations, memory_window, reasoning_effort.
- **ChannelsConfig**: send_progress, send_tool_hints, per-channel configs (telegram, discord, slack, …).
- **ProvidersConfig**: per-provider (api_key, api_base, extra_headers).
- **GatewayConfig**: host, port, heartbeat.
- **ToolsConfig**: web, exec, restrict_to_workspace, mcp_servers.
- **HeartbeatConfig**: enabled, interval_s.

### 3.2 Loader

```python
def load_config(config_path: Path | None = None) -> Config: ...
def save_config(config: Config, config_path: Path | None = None) -> None: ...
def get_config_path() -> Path: ...
def set_config_path(path: Path) -> None: ...
```

### 3.3 Paths

```python
def get_data_dir() -> Path: ...
def get_runtime_subdir(name: str) -> Path: ...
def get_media_dir(channel: str | None = None) -> Path: ...
def get_cron_dir() -> Path: ...
def get_logs_dir() -> Path: ...
def get_workspace_path(workspace: str | None = None) -> Path: ...
```

---

## 4. Cron BASE

### 4.1 Types

```python
@dataclass
class CronSchedule:
    kind: Literal["at", "every", "cron"]
    at_ms: int | None = None
    every_ms: int | None = None
    expr: str | None = None
    tz: str | None = None

@dataclass
class CronPayload:
    kind: Literal["system_event", "agent_turn"] = "agent_turn"
    message: str = ""
    deliver: bool = False
    channel: str | None = None
    to: str | None = None

@dataclass
class CronJob:
    id: str
    name: str
    enabled: bool = True
    schedule: CronSchedule
    payload: CronPayload
    state: CronJobState
    ...
```

### 4.2 Service

```python
class CronService:
    def __init__(self, store_path: Path, on_job: Callable[[CronJob], Coroutine[..., str | None]] | None = None): ...

    async def start(self) -> None: ...
    def stop(self) -> None: ...
    def list_jobs(self, include_disabled: bool = False) -> list[CronJob]: ...
    def add_job(self, name, schedule, message, deliver=False, channel=None, to=None, ...) -> CronJob: ...
    def remove_job(self, job_id: str) -> bool: ...
    async def run_job(self, job_id: str, force: bool = False) -> bool: ...
```

### 4.3 Integration

- `on_job(job)` receives `CronJob`; for `payload.kind == "agent_turn"`, build `req` and call `run_turn` or submit to runner.
- If `deliver`, route result to `channel`/`to`.

---

## 5. Heartbeat BASE

### 5.1 Service

```python
class HeartbeatService:
    def __init__(
        self,
        workspace: Path,
        provider: LLMProvider,
        model: str,
        on_execute: Callable[[str], Coroutine[..., str]] | None = None,
        on_notify: Callable[[str], Coroutine[..., None]] | None = None,
        interval_s: int = 30 * 60,
        enabled: bool = True,
    ): ...

    async def start(self) -> None: ...
    def stop(self) -> None: ...
    async def trigger_now(self) -> str | None: ...
```

### 5.2 Flow

1. Read `workspace/HEARTBEAT.md`.
2. Phase 1: LLM decides `skip` or `run` via virtual tool call.
3. Phase 2: If `run`, `on_execute(tasks)` runs full agent loop; `on_notify(response)` delivers result.

### 5.3 Integration

- `on_execute` = call `run_turn` (or submit to runner) with `req` built from tasks.
- `on_notify` = publish `OutboundMessage` to configured channel.

---

## 6. Skills BASE

### 6.1 Loader

```python
class SkillsLoader:
    def __init__(self, workspace: Path, builtin_skills_dir: Path | None = None): ...

    def list_skills(self, filter_unavailable: bool = True) -> list[dict]: ...
    def load_skill(self, name: str) -> str | None: ...
    def load_skills_for_context(self, skill_names: list[str]) -> str: ...
    def build_skills_summary(self) -> str: ...
    def get_always_skills(self) -> list[str]: ...
    def get_skill_metadata(self, name: str) -> dict | None: ...
```

### 6.2 Integration

- `ContextBuilder` uses `SkillsLoader` for:
  - `get_always_skills()` → inject into system prompt.
  - `build_skills_summary()` → inject summary; agent reads full SKILL.md via `read_file`.

---

## 7. Templates BASE

### 7.1 Files

| File | Purpose |
|------|---------|
| AGENTS.md | Agent instructions (scheduled reminders, heartbeat tasks) |
| SOUL.md | Identity/personality |
| USER.md | User context |
| TOOLS.md | Tool usage guidance |
| HEARTBEAT.md | Heartbeat task list template |
| memory/MEMORY.md | Long-term memory template |
| memory/HISTORY.md | Empty, append-only log |

### 7.2 Sync

```python
def sync_workspace_templates(workspace: Path, silent: bool = False) -> list[str]:
    """Create missing template files in workspace. Returns list of created paths."""
    ...
```

---

## 8. Utils BASE

```python
def ensure_dir(path: Path) -> Path: ...
def detect_image_mime(data: bytes) -> str | None: ...
def timestamp() -> str: ...
def safe_filename(name: str) -> str: ...
def split_message(content: str, max_len: int = 2000) -> list[str]: ...
def sync_workspace_templates(workspace: Path, silent: bool = False) -> list[str]: ...
```

---

## 9. Runner (Adapter, Not Kernel)

```python
class TurnRunner:
    def __init__(self, deps, on_emit): ...

    def submit(self, req: dict) -> None: ...
    async def loop(self) -> None: ...
```

- Queue + per-session cancellation.
- `on_emit(result)` routes to channels.

---

## 10. Deps Wiring (Bootstrap)

```python
deps = SimpleNamespace(
    store       = store,      # Store(history, memory, summarizer, policy)
    llm         = llm_fn,     # (msgs, schemas) -> {text, tc}
    tools       = tools,      # Tools(registry, policy_fn)
    build       = build_messages,
    on_progress = on_progress_fn,
)
```

---

## 11. File Layout (BASE-Aligned)

```text
nanobot/
  kernel.py       # run_turn, asst, tool_result, emit
  store.py        # Store, _sanitize
  tools.py        # Tools, ToolRegistry, Tool base
  runner.py       # TurnRunner
  bus/
    events.py     # InboundMessage, OutboundMessage
    queue.py      # MessageBus
  config/
    schema.py     # Config, channel/provider/tool configs
    loader.py     # load_config, save_config
    paths.py      # get_data_dir, get_cron_dir, ...
  cron/
    types.py      # CronJob, CronSchedule, CronPayload
    service.py    # CronService
  heartbeat/
    service.py    # HeartbeatService
  agent/
    context.py    # ContextBuilder (identity, bootstrap, memory, skills)
    memory.py    # MemoryStore (MEMORY.md, HISTORY.md, consolidate)
    skills.py    # SkillsLoader
    subagent.py  # SubagentManager
    tools/       # read_file, write_file, exec, web_search, message, spawn, cron, mcp
  templates/     # AGENTS.md, SOUL.md, USER.md, TOOLS.md, HEARTBEAT.md, memory/
  utils/
    helpers.py   # ensure_dir, detect_image_mime, split_message, sync_workspace_templates
```

---

## 12. Invariants

1. **One turn function.** All triggers (manual, cron, heartbeat, subagent, system) call `run_turn`.
2. **One commit path.** All traces persist through `deps.store.commit`.
3. **One ingress shape.** Every event becomes `req` with `sk`, `content`, `media`, `meta`.
4. **Kernel ignores `meta`.** Trigger, channel, chat_id — opaque to kernel.
5. **Tools are stateless.** Context flows via `req`; no mutable shared state.
6. **Runner owns cancellation.** One active task per `sk`; new request cancels prior.
7. **Cron/heartbeat/subagent re-enter through queue.** No background shortcut.
8. **Config drives all adapters.** No hardcoded paths or keys in BASE.

---

## 13. Plug-In Map (CORE-FINAL → BASE)

| CORE-FINAL plug | BASE implementation |
|-----------------|----------------------|
| `deps.store` | Store + MemoryStore + SessionManager + ContextBuilder |
| `deps.llm` | Provider.chat (OpenAI/Anthropic/…) |
| `deps.tools` | ToolRegistry + Tool implementations |
| `deps.build` | ContextBuilder.build_messages |
| `deps.on_progress` | Optional streaming to bus |
| `make_request` | InboundMessage → req |
| `on_emit` | result → OutboundMessage → bus |
| Cron | CronService.on_job → req → run_turn |
| Heartbeat | HeartbeatService.on_execute → run_turn |
| Subagent | SpawnTool → SubagentManager → bus.publish_inbound |

---

## One-Sentence Summary

The BASE layer defines the minimal contracts (Store, Tools, ContextBuilder, bus events, config, cron, heartbeat, skills, templates, utils) that let the lean kernel (`run_turn`) support the full nanobot feature set without re-implementing it.
