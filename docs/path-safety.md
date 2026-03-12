# Path Safety & Canonicalization

`SymphonyElixir.PathSafety` keeps every workspace path firmly rooted where the operator expects it. When Symphony builds workspace directories, resolves workspace roots, or prepares Codex runtime sandboxes, it canonicalizes the candidate path instead of trusting the raw string from configuration or the filesystem. That guards against symlinks, `..` segments, and other tricks that would otherwise let a workspace escape `Config.settings!().workspace.root`.

## `canonicalize/1`

`PathSafety.canonicalize/1` is the only public API. It expands the given path, splits it into a drive/root prefix and the remaining segments, then walks every segment while keeping track of the resolved path so far. For each part it:

1. Joins the candidate (`root + resolved + segment`) and inspects it with `File.lstat/1`.
2. If the entry is a symlink it reads every link target with `:file.read_link_all/1`, resolves that target relative to the previously resolved path, and restarts evaluation from the target's canonical split so symbolic loops are unraveled.
3. If the entry exists, it adds the segment to the resolved list and continues; if it does not exist, it assumes the remaining segments are new directories and joins them directly.
4. When `File.lstat/1` fails for other reasons (permission denied, too long, etc.) it returns `{:error, reason}` and the caller wraps it as `{:path_canonicalize_failed, expanded_path, reason}`.

This deterministic recursion lets Symphony understand exactly where a workspace would land before it ever calls `File.mkdir_p/1` or passes the path to Codex.

## Where the canonical path is used

- `SymphonyElixir.Workspace.create_for_issue/2` canonicalizes the configured root plus the sanitized identifier so every workspace directory reflects the real filesystem location. Later hooks (`hooks.after_create`, `hooks.before_run`) rely on the same value.
- `SymphonyElixir.Codex.AppServer.start_session/2` canonicalizes both the workspace and the workspace root before launching `codex app-server`. That ensures sandbox policies such as `workspaceWrite` receive a stable path and that turns happen inside a directory the orchestrator already vetted.
- `SymphonyElixir.Config.Schema.default_runtime_turn_sandbox_policy/2` canonicalizes the workspace root before embedding it in sandbox policies, so `worker`/`codex` settings cannot point a policy at a symlinked escape route.

## Remote hosts & failure handling

Remote worker hosts do not go through `PathSafety.canonicalize/1` because the local process cannot inspect a remote filesystem. Instead, remote commands only verify the workspace path is non-empty, lacks newline/NULL characters, and relies on the remote SSH helper to create canonical directories on the far side. When `canonicalize/1` does return `{:error, {:path_canonicalize_failed, path, reason}}`, the orchestrator logs the failure (including the expanded path) and refuses to launch Codex or mutate the workspace until the configuration is corrected.
