---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.5 LiveView & PubSub Basics"
subtitle: "Create a Dynamic Web GUI without JavaScript!"
summary: "Create a simple web counter app to learn how Phoenix 1.5 LiveView works.  Phoenix LiveView allows dynamic webpages with fast update times -- without JavaScript."
authors: ["btihen"]
tags: ["Elixir", "LiveView", "PubSub", "Interactive"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2020-05-10T17:01:53+02:00
lastmod: 2020-08-07T01:01:53+02:00
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
I have been watching Phoenix and Elixir for a while, and the idea of writing dynamic Web Applications without needing a ton of JavaScript is very interesting.  I recently saw this video by Chris McCord:

* https://www.youtube.com/watch?v=MZvmYaFkNJI&feature=youtu.be

which is very cool.  I wanted to learn more and found this Phoenix 1.4 tutorial:

* https://www.youtube.com/watch?v=2bipVjOcvdI
* https://dennisbeatty.com/2019/03/19/how-to-create-a-counter-with-phoenix-live-view.html

and decided to translate that into Phoenix 1.5. This is what follows.

**NOTE:** Since I am just learning the Phoenix Framework and will need to refer to this for my self to remember how to do basic things -- I've documented every little detail.

### Step 0 - setup environment

Setup environment & newest version of elixir:

```
exenv install 1.10.3
exenv global
exenv local 1.10.3
```

Install the 1.5.1 phx_new generator:

`mix archive.install hex phx_new 1.5.1`

### Step 1: Create a Phoenix Project with LiveView

Create the project (notice the `--live` - that enables LiveView, `--no-ecto` - keeps the project smaller since we won't be persisting any data):

`mix phx.new counter --no-ecto --live`

enter project and create init commit:
```
cd counter
git init && git add -A && git commit -m "init"
```

### Step 2 - simple counter page using LiveView

Make a counter_live folder & an index.ex file:
```elixir
mkdir lib/counter_web/live/counter_live
touch lib/counter_web/live/counter_live/index.ex
cat <<EOF > lib/counter_web/live/counter_live/index.ex
# lib/counter_web/live/counter_live/index.ex
defmodule CounterWeb.CounterLive.Index do
  use CounterWeb, :live_view

  # since we don't have a db to pull from we initialize on mount
  @impl true
  def mount(_params, _session, socket) do
    {:ok, assign(socket, :val, 0)}
  end

  def render(assigns) do
    ~L"""
    <h1>Live Counter</h1>
    <p>
      <b>Here is a great complex page</b>
    </p>

    <div>
      <h2>The count is: <%= @val %></h2>
      <button phx-click="dec">-</button>
      <button phx-click="inc">+</button>
    </div>
    <div>
      <button phx-click="clear">clear</button>
    </div>

    <p>
      <i>even more awesome content</i>
    </p>
    """
  end

  # event handler for <button phx-click="inc">
  def handle_event("inc", _, socket) do
    {:noreply, update(socket, :val, &(&1 + 1))}
  end

  # event handler for <button phx-click="dec">
  def handle_event("dec", _, socket) do
    {:noreply, update(socket, :val, &(&1 - 1))}
  end

  # event handler for <button phx-click="clear">
  def handle_event("clear", _, socket) do
    {:noreply, update(socket, :val, &(&1 - &1))}
    # {:noreply, update(socket, :val, 0)} # very slow - why?
  end

end
```

Now update the routers (so we can get to the new webpage -- now our app should work:
```elixir
  scope "/", CounterWeb do
    pipe_through :browser

    # live "/", PageLive, :index        # remove this line
    live "/", CounterLive.Index, :index # add this line
  end
```

Start pheonix:

`mix phx.server`

Go to:

`localhost:4000`

You should now see the website and the counter should function

Assuming all is good, I'll take a git snapshot:
```
git add .
git commit -m "counter with live update"
```

### Step 3 - Running tests

In order to run the tests we type:

```
mix test
```

We see that PageLive test fails.  This is because we replaced this behavior with `CounterLive`

To fix this we will create a **CounterLive** test and delete **PageLive** test.
```elixir
rm test/counter_web/live/page_live_text.exs
touch test/counter_web/live/counter_live_text.exs
cat <<EOF > test/counter_web/live/counter_live_text.exs
# test/counter_web/live/counter_live_text.exs
defmodule CounterWeb.CounterLiveTest do
  use CounterWeb.ConnCase

  import CounterWeb.CounterLive.Index

  test "disconnected and connected render", %{conn: conn} do
    {:ok, page_live, disconnected_html} = live(conn, "/")
    assert disconnected_html =~ "Live Counter"
    assert render(page_live) =~ "Live Counter"
  end

end
```

Now we can test again: `mix test`

Now that works, lets take another git snapshot:
```
git add .
git commit -m "counter with live update"
```

### Step 4 -- LiveView Templates

Create a template file (helpful for complex html pages, but simple to create):

`touch lib/counter_web/live/counter_live/index.html.leex`

Now just copy the html (from the render method into this file):

{{< highlight "linenos=table,linenostart=1" >}}
# lib/counter_web/live/counter_live/index.html.leex
<h1>Live Counter</h1>
<p>
  <b>Here is a great complex page</b>
</p>

<div>
  <h2>The count is: <%= @val %></h2>
  <button phx-click="dec">-</button>
  <button phx-click="inc">+</button>
</div>
<div>
   <button phx-click="clear">clear</button>
</div>

<p>
  <i>even more awesome content</i>
</p>
{{< / highlight >}}

Now point `lib/counter_web/live/counter_live/index.ex` to this file by replacing render with an apply command:

{{< highlight elixir "linenos=table,linenostart=1" >}}
  # add this new function
  defp apply_action(socket, :index, _params) do
    socket
  end
  # remove this funtion
  # def render(assigns) do
  #  ~L"""
  #   ...
  #   """
  # end
{{< / highlight >}}

**NOTE:** `apply_action` understands the **rest** verbs such as `:new`, `:show` etc.

Now try the app again and it should still work!

Assuming it still works, I'll take another git snapshot:

```
git add .
git commit -m "counter using a template"
```


### Step 5 - Reusable Components (& isolation)

This allows complex components to be **reused** within multiple templates and **isolation** to keep one's mental scope minimal.

Create a file for the component:

`touch lib/counter_web/live/counter_live/counter_component.ex`

Move the dynamic html and it's associated functions into this file, it's important to import the live_components into this file using:

`use CounterWeb, :live_component`

In order to encapsulate the events into the component we will also move the event handlers into the component file.

So this file will now look like:
{{< highlight elixir "linenos=table,linenostart=1" >}}
# lib/counter_web/live/counter_live/counter_component.ex
defmodule CounterWeb.CounterLive.CounterComponent do
  use CounterWeb, :live_component

  def render(assigns) do
    ~L"""
    <div>
      <h2>The count is: <%= @val %></h2>
      <button phx-click="dec" phx-target="<%= @myself %>">-</button>
      <button phx-click="inc" phx-target="<%= @myself %>">+</button>
    </div>
    <div>
      <button phx-click="clear" phx-target="<%= @myself %>">clear</button>
    </div>
    """
  end

  def handle_event("inc", _, socket) do
    {:noreply, update(socket, :val, &(&1 + 1))}
  end

  def handle_event("dec", _, socket) do
    {:noreply, update(socket, :val, &(&1 - 1))}
  end

  def handle_event("clear", _, socket) do
    # {:noreply, update(socket, :val, 0)} # very slow - why?
    {:noreply, update(socket, :val, &(&1 - &1))}
  end

end
{{< / highlight >}}

Notice the button tags are slightly more complex

`<button phx-click="dec" phx-target="<%= @myself %>">`

the **@myself** basically informs the event that the handler is within the component.

Now update the live template to point at the component using:

`<%= live_component @socket, CounterWeb.CounterLive.CounterComponent, id: 0, val: @val %>`

Also note we need to pass the @val value into the component using:

`id: 0, val: @val`

its a little wierd, but we need to pass an **id** even if there is no ecto backed record.

Now the template file looks like a normal template file again (focused on formating):

{{< highlight elixir "linenos=table,linenostart=1" >}}
# lib/counter_web/live/counter_live/index.ex
defmodule CounterWeb.CounterLive.Index do
  use CounterWeb, :live_view

  # since we don't have a db to pull from we initialize on mount
  @impl true
  def mount(_params, _session, socket) do
    {:ok, assign(socket, :val, 0)}
  end

  def render(assigns) do
    ~L"""
    <h1>Live Counter</h1>
    <p>
      <b>Here is a great complex page</b>
    </p>

    <%= live_component @socket, CounterWeb.CounterLive.CounterComponent, id: 0, val: @val %>

    <p>
      <i>even more awesome content</i>
    </p>
    """
  end

end
{{< / highlight >}}

Lets check that this still works.

Assuming it still works, I'll make one last git snapshot:

```
git add .
git commit -m "live pages using isolated components - like JS does"
```
