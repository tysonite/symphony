# Token Accounting

Symphony tracks Codex usage through the orchestrator snapshot data so the terminal dashboard, HTTP API,
and Prometheus-friendly logging can report how many tokens were spent and how fast they are being
consumed. The calculations live inside `SymphonyElixir.Orchestrator` and `SymphonyElixir.StatusDashboard`,
and the orchestrator's strategy is deliberate: it trusts the absolute totals Codex streams (`thread/tokenUsage/updated`),
guards against out-of-order deltas, and only falls back to turn-completed usage when no snapshot is available.

## Absolute totals in the orchestrator

`SymphonyElixir.Orchestrator.State` keeps `:codex_totals` (input/output/total/seconds) plus per-running-entry
fields such as `:codex_total_tokens` and `:codex_last_reported_total_tokens`. Each Codex event flows through
`integrate_codex_update/2`, which calls `extract_token_delta/2`. That helper does three things:

1. It probes the update payload for absolute usage maps by calling `absolute_token_usage_from_payload/1`.
   That function walks every payload path that Codex uses for `thread/tokenUsage/updated` (for example
   `[:params, :tokenUsage, :total]` and `[:tokenUsage, :total]`) and picks the first map whose values look
   like integers named `input_tokens`, `output_tokens`, and `total_tokens`.
2. `compute_token_delta/4` compares the newly observed total against the running entry's `:codex_last_reported_*`
   values. The function only emits a positive delta when the absolute total is greater than or equal to
   the previously reported value, and it records the new high-water mark in the reported slot so the same
   figure is not counted twice.
3. If no absolute total is present, `extract_token_usage/1` falls back to `turn_completed_usage_from_payload/1`.
   This helper handles the `turn/completed` payload by reading its `usage` map. These turn completions are
   the least trusted source and are only used when nothing better is streaming.

`integrate_codex_update/2` sums the deltas into both the running entry fields and the global `:codex_totals` via
`apply_codex_token_delta/2`/`apply_token_delta/2`. Those helpers keep every counter non-negative and accumulate
`seconds_running` whenever a session completes (`record_session_completion_totals/2`). This approach guarantees
that dashboards only see monotonically increasing totals even if Codex resends older events.

## Preferred Codex payloads

The orchestrator's payload probes encode the same preference order expressed in the Codex reference docs:

- `thread/tokenUsage/updated` is the canonical live notification. Its `tokenUsage.total` map is used whenever
  `absolute_token_usage_from_payload/1` can find it, so the orchestrator names it the primary source of truth.
- `codex/event/token_count` carries a nested `info.total_token_usage`. When `thread/tokenUsage/updated` is missing,
  the orchestrator still recognizes this structure thanks to the same `absolute_paths` lookup list.
- `turn/completed` usage maps are parsed via `turn_completed_usage_from_payload/1` and treated as best-effort
  backups. Code paths such as `SymphonyElixir.AgentRunner` ensure a session resets counters between attempts,
  so the turn-completed payload is not used to compute totals unless nothing else is available.

Because every delta is keyed on `thread_id`/`turn_id`, the orchestrator treats multiple turns on the same thread as
continuations instead of resets. The `3.1` helpers keep `codex_last_reported_*` values aligned with the latest
absolute total before adding the delta to `codex_input_tokens`, `codex_output_tokens`, and `codex_total_tokens`.

## Dashboard & API surfaces

The terminal dashboard and Phoenix API both call `SymphonyElixir.Orchestrator.snapshot/0` (and the `snapshot/2`
helper), which already exposes `state.codex_totals` plus running-entry token counters. `SymphonyElixir.StatusDashboard`
samples those totals in `snapshot_with_samples/2`; each render ticks through `snapshot_total_tokens/1` to compute
the latest absolute `total_tokens` value and stores `(timestamp, total_tokens)` pairs in the `:token_samples` list
via `update_token_samples/3`. When the dashboard paints the TPS graph it calls `tps_graph/3`/`rolling_tps/4`, which
calculate deltas from the oldest sample within the window and divide by the window length.

Events are humanized through `token_usage_paths/0`, which reuses the same sequence of payload keys as the
orchestrator. `humanize_codex_method/2` and `humanize_codex_wrapper_event/2` inspect the `token_usage_paths`
list to render text such as "thread token usage updated (input=123 output=456)". The dashboard therefore shows
the most recent `thread/tokenUsage/updated` totals and the derived TPS graphs while ignoring `last`/delta-only
fields to avoid double counting. Meanwhile, the LiveView and API controllers (`SymphonyElixirWeb.Presenter`) report
`codex_totals` directly, ensuring the JSON surface matches the terminal output.

Every time the orchestrator mutates tokens or rate limits, it calls `SymphonyElixir.StatusDashboard.notify_update/0`
and publishes the new snapshot via `ObservabilityPubSub`, so both the terminal dashboard and API stay synced.

## Key takeaways

- The orchestrator prefers Codex absolute totals (`tokenUsage.total`, `info.total_token_usage`) and uses `compute_token_delta/4`
  to avoid counting the same data twice.
- When only turn-completed usage is available, it is treated as a fallback and not aggregated twice.
- `SymphonyElixir.StatusDashboard` samples the `codex_totals`, uses `token_usage_paths/0` to display the event text,
  and derives TPS from `token_samples`.
- The HTTP API (`SymphonyElixirWeb.Presenter`) reuses the orchestrator snapshot, guaranteeing that all observability
  surfaces show the same absolute token totals.
