---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Phoenix 1.7.11 with LiveView 0.20.12 - Auth Chat"
subtitle: "Article 2 - Exploring Authorization for Various Users"
# Summary for listings and search engines
summary: "Restricting access to pages - for various users"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "LiveView", "Authorization"]
categories: ["Code", "Elixir Language", "Phoenix Framework", "LiveView"]
date: 2024-03-09T01:01:53+02:00
lastmod: 2024-03-24T01:01:53+02:00
featured: true
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

In this article we start with the code from [Phoenix 1.7.11 with LiveView 0.20.9 - Authorization Intro](https://btihen.dev/posts/elixir/phoenix_1_7_11_liveview_0_20_9_authorize/).  The starting code can be found at: https://github.com/btihen-dev/phoenix_authorize

This code can be found at: https://github.com/btihen-dev/auth_buzz

## Article Series

* Article 1 (Admin Panel) - [Authorization Intro](https://btihen.dev/posts/elixir/phoenix_1_7_11_liveview_0_20_9_authorize/)
* Article 2 (Model Auth) - Authorization Usage

## Getting started

Before we get started let's update our packages:
```
mix deps.update --all
```

## Add Chat Topics

These will eventually only be available to change by "Admins" - but we won't start with this.

We will start with making topics that can only be created and removed by Admins
Then we will add messages that any user can add to a topic

Let's start by generating our new LiveViews and Models
```bash
mix phx.gen.live Topics Topic topics title:string --web Buzz

    scope "/buzz", AuthorizeWeb.Buzz, as: :buzz do
      pipe_through :browser
      ...

      live "/topics", TopicLive.Index, :index
      live "/topics/new", TopicLive.Index, :new
      live "/topics/:id/edit", TopicLive.Index, :edit

      live "/topics/:id", TopicLive.Show, :show
      live "/topics/:id/show/edit", TopicLive.Show, :edit
    end
```

Lets move all these new files into the namespace of Buzz:

```
mkdir lib/authorize/buzz
mkdir test/authorize/buzz
mkdir test/support/fixtures/buzz

mv lib/authorize/topic* lib/authorize/buzz/.
mv test/authorize/topic* test/authorize/buzz/.
mv test/support/fixtures/topic* test/support/fixtures/buzz/.
```
Lets update the Module names with the Buzz namespace too:
`Authorize.Topics` to become `Authorize.Buzz.Topics`

now lets migrate and run tests:
```
mix ecto.migrate
mix test
```

now go to: `http://localhost:4040/buzz/topics` and be sure you can create, delete topics, etc.


now we can add to the seeds file:
```elixir
# priv/repo/seeds.exs
alias Authorize.Buzz.Topics

topics = [
  %{title: "cats"},
  %{title: "dogs"}
]
Enum.map(topics, fn topic -> Topics.create_topic(topic) end)
```

cool let's snapshot this:
```
git add .
git commit -m "add chat topics"
```

## Manage Topics (Admins only)

let's move the live-topics to admin area
```
mv lib/authorize_web/live/buzz/topic_live lib/authorize_web/live/admin/.
mv test/authorize_web/live/buzz/topic* test/authorize_web/live/admin/.
```

Replace all `AuthorizeWeb.Buzz.TopicLive` with `AuthorizeWeb.Admin.TopicLive`

now replace all `~p"/buzz/topics` with `~p"/admin/topics`

now lets update the routes and put our new routes in the `Admin` the section - so:
```elixir
# lib/authorize_web/router.ex

  scope "/admin", AuthorizeWeb.Admin, as: :admin do
    pipe_through [:browser, :require_authenticated_user, :require_admin_user]

    live_session :admin_live,
      on_mount: [
        {AuthorizeWeb.Access.UserAuth, :ensure_authenticated},
        {AuthorizeWeb.Access.UserAuth, :ensure_admin}
      ] do
      live("/admin_roles", AdminRolesLive, :index)
    end
  end
```

becomes:
```elixir
# lib/authorize_web/router.ex

  scope "/admin", AuthorizeWeb.Admin, as: :admin do
    pipe_through [:browser, :require_authenticated_user, :require_admin_user]

    live_session :admin_live,
      on_mount: [
        {AuthorizeWeb.Access.UserAuth, :ensure_authenticated},
        {AuthorizeWeb.Access.UserAuth, :ensure_admin}
      ] do
      live("/admin_roles", AdminRolesLive, :index)

      # Add other live routes here that require the same authentication
      live "/topics", TopicLive.Index, :index
      live "/topics/new", TopicLive.Index, :new
      live "/topics/:id/edit", TopicLive.Index, :edit

      live "/topics/:id", TopicLive.Show, :show
      live "/topics/:id/show/edit", TopicLive.Show, :edit
    end
  end
```

and of course remove the routes:
```elixir
# lib/authorize_web/router.ex
  scope "/buzz", AuthorizeWeb.Buzz, as: :buzz do
    pipe_through :browser

    live "/topics", TopicLive.Index, :index
    live "/topics/new", TopicLive.Index, :new
    live "/topics/:id/edit", TopicLive.Index, :edit

    live "/topics/:id", TopicLive.Show, :show
    live "/topics/:id/show/edit", TopicLive.Show, :edit
  end
```

now only admins should be able to manage topics - you can test with an incognito browser session.

lets run our tests;
```
mix test
```

oops - we get lots of errors like:
```
  1) test Index updates topic in listing (AuthorizeWeb.Buzz.TopicLiveTest)
     test/authorize_web/live/admin/topic_live_test.exs:49
     ** (MatchError) no match of right hand side value: {:error, {:redirect, %{to: "/access/users/log_in", flash: %{"error" => "You must log in to access this page."}}}}
     code: {:ok, index_live, _html} = live(conn, ~p"/admin/topics")
     stacktrace:
       test/authorize_web/live/admin/topic_live_test.exs:50: (test)
```

Now we need to provide an **admin** user now in the tests:
`test/authorize_web/live/admin/topic_live_test.exs`

```bash
git commit -am "topics are managed by admins
```

## Topic Members (Admin only)

We need the data, lets look at the following hex [mix phx.gen page](https://hexdocs.pm/phoenix/Mix.Tasks.Phx.Gen.html#content)

but no live pages since we will integrate this using `phx.gen.schema` - this builds a migration, schema, but no live page nor context (we will write a simple context on our own) but we could have used `mix phx.gen.context TopicMember ...` and deleted unnecessary code and then had tests automatically generated.
.

```
mix phx.gen.schema TopicMember topic_members \
    topic_id:references:topics member_id:references:users
```

now lets move `lib/authorize/topic_member.ex` to the `buzz` area.

```
mkdir lib/authorize/buzz/topic_members
mv lib/authorize/topic_member.ex lib/authorize/buzz/topic_members/.
```

and rename all `Authorize.TopicMember` to `Authorize.Buzz.TopicMembers.TopicMember`

now lets update the migration to only allow a user to be a member of a given topic ONCE - we need to add the line:
`create unique_index(:topic_members, [:topic_id, :member_id])` - we will leave the on-delete in this case alone.

```elixir
# priv/repo/migrations/yyyymmddHHMMss_create_topic_members.exs
defmodule Authorize.Repo.Migrations.CreateTopicMembers do
  use Ecto.Migration

  def change do
    create table(:topic_members, primary_key: false) do
      add :id, :binary_id, primary_key: true
      add :topic_id, references(:topics, on_delete: :delete_all, type: :binary_id)
      add :member_id, references(:users, on_delete: :delete_all, type: :binary_id)

      timestamps(type: :utc_datetime)
    end

    create index(:topic_members, [:topic_id])
    create index(:topic_members, [:member_id])
    # add the following index to prevent duplicate entriess
    create unique_index(:topic_members, [:topic_id, :member_id]), name: :unique_topic_members
  end
end
```

now lets build our relationships - lets start with our new file `Authorize.Buzz.TopicMembers.TopicMember` and both the relationships and the unique constraint:
let's build associations by adding:
```elixir
    belongs_to :topic, Topic, foreign_key: :topic_id
    belongs_to :member, User, foreign_key: :member_id
```

and let's add a validation for our joint unique entry index by adding:
```elixir
    |> unique_constraint(:topic_id, name: :unique_topic_members) # validates unique
    |> unique_constraint(:member_id, name: :unique_topic_members) # validates unique
```

```elixir
# lib/authorize/buzz/topic_members/topic_member.ex
defmodule Authorize.Buzz.TopicMembers.TopicMember do
  use Ecto.Schema
  import Ecto.Changeset

  # simplify with aliases
  alias Authorize.Buzz.Topics.Topic
  alias Authorize.Core.Accounts.User

  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
  schema "topic_members" do
    # field :topic_id, :binary_id
    # field :member_id, :binary_id
    belongs_to :topic, Topic, foreign_key: :topic_id
    belongs_to :member, User, foreign_key: :member_id

    timestamps(type: :utc_datetime)
  end

  @doc false
  def changeset(topic_member, attrs) do
    topic_member
    |> cast(attrs, [:topic_id, :member_id])
    |> validate_required([:topic_id, :member_id])
    |> unique_constraint(:topic_id, name: :unique_topic_members) # validates unique
    |> unique_constraint(:member_id, name: :unique_topic_members) # validates unique
  end
end
```

Now we need to add relationships to `Topics` and `Users` by adding has-many relationships this would look like:
```elixir
    has_many :topic_members, TopicMember
    has_many :members, through: [:topic_members, :user]
```

so now `Topics` looks like:
```elixir
# lib/authorize/buzz/topics/topic.ex
defmodule Authorize.Buzz.Topics.Topic do
  use Ecto.Schema
  import Ecto.Changeset

  # new alias
  alias Authorize.Buzz.TopicMembers.TopicMember

  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
  schema "topics" do
    field :title, :string

    # new code
    has_many :topic_members, TopicMember
    has_many :members, through: [:topic_members, :user]

    timestamps(type: :utc_datetime)
  end

  @doc false
  def changeset(topic, attrs) do
    topic
    |> cast(attrs, [:title])
    |> validate_required([:title])
  end
end
```

and for `Users` we add a similar, has_many association using:
```elixir
    has_many :topic_members, TopicMember, foreign_key: :member_id
    has_many :topics, through: [:topic_members, :topic]
```

also if not yet done we should change:
```elixir
  def admin?(user), do: "admin" in user.roles || user.email == "batman@example.com"
```
to
```elixir
  def admin?(user), do: "admin" in user.roles
```
so it now looks like:
```elixir
# lib/authorize/core/accounts/user.ex
defmodule Authorize.Core.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  # new alias
  alias Authorize.Buzz.TopicMembers.TopicMember

  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
  schema "users" do
    field :email, :string
    field :password, :string, virtual: true, redact: true
    field :hashed_password, :string, redact: true
    field :confirmed_at, :naive_datetime
    field :roles, {:array, :string}, default: ["user"]

    # new association scode
    has_many :topic_members, TopicMember, foreign_key: :member_id
    has_many :topics, through: [:topic_members, :topic]

    timestamps(type: :utc_datetime)
  end
  # ...
end
```

Now lets stop and see if our tests still work.
```
mix ecto.migrate
mix test
```

