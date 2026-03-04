# Nanobot Leanest V2 (Lean + Manageable)

This version is the recommended balance:

- Keep the small architecture from `CORE-LEANEST.md`
- Re-add operational clarity from `CORE-LEAN.md`

The goal is not maximum minimalism; it is minimum complexity **with explicit safety rails**.

## Why V2

`CORE-LEANEST.md` is compact and clean, but some responsibilities are implicit.  
`CORE-LEAN.md` is explicit and safe, but heavier.

V2 combines both:

- Small module surface
- Explicit invariants
- Clear migration path

## Target Architecture (Keep It Small)

Four core modules:

- `engine.py` — one turn orchestration entry and LLM/tool loop
- `runner.py` — queue + workers + cancellation
- `store.py` — history + memory rollover
- `tools.py` — schemas + execution

Everything else stays adapter/plugin level.

## Core Contracts

## TurnRequest

One request shape for all triggers:

- `session_key`
- `trigger` = `manual|cron|heartbeat|subagent|api`
- `content`
- `media`
- `metadata`

No separate `kind` field unless it creates a real runtime branch.

## Turn Entry

One engine entry:

- `TurnEngine.run(request) -> response`

All inbound sources must pass through this same path.

## Store Contract

Use explicit naming so behavior is not hidden:

- `context = store.load_context(session_key, window)`
- `store.commit_turn_and_roll_memory_if_needed(session_key, trace)`

Memory rollover is guaranteed by contract, not by caller memory.

## Tool Contract

Keep small, explicit contracts:

- `tool_registry.schemas()`
- `tool_registry.execute(call, ctx)`

Minimal tool context:

- `ctx = {session_key, permissions}`
- Add channel/chat/message only for tools that truly require routing.

## Explicit Policies (Re-added from Lean)

Policies are data/config, not ad-hoc branching:

- `ToolPolicy`:
  - workspace restrictions
  - allow/deny patterns
  - timeout profile
- `MemoryPolicy`:
  - context window
  - summarize threshold
  - retention split rule
- `ProgressPolicy`:
  - emit mode: `none|text|text_and_tool_hints`
  - channel-specific suppression if needed

## Explicit Invariants (Must Hold)

1. **Single turn path**
   - All triggers must use `TurnEngine.run`.
2. **Single queue type**
   - Runner queue holds only `TurnRequest`.
3. **Single commit path**
   - All successful/command turns persist via `commit_turn_and_roll_memory_if_needed`.
4. **Single-session cancellation rule**
   - New request for same `session_key` cancels previous in-flight request.
5. **Stateless tool execution**
   - No hidden mutable tool context as default mechanism.

## Simplicity Guardrails

- No wrapper class that only forwards calls.
- No new field without at least one behavior branch using it.
- No per-feature lock maps inside engine loop.
- No ad-hoc synthetic message formats outside one adapter boundary.
- No second turn entrypoint.

## Leanest V2 End-to-End Pseudocode

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
    trace = handle_command_inline(request)
    store.commit_turn_and_roll_memory_if_needed(request.session_key, trace)
    return response_from(trace)

  ctx = store.load_context(request.session_key, memory_policy.window)
  messages = build_messages(ctx, request)
  trace = []

  repeat max_tool_iterations:
    llm = provider.chat(messages, tools.schemas(), model_settings)
    trace += assistant_message(llm)

    if llm.tool_calls:
      for call in llm.tool_calls:
        result = tools.execute(call, {
          "session_key": request.session_key,
          "permissions": tool_policy.permissions_for(request),
        })
        trace += tool_result(call, result)
      messages = rebuild(messages, trace)
      if progress_policy.emit_text_or_hints:
        on_progress(request, llm, trace)
      continue

    final = final_text_or_fallback(llm)
    trace += assistant_final(final)
    break

  store.commit_turn_and_roll_memory_if_needed(request.session_key, trace)
  return to_response(request, final, trace)
```

## Phased Migration Plan (Re-added from Lean)

1. **Extract TurnEngine wrapper first**
   - Wrap current process path without behavior changes.
2. **Introduce Store facade**
   - Keep current session/memory files behind one contract.
3. **Introduce explicit tool call context**
   - Keep compatibility shim if needed, then remove mutable context.
4. **Unify all turn producers**
   - System/direct/cron/heartbeat/subagent -> `TurnRequest`.
5. **Create runner facade**
   - Move active-task tracking and cancellation there.
6. **Flatten loop responsibilities**
   - Engine handles loop only; policies + store handle side concerns.
7. **Delete compatibility branches**
   - Remove old duplicated paths/locks once tests pass.

## What V2 Optimizes For

- Lean mental model in daily development
- Explicit operational rules in production
- Safe, incremental migration from current code

## Bottom Line

`CORE-LEANEST-V2.md` is the practical optimum:  
**small architecture + explicit invariants + clear rollout path**.
