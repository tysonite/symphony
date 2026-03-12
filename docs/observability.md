# Observability & Dashboard

Symphony exposes its runtime health through three observability surfaces: the terminal status dashboard, a Phoenix-powered HTTP/server dashboard, and JSON APIs. Updates flow from `SymphonyElixir.Orchestrator` into `SymphonyElixir.StatusDashboard`, `SymphonyElixirWeb.ObservabilityPubSub`, and eventually the LiveView/API presenters.

## Terminal dashboard

`SymphonyElixir.StatusDashboard` is a GenServer that renders ANSI art to the local terminal. It polls every `refresh_ms` (default 1_000 ms) and throttles desktop rendering via `render_interval_ms`. Each render collects a snapshot of orchestrator state (`Orchestrator.snapshot/2`) and prints running issue counts, retry counts, codex totals, rate-limit objects, and token throughput sparklines. When the orchestrator updates (tick, worker event, retry, codex usage), `notify_update/0` broadcasts via `SymphonyElixirWeb.ObservabilityPubSub` so the dashboard can redraw immediately.

User-visible metrics include session IDs, token counters, `last_codex_event` text, and `turn_count`. The dashboard also tracks `last_snapshot_fingerprint` to skip identical renders and schedules flush timers (`:flush_render`) when renders are deferred.

## HTTP server & LiveView

`SymphonyElixir.HttpServer` (started when `Config.server_port/0` returns a non-negative integer) starts a Phoenix `Endpoint` bound to the configured IP/port/host. It merges runtime opts such as `orchestrator`, `snapshot_timeout_ms` (default 15 000 ms), and a random `secret_key_base` into the endpoint config stored under `:symphony_elixir` app env before calling `Endpoint.start_link/0`. When no port is configured, `start_link/1` returns `:ignore`.

`SymphonyElixirWeb.Router` mounts two scopes: one for static assets (`Dashboard.css`, `phoenix.js`, etc.) and another for the browser pipeline with `DashboardLive`. The LiveView (`SymphonyElixirWeb.DashboardLive`) subscribes to `ObservabilityPubSub` updates and polls for timestamp ticks every second. Its `render/1` template shows summary cards (running/retrying counts, total tokens, runtime), rate-limit dumps, and a table of running sessions with link to `/api/v1/:issue_identifier`.

`Endpoint` also wires the API routes (`/api/v1/state`, `/api/v1/refresh`, `/api/v1/:issue_identifier`) handled by `SymphonyElixirWeb.ObservabilityApiController`. Each action calls into `SymphonyElixirWeb.Presenter`, which formats snapshots and orchestrator request/refresh payloads.

## API & presenter payloads

`Presenter.state_payload/2` transforms `Orchestrator.snapshot/2` data into JSON with counts, running/retrying arrays, codex totals, and rate limits. `Presenter.issue_payload/3` looks up a specific issue by identifier and returns workspace metadata, attempt counts, and the last Codex event. `Presenter.refresh_payload/1` calls `Orchestrator.request_refresh/1` so clients can trigger an immediate poll cycle; the API returns `202 Accepted` with timing data when the orchestrator accepts the refresh, or `503` when it’s unavailable.

Errors during snapshot capture (timeout/unavailable) are surfaced as `snapshot_timeout`/`snapshot_unavailable` errors so the dashboard can display offline hints.

## Snapshot coordination

Both the terminal dashboard and Phoenix stack subscribe to `SymphonyElixirWeb.ObservabilityPubSub`. `StatusDashboard` returns the latest payload and `notify_update/0` broadcasts `:observability_updated` on `SymphonyElixir.PubSub`. LiveView handles that message by reloading presenter payloads, while the API endpoints query the orchestrator on demand. This design keeps the UI, API, and terminal renderer synchronized without direct coupling.
