# PR Body Validation

Symphony ships `mix pr_body.check` to guard that every pull-request description matches the repository's canonical template before the workspace is recycled. The command is meant to run from a workspace hook (see `WORKFLOW.md`'s `hooks.before_remove`) so the agent closes or archives a workspace only after its PR write-up follows the expected structure.

## Usage

Invoke it with the `--file` flag pointing at the drafted PR body:

```
mix pr_body.check --file /path/to/pr_body.md
```

The task reads `.github/pull_request_template.md` (falling back to `../.github/pull_request_template.md` when the repo root lives one level up) and extracts the template sections via a regex that matches headings with four-to-six `#` characters.

## Lint rules

`lint_and_print/4` applies a fixed checklist:

- `check_required_headings/3` ensures every heading from the template appears in the candidate body using `String.match?` checks.
- `check_order/3` enforces the headings remain in the same order by comparing their byte offsets.
- `check_no_placeholders/1` fails when the body still embeds template placeholder comments like `<!-- ... -->`.
- `check_sections_from_template/4` iterates each template section and validates that:
  - The corresponding section is non-empty.
  - When the template resumes with bullets, the body has at least one `- ` item (`maybe_require_bullets/4`).
  - When the template demands checkboxes, the body includes `- [ ]` or `- [x]` entries (`maybe_require_checkboxes/4`).

Errors are printed via `Mix.shell().error/1` and the task raises `Mix.raise/1` with a summary message so CI jobs or workspace hooks fail fast.

## Implementation notes

- The helper `capture_heading_section/3` slices the markdown between one heading and the next, taking care to ignore subsections.
- Headings are sourced from templates that usually live under `.github`, so forks or workspaces can override the template by keeping a file at that path.
- Running `mix pr_body.check` inside `hooks.before_remove` (see `WORKFLOW.md`) automatically treats missing headings or placeholders as routine failures, giving agents a chance to refresh the PR description before closing a workspace.

Documenting this helper keeps the PR hygiene step visible alongside the runtime features described in the other docs.
