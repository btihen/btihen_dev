---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Phoenix 1.7 with LiveView 0.20 Starter Demo"
subtitle: "Exploring and new Phoenix / LiveView features"
# Summary for listings and search engines
summary: "Quick features and sample usage of Phoenix and LiveView (with minimal explanations)"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "LiveView"]
categories: ["Code", "Elixir Language", "Phoenix Framework", "LiveView"]
date: 2023-12-31T01:01:53+02:00
lastmod: 2024-01-02T01:01:53+02:00
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
```html
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

```html
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

## a reactive navbar

**Resources**
* https://pjullrich.gumroad.com/l/bmvp (valuable resource on many levels!)

Let's start making an landing page with a menu bar.  To do this we will start by updating: `lib/taskboard_web/controllers/page_html/root.html.heex`

By putting the navbar in `root.html.heex` we won't be able to add user notifications into the menubar.  If this is desired they you will need to put the dynamic navbar in `app.html.heex` and the homepage static navbar in `home.html.heex` (There might be a better way, but I don't know it yet).

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

we also want to add links to our **user** settings:

```html
      <!-- right justified links (login, etc) -->
      <div class="hidden lg:flex lg:flex-1 lg:justify-end lg:gap-x-4">
        <%= if @current_user do %>
          <.link
            href={~p"/auth/users/settings"}
            class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
          >
            <%= @current_user.email %>
          </.link>
          <.link
            href={~p"/auth/users/log_out"}
            method="delete"
            class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
          >
            Log out
          </.link>
        <% else %>
          <.link
            href={~p"/auth/users/register"}
            class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
          >
            Register
          </.link>
          <.link
            href={~p"/auth/users/log_in"}
            class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
          >
            Log in
          </.link>
        <% end %>
      </div>
```

now the page might look something like:

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
    <header class="absolute inset-x-0 top-0 z-50">
      <nav class="mx-auto flex max-w-7xl items-center justify-between p-6 lg:px-8" aria-label="Global">
        <div class="flex lg:flex-1">
          <a href="#" class="-m-1.5 p-1.5">
            <span class="sr-only">Taskboard</span>
            <img class="h-8 w-auto" src="https://tailwindui.com/img/logos/mark.svg?color=indigo&shade=600" alt="">
          </a>
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
          <a href="#" class="text-sm font-semibold leading-6 text-gray-900 hover:text-blue-700">
            Admin
          </a>
          <a href="#" class="text-sm font-semibold leading-6 text-gray-900 hover:text-blue-700">
            Tasks
          </a>
          <a href="#" class="text-sm font-semibold leading-6 text-gray-900 hover:text-blue-700">
            Collaborate
          </a>
        </div>
        <!-- right justified links (login, etc) -->
        <div class="hidden lg:flex lg:flex-1 lg:justify-end lg:gap-x-4">
          <%= if @current_user do %>
            <.link
              href={~p"/auth/users/settings"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              <%= @current_user.email %>
            </.link>
            <.link
              href={~p"/auth/users/log_out"}
              method="delete"
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Log out
            </.link>
          <% else %>
            <.link
              href={~p"/auth/users/register"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Register
            </.link>
            <.link
              href={~p"/auth/users/log_in"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Log in
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
            <a href="#" class="-m-1.5 p-1.5">
              <span class="sr-only">Taskboard</span>
              <img class="h-8 w-auto" src="https://tailwindui.com/img/logos/mark.svg?color=indigo&shade=600" alt="">
            </a>
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
              <div class="space-y-2 py-6">
                <a href="#" class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700">
                  Admin
                </a>
                <a href="#" class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700">
                  Tasks
                </a>
                <a href="#" class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700">
                  Collaborate
                </a>
              </div>
              <div class="space-y-2 py-6">
                <%= if @current_user do %>
                  <.link
                    href={~p"/auth/users/settings"}
                    class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700"
                  >
                    <%= @current_user.email %>
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

    <%= @inner_content %>

  </body>
</html>
```

When we look at a `dead` homepage: `http://localhost:4000/` and when we look at a `liveview` page: `http://localhost:4000/auth/users/settings` we will have our new reactive navbar!

### Add links and images to the navbar

Let's put a new custom icon in to the path: `priv/static/images/logo.png` - ideally an SVG, but for now I will just use a png.

Now we can update our logo and the url links with:

```html
        <div class="flex lg:flex-1">
          <.link href={~p"/"} class="-m-1.5 p-1.5" >
            <span class="sr-only">TaskBoard</span>
            <img
              class="h-8 w-auto"
              src={~p"/images/logo.png"}
              alt="logo"
            >
          </.link>
        </div>
```

This uses the internal route checking to ensure we are linking to our urls and images properly.

we can also link our SiteAdmin link in the menubar with:
```html
<a
    href={~p"/admin/projects"}
    class="text-sm font-semibold leading-6 text-gray-900 hover:text-blue-700">
  SiteAdmin
</a>
```

Now the navbar `root.html.heex` looks like:
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
    <header class="absolute inset-x-0 top-0 z-50">
      <nav class="mx-auto flex max-w-7xl items-center justify-between p-6 lg:px-8" aria-label="Global">
        <div class="flex lg:flex-1">
          <.link href={~p"/"} class="-m-1.5 p-1.5" >
            <span class="sr-only">TaskBoard</span>
            <img
              class="h-8 w-auto"
              src={~p"/images/logo.png"}
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
        <div class="hidden lg:flex lg:flex-1 lg:justify-end lg:gap-x-4">
          <%= if @current_user do %>
            <.link
              href={~p"/auth/users/settings"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              <%= @current_user.email %>
            </.link>
            <.link
              href={~p"/auth/users/log_out"}
              method="delete"
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Log out
            </.link>
          <% else %>
            <.link
              href={~p"/auth/users/register"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Register
            </.link>
            <.link
              href={~p"/auth/users/log_in"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Log in
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
            <a href="#" class="-m-1.5 p-1.5">
              <span class="sr-only">Taskboard</span>
              <img class="h-8 w-auto" src="https://tailwindui.com/img/logos/mark.svg?color=indigo&shade=600" alt="">
            </a>
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
              <div class="space-y-2 py-6">
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
              <div class="space-y-2 py-6">
                <%= if @current_user do %>
                  <.link
                    href={~p"/auth/users/settings"}
                    class="-mx-3 block rounded-lg px-3 py-2 text-base font-semibold leading-7 text-gray-900 hover:bg-gray-50 hover:text-blue-700"
                  >
                    <%= @current_user.email %>
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

    <%= @inner_content %>

  </body>
</html>
```

### make email a responsive dropdown with settings

Basic html with css classes (non-reactive)
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

Now all togehter:
```html
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
          <% else %>
            <.link
              href={~p"/auth/users/register"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Register
            </.link>
            <.link
              href={~p"/auth/users/log_in"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Log in
            </.link>
          <% end %>
```

now the root.html.heex should look like:
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
    <header class="absolute inset-x-0 top-0 z-50">
      <nav class="mx-auto flex max-w-7xl items-center justify-between p-6 lg:px-8" aria-label="Global">
        <div class="flex lg:flex-1">
          <!-- Logo -->
          <.link href={~p"/"} class="-m-1.5 p-1.5" >
            <span class="sr-only">TaskBoard</span>
            <img
              class="h-8 w-auto"
              src={~p"/images/logo.png"}
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
        <div class="hidden lg:flex lg:flex-1 lg:justify-end lg:gap-x-4">
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
          <% else %>
            <.link
              href={~p"/auth/users/register"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Register
            </.link>
            <.link
              href={~p"/auth/users/log_in"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Log in
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
            <!-- Logo -->
            <.link href={~p"/"} class="-m-1.5 p-1.5" >
              <span class="sr-only">TaskBoard</span>
              <img
                class="h-8 w-auto"
                src={~p"/images/logo.png"}
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

    <%= @inner_content %>

  </body>
</html>
```

### Sticky navbbar

Resources:
* https://stackoverflow.com/questions/60169463/tailwindcss-fixed-navbar

```html
<header class="sticky top-0 z-50"></header>
```

but now transparent - lets add a background color

### Navbar color (non-transparent)

https://tailwindcss.com/docs/background-color

```
bg-slate-100
```


now Navbar (in `root.html.heex` should look like:
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
    <header class="absolute sticky inset-x-0 top-0 z-50 bg-slate-100">
      <nav class="mx-auto flex max-w-7xl items-center justify-between p-6 lg:px-8" aria-label="Global">
        <div class="flex lg:flex-1">
          <!-- Logo -->
          <.link href={~p"/"} class="-m-1.5 p-1.5" >
            <span class="sr-only">TaskBoard</span>
            <img
              class="h-8 w-auto"
              src={~p"/images/logo.png"}
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
        <div class="hidden lg:flex lg:flex-1 lg:justify-end lg:gap-x-4">
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
          <% else %>
            <.link
              href={~p"/auth/users/register"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Register
            </.link>
            <.link
              href={~p"/auth/users/log_in"}
              class="text-[0.8125rem] leading-6 text-zinc-900 font-semibold hover:text-blue-700"
            >
              Log in
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
            <!-- Logo -->
            <.link href={~p"/"} class="-m-1.5 p-1.5" >
              <span class="sr-only">TaskBoard</span>
              <img
                class="h-8 w-auto"
                src={~p"/images/logo.png"}
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

    <%= @inner_content %>

  </body>
</html>
```

### navbar as a component

## add attributes to the user

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

### add user and new attributes to the admin page

### Add name to the settings page

### add name to the signup page

### add migration backfill to user name?

## restrict Admin page to the is_admin users

Resources:
* https://www.youtube.com/watch?v=6TlcVk-1Tpc
* https://www.knowthen.com/implementing-authorization-using-role-based-access-control-rbac-in-phoenix-web-applications/
* https://peterullrich.com/build-a-rap-for-phoenix-part-1
* https://www.knowthen.com/elixir-and-phoenix-for-beginners/
* https://elixirforum.com/t/phx-gen-auth-and-role-based-authentication/49428/7
* https://elixirforum.com/t/protecting-link-creation-functions-as-well-as-resource-controllers/1336
* https://stackoverflow.com/questions/26055501/how-to-restrict-access-to-certain-routes-in-phoenix
* https://blog.appsignal.com/2021/11/02/authorization-and-policy-scopes-for-phoenix-apps.html

## add the project owner area

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
