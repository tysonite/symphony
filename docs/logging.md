# Logging & Disk Handler

Logging starts before the orchestrator itself: `SymphonyElixir.Application.start/2` calls `SymphonyElixir.LogFile.configure/0` at the top of the supervision startup, so every child (dashboard, orchestrator, workflow store, HTTP server) inherits logger configuration that already routes to disk.

`LogFile.configure/0` reads three application environment values:

- `:log_file` (default `default_log_file/0`, which uses `File.cwd!/1` to anchor `log/symphony.log`).
- `:log_file_max_bytes` (default `10 * 1024 * 1024`).
- `:log_file_max_files` (default `5`).

The helper `default_log_file/1` simply joins the provided `logs_root` with `log/symphony.log`, giving CLI authors an easy override path via the `--logs-root` flag described in `docs/cli.md`.

`setup_disk_handler/3` expands the requested path, ensures the directory hierarchy exists (`File.mkdir_p/1`), removes any previously registered handler with the id `:symphony_disk_log`, and installs a `:logger_disk_log_h` handler that:

- logs at `:all` levels with a single-line formatter (`{:logger_formatter, %{single_line: true}}`).
- writes to the configured file in ``wrap`` mode, honoring `max_no_bytes`/`max_no_files` for rotation.

When the handler is added successfully the default `:logger` console handler is removed so the live terminal stays clean. When the handler fails to install (for example due to filesystem permissions) a warning is emitted but the application continues running with the console handler intact.

`setup_disk_handler/3` also calls `remove_existing_handler/0` before the add attempt, ensuring restarts or config changes do not leave duplicate handlers registered.

The rotating log file ensures every Symphomy component (orchestrator, worker tasks, HTTP server, dashboard) writes into `log/symphony.log` by default, and CLI callers can override the log root to direct logs into `--logs-root <path>` prior to supervisor startup.
