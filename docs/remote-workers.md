# Remote Worker Hosts & SSH Orchestration

Symphony can launch agents on remote machines over SSH so a single orchestrator process can dispatch work across a fleet instead of running every issue locally. The feature wires together `SymphonyElixir.AgentRunner`, `SymphonyElixir.Orchestrator`, `SymphonyElixir.Workspace`, `SymphonyElixir.Codex.AppServer`, and `SymphonyElixir.SSH` through the `worker` section of the workflow config.

## Worker host configuration & selection

The `WORKFLOW.md` front matter exposes `worker.ssh_hosts` and `worker.max_concurrent_agents_per_host` through the `SymphonyElixir.Config.Schema.Worker` embedded schema. When `ssh_hosts` is empty, runs stay local (`worker_host = nil`); otherwise each agent attempt tries those hosts in order.

`SymphonyElixir.AgentRunner.candidate_worker_hosts/2` builds the host list: it trims, deduplicates, and respects a `preferred_host` option (passed by the orchestrator). The orchestrator maintains running state metadata via `SymphonyElixir.Orchestrator.State.running`, so `select_worker_host/2` can tell whether a host still has room (`worker_host_slots_available?/2`) or should be skipped because it already hit `worker.max_concurrent_agents_per_host`.

If a preferred host is still available it is reused; otherwise the orchestrator picks the least-loaded host by counting how many running entries already claim it (`running_worker_host_count/2`). When no hosts have capacity, `select_worker_host/2` returns `:no_worker_capacity` and `worker_slots_available?/2` prevents new dispatches.

Each successful `AgentRunner` call reports its chosen host and workspace path via `send_worker_runtime_info/4`, so the orchestrator and dashboards can show where each Codex session is running.

## Workspace lifecycle over SSH

`Workspace.create_for_issue/2` derives a safe identifier, joins it with `Config.settings!().workspace.root`, and, for remote hosts, skips canonicalizing the path (remote filesystems are only verified for emptiness and invalid characters). The remote branch of `ensure_workspace/2` shells out over SSH:

1. `remote_shell_assign/2` ensures `~` expands to `$HOME` so workflow hooks can rely on normalized paths.
2. The script creates or recreates the directory and prints `__SYMPHONY_WORKSPACE__\t<created?>\t<path>` so `parse_remote_workspace_output/1` returns both the actual workspace path and whether it was newly created.
3. Hook execution (`after_create`, `before_run`, `after_run`, `before_remove`) runs through `run_remote_command/3`, which delegates to `SymphonyElixir.SSH.run/3` and enforces `hooks.timeout_ms`.

When a remote run finishes or an issue hits a terminal state, `Workspace.remove_issue_workspaces/2` iterates `Config.settings!().worker.ssh_hosts` and, for each host, executes `rm -rf <workspace>` via `run_remote_command/3`. Remote hooks and cleanup silently ignore failures so the orchestrator can continue cleanup on other hosts.

## Remote Codex sessions

`AgentRunner.run/3` passes the selected `worker_host` to `AppServer.start_session/2`. The session builder uses `validate_workspace_cwd/2` to skip canonicalization for remote paths and `start_port/2` to spawn SSH ports through `SymphonyElixir.SSH.start_port/3`. The remote command is built by `remote_launch_command/1` (`cd <workspace> && exec <codex command>`), so Codex always runs inside the prepared directory. `AppServer.start_session/2` also calls `Config.codex_runtime_settings(workspace, remote: true)` so `SymphonyElixir.Config.Schema.resolve_runtime_turn_sandbox_policy/3` trusts the remote root without inspecting symlinks locally.

If the session fails on one host the runner logs the reason, notifies the orchestrator, and continues with the next candidate host (see `run_on_worker_hosts/4`). When a session runs, `AppServer.run_turn/4` still streams the same Codex JSON-RPC protocol, and `send_codex_update/3` tags each update with `worker_host` so dashboards can associate events with the remote machine.

## SSH helper layer

`SymphonyElixir.SSH` encapsulates the low-level SSH plumbing. `run/3` locates the local `ssh` binary, adds `-T` (no tty), optionally injects a `-F <path>` flag when `SYMPHONY_SSH_CONFIG` points to a custom config file, and passes through any `:stderr_to_stdout` or `:line` options from the caller. `start_port/3` uses the same machinery but opens an Erlang port so Codex can stream newline-delimited JSON. Both helpers accept host shortcuts like `host:2222` and parse them so the real `ssh` command gets `-p 2222`.

Remote commands are wrapped in `remote_shell_command/1` before being sent over SSH, and every path that flows into those commands is sanitized by `shell_escape/1` to avoid injection.

Together, these utilities let Symphony keep worker hosts isolated, respect host-level capacity limits, and run both hooks and Codex sessions remotely with the same runtime config that drives local runs.
