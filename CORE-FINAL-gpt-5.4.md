# Nanobot Final Core (GPT-5.4)

This is the leanest kernel I can justify after reading all `CORE-*.md` variants and `CORE-IDEA.md`.

It keeps only the machinery that is necessary for an agent to be:

- stateful
- tool-using
- cancellable
- async-friendly
- extensible enough to plug the larger system back in

## Stored Prompt

```text
read all CORE-*.md and try to DESIGN as lean agentic kernel as possible BUT it must be able to plug in the features from the big system (CORE-IDEA.md);
save the result in CORE-FINAL-{model-name-and-version}.md; also - store this promp in the resulting file
```

## Design Thesis

The minimal viable agentic kernel is:

1. One canonical `TurnRequest`
2. One per-session single-flight worker loop
3. One `run_turn()` orchestration function
4. One `Store` contract for history + memory
5. One `Tools` gateway for schemas + execution

Everything else is a plugin, adapter, or policy.

That is the smallest shape that still supports the big-system features from `CORE-IDEA.md` without creating a second architecture later.

## Kernel Boundary

### Inside the kernel

- request normalization
- queue + cancellation
- command short-circuit
- context load
- LLM/tool iteration loop
- commit path
- optional outbound emit

### Outside the kernel

- channel adapters
- internal bus implementation
- cron scheduler
- heartbeat scheduler
- subagent runtime
- MCP integration
- storage backend details
- provider SDK details
- bootstrap file loading details
- progress UX details

The rule is simple:

If a feature can enter as a `TurnRequest`, leave as an outbound message, or appear as a tool/store/provider adapter, it does not belong inside the kernel.

## Minimal Runtime Shape

Use plain dicts and plain functions first. Add classes only where state ownership is real.

### Canonical request

```python
TurnRequest = {
    "session_key": str,
    "trigger": "manual|cron|heartbeat|subagent|api|system",
    "content": str,
    "media": list,
    "metadata": dict,
}
```

### Canonical result

```python
TurnResult = {
    "emit": bool,
    "content": str,
    "outbound": {
        "channel": str,
        "chat_id": str,
        "content": str,
        "metadata": dict,
    },
}
```

## The Five Required Contracts

### 1. Request factory

Normalizes any inbound event into `TurnRequest`.

```python
def make_request(event: dict) -> dict:
    return {
        "session_key": resolve_session_key(event),
        "trigger": resolve_trigger(event),
        "content": event.get("content", ""),
        "media": event.get("media", []),
        "metadata": event.get("metadata", {}),
    }
```

### 2. Runner

Owns queue and per-session cancellation. This is the only place that should know about in-flight tasks.

```python
async def worker_loop(queue, active, run_turn, emit):
    while True:
        request = await queue.get()
        key = request["session_key"]

        prev = active.pop(key, None)
        if prev:
            prev.cancel()

        task = asyncio.create_task(run_turn(request))
        active[key] = task

        try:
            result = await task
            if result.get("emit"):
                await emit(result["outbound"])
        finally:
            active.pop(key, None)
```

### 3. Turn orchestration

The core logic should fit on one screen.

```python
async def run_turn(request, deps, policy):
    cmd = try_handle_command(request, deps["store"])
    if cmd is not None:
        await deps["store"].commit_turn(request["session_key"], cmd["trace"], policy["memory"])
        return cmd["result"]

    context = deps["store"].load_context(request["session_key"], policy["memory"])
    messages = build_messages(context, request)
    schemas = deps["tools"].schemas(policy["tool"], request)
    trace = []

    for _ in range(policy["loop"]["max_iterations"]):
        llm = await deps["provider"].chat(messages, schemas, policy["model"])
        trace.append({"role": "assistant", "content": llm["text"], "tool_calls": llm["tool_calls"]})

        if not llm["tool_calls"]:
            final = llm["text"] or policy["loop"]["fallback_text"]
            break

        for call in llm["tool_calls"]:
            tool_ctx = {
                "session_key": request["session_key"],
                "trigger": request["trigger"],
                "metadata": request.get("metadata", {}),
                "permissions": policy["tool"],
            }
            result = await deps["tools"].execute(call, tool_ctx)
            trace.append({"role": "tool", "call": call, "content": result})

        if deps.get("on_progress"):
            await deps["on_progress"]({"request": request, "trace": trace[-len(llm["tool_calls"]):]})

        messages = build_messages(context, request, trace)
    else:
        final = policy["loop"]["fallback_text"]

    if not trace or trace[-1].get("role") != "assistant" or trace[-1].get("content") != final:
        trace.append({"role": "assistant", "content": final})

    await deps["store"].commit_turn(request["session_key"], trace, policy["memory"])
    return to_result(request, final)
```

### 4. Store

One store contract must hide session history, memory files, checkpoints, and rollover.

```python
class Store:
    def load_context(self, session_key: str, memory_policy: dict) -> dict: ...
    async def commit_turn(self, session_key: str, trace: list[dict], memory_policy: dict) -> None: ...
    def archive_and_clear(self, session_key: str) -> None: ...
```

Required behavior inside `commit_turn()`:

- sanitize trace
- append history
- trigger summarize/rollover when threshold is exceeded
- advance memory checkpoint

This keeps memory orchestration out of the turn loop.

### 5. Tools gateway

One surface for all capabilities, whether local, sandboxed, or MCP-backed.

```python
class Tools:
    def schemas(self, tool_policy: dict, request: dict) -> list[dict]: ...
    async def execute(self, call: dict, ctx: dict) -> dict: ...
```

No hidden mutable tool context.

## One Tiny Helper That Is Worth Keeping

`build_messages()` is not a separate subsystem. It is a helper function.

```python
def build_messages(context: dict, request: dict, trace: list[dict] | None = None) -> list[dict]:
    system = "\n\n".join(x for x in [
        context.get("identity", ""),
        context.get("bootstrap", ""),
        context.get("memory", ""),
    ] if x)

    messages = [{"role": "system", "content": system}]
    messages.extend(context.get("history", []))
    messages.append({"role": "user", "content": request.get("content", "")})
    if trace:
        messages.extend(trace)
    return messages
```

This helper is worth keeping because prompt assembly is genuinely distinct from storage and loop execution, but it is still small enough not to deserve its own heavy abstraction.

## Plug-In Map For Big-System Features

This is the critical requirement from `CORE-IDEA.md`: the kernel must stay lean but accept the larger system cleanly.

| Big-system feature | Where it plugs in | Why it does not need to be in the kernel |
|---|---|---|
| Message bus / transport | adapters call `make_request()` and `queue.put()`; outbound goes through `emit()` | ingress/egress only |
| Session persistence | `Store.load_context()` / `Store.commit_turn()` | persistence is not orchestration |
| Memory consolidation | inside `Store.commit_turn()` | rollover is persistence policy |
| Identity + bootstrap docs | `Store.load_context()` or a small context loader used by it | just prompt material |
| Provider SDK normalization | `deps["provider"].chat(...)` | provider differences are adapter concerns |
| Local tools | `Tools.execute()` | capability surface only |
| MCP tools | wrapped into the same `Tools` gateway | same contract, different backend |
| Slash commands | `try_handle_command()` before loop | cheap short-circuit |
| Progress streaming | optional `on_progress(event)` callback | UX concern, not core control flow |
| Cron | scheduler emits `TurnRequest(trigger="cron")` | producer only |
| Heartbeat | scheduler emits `TurnRequest(trigger="heartbeat")` | producer only |
| Subagents | tool or service emits a follow-up `TurnRequest(trigger="subagent")` | same turn path reused |
| Background autonomy | any background worker re-enters through queue with `TurnRequest` | no second engine needed |
| Multi-channel routing | stays in request metadata and outbound emitter | transport concern |
| Images / media | carried as `request["media"]` and interpreted by provider adapter | no new kernel branch needed |

## The Leanest Correct Invariants

These are the rules that preserve future extensibility without letting the core bloat again.

1. One ingress path: every trigger becomes `TurnRequest`.
2. One turn path: every request goes through `run_turn()`.
3. One commit path: all persisted turns go through `Store.commit_turn()`.
4. One cancellation rule: newest request wins within the same `session_key`.
5. One tool surface: all tools use `schemas()` and `execute()`.
6. No hidden mutable per-turn tool state.
7. No feature-specific queue types.
8. No direct store/file access from the turn loop.
9. No background shortcut that bypasses the queue.
10. No second orchestration engine for subagents, cron, or heartbeat.

## What I Would Deliberately Not Put In The Kernel

- a separate bus abstraction
- a separate context-builder class
- a separate memory manager class
- a separate command framework
- hook registries
- provider wrapper classes that only forward calls
- a generic job abstraction beyond `TurnRequest`
- a second trace format for background work

Each of those may be useful later in a full product, but none is necessary for the minimal kernel.

## Recommended File Layout

```text
nanobot/
  kernel.py      # make_request, worker_loop, run_turn, build_messages, helpers
  store.py       # Store implementation, trace sanitize, memory rollover
  tools.py       # Tools gateway, local/MCP tool adapters
  adapters/      # chat, api, cron, heartbeat, subagent, bus, provider, channels
```

This is smaller than the heavier proposals, but still leaves stable plug-in seams for the larger system.

## Why This Is Leaner Than The Other Variants

- Leaner than `CORE-LEAN.md` because it removes extra named subsystems from the kernel boundary.
- Leaner than `CORE-BRICKS*.md` because it collapses multiple "good separations" into only the ones that are operationally necessary.
- Safer than `CORE-MINIMAL.md` because it keeps explicit store and tools contracts instead of letting everything smear into one file.
- More future-proof than `CORE-LEANEST.md` because it explicitly reserves the plug-in seams needed by `CORE-IDEA.md`.

## Final Kernel Statement

The lean agentic kernel for nanobot should be:

- one canonical request shape
- one queue with per-session single-flight cancellation
- one `run_turn()` loop
- one `Store` contract that owns history and memory rollover
- one `Tools` gateway that owns capability exposure and execution

Everything else should plug in at the edges.

That is the smallest kernel that can still grow back into the full nanobot system without a rewrite.
