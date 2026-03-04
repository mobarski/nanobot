# Nanobot Core Idea

The "core" of nanobot is an event-driven orchestration engine that turns incoming chat events into tool-augmented LLM turns, while preserving continuity through session history and long-term memory.

At a high level:

1. A message arrives on the internal bus.
2. The agent rebuilds context (identity + bootstrap docs + memory + recent session history + runtime metadata).
3. The LLM runs in a loop that can call tools.
4. Tool outputs are fed back into the same turn until a final assistant response is produced.
5. The turn is persisted to session storage, and old context is periodically consolidated into durable memory files.

The design keeps channels/providers/adapters outside the core and concentrates the core on these concerns:

- **Stateful reasoning loop** (`agent/loop.py`)
- **Prompt/context assembly** (`agent/context.py`)
- **Session persistence and replay window** (`session/manager.py`)
- **Memory consolidation** (`agent/memory.py`)
- **Tool abstraction + safe execution surfaces** (`agent/tools/*`)
- **Decoupled transport via async bus** (`bus/*`)
- **Background autonomy** (`agent/subagent.py`, `cron/*`, `heartbeat/*`)

## Core Architectural Pattern

The core follows a "single orchestrator + pluggable capabilities" pattern:

- `AgentLoop` is the orchestrator.
- `ToolRegistry` provides capabilities as JSON-schema functions.
- `ContextBuilder` controls what the LLM can see.
- `SessionManager` and `MemoryStore` control what the agent remembers.
- `MessageBus` decouples producer/consumer timing and channel implementations.

This produces a predictable control surface:

- Deterministic outer flow (receive -> build -> run -> persist -> respond)
- Nondeterministic inner reasoning (LLM + tools) bounded by `max_iterations`
- Asynchronous side systems (subagents, cron, heartbeat) routed back through the same core path

## What Makes This "Core"

These files define core behavior because they answer the essential questions:

- **How does work enter the system?** `MessageBus`, `InboundMessage`
- **How does a turn get processed?** `AgentLoop._process_message` and `_run_agent_loop`
- **How are capabilities exposed?** `Tool` interface + `ToolRegistry`
- **How does memory survive process restarts?** `SessionManager` JSONL + `memory/MEMORY.md` and `memory/HISTORY.md`
- **How does background/autonomous work rejoin the same flow?** subagent result injection as system messages, cron callbacks, heartbeat triggers

## End-to-End Core Pseudocode

```text
bootstrap():
  config = load_config()
  workspace = config.workspace_path
  ensure workspace templates/memory dirs exist

  bus = MessageBus()
  provider = select_llm_provider(config)
  sessions = SessionManager(workspace)
  cron = CronService(store_path=workspace/".internal"/"jobs.json", on_job=handle_cron_job)
  heartbeat = HeartbeatService(workspace, provider, model, on_execute=agent_process_direct, on_notify=notify_user)

  agent = AgentLoop(
    bus=bus, provider=provider, workspace=workspace,
    session_manager=sessions, cron_service=cron,
    model/max_tokens/temperature/memory_window/etc from config
  )
  register default tools in registry:
    read_file, write_file, edit_file, list_dir, exec, web_search, web_fetch, message, spawn, (cron if enabled)
  optionally connect MCP servers and wrap MCP tools

  start cron
  start heartbeat
  start agent.run()


agent.run():
  loop while running:
    msg = bus.consume_inbound(timeout=1s)
    if timeout: continue
    if msg.content == "/stop":
      cancel active tasks + cancel spawned subagents for msg.session_key
      bus.publish_outbound("stopped")
    else:
      create async task dispatch(msg)


dispatch(msg):
  with global processing lock:
    response = process_message(msg)
    if response exists:
      bus.publish_outbound(response)


process_message(msg):
  if msg.channel == "system":
    # subagent/cron/heartbeat callback path
    parse origin channel+chat
    session = sessions.get_or_create(origin_session_key)
    set tool routing context (channel/chat/message_id)
    history = session.get_history(memory_window)
    messages = ContextBuilder.build_messages(history, msg.content, runtime_metadata)
    final, tools_used, all_msgs = run_agent_loop(messages)
    save_turn(session, all_msgs)
    sessions.save(session)
    return outbound(origin_channel, origin_chat, final or default)

  # normal user path
  session = sessions.get_or_create(msg.session_key)
  handle slash commands:
    /new -> consolidate/archival then clear session
    /help -> return command list

  if unconsolidated history >= memory_window:
    schedule background memory consolidation

  set tool routing context
  history = session.get_history(memory_window)
  initial_messages = ContextBuilder.build_messages(
    system_prompt = identity + bootstrap files + memory + skills summary + always skills,
    history = history,
    user = runtime_context_tag + current message (+optional images)
  )

  final, tools_used, all_msgs = run_agent_loop(initial_messages, on_progress=stream_to_bus)
  if final is empty: final = fallback text

  save_turn(session, all_msgs)
  sessions.save(session)

  if message tool already sent user-facing output in this turn:
    return None
  return outbound(msg.channel, msg.chat_id, final)


run_agent_loop(messages):
  for iteration in 1..max_iterations:
    response = provider.chat(messages, tools=registry.definitions, model settings)

    if response has tool_calls:
      append assistant message with tool_calls
      optionally stream tool-hint/progress
      for each tool_call:
        result = registry.execute(tool_name, args)
          validate args against tool schema
          run tool async
          wrap validation/runtime errors as recoverable text hints
        append tool result message
      continue

    else:
      clean assistant content (strip think tags)
      append assistant message
      return final_content, tools_used, messages

  return "max iterations reached", tools_used, messages


save_turn(session, new_messages):
  drop empty assistant messages
  truncate oversized tool outputs for session storage
  strip injected runtime-context metadata from persisted user content
  replace inline data:image payloads with "[image]" marker
  append timestamps
  session.updated_at = now


memory_consolidation(session):
  old = unconsolidated messages except latest keep-window
  build compact transcript lines with timestamps + tools used
  ask provider to call virtual tool "save_memory" with:
    history_entry (for HISTORY.md append)
    memory_update (full MEMORY.md content)
  on success:
    write files
    advance session.last_consolidated checkpoint


subagent_flow(task):
  spawn background task with reduced toolset (no message/spawn recursion)
  run mini agent loop up to fixed iterations
  package result as synthetic system message
  bus.publish_inbound(system_message_to_origin_chat)
  # main agent summarizes naturally to user through normal path


cron_flow():
  persist jobs in JSON
  compute next run times ("at", "every", "cron expr + tz")
  on due job:
    execute callback (typically agent direct processing)
    update status/next run or delete one-shot jobs


heartbeat_flow():
  periodically read HEARTBEAT.md
  ask provider to emit structured decision via virtual "heartbeat" tool:
    action = skip|run, tasks summary
  if run:
    execute tasks through full agent path
    notify user via callback
```
