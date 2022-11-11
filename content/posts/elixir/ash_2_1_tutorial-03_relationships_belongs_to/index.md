---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Ash 2.1 Tutorial - 03 Relationships: BelongsTo"
subtitle: "Data Persistence"
# Summary for listings and search engines
summary: "Beginner's guide to the Ash framework - Adding Relationships into Ash Resources"
authors: ["btihen"]
tags: ["Elixir", "Ash", "Phoenix", "GraphQL", "JSON API"]
categories: ["Code", "Elixir Language", "Ash Framework"]
date: 2022-11-11T01:01:53+02:00
lastmod: 2022-11-11T01:01:53+02:00
featured: false
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

**Ash Framework 2.1 - A Beginner's Tutorial**

1. [Introduction](/posts_elixir/ash_2_1_tutorial-00_introduction/)
2. [Resources](/posts_elixir/ash_2_1_tutorial-01_resources/)
3. [Data-Layer](/posts_elixir/ash_2_1_tutorial-02_data_layer/)
4. [Relationships](/posts_elixir/ash_2_1_tutorial-03_relationships/)
5. Authentication
6. Authorization
7. Architecture
8. Engines & Flow
9. LiveView
10. GraphQL
11. JSON API

This article builds on the the Ash Article [Ash 2.1 Tutorial - 02 Data Layer](/posts_elixir/ash_2_1_tutorial-02_data_layer/)

-----------

In order to build relationships we will need more than one resource.  So lets quickly build the 'ticket' resource - also using ETS as the data layer (for now).  I won't explain the `actions` and `attributes` since that has already been covered in the [Resources Article](/posts_elixir/ash_2_1_tutorial-01_resources/).

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
      constraints [one_of: [:new, :open, :closed]]
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
      :new_employee, %{first_name: "Nyima", last_name: "Sönam", email: "nyima@example.com",
                       department_name: "Tech Support"}
    )
  |> Support.AshApi.create!()
)

# Customer reports a ticket - reporter_id remains blank! :(
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
  |> Ash.Changeset.for_update(:update, %{status: :open})
  |> Ash.Changeset.manage_relationship(:technician, %{id: technician.id}, type: :append_and_remove)
  |> Support.AshApi.update!()
```

### Update 'belongs_to' Custom Actions

Of course we can simplify with a custom action:

```elixir
# lib/support/resources/ticket.ex

update :assign do
  # No attributes should be accepted
  accept []

  # We accept a representative's id as input here
  argument :technician_id, :uuid do
    # This action requires representative_id
    allow_nil? false
  end

  # We use a change here to replace the related Representative
  # If there is a different representative for this Ticket, it will be changed to the new one
  # The Representative itself is not modified in any way
  change manage_relationship(:technician_id, :technician, type: :append_and_remove)
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
ticket = (
  Support.Ticket
  |> Ash.Changeset.for_create(
      :new, %{subject: "No Power", description: "nothing happens", reporter_id: customer.id}
    )
  |> Support.AshApi.create!()
)

ticket
|> Ash.Changeset.for_update(:assign, %{technician_id: technician.id})
|> Support.AshApi.update!()
```

## Aggregates

Summarizing relationships

# Resources

* <https://www.youtube.com/watch?v=2U3vQHXCF0s>
* <https://hexdocs.pm/ash/relationships.html#loading-related-data>
* <https://www.ash-hq.org/docs/guides/ash/2.4.1/tutorials/get-started.md>
