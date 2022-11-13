---
# Title, summary, and page position.
linktitle: '07-Aggregates'
summary: Ash Belongs To relationships
weight: 8
icon: book-reader
icon_pack: fas

# Page metadata.
title: '07-Aggregates'
date: '2022-11-04T00:00:00Z'
type: book # Do not modify.
draft: false
---

**IN PROGRESS**- Probably lots of errors

## Aggregates

Summarizing relationships

  [Aggregates](https://ash-hq.org/docs/guides/ash/2.4.2/topics/aggregates) in Ash allow for retrieving summary information over groups of related data. A simple example might be to show the “count of published posts for a user”. ...  In cases where aggregates don’t suffice, use Calculations, which are intended to be much more flexible.

There are two aspects to Aggregates:

* [Query.Aggregates](https://ash-hq.org/docs/module/ash/latest/ash-query/#module-docs) - on the fly query aggregates
* [Resource.Aggregates]() - predefined common aggregates associated with the resource

### Resource Aggregates

This is the primary Usage of Aggregates.

We will start with Resource.Aggregates - by extending the [Aggregate Guide](https://ash-hq.org/docs/guides/ash/2.4.2/topics/aggregates) & [Ash Resource DSL - aggregates](https://hexdocs.pm/ash/Ash.Resource.Dsl.html#module-aggregates)

The available Aggregate Types are:

* **count** - counts related items meeting the criteria
* **first** - gets the first related value matching the criteria. Must specify the field to get.
* **sum** - sums the related items meeting the criteria. Must specify the field to sum.
* **list** - lists the related values. Must specify the field to list.

We will focus on counting the number of tickets (& the types associated with each user)

```elixir
# lib/helpdesk/support/resources/user.ex
  # ...
  aggregates do
    count :all_reported_tickets, :reported_tickets
    count :open_reported_tickets, :reported_tickets do
      filter expr(status != :closed)
      # Also possible
      # filter expr(status == :active or status == :new)
    end
    count :active_reported_tickets, :reported_tickets do
      filter expr(status == :active)
    end
    count :closed_reported_tickets, :reported_tickets do
      filter expr(status == :closed)
    end

    count :all_assigned_tickets, :assigned_tickets
    count :active_assigned_tickets, :assigned_tickets do
      filter expr(status != :closed)
    end
    count :closed_assigned_tickets, :assigned_tickets do
      filter expr(status == :closed)
    end
    # ...
  end
```

Make a few Tickets and try our aggregates:

```elixir
iex -S mix phx.server

# Ash.Query is a macro, so it must be required
require Ash.Query

customers =
  for i <- 0..5 do
    customer = (
      Support.User
      |> Ash.Changeset.for_create(
          :new_customer, %{first_name: "Friendly#{i}", last_name: "Customer",
                           email: "friendly#{i}@customer.com"}
        )
      |> Support.AshApi.create!()
    )
  end

technicians =
  for i <- 0..3 do
    technician = (
      Support.User
      |> Ash.Changeset.for_create(
          :new_employee, %{first_name: "Clever#{i}", last_name: "Technician",
                           email: "clever#{i}@technician.com", department_name: "Support"}
        )
      |> Support.AshApi.create!()
    )
  end

tickets =
  for i <- 0..9 do
    technician = Enum.random(technicians)
    # customer = Enum.random(customers)
    customer = Enum.at((customers ++ customers), i)

    ticket = Support.Ticket
      |> Ash.Changeset.for_create(:new, %{subject: "No Power",
                                          description: "nothing happens",
                                          reporter_id: customer.id})
      |> Support.AshApi.create!()

    # active tickets
    if rem(i, 2) == 0 do
      ticket = (
        ticket
        |> Ash.Changeset.for_update(:activate, %{technician_id: technician.id})
        |> Support.AshApi.update!()
      )
    end
    # closed tickets
    if rem(i, 3) == 0 do
      ticket = (
        ticket
        |> Ash.Changeset.for_update(:update, %{technician: %{id: technician.id}})
        |> Ash.Changeset.manage_relationship(:technician, %{id: technician.id},
                                             type: :append_and_remove)
        |> Support.AshApi.update!()
      )
      ticket
        |> Ash.Changeset.for_update(:update, %{status: :closed})
        |> Support.AshApi.update!()
    end
    # always return a ticket for our list
    ticket
  end
```

Now that we have some tickets lets try our aggregators:

```elixir
# Ash.Query is a macro, so it must be required
require Ash.Query

# load a single aggregate into a single customer record
customer0 = (
  Support.User
  |> Ash.Query.filter(contains(first_name, "0") and account_role == :customer)
  |> Ash.Query.load(:all_reported_tickets)
  |> Support.AshApi.read!()
)

# load multiple aggregates into multiple customer records
Support.User
  |> Ash.Query.new()
  |> Ash.Query.filter(account_role == :customer)
  |> Ash.Query.load([:all_reported_tickets, :closed_reported_tickets])
  |> Support.AshApi.read!()

# Technicians
Support.User
  |> Ash.Query.new()
  |> Ash.Query.filter(account_role == :employee)
  |> Ash.Query.load(:all_assigned_tickets)
  |> Support.AshApi.read!()
```

So now we have 4 Tickets in the system and we expect:

### Query Aggregates

Not yet working

```elixir
# load aggregate into all customer records returned
Support.User
  |> Ash.Query.new()
  |> Ash.Query.filter(account_role == :customer)
  |> Ash.Query.load([:all_reported_tickets, :closed_reported_tickets])
  |> Ash.Query.aggregate(:my_all_tickets, :count, :reported_tickets, filter: [])
  |> Ash.Query.aggregate(:my_closed_tickets, :count, :reported_tickets, filter: [status: :closed])
  |> Support.AshApi.read!()


Support.User
  |> Ash.Query.new()
  |> Ash.Query.filter(account_role == :employee)
  |> Ash.Query.load(:all_assigned_tickets)
  |> Ash.Query.aggregate(:my_assigned_tickets, :count, :assigned_tickets, filter: [])
  |> Support.AshApi.read!()
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
Support.User
|> Ash.Query.aggregate(:all_reported_tickets, [:reported_tickets])
|> Support.AshApi.read!()

Support.User
|> Ash.Query.aggregate(:all_reported_tickets, [:reported_tickets],
                       filter: expr(status != :closed))
|> Support.AshApi.read!()
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
    has_many :assigned_tickets, Support.Ticket do
      destination_attribute :representative_id
    end
    has_many :reported_tickets, Support.Ticket do
      destination_attribute :reporter_id
    end
  end
```

To use aggregates, we can access the aggregates them within our queries (filters and sorts).  Here is an example using the closed tickets within a query:

```elixir
iex -S mix

require Ash.Query

users = Support.User
|> Ash.Query.filter(closed_assigned_tickets < 4) # users with less than 4 closed assigned tickets
|> Ash.Query.sort(closed_assigned_tickets: :desc)
|> Support.AshApi.read!()
# we get (as you see only the requested aggregate will be queried / calculated):
[
  #Support.User<
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
  #Support.User<
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
users = Support.AshApi.read!(Support.User)
  Support.User<
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
Support.AshApi.load!(users, :active_assigned_tickets)

# we can load multiple calculation
users = Support.AshApi.read!(Support.User)
Support.AshApi.load!(users, [:active_assigned_tickets, :closed_assigned_tickets])
```

# Resources

* <https://www.youtube.com/watch?v=2U3vQHXCF0s>
* <https://hexdocs.pm/ash/relationships.html#loading-related-data>
* <https://www.ash-hq.org/docs/guides/ash/2.4.1/tutorials/get-started.md>
