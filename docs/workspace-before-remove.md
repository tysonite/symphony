# Workspace Before-Remove Hook

Before deleting a workspace that completed or abandoned an issue, Symphony runs `mix workspace.before_remove` (configured in `WORKFLOW.md`'s `hooks.before_remove`). The task closes any still-open GitHub pull requests for the branch you just finished so stale PRs do not linger after the ticket moves into a terminal state.

## Usage

```
mix workspace.before_remove [--branch feature/my-branch] [--repo owner/name]
```

If you omit `--branch`, the task falls back to the current Git branch (`git branch --show-current`). `--repo` defaults to `openai/symphony` but can point to another owner/repo tuple when the workspace checks out a fork.

## Task behavior

1. It checks for the presence of the `gh` CLI and verifies authentication via `gh auth status`. Without a usable `gh`, the task logs nothing and exits cleanly.
2. It lists open pull requests for the target branch with `gh pr list --repo <repo> --head <branch> --state open --json number --jq '.[].number'`.
3. For each open PR number it calls `gh pr close <number> --repo <repo> --comment '<closing message>'`, where the comment is `"Closing because the Linear issue for branch <branch> entered a terminal state without merge."`.
4. Success and failure messages are printed to the shell so hooks can log the outcome.

`run_command/2` is a helper that locates the requested executable and shells out via `System.cmd/3`, capturing both stdout/stderr (with `stderr_to_stdout: true`). Any command failure is reported but never raises, keeping the hook resilient to GitHub outages.

Documenting this hook makes it clear why the default workflow `WORKFLOW.md` contains a `before_remove` script that calls this task: it preserves the GitHub/Git state that agents leave behind when they shut down a workspace.
