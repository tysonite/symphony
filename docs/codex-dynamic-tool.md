# Codex Dynamic Tool Support

`SymphonyElixir.Codex.DynamicTool` exposes the `linear_graphql` helper that Codex sessions can call to read or write Linear issues without leaving the app-server turn. `AgentRunner` injects the tool spec returned by `DynamicTool.tool_specs/0` into every `thread/start` payload, so Codex always knows the tool name, description, and required schema.

## The `linear_graphql` schema

The tool schema is a strict object that requires a non-empty `query` string and accepts an optional `variables` object (`additionalProperties` are not allowed unless the schema is changed). When Codex invokes the tool, `execute/3` normalizes the arguments so callers can pass either a string (treated as the query) or a map with explicit `query`/`variables`. Empty queries, missing queries, and non-object `variables` trigger early errors (`:missing_query`, `:invalid_variables`) so the tool never sends junk to Linear.

## Dispatching requests to Linear

Once arguments are normalized, the tool calls `SymphonyElixir.Linear.Client.graphql/3` (the default, but it can be overridden for tests via the `:linear_client` option). The response is examined for `errors` to set the `success` flag: if Linear returns a non-empty `errors` list, the tool reports `success=false` even though the HTTP layer succeeded.

## Response shaping and failure helpers

Results are formatted into the Codex-friendly shape with `"success"`, `"output"`, and `"contentItems"` (the latter two are guaranteed to be strings and list entries so Codex renders a user-friendly summary). When the tool fails before hitting Linear (missing query, invalid arguments, missing API token, HTTP errors, or request failures) it builds an `"error"` map with a descriptive message and includes hints such as the HTTP status or the inspectable reason. These errors flow back to Codex inside a `success=false` payload so the turn can log the issue without raising.

## Why it matters

Because the tool reuses the configured Linear auth (`tracker.api_key` resolved by `SymphonyElixir.Config` and `SymphonyElixir.Linear.Client`) and gracefully translates GraphQL errors into readable tool responses, agents can rely on `linear_graphql` inside the same Codex session that implements the rest of the workflow. Tests can stub `:linear_client` or pass fake environment variables to exercise error payloads, making both success and failure cases observable through the orchestrator logs.
