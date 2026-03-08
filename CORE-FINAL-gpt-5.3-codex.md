# Nanobot Core Final - gpt-5.3-codex

## Stored Prompt

`read all CORE-*.md and try to DESIGN as lean agentic kernel as possible BUT it must be able to plug in the features from the big system (CORE-IDEA.md); save the result in CORE-FINAL-{model-name-and-version}.md; also - store this promp in the resulting file`

## Design Target

Build the smallest useful kernel with one execution path, while keeping hard plug-in ports for the larger system features:

- tool-augmented LLM turns
- persistent session + long-term memory
- background autonomy (subagent, cron, heartbeat)
- multi-channel IO adapters

## Lean Kernel (Minimum Viable Core)

Core runtime is four modules:

- `kernel/request.py` - normalize all inbound events to `TurnRequest`
- `kernel/runner.py` - queue + single-flight cancellation per `session_key`
- `kernel/engine.py` - one turn orchestrator and LLM/tool loop
- `kernel/store.py` - context load + commit with optional memory rollover

Everything else is a plug-in adapter.

## Canonical Data Contracts

`TurnRequest`:

```python
{
  "session_key": str,
  "trigger": "manual|cron|heartbeat|subagent|api",
  "content": str,
  "media": list,
  "metadata": dict
}
```

`TurnResult`:

```python
{
  "emit": bool,
  "content": str,
  "outbound": {"channel": str, "chat_id": str, "content": str, "metadata": dict}
}
```

`ToolCallContext` (strict minimum):

```python
{
  "session_key": str,
  "permissions": dict
}
```

## Kernel Ports (Plug-In Interfaces)

These are the only extension points needed to re-attach the full `CORE-IDEA.md` system.

1) `request_factory(event) -> TurnRequest`  
   Used by chat/CLI/API/cron/heartbeat/subagent adapters.

2) `store.load_context(session_key, window) -> dict`  
   Returns merged context (`identity`, `bootstrap_docs`, `memory`, `history`).

3) `store.commit_turn_and_roll_memory_if_needed(session_key, trace) -> None`  
   Persists trace, sanitizes content, and performs threshold-based consolidation.

4) `provider.chat(messages, tool_schemas, settings) -> {"text","tool_calls","raw"}`  
   Provider-agnostic adapter (OpenAI/Anthropic/local).

5) `tools.schemas(policy) -> list[dict]` and `tools.execute(call, ctx) -> dict`  
   Stateless tool exposure and execution.

6) `command_handler(request, store) -> dict | None`  
   Slash-command short-circuit (`/help`, `/new`, `/stop`, custom).

7) `on_emit(outbound) -> None` and `on_progress(event) -> None`  
   Optional side-effect hooks for channel delivery and streaming.

8) `policy_resolver(request) -> dict`  
   Pure-data policy selection (`tool`, `memory`, `progress`, `model`).

## One Turn Path (Kernel Logic)

```text
event -> request_factory -> runner.submit(request)

runner.worker_loop:
  request = queue.get()
  cancel previous task for request.session_key
  result = engine.run(request)
  if result.emit: on_emit(result.outbound)

engine.run(request):
  cmd = command_handler(request, store)
  if cmd:
    store.commit_turn_and_roll_memory_if_needed(session_key, cmd.trace or [])
    return cmd.result

  policies = policy_resolver(request)
  context = store.load_context(request.session_key, policies.memory.window)
  messages = build_messages(context, request)
  trace = []

  repeat <= policies.max_iterations:
    llm = provider.chat(messages, tools.schemas(policies.tool), policies.model)
    trace += assistant(llm.text, llm.tool_calls)
    if not llm.tool_calls: break

    for call in llm.tool_calls:
      out = tools.execute(call, {"session_key": request.session_key, "permissions": policies.tool})
      trace += tool(call, out)

    messages = build_messages(context, request) + trace
    on_progress({"text": llm.text, "tool_calls": llm.tool_calls})

  final = llm.text or "[no response]"
  store.commit_turn_and_roll_memory_if_needed(request.session_key, trace)
  return to_turn_result(request, final)
```

## How Big-System Features Plug In

From `CORE-IDEA.md`, each feature maps to a port, not a kernel rewrite:

- Message bus and channel adapters -> `request_factory` + `on_emit`
- Session JSONL and memory files -> `store.*` implementation
- Memory consolidation workflow -> internal `store` consolidator callback
- Tool registry, schema validation, safe execution -> `tools.*`
- Provider differences and response normalization -> `provider.chat`
- Subagent completion injection -> emits system event -> `request_factory`
- Cron due jobs -> emits cron event -> same path
- Heartbeat decisions -> emits heartbeat event -> same path

This preserves a single deterministic outer flow with pluggable capability modules.

## Hard Invariants (Do Not Break)

1. One ingress path: every trigger becomes `TurnRequest`.
2. One queue type: runner queues only `TurnRequest`.
3. One engine entry: `engine.run(request)`.
4. One commit path: all turns persist via `store.commit_turn_and_roll_memory_if_needed`.
5. Single-flight per session: new request cancels prior in-flight request for same `session_key`.
6. No hidden mutable tool context: execution context is explicit per call.
7. Kernel does not know storage file formats or channel transport details.

## Why This Is the Leanest Compatible Kernel

- Minimal concept count (request, runner, engine, store, tools/provider ports).
- Full compatibility with advanced features by adapters.
- Operational safety retained through strict invariants.
- Migration-friendly: current system can be wrapped behind ports incrementally.

## Immediate Implementation Skeleton

Start with these signatures and keep the rest as adapters:

```python
def make_request(event: dict) -> dict: ...

class TurnRunner:
    def submit(self, request: dict) -> None: ...
    async def worker_loop(self) -> None: ...

class TurnEngine:
    async def run(self, request: dict) -> dict: ...

class Store:
    def load_context(self, session_key: str, window: int) -> dict: ...
    async def commit_turn_and_roll_memory_if_needed(self, session_key: str, trace: list[dict]) -> None: ...
```
