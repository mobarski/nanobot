# Making Nanobot Core Lean And Simple

This is a "keep power, reduce moving parts" redesign.

The core insight: nanobot currently has good primitives, but too many orchestration branches.  
A lean core should have **one execution model**, **one memory model**, and **one async model**.

---

## Design Goal

Keep these capabilities:

- Tool-augmented LLM turns
- Persistent sessions and memory
- Background tasks (cron/heartbeat/subagents)
- Multi-channel routing

But simplify to:

- One canonical turn pipeline
- One storage abstraction
- One worker model for foreground + background
- Fewer special cases and locks

---

## Lean Core Proposal (V2)

## 1) Single Turn Engine

Create a single `TurnEngine` that handles **all** turns:

- user turns
- system turns (subagent/cron/heartbeat callbacks)
- direct turns (CLI / internal API)

All inputs become the same `TurnRequest`:

- `origin` (channel/chat/session)
- `kind` (`user|system|internal`)
- `trigger` (`manual|cron|heartbeat|subagent|api`)
- `content`
- `media`
- `metadata`
- `stream_progress` flags

This removes split logic and duplicate code paths in message processing.

---

## 2) Collapse Session + Memory into a ConversationStore

Today:

- session history lives in JSONL
- long-term memory lives in `MEMORY.md` and `HISTORY.md`
- consolidation has extra scheduling/lock state

Lean model:

- Introduce one `ConversationStore` interface:
  - `load_context(session_key, window)`
  - `append_turn(session_key, messages)`
  - `checkpoint(session_key)`
  - `summarize_if_needed(session_key)`
- Keep existing files for compatibility, but hide behind one API.

Result:

- no memory logic in loop layer
- no direct file format assumptions in orchestration
- easier future switch to SQLite without touching core flow

---

## 3) Unify Background Work under a TurnRunner

Today, there are separate control paths for:

- spawned subagents
- cron execution
- heartbeat execution
- foreground dispatch tasks

Lean model:

- One `TurnRunner` queue that accepts `TurnRequest` directly (no `Job` wrapper)
- Use `request.trigger` to carry source semantics (`manual|cron|heartbeat|subagent|api`)
- Subagent becomes a policy (`mode=isolated_tools`) instead of a separate orchestration framework.
- All completions are emitted as `TurnRequest(kind=system)` back into the same queue.

Result:

- fewer custom cancellation maps
- one retry policy
- one place for task status/metrics

---

## 4) Replace Tool Context Mutation with Explicit Call Context

Today tools are mutated each turn via `set_context(...)`.  
That creates hidden state and coupling.

Lean model:

- `ToolRegistry.execute(name, params, call_context)`
- `call_context` includes channel/chat/message/session and permissions.
- tools become stateless where possible.

Result:

- less implicit state
- easier testability
- safer concurrent execution

---

## 5) Minimize Loop Surface Area

Keep the LLM-tool iteration loop very small:

- `prepare_messages`
- `provider.chat`
- `execute_tool_calls`
- `finalize_response`

Move everything else out:

- slash commands -> `CommandHandler`
- progress streaming -> `TurnHooks`
- persistence/memory -> `ConversationStore`
- task control -> `TurnRunner`

Result:

- the "core loop" becomes understandable in one screen

---

## 6) Remove Ambiguous Modes, Keep Policies

Instead of multiple hardcoded behaviors in the loop, define policies:

- `ToolPolicy` (allowlist, workspace restrictions, timeout profile)
- `MemoryPolicy` (window, summarize threshold)
- `ProgressPolicy` (emit none/text/tool-hints)

Policies are config data, not branch-heavy code paths.

---

## 7) Strong Simplicity Rules

Adopt explicit constraints:

- No per-feature lock collections in loop layer
- No ad-hoc synthetic message formats outside one adapter
- No tool-side global mutable context
- Every async task must have:
  - owner session key
  - cancellation scope
  - status record

---

## Suggested Target Architecture

Minimal components:

- `TurnEngine` (single orchestration path)
- `ToolRegistry` (stateless execution + schemas)
- `ConversationStore` (history + memory + summarize)
- `TurnRunner` (foreground/background scheduler)
- `ChannelGateway` (adapters in/out)
- `ProviderClient` (LLM API adapter)

If a feature does not fit into one of those six, it is likely cross-cutting noise.

---

## Migration Plan (Low Risk)

1. **Extract TurnEngine without behavior changes**
   - Wrap existing `_process_message`/`_run_agent_loop` behind `TurnEngine.run(request)`.
2. **Introduce ConversationStore facade**
   - Internally call existing `SessionManager` + `MemoryStore`.
3. **Add explicit ToolCallContext**
   - Keep `set_context` temporarily; populate from context object for compatibility.
4. **Unify system/direct paths**
   - Convert system and direct handlers into `TurnRequest` producers.
5. **Create TurnRunner facade**
   - Move active-task maps and cancellation there; queue raw `TurnRequest`.
6. **Flatten loop**
   - Keep only iteration logic in loop class; delegate everything else.
7. **Delete dead compatibility branches**
   - remove redundant lock/task maps once facades are fully adopted.

---

## What Gets Simpler Immediately

- One place to understand "how a message becomes a response"
- One place to reason about cancellation
- One way to do persistence and memory rollover
- Easier unit tests (engine/store/runner can be mocked cleanly)
- Better confidence changing features without side effects

---

## Lean Pseudocode (Whole Core)

```text
on_inbound_event(event):
  request = TurnRequest.from_event(event)
  turn_runner.submit(request)


TurnRunner.worker():
  while running:
    request = queue.get()
    with cancellation_scope(request.session_key):
      response = turn_engine.run(request)
      if response.emit_to_user:
        channel_gateway.send(response.outbound)


TurnEngine.run(request):
  command_reply = command_handler.try_handle(request)
  if command_reply:
    store.append_turn(request.session_key, command_reply.trace)
    return command_reply

  ctx = store.load_context(request.session_key, memory_policy.window)
  prompt = context_builder.build(ctx, request)

  trace = []
  for i in range(max_tool_iterations):
    llm = provider.chat(prompt.messages, tool_registry.schemas(), model_settings)
    trace.append(llm.assistant_message)

    if llm.tool_calls:
      for call in llm.tool_calls:
        result = tool_registry.execute(call.name, call.args, ToolCallContext.from_request(request))
        trace.append(tool_result_message(call, result))
      prompt = prompt.with_trace(trace)
      hooks.on_progress(request, llm, trace)
      continue

    final = llm.final_text_or_fallback()
    trace.append(final_message(final))
    break

  store.append_turn(request.session_key, trace)
  store.summarize_if_needed(request.session_key)
  return Outbound.from_final(request, final, trace)
```

---

## Bottom Line

The leanest version of nanobot is not "fewer features"; it is **fewer orchestration concepts**.  
Unify all turn sources into one engine, hide persistence behind one store, and run everything through one turn runner.  
That gives a much smaller mental model without sacrificing capability.
