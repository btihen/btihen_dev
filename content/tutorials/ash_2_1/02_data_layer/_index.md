---
# Title, summary, and page position.
linktitle: 02-Data-Layer
summary: Ash Data-Layer handles data Persistence within each resource.
weight: 3
icon: book-reader
icon_pack: fas

# Page metadata.
title: 02-Data-Layer
date: '2022-11-04T00:00:00Z'
type: book # Do not modify.
---

Now it is time to enable data persistence. When we configure a persistent data-layer - Ash makes many new features available - these include:

* Unique Resources (Identities)
* Resource Queries (at the data layer)
* Data Calculations (at the Data Layer)
* Relationships between Resources
* Aggregate Queries between Resource Relationships

We will start with simple persistance (ETS) an OTP based memory based storage
The we will switch to using postgreSQL for long-term storage

## Data Layer (ETS Persistence)

In order to read, update and generally execute queries we will add persistance.  We will start with an OTP based method using (ETS) in memory persistance.  In a separate tutorial we will switch to PostgreSQL.

ETS is an in-memory (OTP based) way to persist data (we will work with PostgreSQL later).
Once we have persisted data we can explore relationships.

To add ETS to the Data Layer we need to change the line `use Ash.Resource` to:

```elixir
# lib/support/resources/user.ex
defmodule Support.User do
  use Ash.Resource,
    data_layer: Ash.DataLayer.Ets
  # ...
end
```

Lets try this out - we will create several user and then query for them.

```elixir
iex -S mix phx.server

customer = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_customer, %{first_name: "Ratna", last_name: "Sönam", email: "nyima@example.com"}
    )
  |> Support.AshApi.create!()
)
employee = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_employee, %{first_name: "Nyima", last_name: "Sönam", email: "nyima@example.com",
                       department_name: "Office Actor", account_type: :employee}
    )
  |> Support.AshApi.create!()
)
admin = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_admin, %{first_name: "Karma", last_name: "Sönam", email: "karma@example.com",
                    department_name: "Office Admin", account_type: :employee, admin: true}
    )
  |> Support.AshApi.create!()
)

# now we should be able to 'read' all our users:
Support.AshApi.read!(Support.User)
```

## Identity (Uniqueness)

In order to ensure that the email is a unique identifier - we use the `identities` feature.  We can test that one or multiple attributes build a unique `identity`.

This feature behaves differently depending on the Data Layer in use.  In particular, from the docs, we see
  Ash.DataLayer.Ets will actually require you to set pre_check_with since the ETS data layer has no built in support for unique constraints.

For a single

```elixir
  identities do
    identity :email, [:email], pre_check_with: Support.AshApi
    identity :names, [:first_name, :middle_name, :last_name], pre_check_with: Support.AshApi
  end
```

Let's test the uniqueness of the email attribute -
  `identity :email, [:email], pre_check_with: Support.AshApi`

```elixir
iex -S mix phx.server

customer = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_customer, %{first_name: "Ratna", last_name: "Sönam", email: "ratna@example.com"}
    )
  |> Support.AshApi.create!()
)
employee = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_employee, %{first_name: "Nyima", last_name: "Sönam", email: "ratna@example.com",
                       department_name: "Office Actor", account_type: :employee}
    )
  |> Support.AshApi.create!()
)
# we should get this error
** (Ash.Error.Invalid) Input Invalid

* email: has already been taken.

# But the following should work
employee = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_employee, %{first_name: "Nyima", last_name: "Sönam", email: "nyima@example.com",
                       department_name: "Office Actor", account_type: :employee}
    )
  |> Support.AshApi.create!()
)
```

Let's test the uniqueness of the all the name attributes -
  `identity :names, [:first_name, :middle_name, :last_name], pre_check_with: Support.AshApi`

```elixir
iex -S mix phx.server

employee1 = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_employee, %{first_name: "Nyima", last_name: "Sönam", email: "nyima@example.com",
                       department_name: "Office Actor"}
    )
  |> Support.AshApi.create!()
)
employee2 = (
  Support.User
  |> Ash.Changeset.for_create(
      :create, %{first_name: "Nyima", last_name: "Sönam", email: "nyima_karma@example.com",
                       department_name: "Office Actor", account_type: :employee}
    )
  |> Support.AshApi.create!()
)
# we should get this error
** (Ash.Error.Invalid) Input Invalid

* email: has already been taken.

# But the following should work
employee = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_employee, %{first_name: "Nyima", last_name: "Sönam", email: "nyima@example.com",
                       department_name: "Office Actor", account_type: :employee}
    )
  |> Support.AshApi.create!()
)
```

## Ash Queries

Zach is clear that he was not interested in recreating something like Active Record.  [Ash Queries](https://hexdocs.pm/ash/Ash.Query.html) are quite flexible.  For now we will start with [filters](https://hexdocs.pm/ash/Ash.Filter.html), [sort](https://hexdocs.pm/ash/Ash.Query.html#sort/3) and [select](https://hexdocs.pm/ash/Ash.Query.html#select/3).  However, there are many [Query Functions](https://hexdocs.pm/ash/Ash.Query.html#functions) available -- including `sort`, `distinct`, `aggregate`, `calculate`, `limit`, `offset`, `subset_of`, etc (more or less any Query mechanism needed).  The nice thing is that this functions with all Data Layer, ETS, SQL, Mnesia, etc.

To learn more visit:

* [Ash Queries](https://hexdocs.pm/ash/Ash.Query.html)
* [Ash Queries](https://www.ash-hq.org/docs/module/ash/2.4.1/ash-query)
* [Writing an Ash Filter](https://www.ash-hq.org/docs/module/ash/2.4.1/ash-filter)

### Critical Query Functions

```elixir
require Ash.Query

# a simple 'read' returns ALL users:
Support.AshApi.read!(Support.User)

# don't return duplicate emails
Support.User
|> Ash.Query.new()
|> Ash.Query.distinct(query, :email)
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
