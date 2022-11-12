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

## Ash Postgres -- Configure

My goal here was to configure Ash so that a pre-existing Phoenix Ecto Repo would keep working and Ash would work along side it.

Here is what I did (a deeper dive into: <https://github.com/ash-project/ash_postgres/blob/main/documentation/tutorials/get-started-with-postgres.md>)

We will make a new Ash Repo:

```elixir
# lib/helpdesk/support/repo.ex
defmodule Support.Repo do
  use AshPostgres.Repo, otp_app: :helpdesk
end
```

Now tell Phoenix Config:

```elixir
# config/config.exs
import Config

# add Ash APIs to config
config :helpdesk,
  ash_apis: [Helpdesk.Support]

config :helpdesk,
  ecto_repos: [
    Support.Repo, # add newly created Support Repo
    Helpdesk.Repo
  ]
# ...
```

In the `config/dev.exs` config add the Support database config alongside the Phoenix Ecto config:

```elixir
# config/dev.exs
import Config

# Phoenix Dev DB config
config :helpdesk, Helpdesk.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  database: "helpdesk_dev",
  stacktrace: true,
  show_sensitive_data_on_connection_error: true,
  pool_size: 10

# Ash dev DB Config
config :helpdesk, Support.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  database: "support_dev",
  stacktrace: true,
  show_sensitive_data_on_connection_error: true,
  pool_size: 10
# ...
```

Update `config/test.exs` database settings with:

```elixir
# config/test.exs
import Config

# Configure your phoenix database
config :helpdesk, Helpdesk.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  database: "helpdesk_test#{System.get_env("MIX_TEST_PARTITION")}",
  pool: Ecto.Adapters.SQL.Sandbox,
  pool_size: 10
# Configure your ash (support) database
config :helpdesk, Support.Repo,
  username: "postgres",
  password: "postgres",
  hostname: "localhost",
  database: "support_test#{System.get_env("MIX_TEST_PARTITION")}",
  pool: Ecto.Adapters.SQL.Sandbox,
  pool_size: 10
# ...
```

Finally, update `config/runtime.exs` with (note I haven't deployed - so this is untested):

```elixir
# config/runtime.exs
import Config

if System.get_env("PHX_SERVER") do
  config :helpdesk, HelpdeskWeb.Endpoint, server: true
end

if config_env() == :prod do
  support_database_url =
    System.get_env("SUPPORT_DATABASE_URL") ||
      raise """
      environment variable DATABASE_URL is missing.
      For example: ecto://USER:PASS@HOST/DATABASE
      """
  phoenix_database_url =
    System.get_env("PHOENIX_DATABASE_URL") ||
      raise """
      environment variable DATABASE_URL is missing.
      For example: ecto://USER:PASS@HOST/DATABASE
      """

  maybe_ipv6 = if System.get_env("ECTO_IPV6"), do: [:inet6], else: []

  config :helpdesk, Support.Repo,
    # ssl: true,
    url: support_database_url,
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10"),
    socket_options: maybe_ipv6

  config :helpdesk, Helpdesk.Repo,
    # ssl: true,
    url: phoenix_database_url,
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10"),
    socket_options: maybe_ipv6
# ...
```

Hopefully you can start `iex -S mix`

## Ash Postgres -- Enable

Now we need to tell our `resources` about our new `Support.Repo`, we will do this in the 'ticket' and 'user' files -- we will replace `use Ash.Resource, data_layer: Ash.DataLayer.Ets` with:

```elixir
# lib/helpdesk/support/resources/ticket.ex
defmodule Helpdesk.Support.Ticket do
  use Ash.Resource,
    data_layer: AshPostgres.DataLayer

  postgres do
    table "tickets"
    repo Support.Repo
  end
# ...
```

and

```elixir
# lib/helpdesk/support/resources/user.ex

defmodule Helpdesk.Support.User do
  use Ash.Resource,
    data_layer: AshPostgres.DataLayer

  postgres do
    table "users"
    repo Support.Repo
  end
# ...
```

Now we should be able to create the 'Support' database with:

```bash
mix ash_postgres.create
```

If you get an --apis error then you probably forgot the API config in `config/config.exs` try adding:

```elixir
# config/config.exs
import Config

config :helpdesk,
  ash_apis: [Helpdesk.Support]

# ...
```

Assuming this worked now you can generate a migration from existing 'resources' with:

```elixir
mix ash_postgres.generate_migrations --name support_add_tickets_and_users
```

This should create a migration that looks like:

```elixir
# priv/repo/migrations/YYYYMMDDHHmmSS_support_add_tickets_and_users.exs
defmodule Support.Repo.Migrations.SupportAddTicketsAndUsers do
  use Ecto.Migration

  def up do
    create table(:users, primary_key: false) do
      add :id, :uuid, null: false, primary_key: true
      add :name, :text
    end

    create table(:tickets, primary_key: false) do
      add :id, :uuid, null: false, primary_key: true
      add :subject, :text, null: false
      add :status, :text, null: false, default: "open"
      add :reporter_id,
          references(:users,
            column: :id,
            name: "tickets_reporter_id_fkey",
            type: :uuid,
            prefix: "public"
          )
      add :representative_id,
          references(:users,
            column: :id,
            name: "tickets_representative_id_fkey",
            type: :uuid,
            prefix: "public"
          )
    end
  end

  def down do
    drop constraint(:tickets, "tickets_representative_id_fkey")
    drop constraint(:tickets, "tickets_reporter_id_fkey")
    drop table(:tickets)
    drop table(:users)
  end
end
```

Finally we can update the database by migrating using:

```bash
mix ash_postgres.migrate
```

## Ash Postgres - Actions

Now all our previous actions and queries should still work, but now persist long-term (even if we kill our iex session).

Let's start a new `iex` session now that we have switched to PostgreSQL and try out our Actions like before.

```elixir
iex -S mix

for i <- 0..5 do
  ticket =
    Helpdesk.Support.Ticket
    |> Ash.Changeset.for_create(:new, %{subject: "Issue #{i}"})
    |> Helpdesk.Support.create!()

  if rem(i, 2) == 0 do
    ticket
    |> Ash.Changeset.for_update(:close)
    |> Helpdesk.Support.update!()
  end
end
```

Now kill `iex` and start it again and ensure the following Queries works (and find the data we stored earlier):

```elixir
iex -S mix

# use `read` to list all users
{:ok, users} = Helpdesk.Support.read(Helpdesk.Support.User)
{:ok, tickets}= Helpdesk.Support.read(Helpdesk.Support.Ticket)

# use 'get' to get one record when you know the id
ticket_last = List.last(tickets)
Helpdesk.Support.get(Helpdesk.Support.Ticket, ticket_last.id)

# use Queries for more complex (nuanced lookups)
require Ash.Query

# Show the tickets where the subject contains "2"
Helpdesk.Support.Ticket
|> Ash.Query.filter(contains(subject, "2"))
|> Helpdesk.Support.read!()

# Show the tickets that are closed and their subject does not contain "4"
Helpdesk.Support.Ticket
|> Ash.Query.filter(status == :closed and not(contains(subject, "4")))
|> Helpdesk.Support.read!()
```

## Aggregates

Aggregates are a tool to include grouped data regarding **relationships** <https://hexdocs.pm/ash/Ash.Resource.Dsl.html#module-aggregates>

Aggregates are powerful because they will be translated to SQL, and can be used in filters and sorts (they are a bit like rails `scopes`).

So to try this out lets add ticket aggregates to our users - so we know how many tickets each user has (per ticket status)

The first argument is the aggregate name and the second is the relationship to count (and of course we can filter the results for mor meaningful grouping)

Possible aggregrates include:

* count
* first
* sum
* list

Let's start with trying aggregates within queries:

```elixir
Helpdesk.Support.User
|> Ash.Query.aggregate(:all_reported_tickets, [:reported_tickets])
|> Helpdesk.Support.read!()

Helpdesk.Support.User
|> Ash.Query.aggregate(:all_reported_tickets, [:reported_tickets],
                       filter: expr(status != :closed))
|> Helpdesk.Support.read!()
```

The possible filters available are found at: <https://hexdocs.pm/ash/Ash.Filter.html>

The basic aggregate format is `method :aggregate_name, :relationship_name`

Let's create some ticket aggregates for our users:

```elixir
# lib/helpdesk/support/resources/user.ex
  aggregates do
    count :all_reported_tickets, :reported_tickets
    count :open_reported_tickets, :reported_tickets do
      filter expr(status == :open || status == :new)
    end
    count :closed_reported_tickets, :reported_tickets do
      filter expr(status == :closed)
    end

    count :active_assigned_tickets, :assigned_tickets do
      filter expr(status == :open || status == :new)
    end
    count :closed_assigned_tickets, :assigned_tickets do
      filter expr(status == :closed)
    end
  end

  relationships do
    has_many :assigned_tickets, Helpdesk.Support.Ticket do
      destination_attribute :representative_id
    end
    has_many :reported_tickets, Helpdesk.Support.Ticket do
      destination_attribute :reporter_id
    end
  end
```

To use aggregates, we can access the aggregates them within our queries (filters and sorts).  Here is an example using the closed tickets within a query:

```elixir
iex -S mix

require Ash.Query

users = Helpdesk.Support.User
|> Ash.Query.filter(closed_assigned_tickets < 4) # users with less than 4 closed assigned tickets
|> Ash.Query.sort(closed_assigned_tickets: :desc)
|> Helpdesk.Support.read!()
# we get (as you see only the requested aggregate will be queried / calculated):
[
  #Helpdesk.Support.User<
    closed_assigned_tickets: 1,
    active_assigned_tickets: #Ash.NotLoaded<:aggregate>,
    closed_reported_tickets: #Ash.NotLoaded<:aggregate>,
    open_reported_tickets: #Ash.NotLoaded<:aggregate>,
    all_reported_tickets: #Ash.NotLoaded<:aggregate>,
    reported_tickets: #Ash.NotLoaded<:relationship>,
    assigned_tickets: #Ash.NotLoaded<:relationship>,
    __meta__: #Ecto.Schema.Metadata<:loaded, "users">,
    id: "1b00d77c-bff4-4f3b-8453-67f7c3748a59",
    name: "Jose",
    aggregates: %{},
    calculations: %{},
    __order__: nil,
    ...
  >,
  #Helpdesk.Support.User<
    closed_assigned_tickets: 0,
    active_assigned_tickets: #Ash.NotLoaded<:aggregate>,
    closed_reported_tickets: #Ash.NotLoaded<:aggregate>,
    open_reported_tickets: #Ash.NotLoaded<:aggregate>,
    all_reported_tickets: #Ash.NotLoaded<:aggregate>,
    reported_tickets: #Ash.NotLoaded<:relationship>,
    assigned_tickets: #Ash.NotLoaded<:relationship>,
    __meta__: #Ecto.Schema.Metadata<:loaded, "users">,
    id: "0c18bf1d-44bb-4499-b08b-48abf6fd27f4",
    name: "Nyima",
    aggregates: %{},
    calculations: %{},
    __order__: nil,
    ...
  >,
]

# even though aggregates are not automatically loaded unless requested
users = Helpdesk.Support.read!(Helpdesk.Support.User)
  Helpdesk.Support.User<
    closed_assigned_tickets: #Ash.NotLoaded<:aggregate>,
    active_assigned_tickets: #Ash.NotLoaded<:aggregate>,
    closed_reported_tickets: #Ash.NotLoaded<:aggregate>,
    open_reported_tickets: #Ash.NotLoaded<:aggregate>,
    all_reported_tickets: #Ash.NotLoaded<:aggregate>,
    reported_tickets: #Ash.NotLoaded<:relationship>,
    assigned_tickets: #Ash.NotLoaded<:relationship>,
    __meta__: #Ecto.Schema.Metadata<:loaded, "users">,
    id: "1b00d77c-bff4-4f3b-8453-67f7c3748a59",
    name: "Jose",
    aggregates: %{},
    calculations: %{},
    __order__: nil,

# we load aggregates as needed after the initial query
Helpdesk.Support.load!(users, :active_assigned_tickets)

# we can load multiple calculation
users = Helpdesk.Support.read!(Helpdesk.Support.User)
Helpdesk.Support.load!(users, [:active_assigned_tickets, :closed_assigned_tickets])
```

## Calculations

We can do SQL calculations too:

```elixir

Helpdesk.Support.User
|> Ash.Query.calculate(:username, expr(name <> "-"))
|> Helpdesk.Support.read!()

# or mixed
Helpdesk.Support.User
|> Ash.Query.calculate(:username, expr(name <> "-"))
|> Ash.Query.aggregate(:all_reported_tickets, [:reported_tickets],
                       filter: expr(status != :closed))
|> Helpdesk.Support.read!()
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

Helpdesk.Support.User
|> Ash.Query.filter(all_assigned_tickets > 0) # prevent divide by zero
|> Ash.Query.filter(assigned_open_percent > 0.25)
|> Ash.Query.sort(:assigned_open_percent)
|> Ash.Query.load(:assigned_open_percent)
|> Helpdesk.Support.read!()

# try out the username calculation
Helpdesk.Support.User
|> Ash.Query.load(:username)
|> Helpdesk.Support.read!()

# calculations can also be loaded in a separate query afterwards
users = Helpdesk.Support.read!(Helpdesk.Support.User)
Helpdesk.Support.load!(users, :username)

# we can load multiple calculations and aggregates
users = Helpdesk.Support.read!(Helpdesk.Support.User)
Helpdesk.Support.load!(users, [:username, :closed_assigned_tickets])
```

# Resources

* <https://www.youtube.com/watch?v=2U3vQHXCF0s>
* <https://hexdocs.pm/ash/relationships.html#loading-related-data>
* <https://www.ash-hq.org/docs/guides/ash/2.4.1/tutorials/get-started.md>
* <https://github.com/ash-project/ash_postgres/blob/main/documentation/tutorials/get-started-with-postgres.md>
