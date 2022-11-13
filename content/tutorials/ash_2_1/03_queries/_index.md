---
# Title, summary, and page position.
linktitle: 03-Queries
summary: Ash Data-Layer handles data Persistence within each resource.
weight: 4
icon: book-reader
icon_pack: fas

# Page metadata.
title: 03-Queries
date: '2022-11-04T00:00:00Z'
type: book # Do not modify.
---

Ash has extensive Query support (across DataLayers)!

Zach (Ash's Author) is clear that he was not interested in recreating something like Active Record.  [Ash Queries](https://hexdocs.pm/ash/Ash.Query.html) are quite flexible.  For now we will start with [filters](https://hexdocs.pm/ash/Ash.Filter.html), [sort](https://hexdocs.pm/ash/Ash.Query.html#sort/3) and [select](https://hexdocs.pm/ash/Ash.Query.html#select/3).  However, there are many [Query Functions](https://hexdocs.pm/ash/Ash.Query.html#functions) available -- including `sort`, `distinct`, `aggregate`, `calculate`, `limit`, `offset`, `subset_of`, etc (more or less any Query mechanism needed).  The nice thing is that this functions with all Data Layer, ETS, SQL, Mnesia, etc.

To learn more visit:

* [Ash Queries](https://hexdocs.pm/ash/Ash.Query.html)
* [Ash Queries](https://www.ash-hq.org/docs/module/ash/2.4.1/ash-query)
* [Writing an Ash Filter](https://www.ash-hq.org/docs/module/ash/2.4.1/ash-filter)

## Critical Query Functions

In order to user Queries we must REQUIRE the Ash.Query macro with: `require Ash.Query` so an iex session might look like:

```elixir
require Ash.Query

# a simple 'read' returns ALL users:
Support.AshApi.read!(Support.User)

# don't return duplicate emails
Support.User
|> Ash.Query.new()
|> Ash.Query.distinct(query, :last_name)
|> Support.AshApi.read!()

# we can sort the results with:
Support.User
|> Ash.Query.sort([last_name: :desc, first_name: :asc])
|> Support.AshApi.read!()

# we can limit our results to the first value - with a limit 1
Support.User
|> Ash.Query.sort([last_name: :desc, first_name: :asc])
|> Ash.Query.limit(1)
|> Support.AshApi.read!()

# with filter we can return users with 'Office' within the department_name
Support.User
|> Ash.Query.filter(contains(department_name, "Office"))
|> Support.AshApi.read!()

# we can add multiple filters and build complex filters
Support.User
|> Ash.Query.filter(contains(department_name, "Office"))
|> Ash.Query.filter(account_type == :employee and not(contains(department_name, "Admin")))
|> Support.AshApi.read!()

# we can limit what values are returned with select
Support.User
|> Ash.Query.filter(contains(department_name, "Office"))
|> Ash.Query.filter(account_type == :employee and not(contains(department_name, "Admin")))
|> Ash.Query.sort([last_name: :desc, first_name: :asc])
|> Ash.Query.limit(1)
|> Ash.Query.select([:first_name, :last_name])
|> Support.AshApi.read!()
# notice our return only contains id, first_name and last_name now
[
  #Support.User<
    __meta__: #Ecto.Schema.Metadata<:loaded>,
    id: "23bb05e1-936a-4dc6-94b4-a2123a37eb65",
    email: nil,
    first_name: "Nyima",
    middle_name: nil,
    last_name: "Sönam",
    admin: nil,
    account_type: nil,
    department_name: nil,
    inserted_at: nil,
    updated_at: nil,
    aggregates: %{},
    calculations: %{},
    __order__: nil,
    ...
  >
]
```
