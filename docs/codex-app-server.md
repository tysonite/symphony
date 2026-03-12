# Codex App-Server Client

`SymphonyElixir.Codex.AppServer` wraps the `codex app-server` JSON-RPC stream so every turn behaves like a safe, observable service call instead of a raw shell process. The module bootstraps the thread/turn handshake, marshals newline-delimited JSON over a Port, and forwards Codex feedback to the orchestrator via structured events.

## Session bootstrapping & workspace validation

`start_session/2` first validates the workspace path. When `worker_host` is `nil` it canonicalizes the directory with `PathSafety.canonicalize/1`, ensures it stays under `Config.settings!().workspace.root`, and rejects symlink escapes. Remote attempts skip canonicalization but still guard against empty or newline-containing paths. Local launches run `bash -lc <codex command>` inside the workspace, while remote launches invoke `SymphonyElixir.SSH.start_port/3` with a `cd <workspace> && exec <codex command>` pipeline built by `remote_launch_command/1`. Shared metadata such as `codex_app_server_pid`, the selected `worker_host`, and workspace path get emitted via `port_metadata/2` so dashboards and retry logic can tie events to the right host.

`Config.codex_runtime_settings/2` supplies every session with the configured `approval_policy`, `thread_sandbox`, and `turn_sandbox_policy`. Remote hosts pass `remote: true` so sandbox policies trust the remote workspace root without looking at local symlinks.

## Turn lifecycle & event routing

`run_turn/4` calls `start_turn/7`, which sends `thread/start`/`turn/start` message pairs and wires the dynamic tool list returned by `SymphonyElixir.Codex.DynamicTool.tool_specs/0` into the handshake. When `start_turn/7` returns a turn ID, `run_turn/4` logs the `session_id` (`thread_id-turn_id`) and notifies the orchestrator through `emit_message/4` before entering `receive_loop/6`.

`receive_loop/6` concatenates partial JSON fragments until newline boundaries and feeds each line into `handle_incoming/7`. Completed turns, failures, and cancellations are translated into `:turn_completed`, `:turn_failed`, and `:turn_cancelled` events. Payloads that request approvals (`item/commandExecution/requestApproval`, `execCommandApproval`, `applyPatchApproval`, `item/fileChange/requestApproval`) auto-approve when `auto_approve_requests` is `true` (which happens when the configured approval policy equals `"never"`). Tool calls (`item/tool/call`) forward arguments to the injected `tool_executor` (defaulting to `DynamicTool.execute/3`) and reply with the normalized tool result.

The client also handles `item/tool/requestUserInput` by first checking whether it can supply canned answers that approve the session. If not, it responds with the fixed non-interactive message `"This is a non-interactive session. Operator input is unavailable."` so Codex never blocks waiting for a human. Notifications, unexpected methods, and non-JSON lines are either logged (`log_non_json_stream_line/2`) or re-emitted as `:notification`/`:malformed` events so debugging tools can trace stray output.

## Timeouts, approvals, and metadata

`await_turn_completion/6` and `with_timeout_response/4` use `Config.settings!().codex.turn_timeout_ms` and `read_timeout_ms` respectively, ensuring a hung turn or stalled RPC throws `:turn_timeout`/`:response_timeout`. When the port exits, the helper returns `{:error, {:port_exit, status}}` so the orchestrator can schedule a retry.

Every event bubbled through `emit_message/4` merges static metadata (workspace, worker host, PID) with `payload`/`details` maps and adds timestamps. That metadata feeds `SymphonyElixir.Orchestrator.integrate_codex_update/2` so the orchestrator can accumulate token totals, rate-limit info, and last-reported events tied to the right session.
