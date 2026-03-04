# Nanobot Leanest Core

This is the smallest practical architecture that still behaves like a capable agent.

Goal: keep core power (LLM + tools + memory + async turns) while reducing concepts to the minimum.

## Core Principle

If two abstractions always move together, merge them.  
If a field always has one value, delete it.  
If a layer only forwards calls, remove it.

## What To Simplify Further

## 1) One Request Shape

Use only one input object: `TurnRequest`.

- Keep: `session_key`, `trigger`, `content`, `media`, `metadata`
- Remove: separate `kind` field if `trigger` already explains origin

Why leaner:
- fewer branch combinations
- simpler tests and logging keys

## 2) One Runner, One Queue Type

Queue `TurnRequest` directly.

- `TurnRunner.submit(request)`
- worker pops a request and calls engine

No generic `Job` wrapper, no `job.type`.

Why leaner:
- no discriminators, no dispatch matrix
- less ceremony in scheduler code

## 3) One Engine Entry

Single orchestration method:

- `TurnEngine.run(request) -> TurnResponse`

Command handling, context build, LLM loop, and finalization happen inside this flow.

Why leaner:
- one place to read "how a message becomes a response"

## 4) One Store Commit API

Replace multiple persistence calls with one:

- `context = store.load_context(session_key, window)`
- `store.commit_turn(session_key, trace)`

`commit_turn` internally decides whether to summarize/roll memory.

Why leaner:
- loop layer no longer coordinates memory policies step-by-step
- store owns consistency

## 5) Minimal Tool Execution Contract

Keep tool execution simple:

- `tool_registry.schemas()`
- `tool_registry.execute(call, ctx)`

Use a tiny context object:

- `ctx = {session_key, permissions}`

Only add channel/chat if a concrete tool truly needs it.

Why leaner:
- avoids large context plumbing
- keeps tools mostly stateless

## 6) Remove Thin Wrappers

If `ProviderClient` only forwards to provider SDK, delete it.

Keep wrappers only when they add one of:

- cross-provider normalization
- retry/backoff policy
- auth/header synthesis

Why leaner:
- fewer files and indirection during debugging

## 7) Replace Hook System with One Callback

Use one optional callback:

- `on_progress(event)`

No hook registries unless there are multiple independent consumers.

Why leaner:
- fewer extension points to reason about

## Leanest Module Set

Keep just 4 core modules:

- `engine.py` — turn orchestration + LLM/tool loop
- `runner.py` — queue + workers + cancellation policy
- `store.py` — session history + memory rollover
- `tools.py` — registry + execution

Everything else is adapter/plugin territory.

## Minimal Interfaces

```python
class TurnRequest:
    session_key: str
    trigger: str      # manual|cron|heartbeat|subagent|api
    content: str
    media: list[str]
    metadata: dict

class TurnResponse:
    content: str
    emit_to_user: bool
    outbound: dict
```

```python
class TurnRunner:
    def submit(self, request: TurnRequest) -> None: ...

class TurnEngine:
    async def run(self, request: TurnRequest) -> TurnResponse: ...

class ConversationStore:
    def load_context(self, session_key: str, window: int): ...
    def commit_turn(self, session_key: str, trace: list[dict]) -> None: ...

class ToolRegistry:
    def schemas(self) -> list[dict]: ...
    async def execute(self, call, ctx) -> str: ...
```

## Leanest End-to-End Flow

```text
on_event(event):
  request = TurnRequest.from_event(event)
  runner.submit(request)

runner.worker():
  while running:
    request = queue.get()
    cancel_previous_if_same_session(request.session_key)
    response = engine.run(request)
    if response.emit_to_user:
      send(response.outbound)

engine.run(request):
  if slash_command(request.content):
    trace = handle_command(request)
    store.commit_turn(request.session_key, trace)
    return response_from(trace)

  ctx = store.load_context(request.session_key, window)
  messages = build_messages(ctx, request)
  trace = []

  repeat max_iterations:
    llm = provider.chat(messages, tools.schemas())
    trace += assistant_message(llm)
    if llm.tool_calls:
      for call in llm.tool_calls:
        result = tools.execute(call, {session_key, permissions})
        trace += tool_result(call, result)
      messages = rebuild(messages, trace)
      continue
    final = final_text_or_fallback(llm)
    trace += assistant_final(final)
    break

  store.commit_turn(request.session_key, trace)
  return to_response(request, final, trace)
```

## Practical Rule Of Thumb

Before adding any new abstraction, ask:

- Does it eliminate repeated logic in at least 2 places?
- Does it reduce state branching, not increase it?
- Would debugging a production issue become easier with it?

If any answer is "no", do not add that abstraction.
