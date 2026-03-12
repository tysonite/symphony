# Workflow Configuration & Prompt Generation

Symphony treats `WORKFLOW.md` as the single canonical source for configuration and agent prompts. The Elixir implementation keeps the file in sync via `SymphonyElixir.Workflow` and `SymphonyElixir.WorkflowStore`, materializes its front matter as typed settings, and renders the Markdown body through the Solid template engine before every Codex turn.

## Workflow loader & live reload

`SymphonyElixir.Workflow` reads `WORKFLOW.md` from `Application.get_env(:symphony_elixir, :workflow_file_path)` or falls back to `./WORKFLOW.md`. It splits YAML front matter from the Markdown body, parses the YAML with `YamlElixir`, rejects non-map front matter, and trims the prompt template before publishing `%{config: map, prompt: string, prompt_template: string}`. Every read is wrapped with descriptive error tuples such as `{:missing_workflow_file, path, reason}` or `{:workflow_parse_error, reason}` so the caller can log the exact failure.

`SymphonyElixir.WorkflowStore` caches the last known good payload and polls the file every second (see `@poll_interval_ms`). It keeps the cached state whenever a reload fails, logging `Failed to reload workflow path=...; keeping last known good configuration`. Both periodic polls (`handle_info(:poll, ...)`) and explicit calls to `WorkflowStore.force_reload/0` ultimately call `Workflow.load/1`, so `set_workflow_file_path/1` simply swaps the env var and forces the store to refresh.

## Config schema & defaults

`SymphonyElixir.Config.settings/0` loads the latest workflow config, normalizes keys, strips `nil` entries, and applies a nested `Ecto.Schema` defined in `SymphonyElixir.Config.Schema`. That schema embeds sections for:

- `tracker` (`kind`, `endpoint`, `api_key`, `project_slug`, `assignee`, `active_states`, `terminal_states`) with defaults such as `https://api.linear.app/graphql` and multi-valued state lists.
- `polling` (`interval_ms`, default `30_000`).
- `workspace` (`root`, default `<system_tmp>/symphony_workspaces`).
- `worker` (`ssh_hosts`, `max_concurrent_agents_per_host`).
- `agent` (`max_concurrent_agents`, `max_turns`, `max_retry_backoff_ms`, `max_concurrent_agents_by_state`) with helpers for `normalize_state_limits/1` and `validate_state_limits/2`.
- `codex` (`command`, `approval_policy`, `thread_sandbox`, `turn_sandbox_policy`, `turn_timeout_ms`, `read_timeout_ms`, `stall_timeout_ms`) with stricter validations such as required `command` and positive timeouts.
- `hooks` (`after_create`, `before_run`, `after_run`, `before_remove`, `timeout_ms`).
- `observability` (dashboard enablement, refresh/render intervals).
- `server` (`port`, `host`).

After casting, `Config.Schema.finalize_settings/1` resolves secret references (`$LINEAR_API_KEY`, `WORKFLOW.md` fields like `tracker.api_key`/`tracker.assignee`) and path overrides. `resolve_path_value/2` expands `~`, honors `$VAR` tokens, and falls back to a safe temp directory. When the pipeline needs sanitized workspace roots, `PathSafety.canonicalize/1` is used to resolve symlinks before handing the value to Codex.

`SymphonyElixir.Config` exposes helpers such as `settings!/0`, `max_concurrent_agents_for_state/1`, `codex_runtime_settings/2`, and `server_port/0`. Validation through `validate!/0` enforce presence of `tracker.kind`, supported tracker kinds (`linear` or `memory`), and Linear credentials (`tracker.api_key`, `tracker.project_slug`) before the orchestrator dispatches work. `codex_turn_sandbox_policy/1` and `resolve_runtime_turn_sandbox_policy/3` either reuse an explicit `turn_sandbox_policy` map or construct a workspace-restricted `workspaceWrite` policy with `excludeTmpdirEnvVar`/`excludeSlashTmp` disabled.

## Prompt rendering

`SymphonyElixir.PromptBuilder.build_prompt/2` pulls the current prompt template from `Workflow.current/0`, defaults to `Config.workflow_prompt/0` when the body is blank, and parses the template with `Solid.parse!/1`. Rendering then passes a map containing the issue (converted via `Solid`-friendly transformations in `to_solid_value/1`) and the optional `attempt` number. The template uses Liquid-style placeholders and runs with `strict_variables: true` and `strict_filters: true`, so any missing field raises a `template_parse_error` at render time. Continuation turns rely on the same prompt but append a helper blurb describing that the previous turn completed normally and the issue is still active.
