# Nanobot Core Bricks

This file identifies small, near self-contained functionality units ("bricks") that can be detached from the core and plugged back in through dependency injection.

Scope: derived from `CORE-IDEA.md`, `CORE-LEAN.md`, `CORE-LEANEST.md`, `CORE-LEANEST-V2.md`, and `CORE-MINIMAL.md`.

## Brick Selection Rules

A candidate brick should satisfy most of these:

- Single primary responsibility.
- Narrow contract (few input/output fields).
- Minimal hidden state.
- Replaceable implementation with same behavior contract.
- Usable by `TurnEngine.run(request)` without direct file/system coupling.

## Injection Spine (Where Bricks Plug In)

Recommended root composition:

- `engine = TurnEngine(store, provider, tools, command_handler, progress_sink, policies)`
- `runner = TurnRunner(engine, emitter, cancel_policy)`
- `io_adapter -> TurnRequest -> runner.submit()`

This keeps one turn path while allowing individual bricks to evolve independently.

## Brick Catalog

### 1) Turn Request Normalizer

- **Responsibility:** convert inbound events (chat/cron/heartbeat/subagent/api) into canonical `TurnRequest`.
- **Contract in:** external adapter event.
- **Contract out:** `{session_key, trigger, content, media, metadata}`.
- **Inject as:** `request_factory`.
- **Swap value:** add new transports without touching engine.

### 2) Turn Runner Queue

- **Responsibility:** queue requests and dispatch workers.
- **Contract in:** `submit(request)`.
- **Contract out:** invokes engine and optional outbound emission.
- **Inject as:** `runner` implementation.
- **Swap value:** replace in-memory queue with durable queue later.

### 3) Session Cancellation Policy

- **Responsibility:** enforce "new request cancels prior in-flight request for same `session_key`".
- **Contract in:** `before_run(request)`.
- **Contract out:** cancellation token/scope for current run.
- **Inject as:** `cancel_policy`.
- **Swap value:** strict single-flight vs relaxed coalescing per channel.

### 4) Command Handler

- **Responsibility:** inline handling of slash commands (`/help`, `/new`, `/stop`, etc.).
- **Contract in:** `try_handle(request, store)`.
- **Contract out:** `None` (not a command) or `{trace, result}` short-circuit.
- **Inject as:** `command_handler`.
- **Swap value:** command set extensibility without changing LLM loop.

### 5) Conversation Store

- **Responsibility:** session history load/append and memory rollover trigger.
- **Contract in:** `load_context(session_key, window)`, `commit_turn_and_roll_memory_if_needed(session_key, trace)`.
- **Contract out:** context bundle and durable write side effects.
- **Inject as:** `store`.
- **Swap value:** JSONL+markdown now, SQLite later, same engine API.

### 6) Memory Consolidator

- **Responsibility:** summarize old history and update long-term memory artifacts.
- **Contract in:** unconsolidated history slice + memory policy.
- **Contract out:** memory update payload + checkpoint advancement.
- **Inject as:** `memory_consolidator` used by store.
- **Swap value:** provider-based summarizer vs deterministic rules.

### 7) Prompt/Context Builder

- **Responsibility:** compose model-visible messages from context + request.
- **Contract in:** `build_messages(context, request)` where context already includes identity, bootstrap docs, memory, and history.
- **Contract out:** ordered LLM messages.
- **Inject as:** `context_builder`.
- **Swap value:** lightweight prompts for latency, richer prompts for quality.

### 8) Provider Chat Adapter

- **Responsibility:** normalize provider API calls and responses.
- **Contract in:** `chat(messages, tool_schemas, model_settings)`.
- **Contract out:** `{assistant_text, tool_calls, raw}`.
- **Inject as:** `provider`.
- **Swap value:** OpenAI/Anthropic/local models without loop changes.

### 9) Tool Schema Source

- **Responsibility:** return provider-compatible tool schemas.
- **Contract in:** none (or current policy).
- **Contract out:** list of JSON-schema-like function specs.
- **Inject as:** `tools.schemas`.
- **Swap value:** static registry vs dynamic policy-filtered exposure.

### 10) Tool Executor

- **Responsibility:** execute a tool call with explicit call context.
- **Contract in:** `execute(call, ctx)` where `ctx` is minimal (`session_key`, `permissions`, optional routing).
- **Contract out:** tool result payload or recoverable error text.
- **Inject as:** `tools.execute`.
- **Swap value:** local callables, MCP, sandboxed remote execution.

### 11) Progress Sink

- **Responsibility:** optional progress emission (text and/or tool hints) during loop.
- **Contract in:** progress event.
- **Contract out:** side-effect only (bus emit, stream callback, no-op).
- **Inject as:** `progress_sink` or `on_progress`.
- **Swap value:** silent batch mode vs interactive streaming UX.

### 12) Outbound Emitter

- **Responsibility:** send final outbound response to channel adapter.
- **Contract in:** normalized outbound `{channel, chat_id, content, metadata}`.
- **Contract out:** delivery side effect.
- **Inject as:** `emitter`.
- **Swap value:** different channel stacks with zero engine changes.

### 13) Policy Set (Data Bricks)

- **Responsibility:** centralize behavior decisions as data, not branch-heavy code.
- **Contract in:** request/session state.
- **Contract out:** selected values for tool, memory, and progress behavior.
- **Inject as:** `policies = {tool_policy, memory_policy, progress_policy}`.
- **Swap value:** per-tenant or per-trigger runtime behavior without branching sprawl.

### 14) Background Trigger Adapters

- **Responsibility:** cron/heartbeat/subagent completion conversion into normal turn inputs.
- **Contract in:** scheduler/background events.
- **Contract out:** canonical `TurnRequest` submitted to runner.
- **Inject as:** adapter callbacks.
- **Swap value:** autonomous features stay separate from turn core.

## Suggested Minimal DI Interfaces

```python
class TurnEngineDeps(Protocol):
    store: ConversationStore
    provider: Provider
    tools: ToolGateway
    context_builder: ContextBuilder
    command_handler: CommandHandler
    progress_sink: ProgressSink
    policies: Policies
```

```python
class RunnerDeps(Protocol):
    engine: TurnEngine
    emitter: OutboundEmitter
    cancel_policy: CancelPolicy
```

The key idea is to inject narrow behavior contracts, not full subsystems.

## Brick Boundaries To Keep Hard

- Engine does not know file formats.
  The engine should consume logical context and emit logical trace/result objects only. If it starts reading JSONL or writing markdown directly, storage migration will require engine changes and break separation. Keep file parsing/serialization inside store repositories.
- Store does not know channel transport.
  Store APIs should only care about session keys, messages, and memory checkpoints. If store learns about channel IDs, chat routing, or outbound delivery rules, persistence becomes coupled to UI/transport decisions. This makes testing and backend replacement much harder.
- Tools do not mutate hidden shared context.
  Tool execution must receive explicit `ctx` for each call and return explicit output. Hidden mutable context introduces order-dependent behavior and concurrency bugs that are difficult to reproduce. Stateless call contracts keep tools deterministic and safer in parallel flows.
- Runner does not know prompt assembly.
  Runner is responsible for scheduling, cancellation, and dispatch only. Prompt construction belongs to engine/context-builder, where model-facing behavior is centralized. Mixing these concerns turns queue logic into application logic and creates duplicate prompt paths.
- Background services do not bypass `TurnRequest` path.
  Cron, heartbeat, and subagent completions should be normalized into `TurnRequest` and sent through the same runner/engine flow. Direct shortcuts around this path create invisible behavior differences in memory, tools, and policies. One ingress path keeps invariants testable and predictable.

These boundaries preserve "one turn path" while keeping parts replaceable.

## Extraction Order (Low Risk)

1. `CommandHandler` (easy early isolation).
2. `ConversationStore` facade around existing session/memory logic.
3. `ToolExecutor` with explicit `ctx` (remove mutable tool context).
4. `ProgressSink` and `OutboundEmitter` separation.
5. `CancellationPolicy` moved into runner.
6. `Provider`/`ContextBuilder` normalization.

After these six, most remaining features become adapters, not core complexity.

## Pseudocode Proposals Per Brick

Notation:

- Prefer `def ...` (function brick).
- Use `class ...` only when persistent state/lifecycle is required.

### 1) Turn Request Normalizer (function)

```python
def make_turn_request(event: dict) -> dict:
    # adapter-owned translation into canonical shape
    return {
        "session_key": resolve_session_key(event),
        "trigger": resolve_trigger(event),  # manual|cron|heartbeat|subagent|api
        "content": event.get("content", ""),
        "media": event.get("media", []),
        "metadata": event.get("metadata", {}),
    }
```

### 2) Turn Runner Queue (object required: owns queue/workers)

```python
class TurnRunner:
    def __init__(self, engine, emitter, cancel_policy, queue_impl):
        self.engine = engine
        self.emitter = emitter
        self.cancel_policy = cancel_policy
        self.q = queue_impl

    def submit(self, request: dict) -> None:
        self.q.put(request)

    async def worker_loop(self) -> None:
        while True:
            request = await self.q.get()
            scope = self.cancel_policy.before_run(request)
            try:
                async with scope:
                    result = await self.engine.run(request)
                if result.get("emit"):
                    await self.emitter(result["outbound"])
            finally:
                self.cancel_policy.done(request)
```

### 3) Session Cancellation Policy (object required: owns active scopes)

```python
class SingleFlightCancelPolicy:
    def __init__(self):
        self.active_by_session = {}  # session_key -> CancellationScope

    def before_run(self, request: dict):
        key = request["session_key"]
        prev = self.active_by_session.get(key)
        if prev:
            prev.cancel()
        scope = CancellationScope()
        self.active_by_session[key] = scope
        return scope

    def done(self, request: dict):
        self.active_by_session.pop(request["session_key"], None)
```

### 4) Command Handler (function)

```python
def try_handle_command(request: dict, store) -> dict | None:
    text = request.get("content", "").strip()
    if not text.startswith("/"):
        return None

    key = request["session_key"]

    if text == "/help":
        trace = [assistant("Available commands: /help /new /stop")]
        return {"trace": trace, "result": {"emit": True, "content": trace[-1]["content"]}}

    if text == "/new":
        store.archive_and_clear(key)
        trace = [assistant("Started a new conversation window.")]
        return {"trace": trace, "result": {"emit": True, "content": trace[-1]["content"]}}

    return {"trace": [assistant("Unknown command.")], "result": {"emit": True, "content": "Unknown command."}}
```

### 5) Conversation Store (object required: owns persistence backend)

```python
class ConversationStore:
    def __init__(self, history_repo, memory_repo, memory_consolidator, sanitize_trace, memory_policy):
        self.history_repo = history_repo
        self.memory_repo = memory_repo
        self.memory_consolidator = memory_consolidator
        self.sanitize_trace = sanitize_trace
        self.memory_policy = memory_policy

    def load_context(self, session_key: str, window: int) -> dict:
        history = self.history_repo.load_recent(session_key, window)
        memory = self.memory_repo.load_memory(session_key)
        return {"history": history, "memory": memory}

    async def commit_turn_and_roll_memory_if_needed(self, session_key: str, trace: list[dict]) -> None:
        clean = self.sanitize_trace(trace)
        self.history_repo.append_turn(session_key, clean)
        if self.history_repo.unconsolidated_count(session_key) >= self.memory_policy["summarize_threshold"]:
            await self.memory_consolidator(session_key)

    def archive_and_clear(self, session_key: str) -> None:
        self.history_repo.archive(session_key)
        self.history_repo.clear(session_key)
```

Note: `sanitize_trace` is a pre-commit transform (truncate oversized tool outputs, strip injected runtime-context tags, replace inline `data:image` payloads with markers). It is a function brick injected into store rather than a separate top-level brick, because it has no consumers outside the commit path.

### 6) Memory Consolidator (function or object; function shown)

```python
async def consolidate_memory(session_key: str, history_repo, memory_repo, summarizer, policy) -> None:
    old_messages = history_repo.read_unconsolidated_slice(session_key, keep_recent=policy["keep_recent"])
    if not old_messages:
        return
    memory_update = await summarizer.summarize(old_messages, memory_repo.load_memory(session_key))
    memory_repo.write_memory(session_key, memory_update)
    history_repo.mark_consolidated(session_key, old_messages)
```

### 6b) Trace Sanitizer (function, sub-brick of store)

```python
def sanitize_trace(trace: list[dict], max_tool_output: int = 20_000) -> list[dict]:
    clean = []
    for msg in trace:
        msg = dict(msg)
        if msg.get("role") == "tool" and len(msg.get("content", "")) > max_tool_output:
            msg["content"] = msg["content"][:max_tool_output] + "\n...[truncated]"
        if msg.get("role") == "user":
            msg["content"] = strip_runtime_context_tags(msg.get("content", ""))
        msg["content"] = replace_inline_images(msg.get("content", ""))
        clean.append(msg)
    return clean
```

### 7) Prompt/Context Builder (function)

```python
def build_messages(context: dict, request: dict) -> list[dict]:
    system_parts = [
        context.get("identity", ""),
        context.get("bootstrap_docs", ""),
        context.get("memory", ""),
    ]
    messages = [{"role": "system", "content": "\n\n".join([p for p in system_parts if p])}]
    messages.extend(context.get("history", []))
    user_msg = {"role": "user", "content": request.get("content", "")}
    if request.get("media"):
        user_msg["media"] = request["media"]
    messages.append(user_msg)
    return messages
```

Note: `context` is expected to already contain runtime metadata (`identity`, `bootstrap_docs`) alongside `history` and `memory`. The store's `load_context` or a pre-build step is responsible for assembling this merged dict, keeping the builder signature at two parameters.

### 8) Provider Chat Adapter (function)

```python
async def provider_chat(messages: list[dict], tool_schemas: list[dict], model_settings: dict, sdk_client) -> dict:
    raw = await sdk_client.chat(messages=messages, tools=tool_schemas, **model_settings)
    return {
        "assistant_text": extract_text(raw),
        "tool_calls": extract_tool_calls(raw),
        "raw": raw,
    }
```

### 9) Tool Schema Source (function)

```python
def tool_schemas(registry: dict, tool_policy: dict, request: dict) -> list[dict]:
    allowed = filter_allowed_tools(registry, tool_policy, request)
    return [tool["schema"] for tool in allowed]
```

### 10) Tool Executor (function)

```python
async def execute_tool_call(call: dict, ctx: dict, registry: dict) -> dict:
    tool = registry.get(call["name"])
    if not tool:
        return {"ok": False, "error": f"unknown tool: {call['name']}"}
    if not is_allowed(call["name"], ctx["permissions"]):
        return {"ok": False, "error": "tool not allowed"}
    try:
        result = await tool["fn"](call.get("args", {}), ctx)
        return {"ok": True, "result": result}
    except Exception as e:
        return {"ok": False, "error": str(e)}
```

### 11) Progress Sink (function)

```python
async def emit_progress(progress_event: dict, mode: str, bus_publish) -> None:
    if mode == "none":
        return
    if mode == "text" and progress_event.get("kind") != "text":
        return
    await bus_publish(progress_event)
```

### 12) Outbound Emitter (function)

```python
async def emit_outbound(outbound: dict, channel_gateway) -> None:
    await channel_gateway.send(
        channel=outbound["channel"],
        chat_id=outbound["chat_id"],
        content=outbound["content"],
        metadata=outbound.get("metadata", {}),
    )
```

### 13) Policy Set (function returning data object)

```python
def resolve_policies(request: dict, tenant_config: dict) -> dict:
    trigger = request["trigger"]
    return {
        "tool_policy": tenant_config["tool_policy_by_trigger"].get(trigger, tenant_config["tool_policy_default"]),
        "memory_policy": tenant_config["memory_policy_by_trigger"].get(trigger, tenant_config["memory_policy_default"]),
        "progress_policy": tenant_config["progress_policy_by_trigger"].get(trigger, tenant_config["progress_policy_default"]),
    }
```

### 14) Background Trigger Adapters (function)

```python
def on_background_event(bg_event: dict, request_factory, runner_submit) -> None:
    # cron/heartbeat/subagent results are normalized into the same turn path
    request = request_factory(bg_event)
    runner_submit(request)
```

## Example Composition (How Bricks Plug Back In)

```python
def bootstrap(config, sdk_client, queue_impl, channel_gateway):
    cancel_policy = SingleFlightCancelPolicy()

    async def provider(messages, schemas, settings):
        return await provider_chat(messages, schemas, settings, sdk_client)

    async def progress_sink(event, mode):
        await emit_progress(event, mode, bus_publish=config["bus_publish"])

    async def emitter(outbound):
        await emit_outbound(outbound, channel_gateway)

    async def memory_consolidator_fn(session_key):
        await consolidate_memory(
            session_key,
            history_repo=config["history_repo"],
            memory_repo=config["memory_repo"],
            summarizer=config["summarizer"],
            policy=config["memory_policy_default"],
        )

    store = ConversationStore(
        history_repo=config["history_repo"],
        memory_repo=config["memory_repo"],
        memory_consolidator=memory_consolidator_fn,
        sanitize_trace=sanitize_trace,
        memory_policy=config["memory_policy_default"],
    )

    engine = TurnEngine(
        store=store,
        provider=provider,
        tools={"schemas": tool_schemas, "execute": execute_tool_call},
        context_builder=build_messages,
        command_handler=try_handle_command,
        progress_sink=progress_sink,
        policy_resolver=resolve_policies,
    )
    return TurnRunner(engine=engine, emitter=emitter, cancel_policy=cancel_policy, queue_impl=queue_impl)
```

