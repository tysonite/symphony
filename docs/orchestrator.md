# Orchestrator & Dispatch

`SymphonyElixir.Orchestrator` is the core GenServer that polls the tracker, claims issues, tracks running Codex sessions, and schedules retries. It maintains the `State` struct (`poll_interval_ms`, `max_concurrent_agents`, `running`, `claimed`, `retry_attempts`, `codex_totals`, `codex_rate_limits`) and updates it every tick while pushing snapshots to `SymphonyElixir.StatusDashboard`.

## Tick loop & poll cycle

On init the orchestrator runs `run_terminal_workspace_cleanup/0`, then calls `schedule_tick(state, 0)` to kickoff the first poll cycle right away. `handle_info(:tick)` refreshes `Config.settings!()` and marks the poll check in progress; it then renders the dashboard via `notify_dashboard/0` and schedules `:run_poll_cycle` after a short delay (`@poll_transition_render_delay_ms`). The poll cycle handler (`:run_poll_cycle`) revalidates the workflow config, reconciles running issues, and calls `maybe_dispatch/1`. After each cycle it schedules the next tick with `schedule_tick(state.poll_interval_ms)` and clears the `poll_check_in_progress` flag.

`maybe_dispatch/1` first reconciles running issues (see below). It then validates the workflow config (`Config.validate!/0`), pulls candidate issues from `Tracker.fetch_candidate_issues/0`, and bails out early when there are no available orchestrator slots or when validations/logs hit config errors (missing tracker kind, missing Linear project slug, YAML parse failures, authentication failures). When conditions look good it delegates to `choose_issues/2`.

## Candidate filtering & dispatch

Issue ordering is resolved by `sort_issues_for_dispatch/1`, which sorts each issue by priority (1–4 with 5 as default), creation time, and identifier. `should_dispatch_issue?/4` enforces the full set of eligibility checks before spawning `AgentRunner`:

- Config-defined active/terminal state sets (`active_state_set/0`, `terminal_state_set/0`) gate which states can run.
- `todo_issue_blocked_by_non_terminal?/2` prevents dispatching a `Todo` issue while it still has blockers outside terminal states.
- `state_slots_available?/2` respects `Config.max_concurrent_agents_for_state/1` and tracks running issue counts keyed by normalized state.
- `available_slots/1` applies the global `agent.max_concurrent_agents` cap.
- `worker_slots_available?/1` ensures SSH worker capacity (see below).

`dispatch_issue/4` re-fetches the issue via `revalidate_issue_for_dispatch/3` before handing off to `spawn_issue_on_worker_host/4`. `Task.Supervisor` runs `AgentRunner.run/3`, and the orchestrator monitors that task via `Process.monitor/1`, storing metadata in `state.running`. If the task spawn fails, the orchestrator immediately schedules a retry using `schedule_issue_retry/4`.

## Running issue reconciliation & cleanup

`reconcile_running_issues/1` first calls `reconcile_stalled_running_issues/1`, enforcing `Config.settings!().codex.stall_timeout_ms` by restarting stalled sessions. Afterwards it refreshes running issue states (`Tracker.fetch_issue_states_by_ids/1`). Each issue goes through `reconcile_issue_state/4`: terminal states clean up workspaces (`terminate_running_issue/3`), non-active states stop the agent without cleanup, and active states update the stored issue struct.

When a worker exits normally (`:DOWN` with `:normal`), `handle_info({:DOWN, ...})` completes the issue, records codex token totals, and issues a continuation retry with `@continuation_retry_delay_ms`. Abnormal exits increment attempt counts (`failure_retry_delay/1` multiplies by powers of two capped by `agent.max_retry_backoff_ms`) and keep workspace metadata (`workspace_path`, `worker_host`) for the next retry. Workspace cleanup is centralized in `cleanup_issue_workspace/2` which delegates to `SymphonyElixir.Workspace.remove_issue_workspaces/2`.

## Retry/backoff & worker host selection

Retry state sits in `state.retry_attempts`. `schedule_issue_retry/4` records metadata, cancels existing timers, and sends `{:retry_issue, issue_id, retry_token}` after the calculated delay. When a retry fires, `handle_info({:retry_issue, issue_id, retry_token})` pops the entry and either finds the issue in `Tracker.fetch_candidate_issues/0` (dispatching if slots are free) or requeues with explanatory error metadata.

When SSH hosts are configured (`Config.settings!().worker.ssh_hosts`), `select_worker_host/2` picks the least-loaded host (`running_worker_host_count/2`) or preserves a previously preferred host when available. Host-level caps come from `worker.max_concurrent_agents_per_host`. If no hosts are configured, local execution (`worker_host = nil`) is assumed.

## Snapshot & status integration

`SymphonyElixir.Orchestrator.snapshot/2` exposes running entries, retry queue rows, codex totals, and polling state for the status dashboard. Each running entry includes token counters, workspace paths, session IDs, and last Codex event data. `integrate_codex_update/2` merges dynamic updates such as token deltas, usage reported via several payload keys, and rate-limit snapshots. `apply_codex_token_delta/2` keeps aggregated `state.codex_totals` for throughput graphs, while `apply_codex_rate_limits/2` retains the latest throttle info shown by `SymphonyElixir.StatusDashboard`.

`SymphonyElixir.StatusDashboard.notify_update/0` is called after every orchestrator mutation that matters (tick, worker updates, retry scheduling), ensuring the terminal UI stays in sync with `StatusDashboard` and the HTTP API (see `SymphonyElixir.HttpServer`).
