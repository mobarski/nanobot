# Nanobot Core Final — cursor

## Stored Prompt

```
read all CORE-*.md and try to DESIGN as lean agentic kernel as possible BUT it must be able to plug in the features from the big system (CORE-IDEA.md);
save the result in CORE-FINAL-{model-name-and-version}.md; also - store this promp in the resulting file
```

---

## Design Thesis

**The kernel is one function.** Everything else is a plug.

The irreducible agentic kernel is a single async function — `run_turn(req, deps)` — that accepts a request dict and a deps namespace. Runner, queue, cancellation, channels, cron, heartbeat, subagents — all are adapters that *call* this function. None live inside it.

---

## The Kernel (≈20 lines)

```python
async def run_turn(req, deps):
    """The entire agentic kernel."""

    cmd = deps.handle_command(req) if deps.handle_command else None
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
    return {"emit": True, "content": llm.get("text", "") or "[no response]", "trace": trace}
```

No classes, no inheritance, no registries, no queues, no locks. Pure data-in, data-out with injected async callables.

---

## Request Shape (One Dict)

```python
req = {
    "sk":      str,   # session key — the only identity the kernel sees
    "content": str,   # user/system/cron message text
    "media":   list,  # optional attachments
    "meta":    dict,  # opaque bag for adapters (channel, chat_id, trigger, ...)
}
```

The kernel never inspects `meta`. Adapters read/write it. Trigger-specific logic (cron vs chat vs heartbeat) stays outside the kernel.

---

## Deps Namespace (9 Plugs)

| Plug | Signature | Purpose |
|------|-----------|---------|
| `load` | `(sk) -> ctx` | Load context (history, memory, identity, bootstrap) |
| `commit` | `(sk, trace) -> None` | Persist trace + roll memory if threshold exceeded |
| `build` | `(ctx, req) -> list[msg]` | Assemble LLM message list |
| `chat` | `(msgs, schemas) -> {text, tc}` | Provider-agnostic LLM call |
| `schemas` | `(req) -> list[dict]` | Policy-filtered tool schemas |
| `exec` | `(tool_call, req) -> str` | Execute tool, return result or error text |
| `handle_command` | `(req) -> dict \| None` | Slash-command short-circuit |
| `on_progress` | `(llm, trace) -> None` | Optional streaming side-effect |
| `max_iter` | `int` | Iteration cap |

Plain callables or values. No protocols, no ABCs. A `SimpleNamespace` or dict works.

---

## How CORE-IDEA.md Features Plug In

| CORE-IDEA feature | Kernel plug | Adapter role |
|-------------------|-------------|--------------|
| Message bus + channels | *caller of* `run_turn` | Normalize events → `req`, route result via `meta` |
| Session JSONL | `deps.load` / `deps.commit` | Read/write JSONL, manage replay window |
| Memory consolidation | inside `deps.commit` | Threshold → summarize → update MEMORY.md |
| Identity + bootstrap docs | inside `deps.load` | Merge into `ctx` |
| Tool registry + schemas | `deps.schemas` / `deps.exec` | Filter by policy, validate, call tool fn |
| Provider normalization | `deps.chat` | Wrap OpenAI/Anthropic/local → `{text, tc}` |
| Subagent spawn + result | `deps.exec` (spawn tool) → *caller* | Spawn starts background `run_turn`; result re-enters as new `req` |
| Cron / heartbeat | *caller of* `run_turn` | Timer → build `req` → `run_turn` → route result |
| Progress streaming | `deps.on_progress` | Emit partial text / tool hints |
| Slash commands | `deps.handle_command` | `/help`, `/new`, `/stop` short-circuit |
| MCP servers | inside `deps.exec` / `deps.schemas` | MCP tools alongside local tools |

No feature requires kernel modification. Every feature is an adapter that either **calls** `run_turn` or **is called by** it through a dep.

---

## Runner (Adapter, Not Kernel)

```python
class TurnRunner:
    def __init__(self, deps):
        self.deps = deps
        self.q = asyncio.Queue()
        self.active = {}  # sk -> Task

    def submit(self, req):
        self.q.put_nowait(req)

    async def loop(self):
        while True:
            req = await self.q.get()
            sk = req["sk"]
            prev = self.active.pop(sk, None)
            if prev:
                prev.cancel()
            self.active[sk] = asyncio.create_task(self._run(req))

    async def _run(self, req):
        try:
            result = await run_turn(req, self.deps)
            if result.get("emit"):
                await self.deps.on_emit(result)
        finally:
            self.active.pop(req["sk"], None)
```

Swap for durable queue, process pool, serverless — `run_turn` unchanged.

---

## Store (Two Callables)

```python
async def load(sk):
    history = history_repo.load_recent(sk, window)
    memory = memory_repo.load(sk)
    identity = read_identity_files()
    return {"history": history, "memory": memory, "identity": identity}

async def commit(sk, trace):
    clean = sanitize(trace)  # truncate, strip runtime tags, replace images
    history_repo.append(sk, clean)
    if history_repo.unconsolidated_count(sk) >= threshold:
        await consolidate(sk, history_repo, memory_repo, summarizer)
```

Internally as complex as needed. Externally two functions.

---

## Tools (Two Callables)

```python
def schemas(req):
    return [t["schema"] for t in REGISTRY.values()
            if policy_allows(t["name"], req.get("meta", {}))]

async def exec(tc, req):
    tool = REGISTRY.get(tc["name"])
    if not tool:
        return f"error: unknown tool {tc['name']}"
    try:
        return await tool["fn"](tc["args"], {"sk": req["sk"]})
    except Exception as e:
        return f"error: {e}"
```

Tool context flows in via `req`. No mutable shared state.

---

## Context Builder (One Function)

```python
def build(ctx, req):
    sys = "\n\n".join(p for p in [ctx.get("identity"), ctx.get("memory")] if p)
    msgs = [{"role": "system", "content": sys}]
    msgs.extend(ctx.get("history", []))
    msg = {"role": "user", "content": req["content"]}
    if req.get("media"):
        msg["media"] = req["media"]
    msgs.append(msg)
    return msgs
```

---

## Bootstrap (Wiring Example)

```python
from types import SimpleNamespace

def bootstrap(config, sdk, channel_gw):
    store = make_store(config)
    tools = make_tools(config)

    deps = SimpleNamespace(
        load          = store.load,
        commit        = store.commit,
        build         = build_messages,
        chat          = lambda msgs, schemas: provider_chat(msgs, schemas, config["model"], sdk),
        schemas       = tools.schemas,
        exec          = tools.exec,
        handle_command = make_command_handler(store),
        on_progress   = config.get("on_progress"),
        on_emit       = lambda result: channel_gw.send(result),
        max_iter      = config.get("max_iterations", 25),
    )
    return TurnRunner(deps)
```

---

## Invariants (Non-Negotiable)

1. **One turn function.** All triggers call `run_turn`. No second entry point.
2. **One commit path.** All traces persist through `deps.commit`.
3. **Explicit deps.** No global mutable state. No hidden singletons.
4. **Kernel ignores trigger type.** The `meta` bag is opaque to the kernel.
5. **Tools are stateless.** Context flows in via `req`, not via mutation.

---

## Hard Boundaries (From CORE-BRICKS)

- **Engine does not know file formats.** Storage serialization stays inside store.
- **Store does not know channels.** Only session keys and messages.
- **Tools receive explicit ctx, return explicit output.** No hidden mutable state.
- **Runner does not build prompts.** Scheduling and dispatch only.
- **Background services enter through `req`.** One ingress path, no shortcuts.

---

## Comparison With Prior Designs

| Design | Core concepts | Classes in kernel |
|--------|---------------|------------------|
| CORE-IDEA | ~12 | 5+ |
| CORE-LEAN | 6 components | 6 |
| CORE-LEANEST | 4 modules | 4 |
| CORE-MINIMAL | 5 functions | 0 |
| CORE-FINAL-gpt-5.3-codex | 4 modules, 8 ports | 3 |
| CORE-FINAL-claude-4.6-opus | 1 function, 9 plugs | 0 |
| **This (cursor)** | **1 function, 9 plugs** | **0** |

---

## One-Sentence Summary

The leanest agentic kernel is one async function — `run_turn(req, deps)` — that loops LLM calls and tool executions until a text response emerges, with all other concerns (persistence, channels, scheduling, memory, policies) injected as plain callables.
