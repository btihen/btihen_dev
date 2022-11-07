---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "GraphQL API with Pheonix"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ["Elixir", "GraphQL"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2020-07-10T11:59:53+02:00
lastmod: 2020-07-10T11:59:53+02:00
featured: false
draft: true

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
## Build a GraphQL API for Phoenix

Ideas taken from:
https://github.com/conradwt/zero-to-graphql-using-phoenix
https://itnext.io/graphql-with-elixir-phoenix-and-absinthe-6b0ffd260094
https://pragmaticstudio.com/tutorials/how-to-setup-graphql-in-a-phoenix-app
https://www.viget.com/articles/getting-started-with-graphql-phoenix-and-react/
https://timber.io/blog/a-gentle-introduction-to-graphql-with-elixir-and-phoenix/
https://schneider.dev/blog/elixir-phoenix-absinthe-graphql-react-apollo-absurdly-deep-dive/
https://medium.com/velotio-perspectives/creating-graphql-apis-using-elixir-phoenix-and-absinthe-486ff38f2549

https://dev.to/joseph_lozano/setting-up-a-new-phoenix-1-5-project-with-phoenix-liveview-309n

find the most recent phoenix version:
https://github.com/phoenixframework/phoenix/releases

Start your Phoenix app with:
    $ mix phx.server

You can also run your app inside IEx (Interactive Elixir) as:
    $ iex -S mix phx.server


```
mix archive.install hex phx_new 1.5.3
mix phx.new ideas --live
cd ideas
mix ecto.create
git init && git add -A && git commit -m "init"

mix phx.gen.live Account User users name:string email:string username:string
# mix phx.gen.html Account User users name:string email:string username:string

update routes:
lib/ideas_web/router.ex:

    live "/users", UserLive.Index, :index
    live "/users/new", UserLive.Index, :new
    live "/users/:id/edit", UserLive.Index, :edit

    live "/users/:id", UserLive.Show, :show
    live "/users/:id/show/edit", UserLive.Show, :edit

# context has no webstuff
mix phx.gen.live Blog Post posts user_id:references:users title:string body:text


lib/ideas_web/router.ex:

    live "/posts", PostLive.Index, :index
    live "/posts/new", PostLive.Index, :new
    live "/posts/:id/edit", PostLive.Index, :edit

    live "/posts/:id", PostLive.Show, :show
    live "/posts/:id/show/edit", PostLive.Show, :edit

Lets update these so there is a relationship:


mix phx.gen.context Account Friendship friendships user_id:references:users friend_id:references:users

lib/ideas/account/friendship.ex
defmodule Ideas.Account.Friendship do
  use Ecto.Schema
  import Ecto.Changeset
  alias Ideas.Account.User
  alias Ideas.Account.Friendship

  @required_fields [:user_id, :friend_id]

  schema "friendships" do
    belongs_to(:user, User)
    belongs_to(:friend, User)

    # field :user_id, :id
    # field :friend_id, :id

    timestamps()
  end

  @doc false
  # def changeset(%Friendship{} = friendship, attrs) do
  def changeset(friendship, attrs) do
    friendship
    |> cast(attrs, @required_fields)
    |> validate_required(@required_fields)
  end
end

also update: lib/ideas/account/user.ex
defmodule Ideas.Account.User do
  use Ecto.Schema
  import Ecto.Changeset
  alias Ideas.Blog.Post
  alias Ideas.Account.User
  alias Ideas.Account.Friendship

  schema "users" do
    field :email, :string
    field :name, :string
    field :username, :string

    has_many(:posts, Post)
    has_many(:friendships, Friendship)
    has_many(:friends, through: [:friendships, :friend])

    timestamps()
  end

  @doc false
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :email, :username])
    |> validate_required([:name, :email, :username])
  end
end


Now update the seeds:
# Script for populating the database. You can run it as:
#
#     mix run priv/repo/seeds.exs
#
# Inside the script, you can read and write to any of your
# repositories directly:
#
#     Ideas.Repo.insert!(%Ideas.SomeSchema{})
#
# We recommend using the bang functions (`insert!`, `update!`
# and so on) as they will fail if something goes wrong.
alias Ideas.Repo
alias Ideas.Blog.Post
alias Ideas.Account.User
alias Ideas.Account.Friendship

# reset the datastore
Repo.delete_all(User)

# insert people
me = Repo.insert!(%User{ name: "Bill Tihen", email: "btihen@gmail.com", username: "btihen" })
Repo.insert!(%Post{ user_id: me.id, title: "Elixir", body: "Very cool ideas" })
Repo.insert!(%Post{ user_id: me.id, title: "Phoenix", body: "live is fascinating" })

ct = Repo.insert!(%User{ name: "Conrad Taylor", email: "conradwt@gmail.com", username: "conradwt" })
Repo.insert!(%Post{ user_id: ct.id, title: "Phoenix-GraphQL", body: "very helpful stuff" })

dhh = Repo.insert!(%User{ name: "David Heinemeier Hansson", email: "dhh@37signals.com", username: "dhh" })
ezra = Repo.insert!(%User{ name: "Ezra Zygmuntowicz", email: "ezra@merbivore.com", username: "ezra" })
matz = Repo.insert!(%User{ name: "Yukihiro Matsumoto", email: "matz@heroku.com", username: "matz" })

ct
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: ct.id, friend_id: matz.id } )
|> Repo.insert

dhh
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: dhh.id, friend_id: ezra.id } )
|> Repo.insert

dhh
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: dhh.id, friend_id: matz.id } )
|> Repo.insert

ezra
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: ezra.id, friend_id: dhh.id } )
|> Repo.insert

ezra
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: ezra.id, friend_id: matz.id } )
|> Repo.insert

matz
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: matz.id, friend_id: ct.id } )
|> Repo.insert

matz
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: matz.id, friend_id: ezra.id } )
|> Repo.insert

matz
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: matz.id, friend_id: dhh.id } )
|> Repo.insert
