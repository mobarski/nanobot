# Nanobot Core Bricks V2

Minimal set of separable units that can be detached from the core and plugged back in via dependency injection.

Each brick: one purpose, one pseudocode block, one swap note.

## Injection Spine

```text
io / cron / heartbeat / subagent  →  make_request(event)  →  runner.submit(request)
                                                                  ↓
                                                            runner.worker_loop
                                                                  ↓
                                                            engine.run(request)
                                                                  ↓
                               ┌──────────────────────────────────┼──────────────────────┐
                           commands?                        build context             commit
                           short-circuit                    provider.chat loop        + rollover
                                                            tools.execute
```

Three composition roots:

- `engine = TurnEngine(store, provider, tools, handle_command, build_messages, on_progress)`
- `runner = TurnRunner(engine, on_emit, cancel_policy)`
- Each IO adapter just calls `make_request(event)` then `runner.submit(request)`

---

## Bricks

### 1) Request Normalizer

All inbound sources (chat, cron, heartbeat, subagent, API) produce the same dict.

```python
def make_request(event: dict) -> dict:
    return {
        "session_key": resolve_session_key(event),
        "trigger":     resolve_trigger(event),  # manual|cron|heartbeat|subagent|api
        "content":     event.get("content", ""),
        "media":       event.get("media", []),
        "metadata":    event.get("metadata", {}),
    }
```

**Swap:** add transports without touching engine or runner.

### 2) Runner + Cancellation

Object — owns queue, workers, and per-session cancel scopes.

```python
class TurnRunner:
    def __init__(self, engine, on_emit, cancel_policy):
        self.engine = engine
        self.on_emit = on_emit
        self.active = {}  # session_key -> CancellationScope
        self.q = asyncio.Queue()

    def submit(self, request):
        self.q.put_nowait(request)

    async def worker_loop(self):
        while True:
            req = await self.q.get()
            key = req["session_key"]
            prev = self.active.pop(key, None)
            if prev:
                prev.cancel()
            scope = CancellationScope()
            self.active[key] = scope
            try:
                async with scope:
                    result = await self.engine.run(req)
                if result.get("emit"):
                    await self.on_emit(result["outbound"])
            finally:
                self.active.pop(key, None)
```

**Swap:** in-memory queue now, durable queue later. Cancellation policy can be extracted to a separate callable if needed.

### 3) Command Handler

Function — short-circuits before the LLM loop.

```python
def handle_command(request: dict, store) -> dict | None:
    text = (request.get("content") or "").strip()
    if not text.startswith("/"):
        return None
    key = request["session_key"]
    if text == "/new":
        store.archive_and_clear(key)
        return {"emit": True, "content": "New conversation started."}
    if text == "/help":
        return {"emit": True, "content": "Commands: /help /new /stop"}
    return {"emit": True, "content": "Unknown command."}
```

**Swap:** extend command set without touching LLM loop.

### 4) Store

Object — owns history + memory behind one API. Internally delegates to repos and a consolidator.

```python
class Store:
    def __init__(self, history, memory, consolidate, sanitize, threshold):
        self.history = history          # repo: load_recent, append, archive, clear, ...
        self.memory = memory            # repo: load, write
        self.consolidate = consolidate  # async (session_key) -> None
        self.sanitize = sanitize        # (trace) -> trace
        self.threshold = threshold

    def load_context(self, session_key: str, window: int) -> dict:
        return {
            "history": self.history.load_recent(session_key, window),
            "memory":  self.memory.load(session_key),
        }

    async def commit(self, session_key: str, trace: list[dict]) -> None:
        self.history.append(session_key, self.sanitize(trace))
        if self.history.unconsolidated_count(session_key) >= self.threshold:
            await self.consolidate(session_key)

    def archive_and_clear(self, session_key: str) -> None:
        self.history.archive(session_key)
        self.history.clear(session_key)
```

`sanitize` is a pre-commit transform injected as a function (truncate oversized tool output, strip runtime tags, replace inline images).

`consolidate` is an async function that summarizes old history into long-term memory and advances the checkpoint.

```python
async def consolidate(key, history, memory, summarizer, keep_recent):
    old = history.unconsolidated_slice(key, keep_recent=keep_recent)
    if not old:
        return
    update = await summarizer(old, memory.load(key))
    memory.write(key, update)
    history.mark_consolidated(key, old)
```

**Swap:** JSONL+markdown repos now, SQLite later — same Store API.

### 5) Context Builder

Function — assembles the message list the model will see.

```python
def build_messages(context: dict, request: dict) -> list[dict]:
    system = "\n\n".join(p for p in [
        context.get("identity"),
        context.get("bootstrap_docs"),
        context.get("memory"),
    ] if p)
    msgs = [{"role": "system", "content": system}]
    msgs.extend(context.get("history", []))
    user = {"role": "user", "content": request.get("content", "")}
    if request.get("media"):
        user["media"] = request["media"]
    msgs.append(user)
    return msgs
```

`context` must already contain `identity`, `bootstrap_docs`, `memory`, `history` — assembled by `store.load_context` plus a thin runtime-meta step before calling this function.

**Swap:** lightweight prompts for latency, richer prompts for quality.

### 6) Provider

Function — normalize any LLM SDK into one return shape.

```python
async def provider_chat(messages, tool_schemas, settings, sdk):
    raw = await sdk.chat(messages=messages, tools=tool_schemas, **settings)
    return {
        "text":       extract_text(raw),
        "tool_calls": extract_tool_calls(raw),
        "raw":        raw,
    }
```

**Swap:** OpenAI / Anthropic / local — engine sees the same dict.

### 7) Tools

Two functions behind one namespace — schemas + execute. Always injected together.

```python
def get_schemas(registry: dict, policy: dict) -> list[dict]:
    return [t["schema"] for name, t in registry.items() if policy_allows(name, policy)]

async def execute(call: dict, ctx: dict, registry: dict) -> dict:
    tool = registry.get(call["name"])
    if not tool:
        return {"ok": False, "error": f"unknown tool: {call['name']}"}
    if not policy_allows(call["name"], ctx.get("permissions", {})):
        return {"ok": False, "error": "not allowed"}
    try:
        return {"ok": True, "result": await tool["fn"](call.get("args", {}), ctx)}
    except Exception as e:
        return {"ok": False, "error": str(e)}
```

`ctx` is minimal: `{session_key, permissions}`. Add routing fields only for tools that truly need them.

**Swap:** local callables, MCP, sandboxed remote — same two functions.

---

## TurnEngine (The Glue)

Not a brick — it is the single orchestration path that consumes all bricks.

```python
class TurnEngine:
    def __init__(self, store, provider, tools, handle_command, build_messages, on_progress=None):
        self.store = store
        self.provider = provider
        self.tools = tools
        self.handle_command = handle_command
        self.build_messages = build_messages
        self.on_progress = on_progress or (lambda event: None)

    async def run(self, request: dict, policies: dict = None) -> dict:
        cmd = self.handle_command(request, self.store)
        if cmd is not None:
            await self.store.commit(request["session_key"], cmd.get("trace", []))
            return cmd

        ctx = self.store.load_context(request["session_key"], policies["memory"]["window"])
        messages = self.build_messages(ctx, request)
        registry = self.tools["registry"]
        schemas = self.tools["schemas"](registry, policies.get("tool", {}))
        tool_ctx = {"session_key": request["session_key"], "permissions": policies.get("tool", {})}
        trace = []

        for _ in range(policies.get("max_iterations", 25)):
            llm = await self.provider(messages, schemas, policies.get("model", {}))
            trace.append({"role": "assistant", "content": llm["text"], "tool_calls": llm["tool_calls"]})

            if not llm["tool_calls"]:
                break

            for call in llm["tool_calls"]:
                result = await self.tools["execute"](call, tool_ctx, registry)
                trace.append({"role": "tool", "call": call, "content": result})
            messages = self.build_messages(ctx, request) + trace
            await self.on_progress({"text": llm["text"], "tool_calls": llm["tool_calls"]})

        final = llm["text"] or "[no response]"
        await self.store.commit(request["session_key"], trace)
        return {"emit": True, "content": final, "outbound": {
            "channel": request["metadata"].get("channel"),
            "chat_id": request["metadata"].get("chat_id"),
            "content": final,
            "metadata": request.get("metadata", {}),
        }}
```

Everything that is not iteration logic is an injected brick:

| Concern | Brick called |
|---|---|
| Slash commands | `self.handle_command(request, store)` |
| History + memory | `self.store.load_context(...)` / `self.store.commit(...)` |
| Prompt assembly | `self.build_messages(ctx, request)` |
| LLM call | `self.provider(messages, schemas, settings)` |
| Tool filtering | `self.tools["schemas"](registry, policy)` |
| Tool execution | `self.tools["execute"](call, ctx, registry)` |
| Streaming | `self.on_progress(event)` |

---

## Injected Callbacks (Not Full Bricks)

These are `async (data) -> None` signatures passed into runner or engine. They don't carry enough internal logic to be bricks.

| Callback | Injected into | Purpose |
|---|---|---|
| `on_emit(outbound)` | runner | send final response to channel adapter |
| `on_progress(event)` | engine loop | stream partial text / tool hints during iteration |

Both default to no-op if not provided.

## Policies (Config, Not Code)

Policies are plain dicts resolved once per request from config:

```python
policies = {
    "tool":     {"allow": [...], "deny": [...], "timeout": 30},
    "memory":   {"window": 40, "summarize_threshold": 60, "keep_recent": 20},
    "progress": {"mode": "text"},  # none | text | text_and_tool_hints
}
```

Per-trigger or per-tenant overrides are a config concern, not a brick concern.

---

## Hard Boundaries

- **Engine does not know file formats.** Storage serialization stays inside store repos.
- **Store does not know channels.** Only session keys and messages, never transport routing.
- **Tools receive explicit ctx, return explicit output.** No hidden mutable shared state.
- **Runner does not build prompts.** Scheduling and dispatch only.
- **Background services enter through `make_request`.** One ingress path, no silent shortcuts.

---

## Extraction Order

1. `handle_command` — easiest to isolate, no dependencies.
2. `Store` facade — wrap existing session/memory behind `load_context` + `commit`.
3. `execute` with explicit `ctx` — remove mutable tool context.
4. `on_progress` / `on_emit` as callbacks — decouple IO from engine.
5. Cancellation into runner — single owner for cancel scopes.
6. `provider_chat` / `build_messages` — normalize last.

---

## Bootstrap

```python
def bootstrap(config, sdk, channel_gw):
    registry = load_tool_registry(config)

    async def provider(msgs, schemas, settings):
        return await provider_chat(msgs, schemas, settings, sdk)

    async def on_emit(outbound):
        await channel_gw.send(**outbound)

    store = Store(
        history=config["history_repo"],
        memory=config["memory_repo"],
        consolidate=lambda key: consolidate(
            key, config["history_repo"], config["memory_repo"],
            config["summarizer"], config["memory_policy"]["keep_recent"],
        ),
        sanitize=sanitize_trace,
        threshold=config["memory_policy"]["summarize_threshold"],
    )

    engine = TurnEngine(
        store=store,
        provider=provider,
        tools={"schemas": get_schemas, "execute": execute, "registry": registry},
        handle_command=handle_command,
        build_messages=build_messages,
        on_progress=config.get("on_progress"),
    )
    return TurnRunner(engine, on_emit, cancel_policy=None)
```
