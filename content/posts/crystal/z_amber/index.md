---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Amber Basics"
subtitle: "nested pre-loads and nested resources"
summary: "Exploring more nested Reources with Phoenix"
authors: ["btihen"]
tags: ["crystal", "Relationships", "Templates", "Nested Preloading", "Nested Resources", "Render Foriegn Views", "User Error Handling"]
categories: ["Code", "Crystal Language", "Amber Framework"]
date: 2020-05-06T09:43:51+02:00
lastmod: 2020-05-06T09:43:51+02:00
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

This article builds on the existing article: https://btihen.me/post_tech/phoenix_1_5_blog_intro/ and adds nested relationships and has_many_through.

## now lets create comments (a has many through for users)

we will use `mix phx.gen.context` this time since we will use the posts page to add comments.  We will use the context generator since we don't need any views or templates generated.  Answer `Y` to the question about the context already existing.  We could create to API files within the Context before the one file gets too large, but we will skip that.

```
mix phx.gen.context Blogs Comment comments message:text post_id:references:posts  user_id:references:users
```

## Update Relationships

We need to create the relationships and update the migration to delete comments when post is deleted:

Now lets create the relationship between posts and comments:
```
# lib/feenix_intro/blogs/comment.ex
efmodule FeenixIntro.Blogs.Comment do
  use Ecto.Schema
  import Ecto.Changeset
  alias FeenixIntro.Blogs.Post
  alias FeenixIntro.Accounts.User

  @required_fields [:user_id, :post_id, :message]

  schema "comments" do
    # remove these
    # field :post_id, :id
    # field :user_id, :id
    # add these:
    belongs_to(:user, User)
    belongs_to(:post, Post)

    field :message, :string

    timestamps()
  end

  @doc false
  def changeset(comment, attrs) do
    comment
    |> cast(attrs, @required_fields)
    |> validate_required(@required_fields)
  end
```

Now lets update posts relationship to comments:
```
# lib/feenix_intro/blogs/post.ex
  # ...
  alias FeenixIntro.Blogs.Comment
  # ...
  schema "posts" do
    # ...
    # add this
    has_many(:comments, Comment)
  # ...
```

We could do the same `has_many` relationship with users - but its not needed.  It is unlikely we would want to look-up all a user's comments outside the context of a Blog.

## Update Migration to delete sub-resource when top-resource is deleted

To create the rails equivalent of dependent_delete we change the migration to the following:
```
# priv/repo/migrations/20200704161651_create_comments.exs
      # ...
      # replce
      # add :post_id, references(:posts, on_delete: :nothing)
      # add :user_id, references(:users, on_delete: :nothing)
      # with
      add :post_id, references(:posts, on_delete: :delete_all), null: false
      add :user_id, references(:users, on_delete: :delete_all), null: false
      # ...
```
Now we should be able to migrate:
```
mix ecto.migrate
```

## Testing

**Start simple with the seed file**

Lets add a comment to our prebuild posts:

```
# priv/repo/seeds.exs
# ...
# add the alias to keep things short
alias FeenixIntro.Blogs.Comment

# ...
# this ensures all we have all the correct fields:
Repo.insert!(%Comment{user_id: dog.id, post_id: post1.id, message: "woof" })

# this also checks the relationships
post2
|> Ecto.build_assoc(:comments)
|> Comment.changeset(%{user_id: dog.id, post_id: post2.id, message: "BARK" })
|> Repo.insert!()
```

Lets run the seed and see if all is working:
```
mix run priv/repo/seeds.exs
```

Nice lets make a quick git snapshot before we work on the html aspects
```
git add .
git commit -m "Comments added as a resource and relationship to Posts established"
```

## Preload comments within get_post

To show the comments within a post we will need to preload the comments -- this is done by adding `Repo.preload(:comments)` to our function: `def get_post!(id)` -- however, we will also want to display the comment's author -- so we need to do a nested preload with: `Repo.preload([comments: [:user]])`

So now this function looks like:
```
# lib/feenix_intro/blogs.ex
def get_post!(id) do
    Post
    |> Repo.get!(id)
    |> Repo.preload(:user)
    |> Repo.preload([comments: [:user]])
  end
```

This can actually be shortened to (this will be helpful later):
```
lib/feenix_intro/blogs.ex
def get_post!(id) do
    Post
    |> Repo.get!(id)
    |> Repo.preload([:user, comments: [:user]])
  end
```

## Display the comments within the Post show

Now that we have updated the get_post! to preload comments we can display the comments too by adding to the end of our post's - show template:

```
# lib/feenix_intro_web/templates/post/show.html.eex

# ...
<table>
  <thead>
    <tr>
      <th>Comment Author</th>
      <th>Message</th>
    </tr>
  </thead>
  <tbody>
    <%= for comment <- @post.comments do %>
    <tr>
      <td><%= comment.user.name %></td>
      <td><%= comment.message %></td>
    </tr>
    <% end %>
  </tbody>
</table>

<span><%= link "Edit", to: Routes.post_path(@conn, :edit, @post) %></span>
<span><%= link "Back", to: Routes.post_path(@conn, :index) %></span>
```

Start the server `mix phx.server` and be sure this works

Assuming it works, lets commit:
```
git add .
git commit -m "display comments and comment author on post show page"
```

## Creating Comments (as a nested resource)

Since we have added comments within the Blogs context and they are associated with a post - it makes sense to create and display comments as a nested resource.  To set this up lets change our routes file:

```
# lib/feenix_intro_web/router.ex
# ...
  scope "/", FeenixIntroWeb do
    pipe_through :browser

    get "/", PageController, :index
    resources "/users", UserController

    # replace this line:
    # resources "/posts", PostController
    # with:
    resources "/posts", PostController do
      resources "/comments", CommentController, only: [:create]
    end
  end
  # ...
```

This means we will be able to create a comment only within the context of an existing post (seems reasonable) -- more actions can be added later such as `edit` or `delete` possibly.

This also means we need to display our comments within the context of existing posts (the best place for this is the `show` - where all the details of the post are shown).

Let's create the controller we just defined - we will need to make a new file:

```
# lib/feenix_intro_web/controllers/comment_controller.ex
defmodule FeenixIntroWeb.CommentController do
  use FeenixIntroWeb, :controller

  alias FeenixIntro.Blogs

  def create(conn, %{"post_id" => post_id, "comment" => comment_params}) do
    # define the post we are nested within
    post = Blogs.get_post!(post_id)

    # create our new comment and handle (success or failure)
    case Blogs.create_comment(post, comment_params) do
      {:ok, _comment} ->
        conn
        |> put_flash(:info, "Comment created")
        |> redirect(to: Routes.post_path(conn, :show, post))

      # TODO: return to form and show errors
      {:error, _changeset} ->
        conn
        |> put_flash(:error, "Comment creation failed")
        |> redirect(to: Routes.post_path(conn, :show, post))
    end
  end

end
```

Note: at the moment we don't handle errors, and allow those to be fixed.  We will get to that in a second step.

We need to update the function `create_comment` in order to work as a nested resource:
```
#  @doc """
  Creates a comment.

  ## Examples
      # also update our function docs
      # replace
      # iex> create_comment(%{field: value})
      # with
      iex> create_comment(post, %{field: value})
      {:ok, %Comment{}}

      # replace:
      # iex> create_comment(%{field: bad_value})
      # with:
      iex> create_comment(post, %{field: bad_value})
      {:error, %Ecto.Changeset{}}

  """
  # replace
  # def create_comment(attrs \\ %{}) do
  #   %Comment{}
  #   |> Comment.changeset(attrs)
  #   |> Repo.insert()
  # end

  # with (this uses the passed in post and creates an association with the new comment)
  def create_comment(%Post{} = post, attrs \\ %{}) do
    post
    |> Ecto.build_assoc(:comments)
    |> Comment.changeset(attrs)
    |> Repo.insert()
  end
```

In order to create a new Comment **form** the `show` function will need to borrow from a typical `new` function and send and empty struct (changeset) for the form -- lets start by updating the PostController show function:
```
# lib/feenix_intro_web/controllers/post_controller.ex
  # ...
  alias FeenixIntro.Blogs.Comment

  def show(conn, %{"id" => id}) do
    post = Blogs.get_post!(id)
    users = Accounts.list_users()
    # replace:
    # render(conn, "show.html", post: post, users: users)

    # with: This allows us to add comments on the Post show form!
    comment_changeset = Blogs.change_comment(%Comment{})
    render(conn, "show.html", post: post, users: users,
                              comment_changeset: comment_changeset)
  end
```

Now that we have an empty changeset for the form - we can add the form to the show page with:
```
# lib/feenix_intro_web/templates/post/show.html.eex
# ...
<h3>Add a Comment</h3>
<%= form_for @comment_changeset, Routes.post_comment_path(@conn, :create, @post), fn form -> %>

  <%= label form, "Author" %>
  <%= select form, :user_id, Enum.map(@users, &{&1.name, &1.id}) %>
  <%= error_tag form, :user %>

  <%= label form, :message %>
  <%= textarea form, :message %>
  <%= error_tag form, :message %>

  <div>
    <%= submit "Save"%>
  </div>
<% end %>
# ...
```

Let's try this out with: `mix phx.server`

assuming all works as expected let's make another git commit:
```
git add .
git commit -m "comment creation as a nested resource within posts"
```

## Handle Input Errors

Prevent empty strings:

* https://stackoverflow.com/questions/32784008/phoenix-render-template-of-other-folder

lets add a minimum message legth to comments:
```
# lib/feenix_intro/blogs/comment.ex
  def changeset(comment, attrs) do
    comment
    |> cast(attrs, @required_fields)
    |> validate_required(@required_fields)
    |> validate_length(:message, min: 3)
  end

```

Now, change the controller to prep the data just like a post `show` and send the changeset - with the errors. `|> put_view(FeenixIntroWeb.PostView)` is how we redirect to other external views as of Phoenix 1.5.1:
```
# lib/feenix_intro_web/controllers/comment_controller.ex
  # add the alias
  alias FeenixIntro.Accounts

  # ...

  def create(conn, %{"post_id" => post_id, "comment" => comment_params}) do
    # ...

      # replace:
      # {:error, _changeset} ->
      #   conn
      #   |> put_flash(:error, "Comment creation failed, please fix the errors")
      #   |> redirect(to: Routes.post_path(conn, :show, post))

      # with:
      {:error, %Ecto.Changeset{} = changeset} ->
        users = Accounts.list_users()
        conn
        |> put_flash(:error, "Comment creation failed, please fix the errors")
        |> put_view(FeenixIntroWeb.PostView)   # as of Phoenix 1.5.1
        |> render("show.html", post: post, users: users, comment_changeset: changeset)
# ...
```

Assuming this works make a new git commit:
```
git add .
git commit -m "handle comment creation errors"
```

## Flexible preloading

You may have noticed the pre-loading is hard-coded -- in this case it is ok, but might not always be good.  Here is a flexible alternative:

We can update / replace the following functions with the following:
```
# lib/feenix_intro/blogs.ex
  def list_posts(opts \\ [:user]) do
    preloads = Keyword.get(opts, :preloads, [])
    Post
    |> Repo.all()
    |> Repo.preload(preloads)
  end

  def get_post!(id, opts \\ [:user, comments: [:user]]) do
    preloads = Keyword.get(opts, :preloads, [])
    Post
    |> Repo.get!(id)
    |> Repo.preload(preloads)
  end

  def get_comment!(id, opts \\ [:user]) do
    preloads = Keyword.get(opts, :preloads, [])
    Comment
    |> Repo.get!(id)
    |> Repo.preload(preloads)
  end
```

And now we can change our show post controller to look like - so that we can use this flexibility:
```
# lib/feenix_intro_web/controllers/post_controller.ex
  # ...

  def index(conn, _params) do
    # posts = Blogs.list_posts()
    preloads = [:user]
    posts = Blogs.list_posts(preloads: preloads)
    render(conn, "index.html", posts: posts)
  end

  def new(conn, _params) do
    users = Accounts.list_users()
    changeset = Blogs.change_post(%Post{})
    render(conn, "new.html", changeset: changeset, users: users)
  end

  # ...

  def show(conn, %{"id" => id}) do
    # post = Blogs.get_post!(id)
    preloads = [:user, comments: [:user]]
    post = Blogs.get_post!(id, preloads: preloads)
    users = Accounts.list_users()
    # This allows us to add comments on the Post show form!
    comment_changeset = Blogs.change_comment(%Comment{})
    render(conn, "show.html", post: post,
                              users: users,
                              comment_changeset: comment_changeset)
  end
```

Now we have the flexibilty to preload or not depending on what we want to do,
