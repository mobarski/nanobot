# Nanobot Core Final — claude-4.6-sonnet-medium-thinking

## Stored Prompt

```
read all CORE-*.md and try to DESIGN as lean agentic kernel as possible BUT it must be able to plug in
the features from the big system (CORE-IDEA.md); save the result in CORE-FINAL-{model-name-and-version}.md;
also - store this promp in the resulting file
```

## Source Documents Read

CORE-IDEA.md · CORE-LEAN.md · CORE-LEANEST.md · CORE-LEANEST-V2.md · CORE-MINIMAL.md ·
CORE-BRICKS.md · CORE-BRICKS-V2.md · CORE-FINAL-gpt-5.3-codex.md · CORE-FINAL-gpt-5.4.md ·
CORE-FINAL-claude-4.6-opus.md

---

## Synthesis Thesis

Every prior design converges. After reading all ten documents, the irreducible kernel is clear:

> **One async function. One opaque deps bag. All else is adapter.**

The claude-4.6-opus design (`run_turn(req, deps)`) already reached this conclusion.
This design agrees, then pushes the deps bag itself through one final compression:
collapse nine named callables into **five typed slots** that map cleanly onto the five
concerns that can never be eliminated from any agent system:

| Slot | Concern | Can it be removed? |
|---|---|---|
| `deps.store` | Memory — what did I see before? | No |
| `deps.llm` | Reasoning — what should I do next? | No |
| `deps.tools` | Action — what can I change? | No |
| `deps.build` | Representation — what does the LLM see? | No (but tiny) |
| `deps.on_progress` | Observability — what is happening now? | Yes — defaults to no-op |

Command handling collapses into `deps.store` (it's a stateful pre-hook with store access).
Schema filtering collapses into `deps.tools` (tools own their own exposure contract).
Policy resolution collapses into `deps.store` and `deps.tools` callsites (policies are data, not a slot).

Result: the fewest independent concerns an agent can have.

---

## The Kernel (~18 lines)

```python
async def run_turn(req: dict, deps) -> dict:
    """Irreducible agentic kernel."""

    # pre-hook: slash commands and other short-circuits
    early = deps.store.handle(req)
    if early is not None:
        return early

    # context + prompt
    ctx    = await deps.store.load(req["sk"])
    msgs   = deps.build(ctx, req)
    trace  = []

    # LLM–tool loop
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

Two tiny pure helpers (not kernel logic, just dict construction):

```python
def asst(llm):
    return {"role": "assistant", "content": llm["text"], "tool_calls": llm["tc"]}

def tool_result(tc, out):
    return {"role": "tool", "call_id": tc["id"], "content": out}

def emit(req, text):
    return {"emit": True, "content": text, "sk": req["sk"], "meta": req["meta"]}
```

---

## Request Shape

```python
req = {
    "sk":    str,   # session key — the only identity the kernel sees
    "content": str,
    "media":   list,
    "meta":    dict, # opaque: channel, chat_id, trigger, permissions, ...
}
```

The kernel never inspects `meta`. Adapters read and write it. This keeps trigger-specific
logic (cron / chat / heartbeat / subagent) entirely outside the kernel.

---

## Deps Contract (Five Slots)

```python
# deps is any object (SimpleNamespace, dataclass, dict) that provides:

deps.store.load(sk)           # async -> ctx dict
                              #   ctx must contain: history, memory, identity,
                              #   bootstrap_docs, max_iter
deps.store.commit(sk, trace)  # async -> None  (sanitize + append + roll memory if needed)
deps.store.handle(req)        # sync -> dict | None  (command short-circuit; None = not a command)
deps.store.archive(sk)        # sync -> None  (/new support)

deps.llm(msgs, schemas)       # async -> {"text": str, "tc": list[tool_call]}

deps.tools.schemas(req)       # sync -> list[json_schema]  (policy-filtered per req)
deps.tools.exec(tc, req)      # async -> str  (result or recoverable error)

deps.build(ctx, req)          # sync -> list[message_dict]

deps.on_progress(llm, trace)  # async -> None  (default: no-op)
```

Why `store.handle` lives on store: commands like `/new` need store access (archive + clear),
and they are a stateful pre-hook. Attaching them to `store` avoids a sixth slot.

---

## How CORE-IDEA.md Features Plug In

| CORE-IDEA feature | Kernel slot | Adapter responsibility |
|---|---|---|
| Message bus / channels | caller of `run_turn` | normalize event → `req`; route `emit` result via `meta` |
| Session JSONL history | `deps.store.load` / `deps.store.commit` | read/write JSONL; manage replay window |
| Memory consolidation | inside `deps.store.commit` | threshold check → summarize old → write MEMORY.md |
| Identity + bootstrap docs | inside `deps.store.load` | merge identity, bootstrap files, memory into `ctx` |
| Tool registry + validation | `deps.tools.schemas` / `deps.tools.exec` | filter by policy; validate args; call fn |
| Provider normalization | `deps.llm` | wrap OpenAI / Anthropic / local SDK → `{text, tc}` |
| Subagent spawn | `deps.tools.exec` (spawn tool) | start background `run_turn`; inject result as new `req` |
| Cron | external scheduler | timer fires → build `req` with `meta.trigger="cron"` → queue |
| Heartbeat | external scheduler | same path as cron |
| Progress streaming | `deps.on_progress` | emit partial text / tool hints to channel |
| Slash commands | `deps.store.handle` | `/help`, `/new`, `/stop`; returns short-circuit dict |
| MCP servers | inside `deps.tools` | MCP tools appear alongside local tools, same two methods |
| Multi-channel routing | `req.meta` + outbound adapter | kernel never sees channel type |

No feature requires modifying the kernel. Every feature either **calls** `run_turn` or
**is called by** `run_turn` through one of the five slots.

---

## Runner — An Adapter, Not Kernel

```python
class TurnRunner:
    def __init__(self, deps, on_emit):
        self.deps    = deps
        self.on_emit = on_emit
        self.q       = asyncio.Queue()
        self.active  = {}  # sk -> Task

    def submit(self, req: dict) -> None:
        self.q.put_nowait(req)

    async def loop(self) -> None:
        while True:
            req = await self.q.get()
            sk  = req["sk"]
            prev = self.active.pop(sk, None)
            if prev:
                prev.cancel()
            self.active[sk] = asyncio.create_task(self._run(req))

    async def _run(self, req: dict) -> None:
        try:
            result = await run_turn(req, self.deps)
            if result.get("emit"):
                await self.on_emit(result)
        finally:
            self.active.pop(req["sk"], None)
```

Swap for a durable queue, a process pool, or serverless invocation without touching `run_turn`.

---

## Store — The Thickest Adapter

```python
class Store:
    def __init__(self, history, memory, summarizer, policy):
        self.history    = history    # repo: load_recent, append, archive, clear, unconsolidated_count
        self.memory     = memory     # repo: load, write
        self.summarizer = summarizer # async (old_msgs, current_memory) -> new_memory_text
        self.policy     = policy     # {window, threshold, keep_recent, max_iter}

    def handle(self, req) -> dict | None:
        text = (req.get("content") or "").strip()
        if not text.startswith("/"):
            return None
        sk = req["sk"]
        if text == "/new":
            self.archive(sk)
            return {"emit": True, "content": "New conversation started.", "sk": sk, "meta": req["meta"]}
        if text == "/help":
            return {"emit": True, "content": "Commands: /help /new /stop", "sk": sk, "meta": req["meta"]}
        return {"emit": True, "content": "Unknown command.", "sk": sk, "meta": req["meta"]}

    async def load(self, sk: str) -> dict:
        return {
            "history":       self.history.load_recent(sk, self.policy["window"]),
            "memory":        self.memory.load(sk),
            "identity":      load_identity_files(),
            "bootstrap_docs": load_bootstrap_files(),
            "max_iter":      self.policy["max_iter"],
        }

    async def commit(self, sk: str, trace: list) -> None:
        self.history.append(sk, _sanitize(trace))
        if self.history.unconsolidated_count(sk) >= self.policy["threshold"]:
            await self._consolidate(sk)

    def archive(self, sk: str) -> None:
        self.history.archive(sk)
        self.history.clear(sk)

    async def _consolidate(self, sk: str) -> None:
        old    = self.history.unconsolidated_slice(sk, keep=self.policy["keep_recent"])
        update = await self.summarizer(old, self.memory.load(sk))
        self.memory.write(sk, update)
        self.history.mark_consolidated(sk, old)
```

`_sanitize` is a pure function (truncate oversized tool output, strip runtime-context tags,
replace `data:image` payloads with markers). Not a separate brick — it has no other consumers.

---

## Context Builder — One Function

```python
def build_messages(ctx: dict, req: dict) -> list[dict]:
    system = "\n\n".join(p for p in [
        ctx.get("identity"),
        ctx.get("bootstrap_docs"),
        ctx.get("memory"),
    ] if p)
    msgs = [{"role": "system", "content": system}]
    msgs.extend(ctx.get("history", []))
    user = {"role": "user", "content": req.get("content", "")}
    if req.get("media"):
        user["media"] = req["media"]
    msgs.append(user)
    return msgs
```

---

## Tools — Two Methods, One Object

```python
class Tools:
    def __init__(self, registry: dict, policy_fn=None):
        self.registry  = registry   # name -> {"schema": dict, "fn": async callable}
        self.policy_fn = policy_fn  # optional (name, req) -> bool

    def schemas(self, req: dict) -> list[dict]:
        return [
            t["schema"] for name, t in self.registry.items()
            if not self.policy_fn or self.policy_fn(name, req)
        ]

    async def exec(self, tc: dict, req: dict) -> str:
        tool = self.registry.get(tc["name"])
        if not tool:
            return f"error: unknown tool {tc['name']}"
        try:
            result = await tool["fn"](tc.get("args", {}), {"sk": req["sk"], "meta": req["meta"]})
            return str(result)
        except Exception as e:
            return f"error: {e}"
```

---

## Bootstrap (Full Wiring Example)

```python
from types import SimpleNamespace

def bootstrap(config, sdk, channel_gw):
    store = Store(
        history    = config["history_repo"],
        memory     = config["memory_repo"],
        summarizer = config["summarizer"],
        policy     = config["memory_policy"],
    )

    async def llm(msgs, schemas):
        raw = await sdk.chat(messages=msgs, tools=schemas, **config["model"])
        return {"text": extract_text(raw), "tc": extract_tool_calls(raw)}

    tools = Tools(
        registry  = load_tool_registry(config),
        policy_fn = config.get("tool_policy_fn"),
    )

    deps = SimpleNamespace(
        store       = store,
        llm         = llm,
        tools       = tools,
        build       = build_messages,
        on_progress = config.get("on_progress", lambda *_: None),
    )

    async def on_emit(result):
        await channel_gw.send(
            channel  = result["meta"].get("channel"),
            chat_id  = result["meta"].get("chat_id"),
            content  = result["content"],
        )

    return TurnRunner(deps, on_emit)
```

---

## File Layout

```text
nanobot/
  kernel.py      # run_turn, asst, tool_result, emit  (~25 lines)
  store.py       # Store, _sanitize                   (~60 lines)
  tools.py       # Tools, load_tool_registry          (~40 lines)
  runner.py      # TurnRunner                         (~25 lines)
  adapters/      # channel, cron, heartbeat, subagent, provider, bus
```

Four files. The entire kernel is in `kernel.py`. The rest is configuration.

---

## Invariants

1. **One turn function.** All triggers call `run_turn`. No second entry point.
2. **One commit path.** All traces persist through `deps.store.commit`.
3. **One ingress shape.** Every event becomes a `req` dict with `sk`, `content`, `media`, `meta`.
4. **Kernel ignores `meta`.** Trigger type, channel, chat_id — all opaque to the kernel.
5. **Tools are stateless.** Context flows in via `req`; no mutable shared state in tools.
6. **Runner owns cancellation.** One active task per `sk`; new request cancels prior.
7. **No background shortcut.** Cron / heartbeat / subagent results re-enter through `runner.submit`.

---

## Comparison With Prior Designs

| Design | Core concept count | Approx kernel LoC | Classes in kernel |
|---|---|---|---|
| CORE-IDEA | ~12 | ~150 | 5+ |
| CORE-LEAN | 6 components | ~60 | 6 |
| CORE-LEANEST | 4 modules | ~40 | 4 |
| CORE-MINIMAL | 5 functions, 3 files | ~35 | 0 |
| CORE-BRICKS-V2 | 7 bricks + TurnEngine | ~80 | 3 |
| CORE-FINAL-gpt-5.3-codex | 4 modules, 8 ports | ~40 | 3 |
| CORE-FINAL-gpt-5.4 | 5 contracts | ~45 | 2 |
| CORE-FINAL-claude-4.6-opus | 1 function, 9 plugs | ~20 | 0 |
| **This (claude-4.6-sonnet-medium-thinking)** | **1 function, 5 slots** | **~18** | **0** |

The reduction from 9 plugs to 5 slots comes from collapsing command handling into `store`,
schema filtering into `tools`, and policy resolution into the two callsites that need it —
without losing any extensibility.

---

## One-Sentence Summary

The leanest agentic kernel is one async function — `run_turn(req, deps)` — with five dependency
slots (store, llm, tools, build, on\_progress) that cover the only five concerns an agent cannot
eliminate: memory, reasoning, action, representation, and observability.
