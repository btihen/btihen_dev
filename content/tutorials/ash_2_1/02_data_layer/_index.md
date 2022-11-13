---
# Title, summary, and page position.
linktitle: 02-DataLayer(ETS)
summary: Ash Data-Layer handles data Persistence within each resource.  ETS is wonderful for experimenting and prototyping without having to do migrations, etc.
weight: 3
icon: book-reader
icon_pack: fas

# Page metadata.
title: 02-DataLayer(ETS)
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
The we will switch to using postgreSQL for long-term storage.

**ETS is wonderful for experimenting and prototyping without having to do migrations, etc.**

## ETS Persistence

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

We can also now read back from our DataLayer and we should now be able to read back the users we have created with:
```elixir
Support.User
|> Support.AshApi.read!()
```
