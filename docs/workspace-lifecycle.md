# Workspace Lifecycle & Hooks

`SymphonyElixir.Workspace` keeps one isolated filesystem workspace per issue so Codex runs can execute in parallel without interfering. The module handles safe path computation, creation, hook execution, remote workspaces, and cleanup.

## Workspace creation

Users call `Workspace.create_for_issue(issue, worker_host)` as soon as `AgentRunner` wants to bootstrap a run. Internally the module:

1. Normalizes the issue identifier via `safe_identifier/1` (non-alphanumeric characters become `_`).
2. Derives the workspace path from `Config.settings!().workspace.root` and the safe identifier, then canonicalizes it with `PathSafety.canonicalize/1` to prevent symlink escape.
3. Runs `File.dir?`/`File.exists?` checks, removing stale files when necessary, and returns `{created?, path}` from `ensure_workspace/2`.
4. Invokes `hooks.after_create` only when `created?` is `true`; this hook runs through `run_hook/4` which shells out with `sh -lc` and the configured `hooks.timeout_ms`.

When a remote worker host is selected (`worker_host` is a binary), the same behavior happens over SSH: `ensure_workspace/2` builds a script that reports a tab-delimited marker (`__SYMPHONY_WORKSPACE__	created	path`) so the orchestrator knows whether the directory was just created. Remote command execution uses `SymphonyElixir.SSH.run/3` and `SSH.start_port/3` under the hood, with `SYMPHONY_SSH_CONFIG` controlling the `-F` flag.

## Hooks surrounding Codex work

All hooks live in the workflow schema (`hooks.after_create`, `hooks.before_run`, `hooks.after_run`, `hooks.before_remove`). `AgentRunner` calls `Workspace.run_before_run_hook/3` before each turn attempt and `Workspace.run_after_run_hook/3` afterward; both commands respect `hooks.timeout_ms` and log results. Hook execution uses `System.cmd/3` locally or remote SSH commands when `worker_host` is provided. Failures in `after_run` and `before_remove` are logged but ignored so cleanup continues; hooks can return truncated output for logging.

`run_before_run_hook/3`/`run_after_run_hook/3` also forward the issue context (ID + identifier) into log lines to aid debugging.

## Cleanup & removal

Workspaces are removed by `Workspace.remove/2` and `Workspace.remove_issue_workspaces/2`. Local removal validates the path again to prevent touching the workspace root directly. Remote removal runs `rm -rf` over SSH and reports errors with hook-style metadata. The orchestrator calls `remove_issue_workspaces/2` when a terminal state is detected or on startup via `run_terminal_workspace_cleanup/0`.

`Workspace.remove_issue_workspaces/2` iterates over every configured SSH host (from `Config.settings!().worker.ssh_hosts`) and deletes the per-issue directory on each host.

## Path safety & validation

`validate_workspace_path/2` enforces that a workspace path is not the workspace root itself, escapes via symlinks, or lives outside the configured root. It uses `PathSafety.canonicalize/1` to resolve the path and compares against the canonical root. Remote validation is simpler: it ensures the path is non-empty and contains no newline/NULL characters.

All shell commands that interpolate workspace paths use `shell_escape/1` or `remote_shell_assign/2` (which resolves `~`/`~/` to `$HOME`). This keeps remote scripts portable when `hooks.after_create` clones a repo or populates dependencies.
