---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Phoenix 1.7 with LiveView 0.20 Starter Demo"
subtitle: "Article 0 - Exploring and new Phoenix / LiveView features"
# Summary for listings and search engines
summary: "Quick features and sample usage of Phoenix and LiveView (with minimal explanations)"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "LiveView"]
categories: ["Code", "Elixir Language", "Phoenix Framework", "LiveView"]
date: 2023-12-31T01:01:53+02:00
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

It's been a while since I have had time to code in Elixir.  In this article I wanted to explore its current usage and features.

We will build toward a flexible Kanban / Task Tracker.  We will see how far we get.  The main point is to explore features and create a quasi-tutorial (assuming some pre-existing knowledge).

## Article Series

* Article 0 (Getting Started) - LiveView v0.20.x Intro
* Article 1 (Admin Panel) - [Authorization Intro](https://btihen.dev/posts/elixir/phoenix_1_7_11_liveview_0_20_9_authorize/)
* Article 2 (Model Auth) - [Using Authorization](https://btihen.dev/posts/elixir/phoenix_1_7_11_liveview_0_20_12_auth_chat/)

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

## Binary IDs

if you prefer UUID keys to serial IDs then you can easily do that with the following changes.  _This can be very helpful if you need to sync with external frontend or other external apps_

update `config/config.ex` from:
```elixir
config :taskboard,
  ecto_repos: [Taskboard.Repo],
  generators: [timestamp_type: :utc_datetime]
```

to:

```elixir
config :taskboard,
  ecto_repos: [Taskboard.Repo],
  generators: [
    timestamp_type: :utc_datetime,
    binary_id: true
  ]
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
mv test/taskboard/accounts* test/taskboard/core/.
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

## System Emails

you can find (in Dev) any emails that would have been sent for user confirmation or password reminders, etc. at:

`http://localhost:4000/dev/mailbox`

Here you can read and test the messages - without them being sent to `real` people.


## add Projects / Projects
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

    # add the foreign_key here to ensure the custom association name works
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
```h
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

```h
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

```html
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

### Empty Page Component

**Resources**
* https://hexdocs.pm/phoenix/components.html
* https://pjullrich.gumroad.com/l/bmvp (valuable resource on many levels!)

Let's say you want a reusable 'empty page' component with an interesting logo which will be displayed on ANY index page with nothing to show - we will start with our project page.

Lets make a new file at: `lib/taskboard_web/components/empty_state.ex` with the contents:

```elixir
defmodule TaskboardWeb.Components.EmptyState do
  use TaskboardWeb, :html

  # this help heex know if the component is used properly
  attr(:text, :string)
  attr(:image, :string)
  def empty_state(assigns) do
    ~H"""
    <div>
      <h2 class="text-2xl font-semibold tracking-tight sm:text-center sm:text-4xl">
        <%= @text %>
      </h2>
      <div class="mt-5 mx-auto w-full max-w-xs">
        <img
          alt={@text}
          src={~p"/images/#{@image}"}
          class="w-full max-w-none rounded-xl ring-1 ring-gray-400/10 md:-ml-4 lg:-ml-0"
        />
      </div>
    </div>
    """
  end
end
```

These two line:
```elixir
  attr(:text, :string)
  attr(:image, :string)
```
will let HEEX enforce that they are included in the call (or provide an error)

### usage

Now we can change: `` with the content:
```html

<%= if Enum.count(@streams.projects) == 0 do %>
  <div class="mt-20">
    <.empty_state show text="no Projects yet" image="task.png"/>
    <%!-- <TaskboardWeb.Components.EmptyState.empty_state text="no Projects yet" image="task.png"/> --%>
  </div>
<% else %>
  <.table
    id="projects"
    rows={@streams.projects}
    row_click={fn {_id, project} -> JS.navigate(~p"/admin/projects/#{project}") end}
  >
    <:col :let={{_id, project}} label="Title"><%= project.title %></:col>
    <:col :let={{_id, project}} label="Description"><%= project.description %></:col>
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
<% end %>
```
Now we only show the table when there are items to show.

### Simplify Usage

We can actually simplify the line:
`<TaskboardWeb.Components.EmptyState.empty_state text="no Projects yet" image="task.png"/>`
to
`<.empty_state show text="no Projects yet" image="task.png"/>`
by importing our Component with:
`import TaskboardWeb.Components.EmptyState, only: [empty_state: 1]`
into `lib/taskboard_web/live/admin/project_live/index.ex`

So now the beginning of this file should look like:
```elixir
defmodule TaskboardWeb.Admin.ProjectLive.Index do
  use TaskboardWeb, :live_view

  alias Taskboard.Core.Programs
  alias Taskboard.Core.Programs.Project
  import TaskboardWeb.Components.EmptyState, only: [empty_state: 1]

  ...
end
```

Thus now we can use:
```html

<%= if Enum.count(@streams.projects) == 0 do %>
  <div class="mt-20">
    <.empty_state show text="no Projects yet" image="task.png"/>

  </div>
<% else %>
  ...
<% end %>
```

## Create a Reactive navbar

Since Navbar is a lot of messy code lets make a Navbar component.

To make it reactive we will need to inject some 'JS' that Phoenix provides (and avoids us needing to install our own JavaScript framework)

Let's start making an landing page with a menu bar.  To do this we will start by updating: `lib/taskboard_web/controllers/page_html/root.html.heex`

By putting the navbar in `root.html.heex` we won't be able to add user notifications into the menubar.  If this is desired they you will need to put the dynamic navbar in `app.html.heex` and the homepage static navbar in `home.html.heex` (There might be a better way, but I don't know it yet).

Be sure to use `<.link navigate={~p"/some/route/here"}>` instead of `<.link patch={~p"/some/route/here"}` in `root.html.heex` because the patch navigation option is only available in LiveViews (LiveViews use both `root.html.heex` and `app.html.heex` - dead pages only use `eoot.html.heex`).

### Navbar Component

Create a new page at `lib/taskboard_web/components/navigation_bar.ex` and copy the following contents into it:

```elixir
defmodule TaskboardWeb.Components.NavigationBar do
  use TaskboardWeb, :html

  def navigation_bar(assigns) do
    ~H"""
    <header class="absolute sticky inset-x-0 top-0 z-50 bg-slate-100">
      <nav class="mx-auto flex max-w-7xl items-center justify-between p-6 lg:px-8" aria-label="Global">
        <div class="flex lg:flex-1">
          <.link href={~p"/"} class="-m-1.5 p-1.5">
            <span class="sr-only">TaskBoard</span>
            <!-- Logo -->
            <img
              class="h-8 w-auto"
              src={~p"/images/task.png"}
              alt="logo"
            >
          </.link>
        </div>
        <!-- hamburger menu button -->
        <div class="flex lg:hidden">
          <button phx-click={JS.toggle(to: "#mobile-menu")} class="-m-2.5 inline-flex items-center justify-center rounded-md p-2.5 text-gray-700">
            <span class="sr-only">Open main menu</span>
            <svg class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" aria-hidden="true">
              <path stroke-linecap="round" stroke-linejoin="round" d="M3.75 6.75h16.5M3.75 12h16.5m-16.5 5.25h16.5" />
            </svg>
          </button>
        </div>
        <div class="hidden lg:flex lg:gap-x-12">
          <a href={~p"/admin/projects"} class="text-sm font-semibold leading-6 text-gray-900 hover:text-blue-700">
            SiteAdmin
          </a>
          <a href="#" class="text-sm font-semibold leading-6 text-gray-900 hover:text-blue-700">
            ProjectOwner
          </a>
          <a href="#" class="text-sm font-semibold leading-6 text-gray-900 hover:text-blue-700">
            Collaborate
          </a>
        </div>
        <!-- right justified links (login, etc) -->
        <div class="flex flex-1 items-center justify-end gap-x-6">
          <%= if @current_user do %>
            <div class="relative">
              <%!-- <button type="button" class="flex items-center gap-x-1 text-sm font-semibold leading-6 text-gray-900" aria-expanded="false"> --%>
              <button phx-click={JS.toggle(to: "#user-dropdown")} class="flex items-center gap-x-1 text-sm font-semibold leading-6 text-gray-900">
                <%= @current_user.email %>
                <svg class="h-5 w-5 flex-none text-gray-400" viewBox="0 0 20 20" fill="currentColor" aria-hidden="true">
                  <path fill-rule="evenodd" d="M5.23 7.21a.75.75 0 011.06.02L10 11.168l3.71-3.938a.75.75 0 111.08 1.04l-4.25 4.5a.75.75 0 01-1.08 0l-4.25-4.5a.75.75 0 01.02-1.06z" clip-rule="evenodd" />
                </svg>
              </button>
              <div id="user-dropdown" class="hidden absolute -left-8 top-full z-10 mt-3 w-56 rounded-xl bg-white p-2 shadow-lg ring-1 ring-gray-900/5">
                <.link
                  phx-click={JS.hide(to: "#user-dropdown")}
                  href={~p"/auth/users/settings"}
                  class="block rounded-lg px-3 py-2 text-sm font-semibold leading-6 text-gray-900 hover:bg-gray-50"
                >
                  Settings
                </.link>
                <.link
                  phx-click={JS.hide(to: "#user-dropdown")}
                  href={~p"/auth/users/log_out"}
                  method="delete"
                  class="block rounded-lg px-3 py-2 text-sm font-semibold leading-6 text-gray-900 hover:bg-gray-50"
                >
                  Log out
                </.link>
              </div>
            </div>
            <.link
              href={~p"/auth/users/log_out"}
              class="rounded-md bg-indigo-600 px-3 py-2 text-sm font-semibold text-white shadow-sm hover:bg-indigo-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-600"
            >
              Log out
            </.link>
          <% else %>
            <.link
              href={~p"/auth/users/log_in"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Log in
            </.link>
            <.link
              href={~p"/auth/users/register"}
              class="rounded-md bg-indigo-600 px-3 py-2 text-sm font-semibold text-white shadow-sm hover:bg-indigo-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-600"
            >
              Register
            </.link>
          <% end %>
        </div>
      </nav>

      <!-- Mobile menu, show/hide based on menu open state. -->
      <div id="mobile-menu" class="lg:hidden" role="dialog" aria-modal="true">
        <!-- Background backdrop, show/hide based on slide-over state. -->
        <div class="fixed inset-0 z-50"></div>
        <div class="fixed inset-y-0 right-0 z-50 w-full overflow-y-auto bg-white px-6 py-6 sm:max-w-sm sm:ring-1 sm:ring-gray-900/10">
          <div class="flex items-center justify-between">
            <.link href={~p"/"} class="-m-1.5 p-1.5" >
              <span class="sr-only">TaskBoard</span>
              <!-- Logo -->
              <img
                class="h-8 w-auto"
                src={~p"/images/task.png"}
                alt="logo"
              >
            </.link>
            <!-- Close button, show/hide based on slide-over state. -->
            <button phx-click={JS.toggle(to: "#mobile-menu")} class="-m-2.5 rounded-md p-2.5 text-gray-700">
              <span class="sr-only">Close menu</span>
              <svg class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" aria-hidden="true">
                <path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
          </div>
          <div class="mt-6 flow-root">
            <div class="-my-6 divide-y divide-gray-500/10">
              <div class="space-y-2 py-3">
                <a href={~p"/admin/projects"} class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700">
                  SiteAdmin
                </a>
                <a href="#" class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700">
                  ProjectOwner
                </a>
                <a href="#" class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700">
                  Collaborate
                </a>
              </div>
              <div class="space-y-2 py-3">
                <%= if @current_user do %>
                  <%= @current_user.email %>
                  <.link
                    href={~p"/auth/users/settings"}
                    class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700"
                  >
                    Settings
                  </.link>
                  <.link
                    href={~p"/auth/users/log_out"}
                    method="delete"
                    class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700"
                  >
                    Log out
                  </.link>
                <% else %>
                  <.link
                    href={~p"/auth/users/register"}
                    class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700"
                  >
                    Register
                  </.link>
                  <.link
                    href={~p"/auth/users/log_in"}
                    class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700"
                  >
                    Log in
                  </.link>
                <% end %>
              </div>
            </div>
          </div>
        </div>
      </div>
    </header>
    """
  end
end
```

### Using the Navbar

now we can add our new component to the `root.html.heex`

```html
<!DOCTYPE html>
<html lang="en" class="[scrollbar-gutter:stable]">

  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="csrf-token" content={get_csrf_token()} />
    <.live_title suffix=" · Phoenix Framework">
      <%= assigns[:page_title] || "Taskboard" %>
    </.live_title>
    <link phx-track-static rel="stylesheet" href={~p"/assets/app.css"} />
    <script defer phx-track-static type="text/javascript" src={~p"/assets/app.js"}>
    </script>
  </head>

  <body class="bg-white antialiased">

    <TaskboardWeb.Components.NavigationBar.navigation_bar current_user={@current_user}/>

    <%= @inner_content %>

  </body>
</html>
```

(this will make it work everywhere - with the limitation that it can't do pop-up notifications).  For popup notifications we need to put it in: `app.html.heex`, but that has some complications with @current_user I haven't yet resolved.

### Notes on Reactivity

Since we want our app to work with mobile and desktop we need to create a collapsable menubar.  We will do this by adding the following JS toggle `JS.toggle(to: "#mobile-menu"` snippets to the menu bar -- the outline is:

```html
<!-- Hamburger Button -->
<button phx-click={JS.toggle(to: "#mobile-menu")} class="-m-2.5 inline-flex items-center justify-center rounded-md p-2.5 text-gray-700">
  <!-- SVG for the icon -->
</button>

<!-- Mobile Menu -->
<div id="mobile-menu" class="lg:hidden" role="dialog" aria-modal="true">
  <div class="flex items-center justify-between">
    <!-- ... other content ... -->
    <!-- Close Button -->
    <button phx-click={JS.toggle(to: "#mobile-menu")} class="-m-2.5 rounded-md p-2.5 text-gray-700">
      <!-- SVG for the close icon -->
    </button>
  </div>
  <!-- ... other content ... -->
</div>
```

When we look at a `dead` homepage: `http://localhost:4000/` and when we look at a `liveview` page: `http://localhost:4000/auth/users/settings` we will have our new reactive navbar!

### Add links and images to the navbar

Let's put a new custom icon in to the path: `priv/static/images/logo.png` - ideally an SVG, but for now I will just use a png.

Now we can update our logo and the url links with - note variables and information INSIDE an html tag must be places within `{}` and information we place in HTML will use `<% %?` or `<%= %>` formatting.

```html
        <div class="flex lg:flex-1">
          <!-- link to homepage -->
          <.link href={~p"/"} class="-m-1.5 p-1.5" >
            <span class="sr-only">TaskBoard</span>
            <!-- the site logo in the navbar -->
            <img
              class="h-8 w-auto"
              src={~p"/images/logo.png"}
              alt="logo"
            >
          </.link>
        </div>
```

NOTE: `~p` within HEEX uses the internal route checking to ensure we are linking to our urls and images properly.

we can also link our SiteAdmin link in the menubar with:
```html
<a
    href={~p"/admin/projects"}
    class="text-sm font-semibold leading-6 text-gray-900 hover:text-blue-700">
  SiteAdmin
</a>
```

### Reactive Dropdown Menu

non-reactive dropdown css outline
```
<div class="relative">
  <button type="button" class="flex items-center gap-x-1 text-sm font-semibold leading-6 text-gray-900" aria-expanded="false">
    Company
    <svg class="h-5 w-5 flex-none text-gray-400" viewBox="0 0 20 20" fill="currentColor" aria-hidden="true">
      <path fill-rule="evenodd" d="M5.23 7.21a.75.75 0 011.06.02L10 11.168l3.71-3.938a.75.75 0 111.08 1.04l-4.25 4.5a.75.75 0 01-1.08 0l-4.25-4.5a.75.75 0 01.02-1.06z" clip-rule="evenodd" />
    </svg>
  </button>

  <div class="absolute -left-8 top-full z-10 mt-3 w-56 rounded-xl bg-white p-2 shadow-lg ring-1 ring-gray-900/5">
    <a href="#" class="block rounded-lg px-3 py-2 text-sm font-semibold leading-6 text-gray-900 hover:bg-gray-50">Settings</a>
    <a href="#" class="block rounded-lg px-3 py-2 text-sm font-semibold leading-6 text-gray-900 hover:bg-gray-50">Logout</a>
  </div>
</div>
```

now we add our reactive JS components - we want the dropdown hidden by default so we load it with `hidden` we also add `hidden` back after we click on a menu item (and of course we toggle it open and closed when clicking on the menu bar item):

```html
<div class="relative">
  <button phx-click={JS.toggle(to: "#user-dropdown")} class="flex items-center gap-x-1 text-sm font-semibold leading-6 text-gray-900">
    <%= @current_user.email %>
    <!-- Chevron SVG here -->
  </button>

  <div id="user-dropdown" class="hidden absolute -left-8 top-full z-10 mt-3 w-56 rounded-xl bg-white p-2 shadow-lg ring-1 ring-gray-900/5">
    <a phx-click={JS.hide(to: "#user-dropdown")} href="/settings" class="block rounded-lg px-3 py-2 text-sm font-semibold leading-6 text-gray-900 hover:bg-gray-50">Settings</a>
    <a phx-click={JS.hide(to: "#user-dropdown")} href="/logout" class="block rounded-lg px-3 py-2 text-sm font-semibold leading-6 text-gray-900 hover:bg-gray-50">Log out</a>
    <!-- ... more links ... -->
  </div>
</div>
```

### Sticky Navbbar

Resources:
* https://stackoverflow.com/questions/60169463/tailwindcss-fixed-navbar

```html
<header class="sticky top-0 z-50"></header>
```

but now transparent - lets add a background color

### Navbar color (non-transparent)

Once the navbar is fixed, it will need a background color `bg-slate-100` to the navbar header (can be any color desired)

https://tailwindcss.com/docs/background-color

```
<header class="absolute sticky inset-x-0 top-0 z-50 bg-slate-100">
```

## Oft Repeated CSS classes

we can use the tailwind `@apply` to define our own reusable 'easier to read' css class.  Go to `assets/css/app.css` and make it look like:
```css
@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";

/* This file is for your main application CSS */
.nav-link {
  @apply text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700
}
```

now it can be used withsome think like:
```html
<.link class="nav-link" html={~p"/"}>Home</.link>
```


In this way you will no longer see both the navbar and the contents below it - which is quite confusing.

## optional Live Homepage

to make a 'Live' /homehage we an do the following:

```bash
mkdir lib/taskboard_web/live/home_live
touch lib/taskboard_web/live/home_live/index.ex
touch lib/taskboard_web/live/home_live/index.html.heex
```

now lets populate the heex `lib/taskboard_web/live/home_live/index.html.heex` file with:
```html
<.flash_group flash={@flash} />

<div class="bg-white">

  <!-- page hero content -->
  <div class="relative isolate overflow-hidden bg-gradient-to-b from-indigo-100/20 pt-14">
    <div class="absolute inset-y-0 right-1/2 -z-10 -mr-96 w-[200%] origin-top-right skew-x-[-30deg] bg-white shadow-xl shadow-indigo-600/10 ring-1 ring-indigo-50 sm:-mr-80 lg:-mr-96" aria-hidden="true"></div>
    <div class="mx-auto max-w-7xl px-6 py-32 sm:py-40 lg:px-8">
      <div class="mx-auto max-w-2xl lg:mx-0 lg:grid lg:max-w-none lg:grid-cols-2 lg:gap-x-16 lg:gap-y-6 xl:grid-cols-1 xl:grid-rows-1 xl:gap-x-8">
        <h1 class="max-w-2xl text-4xl font-bold tracking-tight text-gray-900 sm:text-6xl lg:col-span-2 xl:col-auto">
          Changing Collaboration.
        </h1>
        <div class="mt-6 max-w-xl lg:mt-0 xl:col-end-1 xl:row-start-1">
          <p class="text-lg leading-8 text-gray-600">
            Share tasks, progress and collaborate with partners
          </p>
          <div class="mt-10 flex items-center gap-x-6">
            <.link href={~p"/auth/users/register"}
              class="rounded-md bg-indigo-600 px-3.5 py-2.5 text-sm font-semibold text-white shadow-sm hover:bg-indigo-500 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-indigo-600">
              Get started
            </.link>
          </div>
        </div>
        <img src={~p"/images/task.png"}
          alt="" class="mt-10 aspect-[6/5] w-full max-w-lg rounded-2xl object-cover sm:mt-16 lg:mt-0 lg:max-w-none xl:row-span-2 xl:row-end-2 xl:mt-36">
      </div>
    </div>
    <div class="absolute inset-x-0 bottom-0 -z-10 h-24 bg-gradient-to-t from-white sm:h-32"></div>
  </div>

</div>
```

Now we can write a minimalistic liveview action `lib/taskboard_web/live/home_live/index.ex` page (without any DB model included):

```elixir
defmodule TaskboardWeb.HomeLive.Index do
  use TaskboardWeb, :live_view

  @impl true
  def mount(_params, _session, socket) do
    {:ok, socket}
  end

  @impl true
  @spec handle_params(any(), any(), %{
          :assigns => atom() | %{:live_action => :index, optional(any()) => any()},
          optional(any()) => any()
        }) :: {:noreply, map()}
  def handle_params(params, _url, socket) do
    {:noreply, apply_action(socket, socket.assigns.live_action, params)}
  end

  defp apply_action(socket, :index, _params) do
    socket
    |> assign(:page_title, "Landing Page")
  end
end
```

now we can update the routes `lib/taskboard_web/router.ex` file.

We need to change the default root route from:
`get "/", PageController, :home`
to:
`live "/", HomeLive.Index, :index`

now this part of the route should look like:
```elixir
defmodule TaskboardWeb.Router do
  use TaskboardWeb, :router

  import TaskboardWeb.Auth.UserAuth

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, html: {TaskboardWeb.Layouts, :root}
    plug :protect_from_forgery
    plug :put_secure_browser_headers
    plug :fetch_current_user
  end

  pipeline :api do
    plug :accepts, ["json"]
  end

  scope "/", TaskboardWeb do
    pipe_through :browser

    # get "/", PageController, :home
    live "/", HomeLive.Index, :index
  end
  ...
end
```

now you can delete the unneeded files with:
```bash
rm lib/taskboard_web/controllers/page_controller.ex
rm lib/taskboard_web/controllers/page_html.ex
rm lib/taskboard_web/controllers/page_html/home.html.heex
```

and tou can if desired move the navbar into `app.html.heex` if user notifications are needed in the navbar.

## add role to user

* **is_admin** - role based access to the admin pages
* **name** - backfill existing with the email and add name to the registration

Resources:
* https://devhints.io/phoenix-migrations
* https://fly.io/phoenix-files/backfilling-data/#batching-deterministic-data

### User Migration

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

### Update the admin page

### Update User settings page

### Update Signup page

### backfill to user name?

## Role based Restrictions

Resources:
* https://www.youtube.com/watch?v=6TlcVk-1Tpc
* https://www.knowthen.com/implementing-authorization-using-role-based-access-control-rbac-in-phoenix-web-applications/
* https://peterullrich.com/build-a-rap-for-phoenix-part-1
* https://www.knowthen.com/elixir-and-phoenix-for-beginners/
* https://elixirforum.com/t/phx-gen-auth-and-role-based-authentication/49428/7
* https://elixirforum.com/t/protecting-link-creation-functions-as-well-as-resource-controllers/1336
* https://stackoverflow.com/questions/26055501/how-to-restrict-access-to-certain-routes-in-phoenix
* https://blog.appsignal.com/2021/11/02/authorization-and-policy-scopes-for-phoenix-apps.html

## project owner area

### add collaborators to projects

### Add Epics to projects

### Add Tasks to projects with status

## Add a collaboration (task update) area for non-owners

### add drag and drop to tasks to update status

### add a todo list for a task?

## fly.io deployment

Resources:
* https://pjullrich.gumroad.com/l/bmvp
* https://fly.io/docs/elixir/getting-started/existing/
* https://supabase.com/blog/postgres-on-fly-by-supabase

PS - Be aware that the 'Free' instances require adding a payment method.

```bash
# authenticate
fly auth login

# region can't be Germany for free account (creates Docker images)
fly launch

#
flyctl deploy

# add DB info
fly secrets set DATABASE_URL="postgres://--------:*********@----.db.elephantsql.com/-------"

fly deploy

# tiny turtle only allows 5 connections and we need one for deployment migrations - thus "4"
fly secrets set POOL_SIZE=4

fly deploy

# if you get the error: `The database does not exist`
# and `To fix the first issue, run "mix ecto.create" for the desired MIX_ENV.``
# then: open `rel/env.sh.eex` and disable: `export ECTO_IPV6="true"` so it looks like:

#!/bin/sh

# configure node for distributed erlang with IPV6 support
export ERL_AFLAGS="-proto_dist inet6_tcp"
# export ECTO_IPV6="true"
export DNS_CLUSTER_QUERY="${FLY_APP_NAME}.internal"
export RELEASE_DISTRIBUTION="name"
export RELEASE_NODE="${FLY_APP_NAME}-${FLY_IMAGE_REF##*-}@${FLY_PRIVATE_IP}"


fly deploy
```
