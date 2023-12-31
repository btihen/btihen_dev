---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Phoenix 1.7 with LiveView 0.20 Starter Demo"
subtitle: "Exploring and new Phoenix / LiveView features"
# Summary for listings and search engines
summary: "Quick features and sample usage of Phoenix and LiveView (with minaml explanations)"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "LiveView"]
categories: ["Code", "Elixir Language", "Phoenix Framework", "LiveView"]
date: 2023-12-31T01:01:53+02:00
lastmod: 2023-12-31T01:01:53+02:00
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

It's been a while since I have had time to code in Elixir.  In this article I wanted to explore its current usage and features.

We will build toward a flexible Kanban / Task Tracker.  We will see how far we get.  The main point is to explore features and create a quasi-tutorial (assuming some pre-existing knowledge).

## Getting started

```bash
mix phx.new taskboard
cd taskboard
mix ecto.create
git init
git add .
git commit -m "initial commit"

iex -S mix phx.server
```

## using phx.gen.auth

Some resources used:
* https://fly.io/phoenix-files/phx-gen-auth/
* https://hexdocs.pm/phoenix/mix_phx_gen_auth.html
* https://blog.logrocket.com/phoenix-authentication/
* https://experimentingwithcode.com/phoenix-authentication-with-phx-gen-auth-part-1/
* https://experimentingwithcode.com/restricting-registrations-when-using-phx-gen-auth-part-1/
* https://levelup.gitconnected.com/a-deep-dive-into-authentication-in-elixir-phoenix-with-phx-gen-auth-9686afecf8bd

also:
* https://nithinbekal.com/posts/phoenix-authentication/
* https://alchemist.camp/episodes/phoenix-live-view-auth

```bash
mix phx.gen.auth Accounts User users --web Auth
mix deps.get
mix ecto.migrate
```

### organize user into core data

I like to organize the code into areas that is associated with usage - so, in this case, I will start by moving all core 'models' into a folder / module called 'core' by doing:

```bash
# create the core lib folder
mkdir lib/taskboard/core/
# create the core test folder
mkdir test/taskboard/core
mkdir test/support/fixtures/core

# move your lib code into the new area
mv lib/taskboard/account* lib/taskboard/core/.
# move the test and support code into 'core'
mv test/taskboard/accounts*test/taskboard/core/.
mv test/support/fixtures/accounts* test/support/fixtures/core/.
```
replace every `Taskboard.Accounts` with `Taskboard.Core.Accounts`

Lets make sure everything works
```bash
mix test

iex -S mix phx.server
# make a new user and login and logout

git add .
git commit -m "add user auth"
```

### add title / home page



### add Projects / Projects
(using web space - and lib module)

Resources used:
* https://hexdocs.pm/phoenix/Mix.Tasks.Phx.Gen.Live.html
* https://elixirforum.com/t/ecto-association-with-custom-name/39981
* https://hexdocs.pm/ecto/Ecto.Schema.html#has_many/3

with a renamed (owner) association for the 'user'

and within a routing namespace (auth)

```bash
mix phx.gen.live Projects Project projects title:string description:text owner_id:references:users --web Admin
```

Again lets move 'Projects' into the core area/
```bash
mv lib/projects* lib/core/.
```

replace: `Taskboard.Projects` with `Taskboard.Core.Projects`

now update **schemas** with the custom association name (owner) - replace:
`field :owner_id, :id`
with:
`belongs_to :owner, User, [foreign_key: :owner_id]`

and in the **changeset** add the `owner_id`

Thus `lib/core/projects/project.ex` should now look like:

```elixir
defmodule Taskboard.Core.Projects.Project do
  use Ecto.Schema
  import Ecto.Changeset
  # add the association alias
  alias Taskboard.Core.Accounts.User

  schema "projects" do
    field :description, :string
    field :title, :string
    # remove this
    # field :owner_id, :id
    # use this for relationship and custom association name
    belongs_to :owner, User, [foreign_key: :owner_id]

    timestamps(type: :utc_datetime)
  end

  @doc false
  def changeset(project, attrs) do
    project
    |> cast(attrs, [:title, :description, :owner_id])
    |> validate_required([:title, :description, :owner_id])
    # |> cast(attrs, [:title, :description])
    # |> validate_required([:title, :description])
  end
end
```

we also need to update user **schema** with the has_many association:
`has_many :projects, Project, foreign_key: :owner_id`
again given the rename of owner / user id we need to add the foreign key.

now `lib/core/accounts/user.ex` should now look like:

```elixir
defmodule Taskboard.Core.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset
  # add the association alias
  alias Taskboard.Core.Projects.Project

  schema "users" do
    field :email, :string
    field :password, :string, virtual: true, redact: true
    field :hashed_password, :string, redact: true
    field :confirmed_at, :naive_datetime

    # add the foreignkey here to ensure the custom association name works
    has_many :projects, Project, foreign_key: :owner_id

    timestamps(type: :utc_datetime)
  end
  ...

end
```

finally we need to update our new **migration** to delete projects when user is deleted - `on_delete: :delete_all` and ensure an association = `null: false - replace:
`add :owner_id, references(:users, on_delete: :nothing)`
with:
`add :owner_id, references(:users, on_delete: :delete_all), null: false`

Thus the migration should look like:

```elixir
defmodule Taskboard.Repo.Migrations.CreateProjects do
  use Ecto.Migration

  def change do
    create table(:projects) do
      add :title, :string
      add :description, :text
      # remove this
      # add :owner_id, references(:users, on_delete: :nothing)
	    # add this to delete projects when user is deleted and require association
      add :owner_id, references(:users, on_delete: :delete_all), null: false

      timestamps(type: :utc_datetime)
    end

    create index(:projects, [:owner_id])
  end
end
```

update the routes too with:
```elixir
    scope "/admin", TaskboardWeb.Admin, as: :admin do
      pipe_through :browser

      live "/projects", ProjectLive.Index, :index
      live "/projects/new", ProjectLive.Index, :new
      live "/projects/:id/edit", ProjectLive.Index, :edit

      live "/projects/:id", ProjectLive.Show, :show
      live "/projects/:id/show/edit", ProjectLive.Show, :edit
    end
```

update routes
```elixir
    scope "/admin", TaskboardWeb.Admin, as: :admin do
      pipe_through :browser

      live "/projects", ProjectsLive.Index, :index
      live "/projects/new", ProjectsLive.Index, :new
      live "/projects/:id/edit", ProjectsLive.Index, :edit

      live "/projects/:id", ProjectsLive.Show, :show
      live "/projects/:id/show/edit", ProjectsLive.Show, :edit
    end
```


lets see if it all works
```bash
mix ecto.migrate
mix test

iex -S mix phx.server
```

go to the **admin** page `http://localhost:4000/admin/projects` and try to make a new project.

### fix the projects form

Resources used:
* https://elixirforum.com/t/phoenix-1-7-liveview-0-20-2-simple-dropdown-selector-question/60643

We can't make new projects (because we required an owner_id), but there is no way to enter the owner!

To add the user to the form, we first need to add the users to the `socket` in the update function (within the file: `lib/taskboard_web/live/admin/project_live/form_component.ex`) thus it should now look like:
```elixir
  @impl true
  def update(%{domains: domains} = assigns, socket) do
    changeset = WorkDomains.change_domains(domains)

    # add these lines
    owners = ClearSync.Core.Accounts.list_users()
    socket = assign(socket, owners: owners)

    {:ok,
     socket
     |> assign(assigns)
     |> assign_form(changeset)}
  end
```

now we need to create the list_users: in `lib/taskboard/core/accounts.ex`
```elixir
  def list_users, do: Repo.all(User)
```


now we can add the user selection to the form `lib/taskboard_web/live/admin/project_live/form_component.ex` page:
```heex
        <.input
          field={@form[:owner_id]}
          type="select"
          label="Owner"
          prompt="Select an owner"
          options={Enum.map(@users, &{&1.email, &1.id})}
        />
```

We will also make the description a textarea by changing:
`<.input field={@form[:description]} type="text" label="Description" />`
to
`<.input field={@form[:description]} type="textarea" label="Description" />`

Thus now `lib/taskboard_web/components/core_components.ex` should look like:

```elixir
defmodule TaskboardWeb.Admin.ProjectLive.FormComponent do
  use TaskboardWeb, :live_component

  alias Taskboard.Core.Projects

  @impl true
  def render(assigns) do
    ~H"""
    <div>
      <.header>
        <%= @title %>
        <:subtitle>Use this form to manage project records in your database.</:subtitle>
      </.header>

      <.simple_form
        for={@form}
        id="project-form"
        phx-target={@myself}
        phx-change="validate"
        phx-submit="save"
      >
        <.input field={@form[:title]} type="text" label="Title" />
		    <!-- use text area for a longer input in the description -->
        <.input field={@form[:description]} type="textarea" label="Description" />
        <!-- add this to allow a relationship -->
        <.input
          field={@form[:owner_id]}
          type="select"
          label="Owner"
          prompt="Select an owner"
          options={Enum.map(@owners, &{&1.email, &1.id})}
        />
        <:actions>
          <.button phx-disable-with="Saving...">Save Project</.button>
        </:actions>
      </.simple_form>
    </div>
    """
  end

  @impl true
  def update(%{project: project} = assigns, socket) do
    changeset = Projects.change_project(project)

    # add these two lines (so we have users available in the heex render)
    owners = Taskboard.Core.Accounts.list_users()
    socket = assign(socket, owners: owners)

    {:ok,
     socket
     |> assign(assigns)
     |> assign_form(changeset)}
  end
  ...
end
```

now be sure we can make a new project.

```bash
git add .
git commit -m "add projects to an admin panel
```

### update show & index pages

Now we also notice that both the show and index page also don't show the 'owner' ('user').  To do this we need to 'preload' the owner within the project - Phoenix / Ecto doesn't do this automatically. (if you forget you get a 'not loaded error')

Preload Resource:
* https://til.hashrocket.com/posts/etrpzkvsm6-how-to-force-reload-associations-in-ecto

update `lib/taskboard/core/Projects.ex` to preload the owner in the index page by changing:
`def list_projects, do: Repo.all(Project)`
to
`def list_projects, do: Repo.all(Project) |> Repo.preload(:owner)`

and for the show page change:
`def get_project!(id), do: Repo.get!(Project, id)`
to
`def get_project!(id), do: Repo.get!(Project, id) |> Repo.preload(:owner)`

Thus now `lib/taskboard/core/Projects.ex` should look like:

```elixir
defmodule Taskboard.Core.Projects do
  import Ecto.Query, warn: false
  alias Taskboard.Repo

  alias Taskboard.Core.Projects.Project

  # def list_projects, do: Repo.all(Project)
  def list_projects, do: Repo.all(Project) |> Repo.preload(:owner)

  # def get_project!(id), do: Repo.get!(Project, id)
  def get_project!(id), do: Repo.get!(Project, id) |> Repo.preload(:owner)

  ...
```

Now we can add
`<:col :let={{_id, project}} label="Owner"><%= project.owner.email %></:col>`
to the index page:

Thus now the index heex page should look like: `lib/taskboard_web/live/admin/project_live/index.html.heex`:

```
<.header>
  Listing Projects
  <:actions>
    <.link patch={~p"/admin/projects/new"}>
      <.button>New Project</.button>
    </.link>
  </:actions>
</.header>

<.table
  id="projects"
  rows={@streams.projects}
  row_click={fn {_id, project} -> JS.navigate(~p"/admin/projects/#{project}") end}
>
  <:col :let={{_id, project}} label="Title"><%= project.title %></:col>
  <:col :let={{_id, project}} label="Description"><%= project.description %></:col>
  <!-- add this line -->
  <:col :let={{_id, project}} label="Owner"><%= project.owner.email %></:col>
  <:action :let={{_id, project}}>
    <div class="sr-only">
      <.link navigate={~p"/admin/projects/#{project}"}>Show</.link>
    </div>
    <.link patch={~p"/admin/projects/#{project}/edit"}>Edit</.link>
  </:action>
  <:action :let={{id, project}}>
    <.link
      phx-click={JS.push("delete", value: %{id: project.id}) |> hide("##{id}")}
      data-confirm="Are you sure?"
    >
      Delete
    </.link>
  </:action>
</.table>

<.modal :if={@live_action in [:new, :edit]} id="project-modal" show on_cancel={JS.patch(~p"/admin/projects")}>
  <.live_component
    module={TaskboardWeb.Admin.ProjectLive.FormComponent}
    id={@project.id || :new}
    title={@page_title}
    action={@live_action}
    project={@project}
    patch={~p"/admin/projects"}
  />
</.modal>
```

now we should see the email for the owner on the **index** page!

Now we are ready to fix the **show** page - add:
`<:item title="Owner"><%= @project.owner.email %></:item>`
to the `<.list>` in the show heex page.

Thus `lib/taskboard_web/live/admin/project_live/show.html.heex` should now look like:

```elixir
<.header>
  Project <%= @project.id %>
  <:subtitle>This is a project record from your database.</:subtitle>
  <:actions>
    <.link patch={~p"/admin/projects/#{@project}/show/edit"} phx-click={JS.push_focus()}>
      <.button>Edit project</.button>
    </.link>
  </:actions>
</.header>

<.list>
  <:item title="Title"><%= @project.title %></:item>
  <:item title="Description"><%= @project.description %></:item>
  <!-- add this line -->
  <:item title="Owner"><%= @project.owner.email %></:item>
</.list>

<.back navigate={~p"/admin/projects"}>Back to projects</.back>

<.modal :if={@live_action == :edit} id="project-modal" show on_cancel={JS.patch(~p"/admin/projects/#{@project}")}>
  <.live_component
    module={TaskboardWeb.Admin.ProjectLive.FormComponent}
    id={@project.id}
    title={@page_title}
    action={@live_action}
    project={@project}
    patch={~p"/admin/projects/#{@project}"}
  />
</.modal>
```

Now if we click on a project on the index page we should see the owner on the show page too.


```bash
git add .
git commit -m "admin panel for projects"
```


### add attributes to the user

* **is_admin** - role based access to the admin pages
* **name** - backfill existing with the email and add name to the registration

Resources:
* https://devhints.io/phoenix-migrations
* https://fly.io/phoenix-files/backfilling-data/#batching-deterministic-data

```bash
mix ecto.gen.migration add_user_attributes
```

```elixir
defmodule Taskboard.Repo.Migrations.AddUserAttributes do
  use Ecto.Migration

  def change do
    alter table(:users) do
      add :name, :string
      add :is_admin, :boolean, default: false, null: false
    end
  end
end
```

### add user to the admin page

### only allow is_admin access to the admin page

Resources:
* https://elixirforum.com/t/phx-gen-auth-and-role-based-authentication/49428/7

### add edit profile to the owner area

### add projects to the owner area

### add a landing / home page
mix phx.gen.html Pages Home index --web static --no-schema --no-context

make this the default route

add login (link)

if logged in show the admin (if admin), projects link and profile link
