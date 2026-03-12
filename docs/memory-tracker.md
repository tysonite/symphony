# Memory Tracker Adapter

`SymphonyElixir.Tracker.Memory` is a lightweight, in-memory implementation of the tracker boundary intended for tests and developer runs that do not talk to a real Linear account. The adapter follows the same `SymphonyElixir.Tracker` callbacks as `SymphonyElixir.Linear.Adapter`, but it reads from application configuration instead of the network and short-circuits write operations with in-process events.

## Enabling the adapter

Set `tracker.kind: memory` in `WORKFLOW.md` or override it via `Application.put_env/3` so `SymphonyElixir.Config.validate!/0` still accepts the workflow. When `tracker.kind` resolves to `"memory"`, `SymphonyElixir.Tracker.adapter/0` returns `SymphonyElixir.Tracker.Memory` instead of `SymphonyElixir.Linear.Adapter`.

The helper expects the repository to provide a list of issue structs via `Application.get_env(:symphony_elixir, :memory_tracker_issues, [])`. Each entry must be a `%SymphonyElixir.Linear.Issue{}` (see `elixir/lib/symphony_elixir/linear/issue.ex` for the field list) so the orchestrator sees the same API shape it would from Linear.

```elixir
config :symphony_elixir,
  tracker: %{kind: "memory"},
  memory_tracker_issues: [
    %SymphonyElixir.Linear.Issue{
      id: "desk-1",
      identifier: "TES-1",
      title: "Document the symphony",
      description: "Doc task for local runs",
      state: "Todo",
      priority: 1
    }
  ]
```

You can change the list dynamically in tests with `Application.put_env(:symphony_elixir, :memory_tracker_issues, new_list)` before starting the orchestrator.

## How the adapter works

- `fetch_candidate_issues/0` simply returns `issue_entries/0`, which filters the configured list through `match?(%SymphonyElixir.Linear.Issue{}, element)` so invalid values are ignored.
- `fetch_issues_by_states/1` and `fetch_issue_states_by_ids/1` search the cached entries, normalizing states with `String.downcase/1` so the lookup is case-insensitive.

Because the adapter reads static in-memory data, there are no HTTP calls or pagination; everything is immediate and deterministic, making it suitable for unit or integration tests where you drive the orchestrator with canned issues.

## Capturing writes & state updates

`create_comment/2` and `update_issue_state/2` do not mutate the configured list. Instead, they call `send_event/1`, which `Application.get_env(:symphony_elixir, :memory_tracker_recipient)` uses to push messages such as `{:memory_tracker_comment, issue_id, body}` and `{:memory_tracker_state_update, issue_id, state_name}`. If no recipient is configured the calls return `:ok` silently.

In tests you can point the recipient at the test process so you can assert that agents attempt the right tracker writes:

```elixir
Application.put_env(:symphony_elixir, :memory_tracker_recipient, self())

# ...run the orchestrator...

assert_receive {:memory_tracker_comment, "desk-1", comment_body}
assert_receive {:memory_tracker_state_update, "desk-1", "Done"}
```

Because the adapter is just a tiny wrapper around `Application.get_env/2`, it is easy to extend with additional helpers (for example, preloading `memory_tracker_issues` from fixtures) without touching the real Linear API.
