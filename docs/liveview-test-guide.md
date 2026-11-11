# LiveView Test Guide for Tattle

We write UI tests with [PhoenixTest](https://hexdocs.pm/phoenix_test). It drives LiveViews and plain controller pages through one API, so a test reads like a user's steps: visit a page, fill a form, check what shows up.

The examples below come from [Uli](https://github.com/tattle-made/Uli), where this guide started. Good test files to copy:

- [`token_generation_live_test.exs`](https://github.com/tattle-made/Uli/blob/main/uli-community/test/uli_community_web/live/token_generation_live_test.exs)
- [`crowdsource_slur_form_live_test.exs`](https://github.com/tattle-made/Uli/blob/main/uli-community/test/uli_community_web/live/crowdsource_slur_form_live_test.exs)

## What to test

A UI test checks what a user can do on a page and what the page shows back. For each page, write tests for:

- **The happy path.** The page's main action succeeds.
- **Bad input.** The user sees an error and nothing is saved.
- **Editing and deleting.** Only if the page offers them.
- **Roles.** Each role sees only what it should.
- **Ownership.** A user can't open someone else's data.

Skip these in UI tests:

- **Styling and layout.** PhoenixTest doesn't run CSS.
- **JavaScript.** PhoenixTest doesn't run it (see the last section).
- **Logic with no UI.** Cover it in the context's own tests.

## How a test file is set up

```elixir
defmodule MyAppWeb.SomePageLiveTest do
  use MyAppWeb.ConnCase, async: true

  setup :register_and_log_in_user
  ...
end
```

- `use MyAppWeb.ConnCase` gives you a `conn` and turns on the database sandbox. For an example ConnCase, check out [Uli's `conn_case.ex`](https://github.com/tattle-made/Uli/blob/main/uli-community/test/support/conn_case.ex).
- **The sandbox** runs every test in a transaction that rolls back at the end. Nothing a test writes survives it, so `async: true` is safe.
- Because of the rollback, **tests can't depend on each other**. If test B needs data from test A, set it up in B (or in a `setup` block).
- `setup :register_and_log_in_user` comes from `mix phx.gen.auth`. It creates a user, logs them in, and puts `conn` and `user` in the test context:

```elixir
test "an admin can also pick the Admin level", %{conn: conn, user: user} do
```

- When a test needs data to exist before it starts, create it with a fixture in a `setup` block. The block returns a map, and its keys land in the test context next to `conn` and `user`. Keep fixtures in `test/support/fixtures/`, one module per context. For examples, see Uli's [fixtures folder](https://github.com/tattle-made/Uli/tree/main/uli-community/test/support/fixtures).

```elixir
setup %{user: user} do
  %{slur: crowdsourced_slur_fixture(%{contributor_user_id: user.id})}
end

test "...", %{conn: conn, user: user, slur: slur} do
```

## The shape of a test

Each test covers one segment of a flow, such as creating a token or deleting one. Don't walk through the whole page in one test.

Every test is a pipe: visit, check the starting state, act, assert.

Check the starting state before you act. The "before" check proves your action made the change:

```elixir
test "removes the row from the table", %{conn: conn} do
  conn
  |> visit(~p"/gentoken")
  |> generate_token("Phoenix Test Token", Date.utc_today())
  |> click_button("Close")
  |> assert_has("#tokens-table tr", text: "Phoenix Test Token")
  |> click_button("Delete")
  |> assert_has("#flash-info", text: "Token deleted successfully.")
  |> refute_has("#tokens-table tr", text: "Phoenix Test Token")
end
```

Pull repeated steps into a private helper that takes and returns the session, like `generate_token/3` in Uli's token test.

Add a short comment above an assertion when it isn't obvious what it proves.

## Interacting with the page

| Function | Use it for |
| --- | --- |
| `visit(conn, ~p"/path")` | Open a page |
| `fill_in("Label", with: value)` | Text, textarea, date inputs |
| `fill_in("css selector", "Label", with: value)` | Inputs whose label repeats or is empty |
| `select("Label", option: "Text")` | `<select>` dropdowns |
| `choose("Label")` | Radio buttons |
| `check("Label")` / `uncheck("Label")` | Checkboxes |
| `click_button("Text")` | Buttons, including form submit |
| `click_link("Text")` | Links |
| `submit()` | Submit the form you just filled, when there is no button to click |
| `within("#form-id", fn s -> ... end)` | Limit everything inside to one part of the page |

The table lists what we use most. The [PhoenixTest function docs](https://hexdocs.pm/phoenix_test/PhoenixTest.html#functions) list every action and assertion.

## Finding the right element

**Prefer labels.** `fill_in("Token Name", ...)` finds the input whose `<label>` says "Token Name". It's how a user finds it, and it breaks if the label disappears, which is what you want.

**Labels match exactly by default.** If a label has extra text, such as an emoji in front, use `exact: false`:

```elixir
|> choose("Low", exact: false)
```

**Repeated labels need a selector.** In Uli's slur form, "Yes" and "No" appear for both "Is it Casual?" and "Has it been appropriated?". Pass a CSS selector first and the label second:

```elixir
|> choose("input[name='crowdsourced_slur[casual]']", "Yes")
```

**Inputs with no label.** Use a CSS selector and an empty label:

```elixir
|> fill_in("input[name='search[query]']", "", with: "phoenix")
```

Don't skip the `""`. Without it, PhoenixTest reads the selector as label text.

Better still, add a `label` to the input in the template.

**Use `within/3` to scope.** It stops a label from matching something elsewhere on the page:

```elixir
within(session, "#token-form", fn form ->
  form
  |> fill_in("Token Name", with: name)
  |> click_button("Save")
end)
```

## Assertions

```elixir
|> assert_has("#flash-info", text: "Slur updated successfully")
|> assert_has("#tokens-table tr", text: "Phoenix Test Token", count: 1)
|> refute_has("#token_access_level option", text: "Admin")
|> assert_path(~p"/crowdsource-contributions")
```

- Make the selector as narrow as you can. `#tokens-table tr` proves the name is in the table. A broad selector like `body` would also pass if the name only showed up in a flash message.
- `text:` matches text inside the element (a substring, unless you pass `exact: true`).
- `count:` checks exactly how many elements match. It is useful for "only one row was added".
- `label:` and `value:` check form inputs: `assert_has("input", label: "Email address", value: user.email)`.
- `refute_has` is the opposite. Use it for "this is gone" or "this should never show".

**Checking which radio or checkbox is selected.** Combine `:checked` with the label:

```elixir
|> assert_has("input[name='crowdsourced_slur[casual]']:checked", label: "Yes")
|> assert_has("input[name='crowdsourced_slur[categories][]']:checked", label: "Other")
```

**Checking the database.** When the page doesn't show everything that was saved, finish the pipe and query the database directly. This is also how a test goes one layer deeper. For example, a token test could check that the generated token is valid, not just that a row appeared. This slur test checks fields the page never shows:

```elixir
saved = Repo.get!(CrowdsourcedSlur, slur.id)
assert saved.level_of_severity == :high
assert Enum.sort(saved.categories) == [:other, :religion]
```

## Gotchas we've hit

- **`phx-update` needs an `id`.** LiveView raises in tests if an element has `phx-update` but no `id`. Add the `id` in the template.
- **Hidden elements are still in the HTML.** PhoenixTest doesn't run CSS. A modal that is always in the DOM always "exists", so assert on what makes it visible (for example, `#example-modal.block`).
- **Don't use `:fl-contains` selectors.** That's Floki-only syntax. LiveView parses HTML with LazyHTML, which rejects it. Use `text:` instead.
- **Ids keep going up.** The sandbox rolls back rows but not Postgres id sequences. Never hardcode an id. Use the one your fixture returned.

## Debugging a failing test

- Read the error. When PhoenixTest can't find an element, it prints the HTML it searched, and the answer is usually there.
- Add `|> open_browser()` to the pipe to open the current page in your browser.
- Run one test by line number (`mix test path/to/file.exs:42`) to keep the output short.

## What PhoenixTest can't test

PhoenixTest doesn't run JavaScript. Anything in a `phx-hook`, an `assets/js` file, or a `data-confirm` dialog won't run. You can check that the server reacts correctly to the event a hook would push. To check the JavaScript itself you need a browser-based tool (such as `phoenix_test_playwright`) or JS unit tests.
