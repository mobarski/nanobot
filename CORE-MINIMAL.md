# Nanobot Minimal Core (Function-Only)

This is the smallest practical blueprint: no framework-like layering, only a few plain functions.

## Target Size

Core runtime = 1 file (`core.py`) with ~5-7 functions.

Adapters (`io.py`) only translate external signals (chat, cron, heartbeat) into `TurnRequest` dictionaries.

## Minimal Data Shapes

Use plain dicts, not classes.

`TurnRequest`:

```python
{
  "session_key": str,
  "trigger": "manual|cron|heartbeat|subagent|api",
  "content": str,
  "media": list[str],
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

## Only Five Core Functions

1. `handle_event(event)`
   - Converts adapter events to `TurnRequest`
   - Pushes request to queue

2. `worker_loop()`
   - Pops request from queue
   - Cancels previous running task in same session
   - Calls `run_turn(request)`
   - Emits outbound if needed

3. `run_turn(request)`
   - Handles slash commands inline
   - Loads context
   - Runs LLM/tool loop
   - Commits trace
   - Returns final result

4. `load_context(session_key)`
   - Returns recent session messages + memory text + runtime metadata block

5. `commit_turn(session_key, trace)`
   - Persists messages
   - Performs summarize/rollover if threshold exceeded

Everything else is helper code, not a new abstraction.

## Tool Model (No Registry Class)

Keep tools as one dictionary:

```python
TOOLS = {
  "read_file": read_file_tool,
  "write_file": write_file_tool,
  "edit_file": edit_file_tool,
  "list_dir": list_dir_tool,
  "exec": exec_tool,
  "web_search": web_search_tool,
  "web_fetch": web_fetch_tool,
  "message": message_tool,
  "spawn": spawn_tool,
  "cron": cron_tool,
}
```

Two helper functions only:

- `tool_schemas(TOOLS)` for provider call
- `exec_tool_call(name, args, ctx)` for safe execution

## Minimal Control Flow

```text
event -> handle_event -> queue.put(request)
worker_loop -> request -> run_turn -> maybe send outbound
```

No secondary pipelines.  
Subagent/cron/heartbeat callbacks must also enter through `handle_event`.

## Minimal `run_turn` Pseudocode

```text
run_turn(request):
  if request.content startswith "/":
    trace = run_inline_command(request)
    commit_turn(request.session_key, trace)
    return result_from_trace(trace)

  context = load_context(request.session_key)
  messages = build_messages(context, request)
  trace = []

  repeat max_iterations:
    llm = provider.chat(messages, tool_schemas(TOOLS))
    trace += assistant_msg(llm)

    if llm has tool_calls:
      for call in llm.tool_calls:
        tool_result = exec_tool_call(call.name, call.args, {
          "session_key": request.session_key,
          "metadata": request.metadata
        })
        trace += tool_msg(call, tool_result)
      messages = messages + trace_delta
      continue

    final = llm.text or fallback
    trace += assistant_final(final)
    break

  commit_turn(request.session_key, trace)
  return to_result(request, final)
```

## Cancellation Policy (Single Rule)

For each `session_key`:

- New request cancels previous in-flight task.
- Keep one active task per session.

No extra lock matrices, no nested cancellation graphs.

## Persistence Policy (Single Rule)

On every turn:

- append trace
- if history exceeds threshold: summarize old half into memory, keep recent half

Do not expose multi-step memory orchestration at runtime layer.

## Progress Policy (Single Rule)

One optional callback:

- `on_progress(text, is_tool_hint=False)`

If missing, do nothing.

No event bus for progress, no plugin hook chain.

## File Layout (Leanest)

```text
nanobot/
  core.py      # queue, worker, run_turn, load_context, commit_turn
  io.py        # adapters only: channels/cron/heartbeat -> handle_event
  tools.py     # tool callables + schemas
```

Everything else is optional or generated later if complexity truly appears.

## Non-Negotiable Simplicity Guardrails

- No new class unless at least 2 independent implementations are required.
- No new field unless it changes runtime behavior in at least 1 branch.
- No wrapper that only forwards arguments.
- No second queue type.
- No second turn entrypoint.

## Tradeoffs (Accepted)

- Less "clean architecture" purity.
- More code in fewer files.
- Refactors later may split modules again.

This is intentional: optimize for smallest mental model now.

## One-Sentence Summary

The minimal core is: **one queue of `TurnRequest`, one `run_turn` function, one persistence commit path, and tools as plain callables.**
