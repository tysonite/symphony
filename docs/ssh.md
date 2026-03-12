# SSH Helpers & Remote Commands

`SymphonyElixir.SSH` standardizes every remote workspace hook, cleanup job, and Codex turn that runs over SSH. The helper makes it easy for the orchestrator to build the exact `ssh` invocation, stream JSON over a port, and keep calls safe regardless of the host, port, or command provided by the workflow.

## Running commands vs. ports

- `run/3` locates the `ssh` executable via `System.find_executable/1` and shells out with `System.cmd/3`, capturing stdout/stderr as a tuple `{"<combined output>", exit_status}`. This is the workhorse for remote workspace hooks (`hooks.before_run`, `hooks.after_run`, workspace creation/cleanup).
- `start_port/3` uses the same argument list but opens an Erlang `Port` (`Port.open/2`) so `SymphonyElixir.Codex.AppServer` can stream newline-delimited JSON from Codex. It accepts the `:line` option verbatim so the caller can receive line-delimited output, and it always requests `:binary`, `:exit_status`, and `:stderr_to_stdout`.

Both helpers return `{:error, :ssh_not_found}` when the `ssh` binary cannot be located, giving the orchestrator a chance to log a meaningful error before retrying.

## Building the SSH arguments

`ssh_args/2` assembles the argument list in a predictable order:

1. `maybe_put_config/1` adds `-F <config>` when the `SYMPHONY_SSH_CONFIG` environment variable points to a custom SSH config file.
2. `-T` (disable pseudo-terminal allocation) keeps remote commands non-interactive.
3. If the target includes a port (see below), `-p <port>` is automatically inserted.
4. The destination host follows.
5. The remote command is wrapped by `remote_shell_command/1`, which returns `"bash -lc '...escaped... '"` so every call runs in a login shell.

`remote_shell_command/1` uses `shell_escape/1` to wrap the command in single quotes and escape any single quotes in the payload (`'` becomes `'"'"'`). That way, even complicated hook scripts never inject shell-breaking characters into the SSH line.

## Host parsing & port shorthand

`parse_target/1` accepts values like `example.com`, `example.com:2222`, or `[::1]:2222`. The helper trims whitespace, splits `host:port` shorthand through a regex, and calls `valid_port_destination?/1` to allow IPv6 literal addresses (which must be bracketed).

- IPv6 addresses such as `[::1]:2222` are permitted because they contain brackets.
- Bare `host:port` strings only become split when the host portion does not itself contain unbracketed colons.

If the shorthand is invalid, the parser treats the entire string as the destination and leaves the port `nil`, letting OpenSSH default to 22.

## Where the helpers are used

- `SymphonyElixir.Workspace` delegates remote workspace creation, hook execution, and removal to `SSH.run/3`. Every call passes through `remote_shell_command/1` so the remote side always runs inside the intended directory and logs the same metadata that local runs do.
- `SymphonyElixir.Codex.AppServer.start_session/2` uses `SSH.start_port/3` when it launches a Codex app-server on a remote worker host. The generated command is `cd <workspace> && exec <codex command>`, so the remote workspace does not need additional setup.

Because both helpers share the same argument builder, hosts configured in `worker.ssh_hosts` always receive the same SSH options (custom config, port passing, disabled tty). That keeps remote worker runs as predictable as local ones while still respecting per-host capacity limits.
