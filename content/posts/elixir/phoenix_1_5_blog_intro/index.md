---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.5 Blog Intro"
subtitle: "Learn Basic Relationships in Phoneix"
summary: "This article covers how to create a new app with contexts, relationships, preloading, etc.  The basics for most dynamic websites (excluding authentication). Comming later."
authors: ["btihen"]
tags: ["Elixir", "Relationships", "Templates", "Preloading", "has_many", "belongs_to", "dependent delete", "selection in form"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2020-07-04T13:06:29+02:00
lastmod: 2021-08-07T13:06:29+02:00
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
## Purpose

This article creates a basic web application backed by a database and creates a few relationships.  I'll use the mix generator commands to make this process quick and easy.  In step two we will add a graphql api.

## Topics Covered

* create a project
* create a resource
* dropdown list of a collection
* pre-load/display sub-reources
* create a has_many relationship
* create a belongs_to relationship
* delete has_many sub-resources when top resource is deleted

## Getting Started - create an app

find the most recent phoenix version:
https://github.com/phoenixframework/phoenix/releases

```bash
mix archive.install hex phx_new 1.5.3
mix phx.new feenix_intro
cd feenix_intro
mix ecto.create
```

test with: `mix phx.server` and go to `http://localhost:4000`

Ideally you see a the Phoenix Start Page.

Let's create a git snapshot
```bash
git init && git add -A && git commit -m "init"
```

## Create Contexts

**Context helps us create areas of code isolation and creates an API for other contexts to use**

In our case we will need a Blogs and Accounts (better would have been Authors) context

Blogs will have the posts and comments and Accounts will have the user and login credentials and user relationships (why not)?  To see the full documentation on Contexts see: https://hexdocs.pm/phoenix/contexts.html

We will generate two resources and Contexts (and add more later) - lets start with users who will post their blogs (users will be within the Accounts context and posts will be within the Blogs context):
```bash
mix phx.gen.html Accounts User users name:string email:string username:string:unique
mix phx.gen.html Blogs Post posts title:string body:text user_id:references:users
```

Notice we can generate unique fields with `:unique`

And we can generate relationships (foriegn keys) with `references`


Now that we have generated our code - we need to make a few updates:

First: we need to update our routes in the scope area to look like:
```elixir
# lib/ideas_web/router.ex
  scope "/", FeenixIntroWeb do
    pipe_through :browser

    get "/", PageController, :index
    resources "/users", UserController
    resources "/posts", PostController
  end
```

NOTE: the API's for our Contexts `Accounts` and `Blogs` is in `lib/feenix_intro/accounts.ex` and `lib/feenix_intro/blogs/post.ex` respectively - as we add more info into these contexts these files will get long!  **Ideally you will always interact with the Context API and not the Repo directly this will help create much more managable code.**

## Define the has_many relationship

Before we migrate we need to define the relationships:

so we update the users with a has_many relationship to posts
```elixir
# lib/feenix_intro/accounts/user.ex
defmodule FeenixIntro.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset
  alias FeenixIntro.Blogs.Post

  @required_fields [:name, :email, :username]

  schema "users" do
    has_many(:posts, Post)

    field :name, :string
    field :email, :string
    field :username, :string

    timestamps()
  end

  @doc false
  def changeset(user, attrs) do
    user
    |> cast(attrs, @required_fields)
    |> validate_required(@required_fields)
    |> unique_constraint(:username)
  end
end
```
If you skip the alias, then `has_many` needs to be written as: `has_many(:posts, FeenixIntro.Blogs.Post)`

## Define the belongs_to relationship

**IMPORTANT:** replace the `field :user_id, :id` with `belongs_to(:user, User)` -- you CAN'T have both!
```elixir
# lib/feenix_intro/blogs/post.ex
defmodule FeenixIntro.Blogs.Post do
  use Ecto.Schema
  import Ecto.Changeset
  alias FeenixIntro.Blogs.Post
  alias FeenixIntro.Accounts.User

  @required_fields [:user_id, :title, :body]

  schema "posts" do
    belongs_to(:user, User)

    # field :user_id, :id
    field :body, :string
    field :title, :string

    timestamps()
  end

  @doc false
  def changeset(post, attrs) do
    post
    |> cast(attrs, @required_fields)
    |> validate_required(@required_fields)
  end
end
```
NOTE: `@required_fields [:user_id, :title, :body]` isn't required, but as things change defining a constant that can be reused can be convient.

## Auto delete sub-resources

To be sure we don't have unreferenced blogs if a user gets deleted we need to change our Blog migration to:
```elixir
# priv/repo/migrations/20200704152318_create_posts.exs
defmodule FeenixIntro.Repo.Migrations.CreatePosts do
  use Ecto.Migration

  def change do
    create table(:posts) do
      add :title, :string
      add :body, :text
      # remove the default
      # add :user_id, references(:users, on_delete: :nothing)
      # add the following to auto delete posts if user is deleted!
      add :user_id, references(:users, on_delete: :delete_all), null: false

      timestamps()
    end

    create index(:posts, [:user_id])
  end
end
```

Now it should be safe to migrate using:
```bash
mix ecto.migrate
```

## Seed Data

Let's create seed data so that one we know how to do that and two have some data to test before we get all our views and forms working:

```elixir
# priv/repo/seeds.exs

# Script for populating the database. You can run it as:
#
#     mix run priv/repo/seeds.exs
#
# We recommend using the bang functions (`insert!`, `update!`
# and so on) as they will fail if something goes wrong.

alias FeenixIntro.Repo
alias FeenixIntro.Blogs.Post
alias FeenixIntro.Accounts.User

# reset the datastore
Repo.delete_all(User) # this should also delete all Posts

# insert people
me = Repo.insert!(%User{ name: "Bill", email: "bill@example.com", username: "bill" })
dog = Repo.insert!(%User{ name: "Nyima", email: "nyima@example.com", username: "nyima" })
Repo.insert!(%Post{ user_id: me.id, title: "Elixir", body: "Very cool ideas" })
Repo.insert!(%Post{ user_id: me.id, title: "Phoenix", body: "live is fascinating" })
Repo.insert!(%Post{ user_id: dog.id, title: "Walk", body: "oh cool" })
Repo.insert!(%Post{ user_id: dog.id, title: "Dinner", body: "YES!" })
```

now as the comments state run:
```bash
mix run priv/repo/seeds.exs
```

## Testing

run:
```bash
mix phx.server
# or if you prefer:
# iex -S mix phx.server
```
**Test USERS:**

Go to: `http://localhost:4000/users`

when we list users and create users - all is well

**TEST POSTS**

Go to: `http://localhost:4000/posts`

when we do the same withe posts - we get an error creating new posts and we don't see the author in index and show

* we can't create a post since we required the user_id and there is not field for that
* we can't list the author's name (just the author's ID) until we preload the author along with the post

## Fix Post creation with a dropdown list of resources

Normally, this would be done with session info to autoselect the authenticated author, but that is for another day.  In this case, we will demonstrate how to load and pass a collection and use that to populate a dropdown entry.

In the controller we must load users and add the user_id to the post form:
whe we look in the Accounts API we see: `list_users()`
```elixir
# lib/feenix_intro_web/controllers/post_controller.ex
  # ...
  # add the accounts context alias
  alias FeenixIntro.Accounts
  # ...
  def new(conn, _params) do
    changeset = Blogs.change_post(%Post{})
    # replace:
    # render(conn, "new.html", changeset: changeset)
    # with:
    # collection of users for post form
    users = Accounts.list_users()
    # include the collection of users to the new form
    render(conn, "new.html", changeset: changeset, users: users)
  end
  # ...
  def edit(conn, %{"id" => id}) do
    post = Blogs.get_post!(id)
    changeset = Blogs.change_post(post)
    # replace:
    render(conn, "edit.html", post: post, changeset: changeset)
    # with:
    users = Accounts.list_users()
    render(conn, "edit.html", post: post, changeset: changeset, users: users)
  end
# ...
```

Now we need to adapt the form to give us a choice of users:
```elixir
# lib/feenix_intro_web/templates/post/form.html.eex
<%= form_for @changeset, @action, fn f -> %>
  <%= if @changeset.action do %>
    <div class="alert alert-danger">
      <p>Oops, something went wrong! Please check the errors below.</p>
    </div>
  <% end %>

  <%= label f, "Author" %>
  <%= select f, :user_id, Enum.map(@users, &{&1.name, &1.id}) %>
  <%= error_tag f, :user %>
  # ...
```

Assuming you can create posts now, lets make another git snapshot:
```bash
git add .
git commit -m "users and posts resources can be created"
```

## Display the Author of Post (with Preloads)

lets display the Blog author - that's often interesting to others.
We can do this with preloading in our Blog context:
```elixir
# lib/feenix_intro/blogs.ex
  # change this line:
  # def list_posts, do: Repo.all(Post)
  def list_posts do
    Post
    |> Repo.all()
    |> Repo.preload(:user)
  end
```

and also our get_post
```elixir
# lib/feenix_intro/blogs.ex
  # change:
  # def get_post!(id), do: Repo.get!(Post, id)
  # into:
  def get_post!(id) do
    Post
    |> Repo.get!(id)
    |> Repo.preload(:user)
  end
```
now we can update our index and show page to display the author's name at the top of the page:
```elixir
# lib/feenix_intro_web/templates/post/show.html.eex
<h1>Show Post</h1>

<ul>

  <li>
    <strong>Author:</strong>
    <%= @post.user.name %>
  </li>
```
and in the index too:
```
# lib/feenix_intro_web/templates/post/index.html.eex
# ...elixir
<%= for post <- @posts do %>
    <tr>
      <td><%= post.user.name %></td>
      <td><%= post.title %></td>
      <td><%= post.body %></td>
# ...
```


Assuming authors and preload works properly, we can make another git snapshot:
```
git add .
git commit -m "authors names are displayed now with preloading"
```

## Source code

https://github.com/btihen/PhoenixIntro


## Helpful Resources used:

* https://elixircasts.io/phoenix-contexts
* https://github.com/conradwt/zero-to-graphql-using-phoenix
* https://medium.com/@damonvjanis/ecto-preloads-in-phoenix-contexts-167d11e5405e
* https://dev.to/joseph_lozano/setting-up-a-new-phoenix-1-5-project-with-phoenix-liveview-309n
* https://medium.com/velotio-perspectives/creating-graphql-apis-using-elixir-phoenix-and-absinthe-486ff38f2549
