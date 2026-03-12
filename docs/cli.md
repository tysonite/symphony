# CLI Entrypoint & Runtime Flags

`SymphonyElixir.CLI` is the escript entrypoint that drives the OTP application when you run `./bin/symphony` (or the `symphony` escript) from a repository.

## Guardrail acknowledgement

Every run must be prefixed with `--i-understand-that-this-will-be-running-without-the-usual-guardrails`. When the flag is absent `CLI.main/1` short-circuits with an error message that prints the red-bordered acknowledgement banner defined in `acknowledgement_banner/0`. The flag ensures operators deliberately opt into an unsupported workflow before any orchestration begins.

## Workflow path selection

`OptionParser.parse/2` validates the supported switches and collects an optional positional argument for the workflow file path. When the path is omitted the CLI expands `WORKFLOW.md` from the current working directory; when a path is supplied it is resolved with `Path.expand/1`.

`CLI.run/2` then checks `File.regular?/1` to ensure the workflow file exists, delegates to `SymphonyElixir.Workflow.set_workflow_file_path/1`, and calls `Application.ensure_all_started(:symphony_elixir)` through the injectable `ensure_all_started` dependency. Errors propagate as `{:error, message}` so the process exits with status `1` if the workflow file is missing or startup fails.

## Runtime overrides

- `--logs-root <path>`: `maybe_set_logs_root/2` trims the provided `logs_root`, rejects empty strings, and writes the computed file path into `Application.put_env(:symphony_elixir, :log_file, LogFile.default_log_file(logs_root))`. The CLI uses `SymphonyElixir.LogFile.default_log_file/1` so the docs in `docs/logging.md` describe how the normalized path is joined with `log/symphony.log`.
- `--port <port>`: `maybe_set_server_port/2` stores a non-negative integer under `:server_port_override`, which `SymphonyElixir.HttpServer` later reads when deciding whether to start the Phoenix/LiveView stack.

`runtime_deps/0` packages filesystem helpers plus the setter functions so tests can drive each path without touching the real filesystem or application environment. Switching the dependencies allows test suites to assert the guardrail branches and failure outputs without starting the full OTP tree.

## Shutdown coordination

Once the path and overrides are configured, `main/1` calls `run/2` and, on success, invokes `wait_for_shutdown/0`. That helper monitors `SymphonyElixir.Supervisor` and halts the BEAM when the supervisor stops. A normal exit results in `System.halt(0)` while abnormal supervisor termination raises `System.halt(1)` so the CLI always reports the overall outcome to the operator.
