---
# Title, summary, and page position.
linktitle: '06-HasMany'
summary: Relationships with one to many
weight: 7
icon: book-reader
icon_pack: fas

# Page metadata.
title: '06-HasMany'
date: '2022-11-02T00:00:00Z'
type: book # Do not modify.
draft: false
---

**IN PROGRESS**- Probably lots of errors

In order to build relationships we will need more than one resource.  So let's build the support 'ticket' and exchange 'comments' resource.  We will continue using ETS as the data layer (for now).  I won't explain the `actions` and `attributes` since that has already been covered in the [Resources Article](/tutorials/ash_2_1/01_resources/).

Relationships declare the relationship between 'resources' we have:

* `belongs_to` - associated with ONE resource, if the remote resource is deleted, this resource will need to be deleted (first).  For example ONE ticket is associated with ONE user who created the ticket.  If the user who created the ticket is deleted then the `ticket` will need to be removed too.
* `has_many` - A user can create MULTIPLE ticket, again, if the remote resource is deleted, this remote resource will need to be deleted (first).
* `has_one` - not commonly used, but its the inverse of belongs_to


## Has Many

Now let's see how we define our `has_many` relationships within Ash. The 'User' may have has `authored` MANY 'comments'.  A 'User' (customer or employee) may have `created` MANY 'tickets'. A 'User' (employees) may have been `assigned` MANY 'Tickets'.  Here we see we need to use the

```elixir
# lib/support/resources/user.ex
  # ...
  relationships do
    has_many :authored_comments, Support.Comment do
      destination_attribute :author_id
    end

    # with overloaded references we need to declare the remote key
    has_many :created_tickets, Support.Ticket do
      destination_attribute :reporter_id
    end
    has_many :assigned_tickets, Support.Ticket do
      destination_attribute :technician_id
    end
  end
  # ...
```

We also want to be able to find the possible MANY comments associated with a ticket.

```elixir
# lib/support/resources/ticket.ex
  # ...
  relationships do
    has_many :comments, Support.Comment do
      destination_attribute :ticket_id
    end

    belongs_to :reporter, Support.User
    belongs_to :technician, Support.User
  end
  # ...
```

Be sure to update the registry!

```elixir
# lib/support/registry.ex
defmodule Support.Registry do
  use Ash.Registry,
    extensions: [
      Ash.Registry.ResourceValidations
    ]

  entries do
    entry Support.Comment
    entry Support.Ticket
    entry Support.User
  end
end
```

**TEST**

```elixir
iex -S mix phx.server

# Create a Customer and a technician
customer = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_customer, %{first_name: "Ratna", last_name: "Sönam", email: "ratna@example.com"}
    )
  |> Support.AshApi.create!()
)
technician = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_employee, %{first_name: "Nyima", last_name: "Sönam", email: "nyima@example.com",
                       department_name: "Tech Support"}
    )
  |> Support.AshApi.create!()
)

# Customer reports a ticket - reporter_id remains blank! :(
ticket1 = (
  Support.Ticket
  |> Ash.Changeset.for_create(
      :create, %{subject: "No Power", description: "nothing happens", reporter_id: customer.id}
    )
  |> Support.AshApi.create!()
)
```

NOTICE:

* we added a reporter_id we created a ticket without a reporter - we don't want that, so we will add a validation.
* we need a `relationship` changeset to add relationships - a normal changeset only updates `attributes`

To require a relationship we can rewrite our relationships to:
```elixir
# lib/support/resources/ticket.ex
  # ...
  relationships do
    has_many :comments, Support.Comment do
      destination_attribute :ticket_id
    end

    belongs_to :reporter, Support.User
    belongs_to :technician, Support.User
  end
  # ...
```
[belongs_to Docs](<https://ash-hq.org/docs/dsl/ash/latest/resource/relationships/belongs_to#allow_nil>)

**TEST**

```elixir
iex -S mix phx.server
# or
recompile()

# Create a Customer and a technician
customer = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_customer, %{first_name: "Ratna", last_name: "Sönam", email: "ratna@example.com"}
    )
  |> Support.AshApi.create!()
)
technician = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_employee, %{first_name: "Nyima", last_name: "Sönam", email: "nyima@example.com",
                       department_name: "Tech Support"}
    )
  |> Support.AshApi.create!()
)

# Customer reports a ticket - reporter_id remains blank! :(
ticket1 = (
  Support.Ticket
  |> Ash.Changeset.for_create(
      :create, %{subject: "No Power", description: "nothing happens", reporter_id: customer.id}
    )
  |> Support.AshApi.create!()
)

# NOW WE SHOULD GET!
** (Ash.Error.Invalid) Input Invalid

* relationship reporter is required
```



## Has Many

```elixir
comment1 = (
  Support.Comment
  |> Ash.Changeset.for_create(
      :create, %{message: "I plugged it in", author_id: customer.id, ticket_id: ticket1.id}
    )
  |> Support.AshApi.update!()
)
```

## many_to_many ?

### Custom Actions using Relationships

Now we need to create additional 'actions' for ticket to manage the relationships:

```elixir
# lib/support/resources/ticket.ex
  actions do
    # Add a set of simple actions. You'll customize these later.
    defaults [:create, :read, :update, :destroy]

    create :new do
      accept [:subject]

      argument :reporter_id, :uuid do
        allow_nil? false # This action requires reporter_id
      end

      change manage_relationship(:reporter_id, :reporter, type: :append_and_remove)
    end

    update :assign do
      # No attributes should be accepted
      accept []
      # We accept a representative's id as input here
      argument :representative_id, :uuid do
        # This action requires representative_id
        allow_nil? false
      end
      # We use a change here to replace the related representative
      change manage_relationship(:representative_id, :representative, type: :append_and_remove)
    end

    update :start do
      # No attributes should be accepted
      accept []
      # We accept a representative's id as input here
      argument :representative_id, :uuid do
        # This action requires representative_id
        allow_nil? false
      end
      # We use a change here to replace the related representative
      change manage_relationship(:representative_id, :representative, type: :append_and_remove)
      change set_attribute(:status, :open)
    end

    update :close do
      # We don't want to accept any input here
      accept []
      change set_attribute(:status, :closed)
    end
  # ...
```

We can learn more about managing ash relationships at: <https://www.ash-hq.org/docs/module/ash/2.4.1/ash-resource-change-builtins#function-manage_relationship-3>

Testing Relationships:

```elixir
iex -S mix
#
recompile()

# Create a reporter
reporter = (
  Support.User
  |> Ash.Changeset.for_create(:create, %{name: "Nyima Dog"})
  |> Support.create!()
)

# Open a ticket
ticket1 = (
  Support.Ticket
  |> Ash.Changeset.for_create(:new, %{subject: "I can't find my hand!", reporter_id: reporter.id})
  |> Support.create!()
)

# Create a representative
representative_joe = (
  Support.User
  |> Ash.Changeset.for_create(:create, %{name: "Joe"})
  |> Support.create!()
)

representative_jose = (
  Support.User
  |> Ash.Changeset.for_create(:create, %{name: "Jose"})
  |> Support.create!()
)

# Assign that representative
ticket2 = (
  ticket
  |> Ash.Changeset.for_update(:assign, %{representative_id: representative_joe.id})
  |> Support.update!()
)

# Start working on the Ticket
ticket = (
  ticket
  |> Ash.Changeset.for_update(:start, %{representative_id: representative_jose.id})
  |> Support.update!()
)

# close the ticket
ticket = (
  ticket
    |> Ash.Changeset.for_update(:close)
    |> Support.update!()
)
```

I've been curious about the Elixir Ash Framework and had some time.  It looks like it helps create an application framework and has many pre-built common solutions.  Authorization, Queries, Application Structure, etc.

As usual, I struggle with API documentation, and I love tutorials.  So I followed the instructions at:
<https://www.ash-hq.org/docs/guides/ash/2.4.1/tutorials/get-started.md#module-docs>
and integrated it with the slightly outdated Ash 1.x [slide notes](https://speakerdeck.com/zachsdaniel1/introduction-to-the-ash-framework-elixir-conf-2020?slide=16) from a 2020 ElixirConf talk by Zach Daniel called
[Introduction to the Ash Framework](https://www.youtube.com/watch?v=2U3vQHXCF0s)

Here is what I had to do (learn and adjust) to get up and running.

----------

### Testing Ash (Tickets)

Let's see if what we built actually works.

We will create a 'ticket' resource and then query for it.

To do this we will need to:

1. build a change-set for the create action (for the Ticket resource)
2. give it the `create!` instruction

To do this we will test within iex:

```elixir
iex -S mix

Support.Ticket
|> Ash.Changeset.for_create(:create)
|> Support.create!()
```

Which hopefully returns something like:

```bash
#Support.Ticket<
  __meta__: #Ecto.Schema.Metadata<:built, "">,
  id: "bcc9729b-7fa2-4e7c-af45-3293be3394ee",
  subject: nil,
  aggregates: %{},
  calculations: %{},
  __order__: nil,
  ...
>
```

Notice that the subject is nil (we aren't yet using validations) and our primary key is a uuid.

### Attribute Constraints

Let's say we don't want to allow blank Subjects and we want to require a restricted list of statuses.

To prevent blanks in a field we can change the attribute subject to look like:

```elixir
# lib/support/resources/ticket.ex
# ...
  attributes do
    attribute :subject, :string do
      allow_nil? false
    end
    # ...
  end
# ...
```

Now lets add a set of restricted statuses (new, open, closed):

```elixir
# lib/support/resources/ticket.ex
# ...
  attributes do
    attribute :status, :atom do
       constraints [one_of: [:new, :open, :closed]]
       default :new
       allow_nil? false
     end
    # ...
  end
# ...
```

Now the `ticket` file should look like:

```elixir
# lib/support/resources/ticket.ex
defmodule Support.Ticket do
  # This turns this module into a resource
  use Ash.Resource

  actions do
    # A set of simple actions.
    defaults [:create, :read, :update, :destroy]
  end

  # Attributes are the simple pieces of data that exist on your resource
  attributes do
    # Add an autogenerated UUID primary key called `:id`.
    uuid_primary_key :id

    # Add a string type attribute called `:subject`
    attribute :subject, :string do
      allow_nil? false # Don't allow `nil` values
    end

    # status is either `open` or `closed`. We can add more statuses later
    attribute :status, :atom do
      # restricted list of statuses
      constraints [one_of: [:new, :open, :closed]]
      default :new # default value when none is given
      allow_nil? false # Don't allow `nil` values
    end
  end
end
```

### Custom Action

We want to create a custom `new` ticket action - that will only accept one attribute `subject` - which is also now required (per default all attributes will be accepted).

```elixir
# lib/support/resources/ticket.ex
# ...
  actions do
    # ...
    create :new do
      # By default all attributes are accepted by an action
      accept [:subject] # This action should only accept the subject
    end
    # ...
  end
# ...
```

We also want a `close` action:

```elixir
# lib/support/resources/ticket.ex
# ...
  actions do
    # ...
    update :close do
      accept [] # accept no input here
      change set_attribute(:status, :closed)
    end
    # ...
  end
# ...
```

Now the Ticket file should look like:

```elixir
# lib/support/resources/ticket.ex
defmodule Support.Ticket do
  # This turns this module into a resource
  use Ash.Resource

  actions do
    # Add a set of simple actions. You'll customize these later.
    defaults [:create, :read, :update, :destroy]

    create :new do
      accept [:subject] # only accept the subject - no other fields
    end

    update :close do
      accept [] # don't accept attributes
      change set_attribute(:status, :closed) # build the change set
    end
  end

    # Attributes are the simple pieces of data that exist on your resource
  attributes do
    # Add an autogenerated UUID primary key called `:id`.
    uuid_primary_key :id

    # Add a string type attribute called `:subject`
    attribute :subject, :string do
      allow_nil? false # Don't allow `nil` values
    end

    # status is either `open` or `closed`. We can add more statuses later
    attribute :status, :atom do
      # restricted list of statuses
      constraints [one_of: [:new, :open, :closed]]
      default :new # default value when none is given
      allow_nil? false # Don't allow `nil` values
    end
  end
end
```

To test this change we need to adjust our change set - if we don't we should get an error:

```elixir
iex -S mix
# just in case iex is already open
recompile()

Support.Ticket
|> Ash.Changeset.for_create(:new)
|> Support.create!()

# we should get the error:
** (Ash.Error.Invalid) Input Invalid
* attribute subject is required
# ...
```

To properly create a Ticket now -- the following should work:

```elixir
iex -S mix
# just in case iex is already open
recompile()

ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(:new, %{subject: "My mouse won't click!"})
  |> Support.create!()
)
```

we should get something like (notice we have a subject and it's status is `pending`):

```elixir
#Support.Ticket<
  __meta__: #Ecto.Schema.Metadata<:loaded, "tickets">,
  id: "b792b4f1-2167-4aa8-b654-4aef4938ba9a",
  subject: "My mouse won't click!",
  status: :new,
  # ...
```

Now let's test the `close` action on our new `ticket`:

```elixir
ticket
|> Ash.Changeset.for_update(:close)
|> Support.update!()
```

Now our ticket should look like:

```elixir
#Support.Ticket<
  __meta__: #Ecto.Schema.Metadata<:loaded, "tickets">,
  id: "ccd5af37-7cad-40c5-badd-71d5c67d50a5",
  subject: "My mouse won't click!",
  status: :closed,
  # ...
```

Since we have no storage at the moment we can't query our records.

Try:

```elixir
Support.AshApi.read!(Support.Ticket)
```

You should get an error - that says: 'there is no data to be read for that resource'

Let's enable queries - we need to configure a data layer for these resources

## Ash Queries

### Simple Data Layer

We can add a default `Simple` data layer with the macro: `require Ash.Query`, so if we create some tickets using the script:

```elixir
# Ash.Query is a macro, so it must be required
require Ash.Query

tickets =
  for i <- 0..5 do
    ticket =
      Support.Ticket
      |> Ash.Changeset.for_create(:open, %{subject: "Issue #{i}"})
      |> Support.create!()

    if rem(i, 2) == 0 do
      ticket
      |> Ash.Changeset.for_update(:close)
      |> Support.update!()
    else
      ticket
    end
  end
```

Now we should be able to query and filter our tickets:

```elixir
# Show the tickets where the subject contains "2"
Support.Ticket
|> Ash.Query.filter(contains(subject, "2"))
|> Ash.DataLayer.Simple.set_data(tickets)
|> Support.AshApi.read!()

# Show the tickets that are closed and their subject does not contain "4"
Support.Ticket
|> Ash.Query.filter(status == :closed and not(contains(subject, "4")))
|> Ash.DataLayer.Simple.set_data(tickets)
|> Support.AshApi.read!()
```

Notice the power of the `filter` command, try adjusting.

To learn more visit:

* [Ash Queries](https://www.ash-hq.org/docs/module/ash/2.4.1/ash-query)
* [Writing an Ash Filter](https://www.ash-hq.org/docs/module/ash/2.4.1/ash-filter)

## Ash Persistence (ETS)

ETS is an in-memory (OTP based) way to persist data (we will work with PostgreSQL later).
Once we have persisted data we can explore relationships.

To add ETS to the Data Layer we need to change the line `use Ash.Resource` to:

```elixir
# lib/support/resources/ticket.ex
  # ...
  use Ash.Resource,
    data_layer: Ash.DataLayer.Ets
  # ...
```

Now the file should look like:

```elixir
# lib/support/resources/ticket.ex
defmodule Support.Ticket do
  # This turns this module into a resource
  use Ash.Resource,
    data_layer: Ash.DataLayer.Ets

  actions do
    # Add a set of simple actions. You'll customize these later.
    defaults [:create, :read, :update, :destroy]

    create :new do
      accept [:subject] # only accept the subject - no other fields
    end

    update :close do
      accept [] # don't accept attributes
      change set_attribute(:status, :closed) # build the change set
    end
  end

    # Attributes are the simple pieces of data that exist on your resource
  attributes do
    # Add an autogenerated UUID primary key called `:id`.
    uuid_primary_key :id

    # Add a string type attribute called `:subject`
    attribute :subject, :string do
      allow_nil? false # Don't allow `nil` values
    end

    # status is either `open` or `closed`. We can add more statuses later
    attribute :status, :atom do
      # restricted list of statuses
      constraints [one_of: [:new, :open, :closed]]
      default :new # default value when none is given
      allow_nil? false # Don't allow `nil` values
    end
  end
end
```

### Ash Queries - Persistent Data Layer

```elixir
iex -S mix
# and or
recompile()

# Actions (create & persist our records)
for i <- 0..5 do
  ticket =
    Support.Ticket
    |> Ash.Changeset.for_create(:new, %{subject: "Issue #{i}"})
    |> Support.create!()

  if rem(i, 2) == 0 do
    ticket
    |> Ash.Changeset.for_update(:close)
    |> Support.update!()
  end
end

# QUERY Data
# enable Ash Query (notice we no longer need to include the data layer in the query!)
require Ash.Query

# use `read` to list all users
{:ok, users} = Support.read(Support.User)
{:ok, tickets}= Support.read(Support.Ticket)

# use 'get' to get one record when you know the id
ticket_last = List.last(tickets)
Support.get(Support.Ticket, ticket_last.id)


# Show the tickets where the subject contains "2"
Support.Ticket
|> Ash.Query.filter(contains(subject, "2"))
|> Support.AshApi.read!()

# Show the tickets that are closed and their subject does not contain "4"
Support.Ticket
|> Ash.Query.filter(status == :closed and not(contains(subject, "4")))
|> Support.AshApi.read!()
```

## Ash Relationships

We will now create a `User` that can create or be assigned a ticket (using ETS as the data layer).

```elixir
# lib/support/resources/user.ex
defmodule Support.User do
  # This turns this module into a resource
  use Ash.Resource,
    data_layer: Ash.DataLayer.Ets

  actions do
    defaults [:create, :read, :update, :destroy]
  end

  attributes do
    uuid_primary_key :id
    attribute :name, :string
  end

  relationships do
    has_many :assigned_tickets, Support.Ticket do
      destination_attribute :representative_id
    end
    has_many :reported_tickets, Support.Ticket do
      destination_attribute :reporter_id
    end
  end
end
```

The `has_many` means that the destination attribute is not unique, meaning many related records could exist.

We also need to register our new `user` resource by adding:

```elixir
# lib/support/registry.ex
  entries do
    # ...
    entry Support.User
  end
```

So now our registry should look like:

```elixir
# lib/support/registry.ex
defmodule Support.Registry do
  use Ash.Registry,
    extensions: [
      Ash.Registry.ResourceValidations
    ]

  entries do
    entry Support.Ticket
    entry Support.User
  end
end
```

Now we need to add the relationship to Tickets too:

```elixir
# lib/support/resources/ticket.ex
  # ...
  relationships do
    belongs_to :reporter, Support.User
    belongs_to :representative, Support.User
  end
  # ...
```

We use `belong_to` meaning that the destination attribute is unique, meaning only one related record could exist.

Now we need to create additional 'actions' for ticket to manage the relationships:

```elixir
# lib/support/resources/ticket.ex
  actions do
    # Add a set of simple actions. You'll customize these later.
    defaults [:create, :read, :update, :destroy]

    create :new do
      accept [:subject]

      argument :reporter_id, :uuid do
        allow_nil? false # This action requires reporter_id
      end

      change manage_relationship(:reporter_id, :reporter, type: :append_and_remove)
    end

    update :assign do
      # No attributes should be accepted
      accept []
      # We accept a representative's id as input here
      argument :representative_id, :uuid do
        # This action requires representative_id
        allow_nil? false
      end
      # We use a change here to replace the related representative
      change manage_relationship(:representative_id, :representative, type: :append_and_remove)
    end

    update :start do
      # No attributes should be accepted
      accept []
      # We accept a representative's id as input here
      argument :representative_id, :uuid do
        # This action requires representative_id
        allow_nil? false
      end
      # We use a change here to replace the related representative
      change manage_relationship(:representative_id, :representative, type: :append_and_remove)
      change set_attribute(:status, :open)
    end

    update :close do
      # We don't want to accept any input here
      accept []
      change set_attribute(:status, :closed)
    end
  # ...
```

We can learn more about managing ash relationships at: <https://www.ash-hq.org/docs/module/ash/2.4.1/ash-resource-change-builtins#function-manage_relationship-3>

Testing Relationships:

```elixir
iex -S mix
#
recompile()

# Create a reporter
reporter = (
  Support.User
  |> Ash.Changeset.for_create(:create, %{name: "Nyima Dog"})
  |> Support.create!()
)

# Open a ticket
ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(:new, %{subject: "I can't find my hand!", reporter_id: reporter.id})
  |> Support.create!()
)

# Create a representative
representative_joe = (
  Support.User
  |> Ash.Changeset.for_create(:create, %{name: "Joe"})
  |> Support.create!()
)

representative_jose = (
  Support.User
  |> Ash.Changeset.for_create(:create, %{name: "Jose"})
  |> Support.create!()
)

# Assign that representative
ticket = (
  ticket
  |> Ash.Changeset.for_update(:assign, %{representative_id: representative_joe.id})
  |> Support.update!()
)

# Start working on the Ticket
ticket = (
  ticket
  |> Ash.Changeset.for_update(:start, %{representative_id: representative_jose.id})
  |> Support.update!()
)

# close the ticket
ticket = (
  ticket
    |> Ash.Changeset.for_update(:close)
    |> Support.update!()
)
```

# Resources

* <https://www.youtube.com/watch?v=2U3vQHXCF0s>
* <https://hexdocs.pm/ash/relationships.html#loading-related-data>
* <https://www.ash-hq.org/docs/guides/ash/2.4.1/tutorials/get-started.md>
