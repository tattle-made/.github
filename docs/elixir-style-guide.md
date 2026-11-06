# Elixir Style Guide for Tattle

## Motivation

As new developers and AI agents begin to contribute code, the lack of a consistent style guide is causing code with subtle variations to get checked into Tattle repositories. Stylistic concerns often take a backseat while we hurry to resolve functional issues in PR reviews, but over time they lead to problems like slow onboarding of new developers and difficulty in grokking the code bases of various Tattle projects.

The goal of the guide is to provide general guidelines but also give specific examples when possible.

## Style Guide

### 1. Prefer Structs

Structs are a powerful way to document types in a codebase. As much as possible, rely on structs when data has a known set of fields, such as function arguments, return values and errors that cross module boundaries.

### 2. Use of |> operator

Limit it to transformations of a single struct. When dealing with transformation of multiple structs and/or enforcing other business logic, use `with`.

### 3. Defining Functions

Public functions that can fail should return `{:ok, result}` or `{:error, reason}`. This also pairs well with `with`. Pure transformations, `?` predicates, `!` functions and framework callbacks keep their usual return shape.

Reason or error can be primitive data types like atoms (`:not_found`), strings or numbers. When they are complex, define structs for them.

### 4. Model and document complex workflows/orchestrations using a `with` block

```elixir
with {:ok, token} <- Auth.get_token(user),
     {:ok, py} <- Python.start(python_path: @python_path, python: @python_executable),
     {:ok, _run_id} <- Instascraper.initialize(py, token) do
  {:ok, py}
else
  {:error, %Instascraper.Error{type: :init_fail}} = error ->
    Logger.info("a")
    error

  {:error, %Python.Error{type: :port_closed}} = error ->
    Logger.info("b")
    error

  {:error, reason} = error ->
    Logger.info(inspect(reason))
    error
end
```

### 5. Defining Helpers

Prefer extracting common expressions into a helper function. You can even go further with pattern matching and guards.

```elixir
def list_posts(size), do: Posts.list(limit: page_size(size))

def list_comments(size), do: Comments.list(limit: page_size(size))

defp page_size(nil), do: @default_page_size

defp page_size(size) when size > @max_page_size, do: @max_page_size

defp page_size(size), do: size
```

### 6. Ordering of parts of the module

1. `@moduledoc`
2. `@behaviour`
3. `use`
4. `import`
5. `alias`
6. `require`
7. module attributes
8. `defstruct`
9. `@type`
10. public functions
11. private functions

### 7. Use pattern matching on struct arguments for function heads

This documents the expected type and raises a `FunctionClauseError` when a caller passes anything else.

```elixir
def update_client(%Client{} = client, attrs) do
  ...
end
```
