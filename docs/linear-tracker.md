# Linear Tracker Integration

Symphony interacts with Linear through the `SymphonyElixir.Tracker` boundary. When `tracker.kind` is `linear`, the orchestrator uses `SymphonyElixir.Linear.Adapter`, which in turn relies on `SymphonyElixir.Linear.Client` and the `SymphonyElixir.Linear.Issue` struct.

## Tracker boundary & adapter

`SymphonyElixir.Tracker` defines the read/write callbacks `fetch_candidate_issues/0`, `fetch_issues_by_states/1`, `fetch_issue_states_by_ids/1`, `create_comment/2`, and `update_issue_state/2` and dispatches them to `Linear.Adapter`. The adapter exposes `@create_comment_mutation` and `@update_state_mutation` GraphQL documents that wrap Linear mutations. Both mutations call `Linear.Client.graphql/3`, then inspect the nested `data` fields for `success` before returning `:ok` or a descriptive error tuple such as `{:error, :comment_create_failed}`.

`update_issue_state/2` first resolves a Linear state ID via `@state_lookup_query`, which fetches the team states filtered by name. If no matching state ID exists the adapter returns `{:error, :state_not_found}`.

## GraphQL client & paging

`SymphonyElixir.Linear.Client` defines `@query` for active-state polling and `@query_by_ids` for targeted refreshes. Candidate issues are fetched with `fetch_candidate_issues/0`, which validates that `tracker.api_key` and `tracker.project_slug` are configured, optionally applies an assignee filter (`routing_assignee_filter/0`), and paginates via `do_fetch_by_states_page/6`. Pagination tolerates cursors (`next_cursor/1`) and transparently merges pages in order.

`fetch_issue_states_by_ids/1` batches IDs in groups of `@issue_page_size` (50) and sorts the results to preserve the requested order. Both read paths rely on `graphql/3` to POST to `tracker.endpoint` with headers from `graphql_headers/0` (including the Authorization token) and log HTTP failures (`{:error, {:linear_api_status, status}}` or `{:error, {:linear_api_request, reason}}`).

## Issue normalization

`Linear.Client` converts every raw GraphQL issue node into `SymphonyElixir.Linear.Issue`. Normalization pulls fields such as `id`, `identifier`, `title`, `description`, `priority`, `state.name`, `branchName`, `url`, `createdAt`, `updatedAt`, and `assignee.id`. Blockers are extracted from `inverseRelations` where the relation type (case-insensitive) equals `blocks`. Labels are normalized to lowercase strings, and `assigned_to_worker` is computed via `assigned_to_worker?/2`: it defaults to `true` but can filter by `assignee` IDs derived from `tracker.assignee` (supporting literal IDs or the special value `"me"` resolved via `@viewer_query`).

`normalize_issue_for_test/2` and helpers such as `parse_datetime/1` and `parse_priority/1` keep the production logic testable.

## Write operations

`Linear.Adapter.create_comment/2` runs `SymphonyCreateComment` to add a comment body to an issue, while `update_issue_state/2` applies `SymphonyUpdateIssueState`. Both mutations inspect `data.commentCreate.success` or `data.issueUpdate.success` and propagate errors when the API does not return `true`. These helpers allow the orchestrator or Codex dynamic tool to push Linear updates without duplicating GraphQL boilerplate.
