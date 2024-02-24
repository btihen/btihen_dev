---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Phoenix 1.7.11 with LiveView 0.20.9 - Authorization Demo"
subtitle: "Exploring simple page restrictions"
# Summary for listings and search engines
summary: "Restricting access to pages - for both static and live pages"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "LiveView", "Authorization"]
categories: ["Code", "Elixir Language", "Phoenix Framework", "LiveView"]
date: 2024-02-24T01:01:53+02:00
lastmod: 2024-02-24T01:01:53+02:00
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

In this article I will explore a simple authorization technique, restricting access to pages - for both static and live pages.

## Getting started

check / install newest erlang & elixir
```bash
# ERLANG First
# see available versions
asdf list all erlang
# see installed versions
asdf list erlang
# asdf install desired version
asdf install erlang 26.2.2
asdf local erlang 26.2.2

# Elixir (OTP must match erlang version)
asdf list all elixir
asdf list elixir
asdf install elixir 1.16.1-otp-26
#                              ^^ match erlang version
asdf local elixir 1.16.1-otp-26
```

ensure postgresql (or sqlite3)
```
https://www.sqlite.org/download.html
https://postgresapp.com/de/downloads.html
```

create project
```bash
mix archive.install hex phx_new
mix phx.new authorize
cd authorize
git init
git add .
git commit -m "initial commit"
```

you can find the repo at: https://github/btihen_code/phoenix_authorize

## Binary IDs

if you prefer UUID keys to serial IDs then you can easily do that with the following changes.  _This can be very helpful if you need to sync with external frontend or other external apps_

update `config/config.ex` from:
```elixir
# config/config.ex
config :authorize,
  ecto_repos: [Authorize.Repo],
  generators: [timestamp_type: :utc_datetime]
```

to:

```elixir
# config/config.ex
config :authorize,
  ecto_repos: [Authorize.Repo],
  generators: [timestamp_type: :utc_datetime, binary_id: true]
```

now lets create the database:
```bash
mix ecto.create
iex -S mix phx.server
```

## using phx.gen.auth

```bash
mix phx.gen.auth Accounts User users --web Accounts
mix deps.get
mix ecto.migrate
# start phoenix and create a user
iex -S mix phx.server
```

Now you should be logged in. Let's see if the binary ID worked (using the iex cli)

```elixir
import Ecto.Query
alias Authorize.Repo
alias Authorize.Accounts
alias Authorize.Accounts.User

# from Context file
Accounts.get_user_by_email("nyima@example.com")

# one user
Repo.get_by(User, email: email)

# all users
Repo.all(User)
```

we can see we have a uuid for an id and it works cool.

```bash
git add .
git commit -m "added users and authorization"
```

### create a seed file


### organize user into core data

I like to organize the code into areas that is associated with usage - so, in this case, I will move users into 'core' - this is for code & data shared by all aspects of the application:

```bash
# create the core lib folder
mkdir lib/authorize/core/
# create the core test folder
mkdir test/authorize/core
mkdir test/support/fixtures/core

# move your lib code into the new area
mv lib/authorize/account* lib/authorize/core/.
# move the test and support code into 'core'
mv test/authorize/accounts* test/authorize/core/.
mv test/support/fixtures/accounts* test/support/fixtures/core/.
```
replace every `Authorize.Accounts` with `Authorize.Core.Accounts`

Lets make sure everything works
```bash
mix test

iex -S mix phx.server
# make a new user and login and logout

git add .
git commit -m "add users to the 'core'"
```

## System Emails

you can find (in Dev) any emails that would have been sent for user confirmation or password reminders, etc. at:

`http://localhost:4000/dev/mailbox`

Here you can read and test the messages - without them being sent to `real` people.


## Authorization and Admin Panel

Let's make an Admin Panel - where we control who has access to what areas of code.

### User Migration

We will start by adding a 'roles' field (an array of roles).  We will start with 'user' and 'admin' - maybe more later.  So we start with the migration, which will look like:

```elixir
mix ecto.gen.migration add_roles_to_user

# priv/repo/migrations/20240224134441_add_roles_to_user.exs
defmodule Vitali.Repo.Migrations.AddRolesToUser do
  use Ecto.Migration

  def change do
    alter table("users") do
      add :roles, {:array, :string}, default: ["user"], null: false
    end
  end
end

mix ecto.migrate
```

Now our existing user (& all new users) should have the roles: `["user"]` let's check in iex:

```elixir
iex -S mix phx.server

alias Authorize.Core.Accounts

# from Context file
Accounts.get_user_by_email("nyima@example.com")


#Authorize.Core.Accounts.User<
  __meta__: #Ecto.Schema.Metadata<:loaded, "users">,
  id: "1349f6b9-e3f1-4d7a-813b-d1f1aa49fbe3",
  email: "btihen@gmail.com",
  confirmed_at: nil,
  inserted_at: ~U[2024-02-24 10:31:40Z],
  updated_at: ~U[2024-02-24 10:31:40Z],
  ...
>
```

hmm - not what I expected - I was hoping to see the roles - lets check the DB to see if the migration worked:

```bash
$ psql -d authorize_dev

# list record vertically
authorize_dev=# \x

# show users
authorize_dev=# select * from users;
-[ RECORD 1 ]---+-------------------------------------------------------------
id              | 1349f6b9-e3f1-4d7a-813b-d1f1aa49fbe3
email           | nyima@example.com
hashed_password | $2b$12$nHO8KooIVj7CIjEKWm5CsOXlp0ruIdHmZvUV2VvP6rLivFR24b4/C
confirmed_at    |
inserted_at     | 2024-02-24 10:31:40
updated_at      | 2024-02-24 10:31:40
roles           | {user}

# exit psql
\q
```

excellent we have the new roles and the default is `["user"]` (in elixir) as you can seen in postgres land it is stored as `{user}`.

### Update User schema

So the problem is on the elixir side - we forgot to update the user `schema` after the migration change. We need to add our new field (column) using:
`field :roles, {:array, :string}, default: ["user"]`

```elixir
# lib/authorize/core/accounts/user.ex
defmodule Authorize.Core.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset
  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
  schema "users" do
    field :email, :string
    field :password, :string, virtual: true, redact: true
    field :hashed_password, :string, redact: true
    field :confirmed_at, :naive_datetime
    # add the roles using the following;
    field :roles, {:array, :string}, default: ["user"]

    timestamps(type: :utc_datetime)
  end
  # ...
  # changesets (to update later)
  # ...
end
```

now let's see if User in Phoenix/Elixir has the `roles`

```bash
iex -S mix phx.server
# or if already within iex
recompile

alias Authorize.Core.Accounts

# from Context file
user = Accounts.get_user_by_email("nyima@example.com")

Authorize.Core.Accounts.User<
  __meta__: #Ecto.Schema.Metadata<:loaded, "users">,
  id: "1349f6b9-e3f1-4d7a-813b-d1f1aa49fbe3",
  email: "btihen@gmail.com",
  confirmed_at: nil,
  roles: ["user"],
  inserted_at: ~U[2024-02-24 10:31:40Z],
  updated_at: ~U[2024-02-24 10:31:40Z],
  ...
>

# to access the info use:
user.roles

["user"]
```

Nice, now we have what is expected in our users.

```bash
git add .
git commit -m "add roles to Users"
```

### Build a restricted Admin Panel (for logged in users)

Since our new page page has no new resources we will make the UsersLive page within the 'Admin' area.

```bash
# lets make an admin area within liveview
mkdir lib/authorize_web/live/admin
# create the file needed
touch lib/authorize_web/live/admin/accounts_live.ex
# starter template code
cat <<EOF > lib/authorize_web/live/admin/accounts_live.ex
defmodule AuthorizeWeb.Admin.AccountsLive do
  use Phoenix.LiveView

  @impl true
  def render(assigns) do
    ~H"""
    <h1>Admin.UsersLive</h1>
    """
  end

  @impl true
  def mount(_params, _session, socket) do
    {:ok, socket}
  end
end
EOF
```

Lets start by simply testing our page by adding a simple route - then we will add restrictions:

we want a scope of "/admin" - so to make it only available to logged we can add the following to the end of  the routes file:
```elixir
# lib/authorize_web/router.ex
defmodule AuthorizeWeb.Router do
  use AuthorizeWeb, :router
  # ...
  ## Admin Routes (We need to add the scope 'Admin' here!)
  scope "/admin", AuthorizeWeb.Admin do
    pipe_through [:browser]

    live("/accounts", AccountsLive, :index)
  end
end
```

##### Add Authentication requirement (via plug routing)

hopefully you can now get to:
`http://localhost:4000/admin/accounts`

we need to add this to the routes - we will start by just making sure it can only be accessed by logged in users.

let's protect the standard routing (plug) - with:
```elixir
  scope "/admin", AuthorizeWeb.Admin do
    pipe_through [:browser, :require_authenticated_user]

    live("/accounts", AccountsLive, :index)
  end
```
hopefully now if you open an 'incognito' non-logged in browser you are not able to access this page and are redirected to the login / signin page.

##### Add Authentication requirement (via liveview session)

we want a scope of "/admin" - so to make it only available to logged we can add the following to the end of  the routes file:
```elixir
# lib/authorize_web/router.ex
defmodule AuthorizeWeb.Router do
  use AuthorizeWeb, :router
  # ...
  ## Admin Routes (We need to add the scope 'Admin' here!)
  scope "/admin", AuthorizeWeb.Admin do
    pipe_through [:browser, :require_authenticated_user]

    # session name `:live_admin` - can be what you want but must MUST be unique
    # otherwise you get the error: `attempting to redefine live_session`
    live_session :live_admin,
      on_mount: [{AuthorizeWeb.Access.UserAuth, :ensure_authenticated}] do
      live("/accounts", AccountsLive, :index)
    end
  end
end
```
lets see if we can get to this page
`http://localhost:4000/admin/users`
when logged in and not while logged out


### Restrict Admin Page to Admins only

```elixir
# lib/authorize/core/accounts/user.ex
def admin?(user), do: "admin" in user.roles || user.email == "nyima@example.com"
```

```elixir
# lib/authorize_web/access/user_auth.ex

  # new static route plug
  def require_admin_user(conn, _opts) do
    IO.inspect(conn.assigns)
    IO.inspect(conn.assigns.current_user)

    if Authorize.Core.Accounts.User.admin?(conn.assigns.current_user) do
      conn
    else
      conn
      |> put_flash(:error, "You must be an admin to access this page.")
      |> maybe_store_return_to()
      |> redirect(to: ~p"/")
      |> halt()
    end
  end

  # new liveview session mount check
  def on_mount(:ensure_admin, _params, _session, socket) do
    if Authorize.Core.Accounts.User.admin?(socket.assigns.current_user) do
      {:cont, socket}
    else
      socket =
        socket
        |> Phoenix.LiveView.put_flash(:error, "You must be admin to access this page.")
        |> Phoenix.LiveView.redirect(to: ~p"/")

      {:halt, socket}
    end
  end
```

```elixir
# lib/authorize_web/router.ex
  scope "/admin", AuthorizeWeb.Admin, as: :admin do
    pipe_through [:browser, :require_authenticated_user, :require_admin_user]

    live_session :live_admin,
      on_mount: [{AuthorizeWeb.Access.UserAuth, :ensure_authenticated}, {AuthorizeWeb.Access.UserAuth, :ensure_admin}] do
      live("/accounts", AccountsLive, :index)
      # add other admin live routes as needed
    end
  end
end
```

ok now we have an admin page that requires an Admin!

```bash
git add .
git commit -m "added an admin page restricted to admins"
```

## add a grant_admin_changeset

```elixir
# user.ex
  def admin_roles_changeset(user, attrs, _opts \\ []) do
    allowed_roles = ["admin", "user"] # allowed roles here

    user
    |> cast(attrs, [:roles])
    |> validate_required([:roles])
    |> validate_roles(:roles, allowed_roles)
  end

defp validate_roles(changeset, field, allowed_roles) do
  roles = get_field(changeset, field)

  if Enum.all?(roles, fn role -> role in allowed_roles end) do
    changeset
  else
    add_error(changeset, field, "has invalid roles")
  end
end
```

```elixir
# accounts.ex
  def grant_admin(user) do
    new_roles =
      ["admin" | user.roles]
      |> Enum.uniq()

    user
    |> User.admin_roles_changeset(%{roles: new_roles})
    |> Repo.update()
  end

  def revoke_admin(user) do
    user
    |> User.admin_roles_changeset(%{roles: user.roles -- ["admin"]})
    |> Repo.update()
  end
```




Let's look to see how this is done for the login pages (## Authentication routes) see `lib/authorize_web/router.ex:64`:
```elixir
# lib/authorize_web/router.ex
defmodule AuthorizeWeb.Router do
  use AuthorizeWeb, :router
  # ...
  scope "/accounts", AuthorizeWeb.Access, as: :accounts do
    pipe_through [:browser, :require_authenticated_user]

    live_session :require_authenticated_user,
      on_mount: [{AuthorizeWeb.Access.UserAuth, :ensure_authenticated}] do
      live "/users/settings", UserSettingsLive, :edit
      live "/users/settings/confirm_email/:token", UserSettingsLive, :confirm_email
    end
  end
  # ...
end
```
