# Agent Runner & Codex App-Server

`SymphonyElixir.AgentRunner` encapsulates the lifecycle of a single Linear issue execution. It orchestrates workspace setup, hook execution, repeated Codex turns, and failure handling while keeping the orchestrator informed via messages.

## Worker host selection & workspace hooks

`AgentRunner.run/3` receives an issue, the orchestrator PID, and options such as `worker_host`. It builds the ordered list of candidate worker hosts via `candidate_worker_hosts/2`, preferring the requested host when possible. The runner tries each host sequentially until one succeeds, logging why remote hosts fail.

For each attempt it calls `Workspace.create_for_issue/2` to get a safe workspace, sends runtime info back to the orchestrator (`{:worker_runtime_info, issue_id, %{worker_host, workspace_path}}`), then executes `hooks.before_run` through `Workspace.run_before_run_hook/3`. After the Codex turn completes (even on errors) it always calls `Workspace.run_after_run_hook/3`.

## Codex turns & continuation logic

`run_codex_turns/5` spawns a Codex app-server session using `SymphonyElixir.Codex.AppServer.start_session/2`. The session parameters include `workspace`, `worker_host`, `Config.codex_runtime_settings/2`, and deduced sandbox policies. `do_run_codex_turns/8` constructs a prompt via `PromptBuilder.build_prompt/2` for turn 1 and appends a continuation notice for later turns (see `build_turn_prompt/4`). It limits consecutive turns to `Config.settings!().agent.max_turns`.

After each `AppServer.run_turn/4`, the runner asks `continue_with_issue?/2` whether the issue is still in an active state (`Config.settings!().tracker.active_states`). If the issue remains active and the turn limit hasn't been reached, it logs continuation and loops; otherwise it returns `:ok` so the orchestrator can schedule retries or finish.

Codex turn updates send messages (`{:codex_worker_update, issue_id, message}`) that include structured metadata: event type, timestamps, usage, rate limits, and human-readable summaries. These updates feed into the orchestrator’s token counters.

## Codex app-server handshake

`SymphonyElixir.Codex.AppServer` implements a minimal JSON-RPC client over `Port`. `start_session/2` first validates the workspace path (local or remote) via `PathSafety.canonicalize/1`, then spawns `bash -lc <codex command>` (or uses SSH for remote hosts) and sends `initialize`, `initialized`, `thread/start`, and `turn/start`. Thread and turn IDs become part of the `session_id` that the orchestrator logs.

`run_turn/4` starts a turn, emits `:session_started`, and enters `receive_loop/6` which reads newline-delimited JSON from the port. It handles these cases:

- `turn/completed`: logs success and returns `{:ok, result}`.
- `turn/failed` or `turn/cancelled`: emits events (`:turn_failed`, `:turn_cancelled`) and returns `{:error, reason}`.
- `item/tool/call`, `item/commandExecution/requestApproval`, `execCommandApproval`, etc.: the client either auto-approves (`result = %{"decision" => ...}`) or replies to tool calls using `SymphonyElixir.Codex.DynamicTool`.
- Non-JSON or unexpected lines: logs them (warning when they look like errors) and continues.

Requests are auto-approved when `auto_approve_requests` is `true` (which happens when `approval_policy == "never"`). When a tool requests user input and the session is non-interactive, `ToolRequestUserInput` prompts are answered with `This is a non-interactive session...`.

`AppServer` also enforces timeouts: `await_response/3` uses `Config.settings!().codex.read_timeout_ms` when waiting for JSON replies, and `receive_loop/6` uses `codex.turn_timeout_ms` to cancel hung turns.

## Dynamic tool support

`SymphonyElixir.Codex.DynamicTool.tool_specs/0` exposes the `linear_graphql` tool with an input schema that requires a `query` string and optional `variables` map. When Codex calls the tool, the adapter validates the arguments, runs `SymphonyElixir.Linear.Client.graphql/3`, and inspects the HTTP response. Success responses become structured `dynamic tool` outputs; GraphQL errors cause the tool to report `success=false` with the error details.

The dynamic tool also reports structured failure reasons for missing queries, invalid `variables`, missing Linear auth, or network errors. This makes it possible to instrument raw Linear GraphQL calls from within Codex turns without leaving the Symphomy execution context.
