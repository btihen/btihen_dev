---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Elixir - Prioritized Algorithms"
subtitle: "Simple Clean Prioritized Algorithms"
summary: "continue on failure, stop on success"
authors: ["btihen"]
tags: ["Elixir", "With", "Prioritize With", "Reverse With"]
categories: ["Code", "Elixir Language", "Prioritized Algorithms", "Prioritized Pattern"]
date: 2024-10-02T01:01:53+02:00
lastmod: 2024-10-02T01:01:53+02:00
featured: true
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

## Context

Recently we at work we had the need to try a variety of different algorithms and stop at the first success. A lot like in the article:

[Ruby Lazy Priorities](https://btihen.dev/posts/ruby/ruby_3_x_lazy_eval_prioritized_algorithms/)

In this case, we were looking for uniquely matching records.

In our actual code we were using complex `ecto` queries.

Here we will use simpler Enums to match, but the prioritized logic architecture is similar.

## Setup

Let's create a test elixir project to look for the best contact person within a list.

```bash
mix new person
cd person
```

Let's add the code to people:
```elixir
# lib/person.ex
defmodule Person do
  defstruct name: nil, rank: nil, role: nil, years_of_service: nil
end
```

and we will want a little run script and test our person struct:
```elixir
iex -S mix

eve = %Person{name: "Evelyn", rank: 5, role: "Contributor", years_of_service: 2}
eve.name

staff = [
  %Person{name: "Alice", rank: 1, role: "Purchaser", years_of_service: 3},
  %Person{name: "Bobby", rank: 2, role: "Manager", years_of_service: 5},
  %Person{name: "Charlie", rank: 3, role: "Executive", years_of_service: 7},
  %Person{name: "David", rank: 4, role: "Contributor", years_of_service: 1},
  eve
]
```

Assuming our person module works - let's continue.

## Chaining - First Attempt

My First attempt was with chaining first filter for the correct names and then the best role.

```elixir
# lib/prioritized_chain.ex

defmodule PrioritizedChain do
  def best_contact(staff, name) do
    with {:ok, filtered_names} <- filter_name(staff, name),
         {:ok, person} <- filter_role(filtered_names) do
      {:ok, person}
    else
      _ -> {:error, :no_unique_match}
    end
  end

  def filter_name(staff, name) do
    exact_match = fn s -> Enum.filter(s, &(&1.name == name)) end
    contains_match = fn s -> Enum.filter(s, &String.contains?(&1.name, name)) end

    {:error, staff}
    |> name_filter(exact_match)
    |> name_filter(contains_match)
    |> case do
      {:ok, filtered_staff} -> {:ok, filtered_staff}
      _ -> {:error, staff}
    end
  end

  # on a match we simply fall through to the next clause
  def filter_role(staff) do
    {:error, staff}
    |> role_filter("Purchaser")
    |> role_filter("Manager")
    |> role_filter("Executive")
    |> role_filter("Contributor")
    |> case do
      {:ok, person} -> {:ok, person}
      _ -> {:error, :no_unique_match}
    end
  end

  defp name_filter({:ok, staff}, _filter), do: {:ok, staff}
  defp name_filter({:error, [_ | _] = staff}, filter) do
    staff
    |> filter.()
    |> case do
      [_ | _] = filtered_staff -> {:ok, filtered_staff}
      [] -> {:error, staff}
    end
  end
  defp name_filter({:error, _}, _filter), do: {:error, []}

  # if we have a match, simply return the success result
  defp role_filter({:ok, person}, _role), do: {:ok, person}

  # if we have a non-empty list, then try to filter it and return the result
  defp role_filter({:error, [_ | _] = staff}, role) do
    staff
    |> Enum.filter(&(&1.role == role))
    |> case do
      [] -> {:error, staff}
      filtered_staff ->
        filtered_staff
        |> Enum.max_by(& &1.years_of_service)
        |> (fn person -> {:ok, person} end).()
    end
  end

  # if we have an empty list (or an unexpected value), fall through with an empty list
  # NOTE: this is a weakness, each filter might be dependent on the previous filter
  defp role_filter({:error, _}, _role), do: {:error, []}
end
```

Let's test this with:
```elixir
iex -S mix

staff = [
  %Person{name: "Alice", rank: 1, role: "Purchaser", years_of_service: 3},
  %Person{name: "Bobby", rank: 2, role: "Manager", years_of_service: 5},
  %Person{name: "Charlie", rank: 3, role: "Executive", years_of_service: 7},
  %Person{name: "David", rank: 4, role: "Contributor", years_of_service: 1},
  %Person{name: "Evelyn", rank: 5, role: "Contributor", years_of_service: 2}
]

IO.inspect(PrioritizedChain.best_contact(with_eve, "Eve"))
{:ok,
 %Person{name: "Evelyn", rank: 5, role: "Contributor", years_of_service: 2}}

IO.inspect(PrioritizedChain.best_contact(with_eve, "Jim"))
{:error, :no_unique_match}
```

This works just fine, but is relatively complex (to read). Worse yet, in the case of an error, we have to recover and pass an [].  This means these `chained` methods are dependent on each other (coupling).

## Reverse With (decoupled)

My colleage [Gernot Kogler](https://www.linkedin.com/in/gernot-kogler-075513162/),
suggested a cool way to decouple and simplify these prioritized matching logic - using a **reverse** `with`.  Using it to guide the failed cases and dropping out with success.

This was a new and surprising use of `with`, I had always used to match the happy path and not the failure path, but this creates a simple, clean and effective logic, that is easy to read _(except I like to add the unnecessary `else` to see the possible returns in the main function)_.

```elixir
# lib/prioritized_with.ex

defmodule PrioritizedWith do
  # normal with - testing for positive results
  def best_contact(staff, name) do
    with {:ok, filtered_names} <- filter_name(staff, name),
          {:ok, person} <- filter_role(filtered_names) do
      {:ok, person}
    else
      _ -> {:error, :no_unique_match}
    end
  end

  # reverse with testing for negative results
  def filter_name(staff, name) do
    exact_match = fn s -> Enum.filter(s, &(&1.name == name)) end
    contains_match = fn s -> Enum.filter(s, &String.contains?(&1.name, name)) end

    with {:error, _} <- staff |> name_filter(exact_match),
         {:error, _} <- staff |> name_filter(contains_match) do
      {:error, :no_unique_match}
    else
      {:ok, matched_names} -> {:ok, matched_names}
      _ -> {:error, :no_unique_match}
    end
  end

  defp name_filter(staff, filter) do
    filter.(staff)
    |> case do
      [_ | _] = filtered_staff -> {:ok, filtered_staff}
      [] -> {:error, :no_unique_match}
    end
  end

  # mostly, reverse with testing for negative results
  def filter_role(staff) do
    with [_ | _] <- staff,
        # assuming we have people in our array filter by the roles we most care about
        {:error, _} <- role_filter(staff, "Purchaser"),
        {:error, _} <- role_filter(staff, "Manager"),
        {:error, _} <- role_filter(staff, "Executive"),
        {:error, _} <- role_filter(staff, "Contributor") do
      {:error, :no_unique_match}
    else
      {:ok, person} -> {:ok, person}
      _ -> {:error, :no_unique_match}
    end
  end

  defp role_filter(staff, role) do
    staff
    |> Enum.filter(&(&1.role == role))
    |> case do
      # if we have results (then filter by years of service)
      [_ | _] = filtered_staff ->
        filtered_staff
        |> Enum.max_by(& &1.years_of_service)
        |> (fn person -> {:ok, person} end).()

      # otherwise return an error
      _-> {:error, :no_unique_match}
    end
  end
end
```

let's test this with:
```elixir
iex -S mix

staff = [
  %Person{name: "Alice", rank: 1, role: "Purchaser", years_of_service: 3},
  %Person{name: "Bobby", rank: 2, role: "Manager", years_of_service: 5},
  %Person{name: "Charlie", rank: 3, role: "Executive", years_of_service: 7},
  %Person{name: "David", rank: 4, role: "Contributor", years_of_service: 1},
  %Person{name: "Evelyn", rank: 5, role: "Contributor", years_of_service: 2}
]

IO.inspect(PrioritizedWith.best_contact(with_eve, "Eve"))
{:ok,
 %Person{name: "Evelyn", rank: 5, role: "Contributor", years_of_service: 2}}

IO.inspect(PrioritizedWith.best_contact(with_eve, "Jim"))
{:error, :no_unique_match}
```

Excellent and it works well and is far simpler to read.

The minimalist code can be written as (50 Lines of Code):

```elixir
# lib/prioritized_minimal_with.ex

defmodule PrioritizedMinimalWith do
  # normal with - testing for positive results
  def best_contact(staff, name) do
    with {:ok, filtered_names} <- filter_name(staff, name),
          {:ok, person} <- filter_role(filtered_names) do
      {:ok, person}
    else
      _ -> {:error, :no_unique_match}
    end
  end

  # reverse with testing for negative results
  def filter_name(staff, name) do
    exact_match = fn s -> Enum.filter(s, &(&1.name == name)) end
    contains_match = fn s -> Enum.filter(s, &String.contains?(&1.name, name)) end

    with {:error, _} <- staff |> name_filter(exact_match),
         {:error, _} <- staff |> name_filter(contains_match) do
      {:error, :no_unique_match}
    end
  end

  defp name_filter(staff, filter) do
    filter.(staff)
    |> case do
      [_ | _] = filtered_staff -> {:ok, filtered_staff}
      [] -> {:error, :no_unique_match}
    end
  end

  # mostly, reverse with testing for negative results
  def filter_role(staff) do
    with [_ | _] <- staff,
        # assuming we have people in our array filter by the roles we most care about
        {:error, _} <- role_filter(staff, "Purchaser"),
        {:error, _} <- role_filter(staff, "Manager"),
        {:error, _} <- role_filter(staff, "Executive"),
        {:error, _} <- role_filter(staff, "Contributor") do
      {:error, :no_unique_match}
    end
  end

  defp role_filter(staff, role) do
    staff
    |> Enum.filter(&(&1.role == role))
    |> case do
      # if we have results (then filter by years of service)
      [_ | _] = filtered_staff ->
        filtered_staff
        |> Enum.max_by(& &1.years_of_service)
        |> (fn person -> {:ok, person} end).()

      # otherwise return an error
      _-> {:error, :no_unique_match}
    end
  end
end
```


let's test this with:
```elixir
iex -S mix

staff = [
  %Person{name: "Alice", rank: 1, role: "Purchaser", years_of_service: 3},
  %Person{name: "Bobby", rank: 2, role: "Manager", years_of_service: 5},
  %Person{name: "Charlie", rank: 3, role: "Executive", years_of_service: 7},
  %Person{name: "David", rank: 4, role: "Contributor", years_of_service: 1},
  %Person{name: "Evelyn", rank: 5, role: "Contributor", years_of_service: 2}
]

IO.inspect(PrioritizedMinimizedWith.best_contact(with_eve, "Eve"))
{:ok,
 %Person{name: "Evelyn", rank: 5, role: "Contributor", years_of_service: 2}}

IO.inspect(PrioritizedMinimizedWith.best_contact(with_eve, "Jim"))
{:error, :no_unique_match}
```

## Conclusion

I appreciate the `prioritized with` or the `reverse with` pattern.

It decouples various algorithms from each other, is relatively easy to read (especially with the redundant `else` to clearly show what will be returned)
