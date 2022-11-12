---
# Title, summary, and page position.
linktitle: 09-PostgreSQL
summary: Ash Data-Layer handles data Persistence within each resource.
weight: 10
icon: book-reader
icon_pack: fas

# Page metadata.
title: 09-PostgreSQL
date: '2022-11-04T00:00:00Z'
type: book # Do not modify.
---

**IN PROGRESS**- Probably lots of errors

Now it is time to enable data persistence - between IEX sessions.
ETS makes it wonderful to experiment, but in production, PostgreSQL (long-term persistance will be needed).

## Configuration

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

## Enable / Activate

Now we need to tell our `resources` about our new `Support.Repo`, we will do this in the 'ticket' and 'user' files -- we will replace `use Ash.Resource, data_layer: Ash.DataLayer.Ets` with:

```elixir
# lib/support/resources/ticket.ex
defmodule Support.Ticket do
  use Ash.Resource,
    data_layer: AshPostgres.DataLayer

  postgres do
    table "tickets"
    repo Support.Repo
  end
# ...
end
```

and

```elixir
# lib/support/resources/user.ex
defmodule Support.User do
  use Ash.Resource,
    data_layer: AshPostgres.DataLayer

  postgres do
    table "users"
    repo Support.Repo
  end
# ...
end
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
    Support.Ticket
    |> Ash.Changeset.for_create(:new, %{subject: "Issue #{i}"})
    |> Support.AshApi.create!()

  if rem(i, 2) == 0 do
    ticket
    |> Ash.Changeset.for_update(:close)
    |> Support.AshApi.update!()
  end
end
```

Now kill `iex` and start it again and ensure the following Queries works (and find the data we stored earlier):

```elixir
iex -S mix

# use `read` to list all users
{:ok, users} = Support.AshApi.read(Support.User)
{:ok, tickets}= Support.AshApi.read(Support.Ticket)

# use 'get' to get one record when you know the id
ticket_last = List.last(tickets)
Support.AshApi.get(Support.Ticket, ticket_last.id)

# use Queries for more complex (nuanced lookups)
require Ash.Query

# Show the tickets where the subject contains "2"
Support.Ticket
|> Ash.Query.filter(contains(subject, "2"))
|> Support.AshApi.read!()

# Show the tickets that are closed and their subject does not contain "4"
Support.Ticket
|> Ash.Query.filter(status == :closed and not(contains(subject, "4")))
|> Support.AshApi.read!()
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
