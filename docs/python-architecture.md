# Python Architecture for the Symphony Rewrite

This document maps the BEAM-based reference implementation in `elixir/` to a Python-friendly architecture that fulfills the goals laid out in `SPEC.md`. The new implementation must re-create the same behavioral layers (config + workflow, orchestrator, agent/worker, logging, and observability) with clear package boundaries so downstream implementation tickets can rely on solid APIs.

## Project layout

- `pyproject.toml` (Poetry) or `setup.cfg`/`requirements.txt` entry point that declares package metadata, pinned dependency versions, and optional extras (`observability`, `ssh`, `dev`).
- `WORKFLOW.md` (root) remains the canonical workflow contract: YAML front matter for config plus a Markdown prompt body, exactly like the BEAM reference and the `elixir/WORKFLOW.md` example.
- `docs/python-architecture.md` (this file) and future docs for observability, onboarding, and workflow tuning.
- `symphony/` (main package)
  - `__init__.py` exposes the default entry point, `__version__`, and shared types.
  - `cli.py` starts the service, wires logging, and launches the `OrchestratorSupervisor` with CLI flags such as `--workflow`, `--port`, and `--logs-root`.
  - Subpackages described below for config, workflow, tracker, orchestrator, agent, workspace, codex, logging, and observability.
- `tests/` follows `pytest` conventions with `unit/` and `integration/` suites that exercise config parsing, scheduler behavior, and agent/workspace hooks.
- `scripts/` (optional) contains helper scripts for lint, formatting, and symbolic data generation (for example, a `generate-workflow-template.py`).

## Package boundaries and APIs

| Package | Responsibility | Key APIs / Notable collaborators |
| --- | --- | --- |
| `symphony.config` | Typed view of `WORKFLOW.md` front matter plus environment overrides, validation, and defaults (poll interval, tracker credentials, agent limits, hooks, worker hosts, observability settings). | `Config.load(workflow_path)` returns a `Config.Settings` dataclass; `config.tracker`, `.agent`, `.hooks`, `.workspace`, `.observability` are strongly typed. | 
| `symphony.workflow` | Raw loader and cache for the workflow prompt template and config map (mirrors `Workflow`/`WorkflowStore`). | `WorkflowLoader(path)` provides `/reload()` and `.current()`; `.split_front_matter()`, `.render_prompt(issue)` (backed by Jinja2 with strict variables). | 
| `symphony.prompt` | Solid-like prompt rendering that turns an `Issue` dataclass into a dict, coerces `datetime` objects to ISO strings, and renders `attempt` and `issue` placeholders. | `PromptBuilder.build(issue, attempt=None)` and `PromptBuilder.default()` fallback to the default prompt. | 
| `symphony.tracker` | Adapter boundary with a `TrackerAdapter` interface plus a `LinearTracker` implementation powered by `httpx`/GraphQL. | `TrackerAdapter.fetch_candidates()`, `fetch_states(ids)`, `create_comment(...)`, `update_state(...)`; the `LinearAdapter` caches state IDs and reads the `active_states`/`terminal_states` lists from config. | 
| `symphony.orchestrator` | Poll loop, eligibility decisions, concurrency tracking, retry queue, and continuation/backoff semantics discussed in `SPEC 3.1` and captured by `SymphonyElixir.Orchestrator`. | `Orchestrator.start()` spawns worker tasks; `Scheduler` exposes `run_cycle()`; `IssueDispatch` decides `max_slots` per-state, enqueues `AgentRunner`, and reacts to completion events via `asyncio.Queue`. | 
| `symphony.agent` | Agent runner that creates/boots a workspace, renders the prompt via `prompt`, invokes the Codex app-server (via `CodexClient`), and handles continuation logic similar to `AgentRunner`. | `AgentRunner.run(issue, recipient_queue)` returns or raises; event callbacks send runtime telemetry back to the orchestrator; built-in retries respect `agent.max_turns`. | 
| `symphony.workspace` | Filesystem helpers that create/remove safe per-issue directories, run hooks, optionally manage remote workspaces through SSH, and enforce `PathSafety`. | `WorkspaceManager.create(issue_id, worker_host)`, `.remove(issue_id)`, `.run_hook(name, context)`; `PathSafety` utilities canonicalize paths before any disk mutations. | 
| `symphony.codex` | Codex app-server client wrapping `asyncio.create_subprocess_exec`, JSON-RPC handshake, tool execution, and streaming updates (mirrors `Codex.AppServer`). | `AppServer.start(workspace, policies)` returns a session; `.run_turn(prompt, issue, on_event)` drives JSON-RPC; `.stop()` ensures cleanup. | 
| `symphony.logging` | Logging configuration that sets `structlog` + `logging` with JSON formatting, rotating disk handler (like `LogFile.configure`), and `ContextVar` helpers to inject `issue_id`, `workspace`, and `session_id`. | `Logging.configure(logs_root, level)` sets handlers; `Logging.with_context(**ctx)` yields loggers with structured fields; `Logging.link_task(task, issue_id)` ensures `issue_id` flows across async tasks. | 
| `symphony.observability` | Metrics, HTTP status snapshots, optional terminal dashboard, and event bus (maps to `StatusDashboard` + `HttpServer`). | `Observability.publish(snapshot)` records running/completed counts; `Metrics.exporter()` exposes Prometheus `/metrics`; `StatusServer` feeds `GET /api/v1/state` snapshots. | 

Additional packages:

- `symphony.clisup` (supervisor) composes the orchestrator, observability server, and optional worker pool (similar to OTP supervision tree).
- `symphony.tools` (optional) hosts helper classes like `LinearPayloadNormalizer`, `CodexToolRegistry`, or `RetryPolicy`.

## Dependency list

| Dependency | Purpose | Notes |
| --- | --- | --- |
| `pydantic` | Typed config: maps workflow YAML to dataclasses, enforces defaults, normalizes states, and validates tracker credentials. | Mirrors `Config.Schema`.
| `PyYAML` | Parse workflow front matter. | Used in `WorkflowLoader` before handing YAML to `pydantic`.
| `tomli` | Read config from `pyproject.toml` if we need defaults or version metadata. | Optional for packaging.
| `jinja2` | Prompt templating (issue + attempt context). | Use `StrictUndefined` to surface missing variables like `issue.branch_name`.
| `httpx[http2]` | Tracker GraphQL client with timeouts/retries. | Use `AsyncClient` for polls.
| `structlog` | Structured logging with context. | Combined with the stdlib `logging` to route events to console and rotating disk file.
| `python-json-logger` | Optional `json` formatter for shipping logs to machines. | Used when `observability.log_format=json`.
| `prometheus_client` | Metric counters/gauges that feed `/metrics` (dispatch rate, agent duration, tokens). | See observability plan below.
| `fastapi`, `uvicorn` | Optional observability HTTP server that exposes `/metrics`, `/status`, and `/issues/{id}` snapshots (mirrors Phoenix dashboard). | Lazy-loaded so telemetry is optional.
| `asyncssh` | Run workspace hooks and Codex inside remote workers via SSH, similar to `SSH.run`. | Use fallbacks: `ssh` binary when `asyncssh` unavailable.
| `watchdog` | Monitor `WORKFLOW.md` for changes and trigger reloads. | Keeps configuration fresh, replicating `WorkflowStore.force_reload`.
| `pytest`, `pytest-asyncio`, `httpx` (test) | Unit and integration testing. | Validate scheduler, config parsing, tracker adapters, and Codex interactions.
| `rich` | Terminal status dashboard (optional) or dynamic CLI output. | Helps mirror `StatusDashboard` renderings while staying Cross-platform.

Development extras: `coverage`, `mypy`, `pre-commit`, `black`, `ruff`.

## Configuration and workflow loader design

1. **WorkflowLoader** reads `WORKFLOW.md` from `Config.settings().workflow.path` (default: `<cwd>/WORKFLOW.md`). It uses `PyYAML` to split front matter (`---`) from the Markdown body, exactly like `Workflow.split_front_matter`. Blank front matter yields `{}`.
2. The loader returns `WorkflowDefinition(config: dict, prompt_template: str)`. The `prompt_template` is rendered with `Jinja2` using the same keys supplied to BEAM's `PromptBuilder`: an `issue` dict (with ISO timestamps) plus a numeric `attempt` value.
3. `Config.load()` feeds the workflow config map into a `pydantic` model hierarchy:
   - `TrackerConfig` (kind, API key, project slug, active/terminal states).
   - `WorkspaceConfig` (root path, path sanitization, hooks for `after_create`, `before_run`, `before_remove`).
   - `AgentConfig` (max concurrent agents, per-state overrides, max turns).
   - `CodexConfig` (command, approval policy, thread sandbox, turn sandbox policy).
   - `ObservabilityConfig` (dashboard_enabled, render_interval_ms, logs_root, metrics_enabled).
4. Each config class exposes helper methods: e.g., `TrackerConfig.normalize_state(name)`, `WorkspaceConfig.workspace_path(issue_id, worker_host)`, `AgentConfig.concurrent_slots(state_name)`, and `CodexConfig.resolve_turn_sandbox(workspace_path)`. These mirror the BEAM-level `Schema` helpers.
5. Workflow config is cached by `WorkflowStore` analog, but the loader also registers a `watchdog` observer so editing `WORKFLOW.md` triggers a reload and revalidation (like `WorkflowStore.force_reload`).
6. Errors surface with helpful messages (`workflow_unavailable`, `workflow_parse_error`). When the front matter is empty, `PromptBuilder.default_prompt` returns the fallback template that references `issue.identifier`, `title`, and `description`.

## Orchestrator scheduling contract

1. **Ticker**: Every `poll.interval_ms` milliseconds, the orchestrator wakes the scheduler (via `asyncio.Event` or `asyncio.Task`). Only one poll cycle runs at a time (`poll_check_in_progress` guard).
2. **Discovery**: Scheduler asks `TrackerAdapter.fetch_candidate_issues()` and normalizes the returned dicts into `Issue` dataclasses (id, identifier, state, priority, updated_at, branch_name, labels). It filters active states defined in the config (default: `Todo`, `In Progress`, `Merging`, `Rework`).
3. **Slot accounting**: The orchestrator keeps `running: Dict[issue_id, RunningEntry]` and `retry_queue: PriorityQueue[ScheduledRetry]`.
   - `max_concurrent_agents` is global; `max_concurrent_agents_by_state` can override it per `state`. The orchestrator only dispatches until the lowest of `available_slots` and `per-state slots` is reached.
   - `MapSet` equivalents (`completed`, `claimed`, `retry_attempts`) track progress and ensure idempotency when issues re-enter active states.
4. **Dispatch**: `IssueDispatch` sorts candidates by `priority` and `updated_at`, taking into account whether an issue is currently being retried. Each dispatched issue calls `AgentRunner.run(issue, event_queue)` inside `asyncio.create_task`. The orchestrator links the task to `running` and listens on a `Queue` for completion events (normal vs. error) and Codex updates (token counts, rate limit). It also publishes snapshots to the observability bus.
5. **Completions**:
   - If the agent task exits normally, the orchestrator records totals and schedules a `continuation_check` after a short delay (`status_render_delay_ms`). It re-checks the issue state before deciding whether another turn is needed.
   - If the agent task exits with error, `retry_attempts` increments, and a backoff (e.g., exponential: `base_ms * 2^attempt`) is scheduled in `retry_queue`. Errors carry metadata (workspace path, worker host) for diagnostics.
6. **Retry queue**: Maintained as a min-heap keyed by `due_at`; the orchestrator drains ready retries before or after a poll cycle. Each retry respects the `max_concurrent_agents` budget and re-fetches the issue state to avoid redundant retries if the issue moved to terminal.
7. **State transitions**: The orchestrator listens for `TrackerAdapter` signals (e.g., CLI or manual) that indicate state changes; it cancels running tasks for terminal states and cleans up workspaces (`WorkspaceManager.remove_issue_workspaces`).
8. **Metrics and logging hooks**: Each major transition emits structured logs and metric updates (poll durations, issues fetched, tokens consumed). The orchestrator also exposes health metrics (last poll, running count, backlog size) for the observability HTTP endpoint.

## Agent, workspace, and tracker boundaries

- **AgentRunner** mirrors `SymphonyElixir.AgentRunner`:
  1. Accepts an `Issue` plus optional `worker_host` and `attempt` metadata.
  2. Calls `WorkspaceManager.create(issue_id, worker_host)` to ensure a safe directory (including symlink canonicalization, workspace hooks, and remote worker support via `asyncssh` or the `ssh` binary).
  3. Starts a `CodexClient` session inside the workspace (`codex_app_server` command defined in config) and streams updates to the orchestrator through an `asyncio.Queue` (events: `session_started`, `session_completed`, `token_update`, `tool_request`).
  4. Handles continuation turns up to `agent.max_turns` by calling `PromptBuilder.build()` with `attempt` context, just like the recursive `do_run_codex_turns` logic.
  5. On teardown, `WorkspaceManager.run_after_run_hook()` runs even if the agent raised an exception.

- **WorkspaceManager** analogs:
  - `workspace_root` is joined with sanitized issue identifiers (`PathSafety.safe_identifier`) to avoid collisions and path escapes.
  - Hooks (`after_create`, `before_run`, `after_run`, `before_remove`) execute via `subprocess.run` (or remote commands when `worker_host` is set). Hook commands inherit environment variables such as `ISSUE_IDENTIFIER`, `WORKSPACE_PATH`, and `LINEAR_TOKEN`.
  - Remote worker support uses `asyncssh` to run shell commands that mirror the `SSH` helper: `set -eu`, `mkdir -p`, and printing metadata before returning to Python for parsing.

- **TrackerAdapter** interface ensures the orchestrator can be swapped to `memory` for tests:
  - `LinearTracker` uses GraphQL operations (create comment, update state, fetch states) built on `httpx.AsyncClient` with exponential backoff.
  - State resolution caches `Linear` `state_id`s, sharing code with config validation so the `state_name -> id` map is available without extra API calls.
  - The orchestrator uses `TrackerAdapter.fetch_issue_states_by_ids` before scheduling retries to detect terminal transitions.

## Logging and observability plan

1. **Structured logging**: Use `structlog` + `logging` to emit JSON/single-line events. A `LoggingContext` context manager injects the following keys: `issue_id`, `issue_state`, `workspace`, `session_id`, `worker_host`, and `request_id`. The base logger writes to STDOUT/console, while `RotatingFileHandler` writes to `logs/symphony.log` with rotation parameters (10 MB max, keep 5 files) mirroring `LogFile.configure`.
2. **Observability events**:
   - `orchestrator.poll_tick`, `orchestrator.cycle`, `orchestrator.dispatch`, `agent.session_started`, `agent.session_finished`, `codex.token_delta`, `workspace.hook_failed` are emitted as metrics/counters.
   - `Prometheus` metrics include `symphony_agent_running`, `symphony_agent_errors_total`, `symphony_retry_queue_length`, `symphony_codex_tokens_total`, `symphony_poll_duration_seconds`, and `symphony_workspace_creation_seconds`.
3. **Status dashboard + HTTP server**:
   - A lightweight `FastAPI` service exposes `/api/v1/state` (latest orchestrator snapshot) and `/api/v1/refresh` endpoints plus `/metrics` (Prometheus). It mirrors `HttpServer` + `StatusDashboard` by publishing the same `running`/`retry`/`codex` counters.
   - `StatusRenderer` optionally prints a terminal snapshot using `rich` (token sparkline, running tasks list, throughput rates). The renderer subscribes to the `ObservabilityBus` (analogous to `ObservabilityPubSub`) and throttles updates to prevent overwhelming the terminal.
4. **Observability config**: `WORKFLOW.md` gains an `observability` section mirroring the existing Elixir defaults:

```yaml
observability:
  dashboard_enabled: true
  refresh_ms: 1000
  render_interval_ms: 100
  logs_root: ./log
  metrics_enabled: true
```

5. **Telemetry contract**: The orchestrator publishes snapshots (tokens, running issues, completed ids) that both the dashboard and HTTP server read. Tokens and rate limits accumulate exactly as in `SymphonyElixir.Orchestrator` so the Python implementation can surface the `codex_totals` and `codex_rate_limits` data in the HTTP snapshot.

## Implementation/ticket handoff notes

- Each component above should have an API defined in a Python module or type stub before implementation begins.
- Tests should cover config parsing, workflow reloads, scheduler dispatch, agent retries, workspace hook execution, and logging contexts.
- Observability and logging plans must include smoke tests (`pytest` fixtures can spin up the `FastAPI` server and hit `/metrics`).
- The design remains faithful to `SPEC.md` while providing Pythonic abstractions, so implementation tickets can target one package at a time.

---

*References: `SPEC.md` (§3.1–3.3), `elixir/lib/symphony_elixir/{config,workflow,orchestrator,agent_runner,workspace,Tracker,log_file,status_dashboard,http_server}`.*
