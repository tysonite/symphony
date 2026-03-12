# Specs Check

`mix specs.check` enforces adjacent `@spec` declarations for every public API defined under `lib/`. The task wraps `SymphonyElixir.SpecsCheck.missing_public_specs/2` and intentionally fails the run if any `def` lacks a neighboring `@spec` (unless the function is marked `@impl` or listed in an exemption file).

## Invocation

```
mix specs.check [--paths path1 --paths path2] [--exemptions-file exceptions.txt]
```

If you omit `--paths`, the task defaults to scanning `lib`. You can pass `--paths` multiple times to add directories or specific files, and `--exemptions-file` can list identifiers (`Module.function/arity`) to ignore.

## How it works

- `SpecsCheck.missing_public_specs/2` walks the provided paths, expanding directories via `Path.wildcard("**/*.ex")`.
- Each file is parsed to an AST (`Code.string_to_quoted/2`), and `module_nodes/1` collects every `defmodule` body.
- `consume_form/5` tracks pending `@spec` signatures and `@impl` markers. When a `def` appears, it verifies that `(name, arity)` was announced by a spec right before it.
- If no spec is present, the helper records a finding (`file`, `module`, `name`, `arity`, `line`) so `Mix.Tasks.Specs.Check` can print `#{file}:#{line} missing @spec for #{Module.function/arity}`.
- After scanning all files, the task raises `Mix.raise/1` when any findings remain, causing CI jobs or workspace hooks to fail and point engineers to the exact location that needs a spec.

Keeping this task documented makes the repository’s code-quality automation visible alongside the runtime features described in the other docs.
