---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.5 Blog with Friendship Relations"
subtitle: "Relationships - join_table, 'has_many_through' and self-referential relations"
summary: "In this article we cover join_tables, has_many_through and how simple it is to handle two ids on the same table with different names / meanings."
authors: ["btihen"]
tags: ["Elixir", "Relationships", "Join Table", "has_many_through", "self-references"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2020-07-15T18:32:15+02:00
lastmod: 2020-07-15T18:32:15+02:00
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
## Purpose

This article builds on the existing article: https://btihen.me/post_tech/phoenix_1_5_blog_w_comments/ and adds a join table so we can excersize a  has_many_through relationship.  Also this is a self-referential join-table (both fields reference the same root table - with different names).

## Join Table for a has_many_through Relationship

**Lets model friendships between authors**
```
mix phx.gen.context Accounts Friendship friendships user_id:references:user friend_id:references:user
```
Answer `Y` - we can split the file later

Update the Friend resorce:
```
lib/ideas/account/friendship.ex
defmodule Ideas.Account.Friendship do
  use Ecto.Schema
  import Ecto.Changeset
  # add aliases
  alias Ideas.Account.User
  alias Ideas.Account.Friendship

  # be sure well formed data
  @required_fields [:user_id, :friend_id]

  schema "friendships" do
    # add relationships
    belongs_to(:user, User)
    belongs_to(:friend, Friend)

    # remove belonds to fields
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
```


Add relationships to user resource
```
# lib/ideas/account/user.ex
# ...

  alias FeenixIntro.Accounts.Friendship
  # ...
  schema "users" do
    has_many(:posts, Post)
    has_many(:friendships, Friendship)
    has_many(:friends, through: [:friendships, :friend])

    # ...
  end
  # ...
end
```

No changes needed to the migration this time!
```
# priv/repo/migrations/20200705162608_create_friendships.exs
defmodule FeenixIntro.Repo.Migrations.CreateFriendships do
  use Ecto.Migration

  def change do
    create table(:friendships) do
      # no change when belongs to two models - won't migrate!
      add :user_id, references(:users, on_delete: :nothing)
      add :friend_id, references(:users, on_delete: :nothing)

      timestamps()
    end

    create index(:friendships, [:user_id])
    create index(:friendships, [:friend_id])
  end
end
```


## update the seeds by adding:
```
# ...
alias FeenixIntro.Accounts.Friendship

# ...

# insert relationships
cat = Repo.insert!(%User{ name: "Cat", email: "cat@example.com", username: "cat" })

me
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: me.id, friend_id: dog.id } )
|> Repo.insert

me
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: me.id, friend_id: cat.id } )
|> Repo.insert

dog
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: dog.id, friend_id: me.id } )
|> Repo.insert

dog
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: dog.id, friend_id: cat.id } )
|> Repo.insert

cat
|> Ecto.build_assoc(:friendships)
|> Friendship.changeset( %{ user_id: cat.id, friend_id: me.id } )
|> Repo.insert
```
test our new relationships with:
`mix run priv/repo/seeds.exs`



## list friends (has_many_though) on user show

## add friends

## has_one (covered in authentication)e (covered in authentication)
