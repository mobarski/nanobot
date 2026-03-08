# Nanobot Core Final — claude-4.6-opus

## Stored Prompt

`read all CORE-*.md and try to DESIGN as lean agentic kernel as possible BUT it must be able to plug in the features from the big system (CORE-IDEA.md); save the result in CORE-FINAL-{model-name-and-version}.md; also - store this promp in the resulting file`

## Source Documents

CORE-IDEA.md, CORE-LEAN.md, CORE-LEANEST.md, CORE-LEANEST-V2.md, CORE-MINIMAL.md, CORE-BRICKS.md, CORE-BRICKS-V2.md, CORE-FINAL-gpt-5.3-codex.md

---

## Core Thesis

**The kernel is one function.** Everything else is a plug.

Previous iterations reduced the core to 4 modules, then 3 files, then 5 functions. This design goes further: the irreducible agentic kernel is a single async function — `run_turn` — that accepts a request dict and a deps namespace. Runner, queue, cancellation, channels, cron, heartbeat, subagents — all are adapters that *call* this function. None live inside it.

---

## Why One Function

Every prior design (LEAN through BRICKS-V2) converges on the same inner loop:

1. Check for command short-circuit
2. Load context
3. Build messages
4. Loop: LLM call → tool execution → repeat until text response
5. Commit trace
6. Return result

Steps 1–6 have zero internal branching that depends on *how* the turn was triggered. A cron turn, a chat turn, a subagent callback — all execute the same steps. The differences are entirely in what gets *injected*: which tools, which context, which provider settings. That means the kernel has no reason to know about triggers, channels, or scheduling. It just runs a turn.

---

## The Kernel

```python
async def run_turn(req, deps):
    """The entire agentic kernel. ~20 lines of logic."""

    # pre-hook: slash commands
    cmd = deps.handle_command(req) if deps.handle_command else None
    if cmd is not None:
        await deps.commit(req["sk"], cmd.get("trace", []))
        return cmd

    # context + prompt
    ctx = await deps.load(req["sk"])
    msgs = deps.build(ctx, req)
    trace = []

    # LLM-tool loop
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

    # persist + return
    await deps.commit(req["sk"], trace)
    return {"emit": True, "content": llm.get("text", ""), "trace": trace}
```

That is the complete kernel. No classes, no inheritance, no registries, no queues, no locks. Pure data-in, data-out with injected async callables.

---

## Request Shape (One Dict, No Classes)

```python
req = {
    "sk":       str,   # session key — the only identity the kernel sees
    "content":  str,   # user/system/cron message text
    "media":    list,  # optional attachments
    "meta":     dict,  # opaque bag for adapters (channel, chat_id, trigger, ...)
}
```

The kernel never inspects `meta`. Adapters read/write it. This keeps trigger-specific logic (cron vs chat vs heartbeat) outside the kernel entirely.

---

## Deps Namespace (All Plugs)

```python
deps.load(sk)                  # -> ctx dict {history, memory, identity, ...}
deps.commit(sk, trace)         # persist trace + roll memory if needed
deps.build(ctx, req)           # -> list[message_dict] for LLM
deps.chat(msgs, schemas)       # -> {"text": str, "tc": list[tool_call]}
deps.schemas(req)              # -> list[json_schema]  (policy-filtered)
deps.exec(tool_call, req)      # -> str (result or error text)
deps.handle_command(req)       # -> dict | None  (short-circuit)
deps.on_progress(llm, trace)   # -> None  (optional streaming side-effect)
deps.max_iter                   # int  (iteration cap)
```

Nine plugs. Each is a plain callable or value. No protocols, no base classes, no ABCs. A `SimpleNamespace` or `dataclass` or even a dict with callables works.

---

## How CORE-IDEA.md Features Plug In

| CORE-IDEA feature | Kernel plug | What the adapter does |
|---|---|---|
| Message bus + channels | *caller of* `run_turn` | Normalizes events into `req`, routes result via `meta` |
| Session JSONL | `deps.load` / `deps.commit` | Reads/writes JSONL, manages replay window |
| Memory consolidation | inside `deps.commit` | Threshold check → summarize old history → update MEMORY.md |
| Identity + bootstrap docs | inside `deps.load` | Merges identity, bootstrap files, memory into `ctx` |
| Tool registry + schemas | `deps.schemas` / `deps.exec` | Filters by policy, validates args, calls tool fn |
| Provider normalization | `deps.chat` | Wraps OpenAI/Anthropic/local SDK into `{text, tc}` |
| Subagent spawn + result | `deps.exec` (spawn tool) → *caller* | Spawn tool starts background `run_turn`; result re-enters as new `req` |
| Cron / heartbeat | *caller of* `run_turn` | Timer fires → builds `req` → calls `run_turn` → routes result |
| Progress streaming | `deps.on_progress` | Emits partial text / tool hints to channel |
| Slash commands | `deps.handle_command` | `/help`, `/new`, `/stop` — returns short-circuit dict |
| MCP servers | inside `deps.exec` / `deps.schemas` | MCP tools appear alongside local tools |

No feature requires kernel modification. Every feature is an adapter that either **calls** `run_turn` or **is called by** `run_turn` through a dep.

---

## Runner / Queue / Cancellation — An Adapter, Not Kernel

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

This is infrastructure, not kernel. Swap it for a durable queue, a process pool, a serverless invocation — `run_turn` doesn't change.

---

## Store — The Thickest Adapter

The store is the most complex plug because it owns two concerns: history replay and memory rollover. But from the kernel's perspective, it's just two callables:

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

## Tools — Two Callables

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

Tool context is the request itself. No mutable shared state. No `set_context`.

---

## Context Builder — One Function

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
        load     = store.load,
        commit   = store.commit,
        build    = build_messages,
        chat     = lambda msgs, schemas: provider_chat(msgs, schemas, config["model"], sdk),
        schemas  = tools.schemas,
        exec     = tools.exec,
        handle_command = make_command_handler(store),
        on_progress    = config.get("on_progress"),
        on_emit        = lambda result: channel_gw.send(result),
        max_iter       = config.get("max_iterations", 25),
    )
    runner = TurnRunner(deps)
    return runner
```

---

## Invariants (Non-Negotiable)

1. **One turn function.** All triggers call `run_turn`. No second entry point.
2. **One commit path.** All traces persist through `deps.commit`.
3. **Explicit deps.** No global mutable state. No hidden singletons.
4. **Kernel ignores trigger type.** The `meta` bag is opaque to the kernel.
5. **Tools are stateless.** Context flows in via `req`, not via mutation.

---

## What This Trades Away

| Sacrifice | Why it's acceptable |
|---|---|
| No classes in kernel | A 20-line function doesn't benefit from OOP structure |
| No formal Protocol/ABC | Duck-typed callables are sufficient at this scale |
| Runner is outside kernel | Queue/cancel strategy varies by deployment; kernel shouldn't constrain it |
| No explicit policy objects | Policies are baked into the deps callables (schemas filters by policy, commit rolls by threshold) — no separate concept needed |
| No command handler in kernel | Commands don't need LLM; they're a pre-hook, not core logic |

---

## Comparison With Prior Designs

| Design | Core concepts | Core LoC (pseudo) | Classes |
|---|---|---|---|
| CORE-IDEA | ~12 | ~150 | 5+ |
| CORE-LEAN | 6 components | ~60 | 6 |
| CORE-LEANEST | 4 modules | ~40 | 4 |
| CORE-LEANEST-V2 | 4 modules + invariants | ~45 | 4 |
| CORE-MINIMAL | 5 functions, 3 files | ~35 | 0 |
| CORE-BRICKS-V2 | 7 bricks + glue | ~80 | 3 |
| CORE-FINAL-gpt-5.3-codex | 4 modules, 8 ports | ~40 | 3 |
| **This (claude-4.6-opus)** | **1 function, 9 plugs** | **~20** | **0** |

---

## One-Sentence Summary

The leanest agentic kernel is one async function — `run_turn(req, deps)` — that loops LLM calls and tool executions until a text response emerges, with all other concerns (persistence, channels, scheduling, memory, policies) injected as plain callables.
