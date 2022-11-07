---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix LiveView - Simple Real-Time SPA"
subtitle: "Phoenix, Elixir, Single Page App Demo"
summary: "Create a dynamic SPA staying mostly in the Pheonix/Elixir mindset"
authors: ["btihen"]
tags: ["Elixir", "LiveView", "SPA"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2021-04-10T17:01:53+02:00
lastmod: 2021-08-07T01:01:53+02:00
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
Here is a quick example of how to create a very simple "real-time"-"single-page-app" using phoenix-liveview.  This provides the same functionality to as [Realtime Rails with Hotwire](post_ruby_rails/rails_6_1_hotwire_simple_realtime) - in order to compare.

The repo can be found here: https://github.com/btihen/live-tweets

## create / config a project

First we will creat the folder / project location
```bash
mkdir tweets
```

Now we will tell it which software to use:
```bash
touch tweets/.tool-versions
cat <<EOF >>tweets/.tool-versions
erlang 23.3.1
elixir 1.11.4-otp-23
nodejs lts-Fermium
Postgres 13.2
EOF
```

## Create a new Phoenix Project

https://carlyleec.medium.com/create-an-elixir-phoenix-app-with-asdf-e918649b4d58

Now you can simply do:
```
mix phx.new tweets --live
```
You will now get the message:
```
The directory /Users/btihen/Dropbox/devel/marpori/tweets already exists. Are you sure you want to continue? [Yn]
```
Say `Y` yes.
Say `Y` yes again when you see:
```
Fetch and install dependencies? [Yn]
```
This can take a few minutes - when done, enter the directory and setup.
```
cd tweets
```
Adjust the DB settings as needed in: `config/dev.exs`

Create the database and lets see if default tests work and we get the start page.
```
mix ecto.create
mix test
mix phx.server
```

assuming all is good lets snapshot:
```
git init
git add .
git commit -m "initial setup commit"
```

This code commit can be seen at: https://github.com/btihen/live-tweets/commit/2eb9016371db3210eaf3a1cb35e4066e3b67bdbe

## create our tweet model

Create this with the generator (notice we are using `mix phx.gen.live` not `mix phx.gen.html`):
```
mix phx.gen.live Messages Post posts body:text
```

Change migration to require data - add `null: false` to our field so it now looks like:
```elixir
# priv/repo/migrations/20210418084643_create_posts.exs
defmodule Tweets.Repo.Migrations.CreatePosts do
  use Ecto.Migration

  def change do
    create table(:posts) do
      add :body, :text, null: false

      timestamps()
    end

  end
end
```

Now lets update the routes as described by the generator - in `lib/tweets_web/router.ex` so the section that looks like:
```elixir
  scope "/", TweetsWeb do
    pipe_through :browser

    live "/", PageLive, :index
  end
```

should be change to:
```elixir
  scope "/", TweetsWeb do
    pipe_through :browser

    # live "/", PageLive, :index
    live "/", PostLive.Index, :index
    live "/posts", PostLive.Index, :index
    live "/posts/new", PostLive.Index, :new
    live "/posts/:id/edit", PostLive.Index, :edit

    live "/posts/:id", PostLive.Show, :show
    live "/posts/:id/show/edit", PostLive.Show, :edit
  end
```

Now check our field `body` is required in validations -- in our changeset.  We see `validate_required([:body])` in the file: `lib/tweets/messages/post.ex` - so we are all set.
```elixir
  def changeset(post, attrs) do
    post
    |> cast(attrs, [:body])
    |> validate_required([:body])
  end
```

So it time to migrate & test:
```bash
mix ecto.migrate
mix test
```

Hmm - the tests generator and html use different html standards: to make the tests pass test that phoenix returns `can&#39;t be blank` instead of `can&apos;t be blank` in `test/tweets_web/live/post_live_test.exs`

also change: `"Welcome to Phoenix!"` to `"Listing Posts"` in `test/tweets_web/live/page_live_test.exs`

Now lets see how our new SPA works:
```
mix phx.server
```

It works, but we want to list the most recent tweets at the top of the page -- let's investigate -- open `lib/tweets_web/live/post_live/index.ex` we see in the `mount` command:
```elixir
# lib/tweets_web/live/post_live/index.ex
  def mount(_params, _session, socket) do
    {:ok, assign(socket, :posts, list_posts())}
  end
```
It uses `list_posts()` to get the list - so let's change this function.

Open: `lib/tweets/messages.ex` and change:
```elixir
# lib/tweets/messages.ex
  def list_posts do
    Repo.all(Post)
  end
```
to
```elixir
# lib/tweets/messages.ex
 def list_posts do
    Post
      |> order_by(desc: :inserted_at)
      |> Repo.all
  end
```

Cool now our SPA works like we want -- but it isn't real-time between two users / browsers.


This code can be seen at: https://github.com/btihen/live-tweets/commit/3f432d7c06d974f9c2349937a35e391dedeb2ad6

## Broadcast changes with Pubsub

Phoenix uses Websockets to do `real-time` communication.  In our "context" we will create our `channel` - the pipeline that the socket uses to send information back and forth to various "subscribers" - viewers of our page.

### Setup the "Messages" Channel

We go into: `lib/tweets/messages.ex` and at the top of the file add the Broadcast Setup:
```elixir
# lib/tweets/messages.ex
defmodule Tweets.Messages do
  @moduledoc """
  The Messages context.
  """

  import Ecto.Query, warn: false
  alias Tweets.Repo
  alias Tweets.Messages.Post

  # Setup Broadcasting
  @topic inspect(__MODULE__)

  def subscribe do
    Phoenix.PubSub.subscribe(Tweets.PubSub, @topic)
  end

  def notify_subscribers({:ok, post}, event) do
    posts = list_posts()
    Phoenix.PubSub.broadcast(Tweets.PubSub, @topic, {__MODULE__, event, posts})
    {:ok, post}
  end

  def notify_subscribers({:error, post}, event) do
    {:error, post}
  end
  # Setup Broadcasting
  ...
```

Lets quickly review this new code:

`@topic inspect(__MODULE__)`

makes @topic named `Tweets.Messages` - but if we change the module name it changes @topic too.

`subscribe` function - allows us to register our index page with channel created automatically by LiveView.

We have two `notify_subscribers` because we will call these after we do our DB actions - writing to the DB could fail or succeed.  If we have success then we will update all subscribers and the last line tuple passes the results of the interaction back to the actual user.   (Of course we don't need to notify when the DB transaction fails, we only need to pass the message back to the user).

### Subscribing to the 'Messages' Channel

Now that we have `notify_subscribers` that broadcasts `Phoenix.PubSub.broadcast(Tweets.PubSub, @topic, {__MODULE__, event, posts})` we need a way to subscribe to this channel and receive these messages in all our index pages.

```elixir
# lib/tweets_web/live/post_live/index.ex
defmodule TweetsWeb.PostLive.Index do
  use TweetsWeb, :live_view

  alias Tweets.Messages
  alias Tweets.Messages.Post

  @impl true
  def mount(_params, _session, socket) do
    # register with the channel if connection to LiveView
    if connected?(socket), do: Messages.subscribe()
    {:ok, assign(socket, :posts, list_posts())}
  end

  @impl true
  def handle_info({Messages, "posts-changed", posts}, socket) do
    socket = assign(socket, :posts, posts)
    {:noreply, socket}
  end
  ...
```

The important changes are to **subscribe to the channel** we we have subscribed to our Websocket we do this in the `mount` function with `if connected?(socket), do: Messages.subscribe()`

Now we need a way to **recieve information from the channel** this is done with the `handle_info` function - so we will simply take the new list of posts and update the socket and index will take care of the rest -- automatically!

### Sending Messages to the Channel

So now to activate our changes - we need to send to the channel via `notify_subscribers` when we successfully change something in the Messages "post" context.  To do this we will make small changes to the create_post, update_post and delete_post functions.  We will add `notify_subscribers({status, post}, "posts-changed")` to the end of each function.  Since we only defined one event `"posts-changed"` in our index page `handle_info` function -- we will hard-code that into our `notify_subscribers` call

So our simple DB calls in Messages currently looks like:
```elixir
# lib/tweets/messages.ex
  def create_post(attrs \\ %{}) do
    %Post{}
      |> Post.changeset(attrs)
      |> Repo.insert()
  end

  def update_post(%Post{} = post, attrs) do
    post
      |> Post.changeset(attrs)
      |> Repo.update()
  end

  def delete_post(%Post{} = post) do
    Repo.delete(post)
  end
```

Now becomes:
```elixir
# lib/tweets/messages.ex
  def create_post(attrs \\ %{}) do
    {status, post} = %Post{}
                      |> Post.changeset(attrs)
                      |> Repo.insert()
    notify_subscribers({status, post}, "posts-changed")
  end

  def update_post(%Post{} = post, attrs) do
    {status, post} = post
                      |> Post.changeset(attrs)
                      |> Repo.update()
    notify_subscribers({status, post}, "posts-changed")
  end

  def delete_post(%Post{} = post) do
    {status, post} = post
                      |> Repo.delete()
    notify_subscribers({status, post}, "posts-changed")
  end
```

Note: in-order to pass the DB transaction information back to the user, we need to capture that information with the tuple: `{status, post}` - which notify_subscribers will pass back and will be returned to the user - the returns values will be either `{:ok, post}` or `{:error, post_changeset}`

Let's be sure we didn't break anything and run our tests again:
```
mix test
```
Ideally, all is still good so lets try our updated app now:
```
mix phx.server
```
Now any changes we make should be seen all users.

Cool, lets snapshot this.

```
git checkout -b liveview_spa_broadcast_with_pubsub
git add .
git commit -m "add realtime broadcast to all users"
```

This code can be seen at: https://github.com/btihen/live-tweets/commit/32c179e05cae68c5a2a6d49f54bf5a8dcf4d4dac

## Summary

In my mind this is far simpler to setup as a single page app - using the LiveView generator and a little more work to add broadcasting than in Rails.  Converting a Standard Phoenix HTML page to LiveView however is considerably more difficult than Converting a Standard Rails page to Hotwire.  I also find adding advanced features much more straight-forward in LiveView - as you write the event_handlers in you liveview pages and it is very clear what is happening.  In rails you need to know what is happening without being able to see the code.  I also like that LiveView - when it can't find an event - you get lots of errors.  This is very helpful.
