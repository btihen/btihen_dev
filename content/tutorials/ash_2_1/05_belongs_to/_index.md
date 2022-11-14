---
# Title, summary, and page position.
linktitle: '05-BelongsTo'
summary: Ash Belongs To relationships
weight: 6
icon: book-reader
icon_pack: fas

# Page metadata.
title: '05-BelongsTo'
date: '2022-11-04T00:00:00Z'
type: book # Do not modify.
draft: false
---


In order to build relationships we will need more than one resource.  So let's build the support 'ticket' and exchange 'comments' resource.  We will continue using ETS as the data layer (for now).  I won't explain the `actions` and `attributes` since that has already been covered in the [Resources Article](/tutorials/ash_2_1/01_resources/).

Relationships declare the relationship between 'resources' we have:

* `belongs_to` - associated with ONE resource, if the remote resource is deleted, this resource will need to be deleted (first).  For example ONE ticket is associated with ONE user who created the ticket.  If the user who created the ticket is deleted then the `ticket` will need to be removed too.
* `has_many` - A user can create MULTIPLE ticket, again, if the remote resource is deleted, this remote resource will need to be deleted (first).
* `has_one` - not commonly used, but its the inverse of belongs_to

## Belongs To

So now we will show how `belongs_to` is defined in Ash.  Notice that in Ticket, we are referencing two different `User` resources, and this establishes two different relationships (with sensible relationship names)

```elixir
# lib/support/resources/ticket.ex
defmodule Support.Ticket do
  # This turns this module into a resource
  use Ash.Resource,
    data_layer: Ash.DataLayer.Ets

  actions do
    defaults [:create, :read, :update, :destroy]
  end

  attributes do
    uuid_primary_key :id

    attribute :status, :atom do
      constraints [one_of: [:new, :active, :closed]]
      default :new
      allow_nil? false
    end

    attribute :priority, :atom do
      constraints [one_of: [:low, :medium, :high]]
      default :low
      allow_nil? false
    end

    attribute :subject, :string do
      allow_nil? false
    end

    attribute :description, :string do
      allow_nil? false
    end
  end

  # ONE 'reporter' creates a ticket
  # ONE 'technician' helps the 'reporter' with the support ticket
  relationships do
    belongs_to :reporter, Support.User
    belongs_to :technician, Support.User
  end
end
```

```elixir
# lib/support/resources/comment.ex
defmodule Support.Comment do
  use Ash.Resource,
    data_layer: Ash.DataLayer.Ets

  actions do
    defaults [:create, :read, :update, :destroy]
  end

  attributes do
    uuid_primary_key :id

    attribute :message, :string do
      allow_nil? false
    end
  end

  # ONE 'author' (reporter or representative)) writes a comment
  # ONE 'ticket' is associated with the comment.
  relationships do
    belongs_to :author, Support.User
    belongs_to :ticket, Support.Ticket
  end
end
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
      :new_employee, %{first_name: "Nyima", middle_name: "Druk",
                       last_name: "Sönam", email: "nyima@example.com",
                       department_name: "Tech Support"}
    )
  |> Support.AshApi.create!()
)

ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(
      :create, %{subject: "No Power", description: "nothing happens", reporter: %{id: customer.id}}
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
ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(:create, %{subject: "No Power", description: "nothing happens"})
  |> Support.AshApi.create!()
)

# NOW WE SHOULD GET!
** (Ash.Error.Invalid) Input Invalid

* relationship reporter is required
```

### Relationship Changesets - Create 'Belongs To' Records

OK, we are protected from creating records without required relationship, now we need to learn to `manage_relationships`

**FIX**

To do this we will make a changeset with `manage_relationship` which looks like:
```elixir
customer = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_customer, %{first_name: "Ratna", last_name: "Sönam", email: "ratna@example.com"}
    )
  |> Support.AshApi.create!()
)

ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(:create, %{subject: "No Power", description: "nothing happens"})
  |> Ash.Changeset.manage_relationship(:reporter, %{id: customer.id}, type: :append_and_remove)
  |> Support.AshApi.create!()
)
```

### Custom Actions with relationships

To simplify relationships we can add them to custom actions:

```elixir
# lib/support/resources/ticket.ex
  # ...
  actions do
    defaults [:create, :read, :update, :destroy]

    create :new do
      accept [:subject, :description, :priority]

      argument :reporter_id, :uuid do
        allow_nil? false # This action requires reporter_id
      end
      # add the reporter
      change manage_relationship(:reporter_id, :reporter, type: :append_and_remove)
    end
  end
  # ...
```

Now we can do:
```elixir
customer = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_customer, %{first_name: "Ratna", last_name: "Sönam", email: "ratna@example.com"}
    )
  |> Support.AshApi.create!()
)

# using custom action - which requires a reporter and is simpler
ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(
      :new, %{subject: "No Power", description: "nothing happens", reporter_id: customer.id}
    )
  |> Support.AshApi.create!()
)
```

### Query Relationships

Notice that you need to pin (^) the customer.id and to get the associated 'repoter' you need to `load` the relationship(s) like in the following query:

```elixir


Support.Ticket
  |> Ash.Query.filter(priority == :low)
  |> Ash.Query.filter(reporter_id == ^customer.id)
  |> Ash.Query.load([:reporter])
  |> Support.AshApi.read!()
```

The `filter` function is very capable - to learn more visit:

* [Ash Queries](https://www.ash-hq.org/docs/module/ash/2.4.1/ash-query)
* [Writing an Ash Filter](https://www.ash-hq.org/docs/module/ash/2.4.1/ash-filter)


### Update 'belongs_to' Relationships

**TODO: fix**

To do this we will make a changeset with `manage_relationship` which looks like:
```elixir
iex -S mix phx.server

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
ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(
      :new, %{subject: "No Power", description: "nothing happens", reporter_id: customer.id}
    )
  |> Support.AshApi.create!()
)

ticket
  |> Ash.Changeset.for_update(:update, %{status: :active})
  |> Ash.Changeset.manage_relationship(:technician, %{id: technician.id}, type: :append_and_remove)
  |> Support.AshApi.update!()
```

### Validate Technician

We want to ensure that if the ticket is `open` or `closed` it has a technician who is responsible.

```elixir
# lib/support/resources/ticket.ex
  # ...
  validations do
    validate present(:technician_id),
             where: attribute_does_not_equal(:status, :new),
             message: "A technician must be assigned to a ticket that is not new"
  end
  # ...
```

**TEST**

```elixir
iex -S mix phx.server

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
ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(
      :new, %{subject: "No Power", description: "nothing happens", reporter_id: customer.id}
    )
  |> Support.AshApi.create!()
)

ticket = (
  ticket
  |> Ash.Changeset.for_update(:update, %{status: :active})
  |> Support.AshApi.update!()
)
# as expected (wanted)
** (Ash.Error.Invalid) Input Invalid

* Invalid value provided for technician_id: A technician must be assigned to a ticket that is not new.


# However This works
ticket = (
  ticket
  |> Ash.Changeset.for_update(:update, %{technician: %{id: technician.id}})
  |> Ash.Changeset.manage_relationship(:technician, %{id: technician.id}, type: :append_and_remove)
  |> Support.AshApi.update!()
)

ticket = (
  ticket
  |> Ash.Changeset.for_update(:update, %{technician_id: technician.id})
  |> Ash.Changeset.manage_relationship(:technician, %{id: technician.id}, type: :append_and_remove)
  |> Support.AshApi.update!()
)

ticket = (
  ticket
  |> Ash.Changeset.for_update(:update, %{status: :active})
  |> Support.AshApi.update!()
)
```

### When Managed Relationships Happen

Our current validation fails under one condition when we might expect it to work - when we update both the technician and the status.  Unfortunately this fails:

```elixir
# oddly this doesn't work (technician and status together) - hmm
ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(
      :new, %{subject: "No Power", description: "nothing happens", reporter_id: customer.id}
    )
  |> Support.AshApi.create!()
)
ticket = (
  ticket
  |> Ash.Changeset.for_update(:update, %{status: :active, technician_id: technician.id})
  |> Support.AshApi.update!()
)
** (Ash.Error.Invalid) Input Invalid

* Invalid value provided for technician_id: A technician must be assigned to a ticket that is not new.
```

This is because the relationship is being set early in the process a straight-forward fix is to change the `belongs_to` to be a writeable `attribute` - not just a 'managed relationship'.  We can do this with the following code:
```elixir
# lib/support/resources/ticket.ex
  # ...
  belongs_to :technician, Support.Technician do
    attribute_writable? true
  end
  # ...
```

Now if we adjust our code to assigning the `technician_id` (that we just enabled) - we can set the status and set the technician at the same time!
```elixir
ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(
      :new, %{subject: "No Power", description: "nothing happens", reporter_id: customer.id}
    )
  |> Support.AshApi.create!()
)
ticket = (
  ticket
  |> Ash.Changeset.for_update(:update, %{status: :active, technician_id: technician.id})
  |> Support.AshApi.update!()
)
```

**Solutions** from the Framework author - _Zach Daniel_:

I think you may be running into an issue around when managed relationships actually happen. Managing relationships is a bit special insofar as the logic to actually do so is handled by calling the action.
So when your validation runs, the technician_id is never going to be set.
You have a few options:

1. don't use managed relationships for the belongs_to relationship
2. run your validation in an after_action hook in a custom change
3. run your validations in the action, based on the arguments instead


**Solution #1:** looks like this (the solution described in detail above):
```elixir
belongs_to :technician, Technician do
  attribute_writable? true
end
```
This makes the belongs_to attribute public and writable. Now you can just say `%{..., technician_id: <id>}` when calling your actions, and you can validate its presence like any other attribute. This is the simplest and is probably what I'd suggest based on how you're using the relationship.

**Solution #2:** - run your validation in an after_action hook in a custom change
```elixir
def change(changeset, _,_) do
  changeset
  |> Ash.Changeset.after_action(fn _changeset, result ->
    if ... do
      {:error, Ash.Error.Invalid.exception(...)}
    else
       {:ok, result}
    end
  end)
end
```

**Solution #3** - is very similar to the custom action describe in the next section - but of course only works with the custom action.
```elixir
update :update do
  # you don't *have* to do the nested input if you don't want to,
  # you can do it like this if you only ever pass an id
  argument :technician_id, :uuid

  # a validation can be placed inside an individual action
  validate present(:technician_id), where: <condition>

  # if the argument name is different from the relationship, you just need to specify both here
  change manage_relationship(:technician_id, :technician, type: :append_and_remove)
end
```

### Update 'belongs_to' Custom Actions

Of course we can simplify this with a custom action:

```elixir
    update :activate do
      # No attributes should be accepted
      accept []
      # We accept a technician's id as input here
      argument :technician_id, :uuid do
        allow_nil? false # requires technician_id
      end
      # We use a change here to replace the related technician
      change manage_relationship(:technician_id, :technician, type: :append_and_remove)
      change set_attribute(:status, :active)
    end
```

**TEST**

```elixir

ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(
      :new, %{subject: "No Power", description: "nothing happens", reporter_id: customer.id}
    )
  |> Support.AshApi.create!()
)

# Assign tech and open in one step
ticket = (
  ticket
  |> Ash.Changeset.for_update(:activate, %{technician_id: technician.id})
  |> Support.AshApi.update!()
)
```
