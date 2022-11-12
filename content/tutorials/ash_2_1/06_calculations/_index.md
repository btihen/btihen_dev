---
# Title, summary, and page position.
linktitle: '06-Calculations'
summary: Ash Belongs To relationships
weight: 7
icon: book-reader
icon_pack: fas

# Page metadata.
title: '06-Calculations'
date: '2022-11-04T00:00:00Z'
type: book # Do not modify.
---

**IN PROGRESS**- Probably lots of errors

### Calculated Queries

[Calculated Queries](https://hexdocs.pm/ash/calculations.html) allow us the logic to extend our resources - and with SQL as the Data Layer, these will generate SQL to do the work instead of elixir code!

The simplest way to create a calculation is to add it to the model - for example:

```elixir
# lib/support/resources/user.ex
# ...
  calculations do
    calculate :full_name, :string, expr(first_name <> " " <> last_name)
    # calculate :formal_name, :string, expr(
    #   last_name  <> ", " <> (
    #                           [first_name, middle_name]
    #                           |> Enum.map(fn string -> is_binary(string) end)
    #                           |> Enum.join(" ")
    #                         )
    # )
  end
  # ...
end
```

Then we can retrieve the calculation with the - 'calculate' and 'load' functions:

```elixir
require Ash.Query

# we can get the calculated resource field with - 'calculate' and 'load':
Support.User
|> Ash.Query.new()
|> Ash.Query.calculate(full_name)
|> Ash.Query.load([:full_name])
|> Support.AshApi.read!()
# you should get something like:
[
  #Support.User<
    full_name: "Nyima Sönam",
    aggregates: %{},
    calculations: %{},
    ...
  >,
...
]

# calucated results can be sorted upon and otherwise used in the query
Support.User
|> Ash.Query.new()
|> Ash.Query.calculate(full_name)
|> Ash.Query.load([:full_name])
|> Ash.Query.sort(full_name: :asc)
|> Support.AshApi.read!()
# you should get something like:
[
  #Support.User<
    full_name: "Nyima Sönam",
    aggregates: %{},
    calculations: %{},
    ...
  >,

# on the fly calculations - don't work, I must be overlooking something
# Support.User
# |> Ash.Query.new()
# |> Ash.Query.calculate(:both_names, :string, expr(first_name <> " " <> last_name))
# |> Ash.Query.load([:full_name])
# |> Support.AshApi.read!()
```

## Calculations

We can do SQL calculations too:

```elixir

Support.User
|> Ash.Query.new()
|> Ash.Query.calculate(:username, :string, expression(name <> "-"))
|> Support.AshApi.read!()

# or mixed
Support.User
|> Ash.Query.calculate(:username, expr(name <> "-"))
|> Ash.Query.aggregate(:all_reported_tickets, [:reported_tickets],
                       filter: expr(status != :closed))
|> Support.AshApi.read!()
```

Pre-built calculations:

```elixir
# lib/helpdesk/support/resources/user.ex
  calculations do
    # calculate :full_name, :string, expr(first_name <> " " <> last_name)
    calculate :username, :string, expr(name <> "-" <> id)
    calculate :assigned_open_percent, :float, expr(active_assigned_tickets / all_assigned_tickets )
  end
```

USAGE:

```elixir
iex -S mix

require Ash.Query

Support.User
|> Ash.Query.filter(all_assigned_tickets > 0) # prevent divide by zero
|> Ash.Query.filter(assigned_open_percent > 0.25)
|> Ash.Query.sort(:assigned_open_percent)
|> Ash.Query.load(:assigned_open_percent)
|> Support.AshApi.read!()

# try out the username calculation
Support.User
|> Ash.Query.load(:username)
|> Support.AshApi.read!()

# calculations can also be loaded in a separate query afterwards
users = Support.AshApi.read!(Support.User)
Support.AshApi.load!(users, :username)

# we can load multiple calculations and aggregates
users = Support.AshApi.read!(Support.User)
Support.AshApi.load!(users, [:username, :closed_assigned_tickets])
```

# Resources

* <https://www.youtube.com/watch?v=2U3vQHXCF0s>
* <https://hexdocs.pm/ash/relationships.html#loading-related-data>
* <https://www.ash-hq.org/docs/guides/ash/2.4.1/tutorials/get-started.md>
* <https://github.com/ash-project/ash_postgres/blob/main/documentation/tutorials/get-started-with-postgres.md>
