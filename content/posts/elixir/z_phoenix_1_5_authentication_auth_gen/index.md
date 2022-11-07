---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.5 authentication with POW"
subtitle: "Phoenix, Elixir, TailwindCSS, AlpineJS, LiveView - PETAL Stack"
summary: "Create a modern Phoenix SPA with tremendous flexibility"
authors: ["btihen"]
tags: ["Elixir", "Authentication", "POW", "Email", "HogMail"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2021-05-06T01:01:53+02:00
lastmod: 2021-05-06T01:01:53+02:00
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

## Auth

Auth with auth.gen
- https://elixircasts.io/using-phx_gen_auth-for-phoenix-authentication
- https://experimentingwithcode.com/phoenix-authentication-with-phx-gen-auth-part-1/
- https://experimentingwithcode.com/phoenix-authentication-with-phx-gen-auth-part-2/
- https://fullstackphoenix.com/tutorials/combining-authentication-solutions-with-guardian-and-phx-gen-auth
- https://medium.com/swlh/how-to-swap-registration-flow-to-a-live-view-with-phx-gen-auth-4966f80b412e
- https://medium.com/swlh/how-to-part-2-swap-registration-flow-to-a-live-view-with-phx-gen-auth-multi-step-form-25371540fce1


Auth with PubSub
- https://curiosum.dev/blog/elixir-phoenix-liveview-messenger-part-3


Auth with POW
* https://www.youtube.com/watch?v=hnD0Z0LGMIk
* https://www.kabisa.nl/tech/real-world-phoenix-lets-auth-some-users/
* https://experimentingwithcode.com/phoenix-authentication-with-pow-part-1/
* https://experimentingwithcode.com/phoenix-authentication-with-pow-part-2/
* https://dev.to/oliverandrich/learn-elixir-and-phoenix-add-authentication-55kl
* https://www.kabisa.nl/tech/real-world-phoenix-sign-up-flow-spa-style-with-liveview/
* https://medium.com/@andreichernykh/phoenix-simple-authentication-authorization-in-step-by-step-tutorial-form-dc93ea350153


Auth with Email
- https://hex.pm/packages/bamboo
- https://hexdocs.pm/bamboo_smtp/readme.html
- https://elixircasts.io/sending-email-with-bamboo-part-1
- https://elixircasts.io/sending-email-with-bamboo-part-2
- https://devato.com/post/use-bamboo-to-send-email-in-phoenix

## Mail in Test Env

- https://github.com/mailhog/MailHog
- https://mailcatcher.me/


## Intro

Pow has the advantage that it updates security patches -- since its a well maintained library.

This repo can be found at: https://github.com/btihen/phoenix_1_5_pow_auth_config

Get the latest version from: https://hex.pm/packages/pow
```
{:pow, "~> 1.0"}
```

Install the dependency:
```
mix deps.get
```

Install POW:
```
mix pow.install
```

Lets verify all is good with the install:
```
mix deps.compile
mix help | grep pow
```

Now hopefully you see some new `pow` commands

## Configure Pow

There are three files you'll need to configure first before you can use Pow.

First, append this to `config/config.exs`:
```
config :fare, :pow,
  user: Fare.Users.User,
  repo: Fare.Repo
```

Next, add `Pow.Plug.Session` plug to `lib/fare_web/endpoint.ex` after `plug Plug.Session`:
```
defmodule FareWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :fare   # add this line HERE!

  # ...

  plug Plug.Session, @session_options
  plug Pow.Plug.Session, otp_app: :fare  # add this line HERE!
  plug FareWeb.Router
end
```

Last, update `lib/fare_web/router.ex` with the Pow routes - this first time we need to do a little extra config:
```
  pipeline :api do
    plug :accepts, ["json"]
  end
  scope "/", MyAppWeb do
    pipe_through [:browser, :protected]

    # Add your protected routes here
  end

  scope "/", MyAppWeb do
    pipe_through :browser

    live "/", PageLive, :index
  end
```

Should now look like:
```
  pipeline :api do
    plug :accepts, ["json"]
  end

  pipeline :protected do
    plug Pow.Plug.RequiredAuthentication,
          error_handler: Pow.Phoenix.PlugErrorHandler
  end

  scope "/" do
    pipe_through :browser

    pow_routes()
  end

  scope "/", MyAppWeb do
    pipe_through [:browser, :protected]

    # Add your protected routes here
    resources "/tasks", TaskController
  end

  scope "/", MyAppWeb do
    pipe_through :browser

    live "/", PageLive, :index
  end
```

Now lets check the routes - that all is well configured:
```
mix phx.routes | grep pow
```
Hopefully you see some new pow routes.


Now we can migrate to create our users table:
```
mix ecto.migrate
```

Now if we start phoenix:
```
mix phx.server
```

and open phoenix: `localhost:4000`

## POW user Links

https://experimentingwithcode.com/phoenix-authentication-with-pow-part-1/

Notice there is no menu option to login - lets build a simple signup/signin/logout link.

In root.html.eex find `<li><a href="https://hexdocs.pm/phoenix/overview.html">Get Started</a></li>` and we will replace it with:
```
            <%= if Pow.Plug.current_user(@conn) do %>
              <li>
                <%= link "#{@current_user.email}", to: Routes.pow_registration_path(@conn, :edit) %>
              </li>
              <li>
                <%= link "Sign-out", to: Routes.pow_session_path(@conn, :delete), method: :delete %>
              </li>
            <% else %>
              <li><%= link "Sign-in", to: Routes.pow_session_path(@conn, :new) %></li>
              <li><%= link "Register", to: Routes.pow_registration_path(@conn, :new) %></li>
            <% end %>
```

Now reload and try it out:

1. you should be able to register
2. sign-out
3. sign in


## Customizable Login pages

Generate the pages to customize with:
```
mix pow.phoenix.gen.templates
```

now be sure to change the config in `config/confix.ex` from:
```
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: Fare.Repo
```
to:
```
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo,
  web_module: MyAppWeb
```
Without updating the config the newly generated pages won't be used!

The new templates to modify are found in:
* `lib/fare_web/templates/pow/registration` &
* `lib/fare_web/templates/pow/session`

Now make a small change to the pages to ensure all works.

## Create a restricted user page

https://www.youtube.com/watch?v=hnD0Z0LGMIk
https://experimentingwithcode.com/phoenix-authentication-with-pow-part-1/


Create a normal html page first:

```
mix phx.gen.html Tasks Task tasks description:string completed:boolean
```

BE SURE TO PUT the new route in the `protected` area of the routes file:
```
# lib/fare_web/router.ex
  scope "/", MyAppWeb do
    pipe_through [:browser, :protected]

    # Add your protected routes here
    resources "/tasks", TaskController
  end
```

Now of course run the migration:
```
mix ecto.migrate
```

now `/tasks` should only be availble to signed in users.  Be sure you are logged out and cannot get to the `/tasks` route (and infact are redirected to sign-in). And once logged in the page works as expected.

## Extensions

### Persistent Login Sessions (Remember me)

https://experimentingwithcode.com/phoenix-authentication-with-pow-part-2/

Currently every time the user closes the browser they are logged out - the login cookie doesn't persist - most users would like the option to change this - with a `remember me` option.

in `config/config.exs` change the `:pow` config to look like:
```
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo,
  web_module: MyAppWeb,
  # add the following two lines
  extensions: [PowPersistentSession],
  controller_callbacks: Pow.Extension.Phoenix.ControllerCallbacks
```

in `/lib/my_app_web/endpoint.ex` we need to add the persistent cookie setting immediately after the `Pow.Plug.Session` plug and before the routing `MyAppWeb.Router` plug -- now the end of the endpoint file should look like:
```
  # enable Pow session based authentication
  plug Pow.Plug.Session, otp_app: :warehouse
  # enable Pow persistent sessions
  plug PowPersistentSession.Plug.Cookie
  # routing plug
  plug MyAppWeb.Router
end
```

just above the login button on the `sign-in` page add the following check-box:
```
# lib/fare_web/templates/pow/session/new.html.eex
  <%= label f, :persistent_session, "Remember me" %>
  <%= checkbox f, :persistent_session %>

  <div>
    <%= submit "Sign in" %>
  </div>
```

restart Phoenix with: `mix phx.server` and now you should be able to close your browser and re-open the link and stay logged in if the `remember-me` is clicked.


## After Logout - go to Landing Page (After Hook Routing)

https://experimentingwithcode.com/phoenix-authentication-with-pow-part-2/

One little annoying thing is that when we logout we go to the sign-in page instead of the landing page.  We can fix that by adding a call_back_route - you can find all the callback routes at: https://github.com/danschultzer/pow/blob/master/lib/pow/phoenix/routes.ex - we will use: the `after_sign_out_path` callback.

To do this we will make a new `pow.routes` file:
```
touch lib/warehouse_web/pow/routes.ex
```

Add the following contents:
```
cat << EOF> lib/my_app_web/pow/routes.ex
defmodule MyAppWeb.Pow.Routes do
  use Pow.Phoenix.Routes
  alias MyAppWeb.Router.Helpers, as: Routes

  def after_sign_out_path(conn), do: Routes.page_path(conn, :index)
end
EOF
```

Now finally update `config/confix.exs` by adding `routes_backend: MyAppWeb.Pow.Routes` to the `:pow` config so now it would look like:
```
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo,
  web_module: MyAppWeb,
  extensions: [PowPersistentSession],
  controller_callbacks: Pow.Extension.Phoenix.ControllerCallbacks,
  routes_backend: MyAppWeb.Pow.Routes    # add this line
```


Assuming all works we will snapshot now!

```
git add .
git commit -m "on logout go to landing page"
```


### Password Reset and Email Confirmation

https://github.com/pow-auth/pow_assent
https://www.youtube.com/watch?v=hnD0Z0LGMIk
https://experimentingwithcode.com/phoenix-authentication-with-pow-part-2/

The following are the possible extensions:

* PowResetPassword
* PowEmailConfirmation
* PowPersistentSession
* PowInvitation

Let's start with password reset and email confirmation.

First we need to do a migration:
```
mix pow.extension.ecto.gen.migrations --extension PowResetPassword --extension PowEmailConfirmation
```

now update the phoenix config `config/config.ex` again from:
```
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo,
  web_module: MyAppWeb
```
to:
```
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo,
  web_module: MyAppWeb,
  extensions: [PowPersistentSession, PowResetPassword, PowEmailConfirmation],
  controller_callbacks: Pow.Extension.Phoenix.ControllerCallbacks
```

now update the `LIB_PATH/users/user.ex` file from:
```
defmodule Fare.Users.User do
  use Ecto.Schema
  use Pow.Ecto.Schema

  schema "users" do
    pow_user_fields()

    timestamps()
  end
end
```
to:
```
defmodule MyApp.Users.User do
  use Ecto.Schema
  use Pow.Ecto.Schema
  use Pow.Extension.Ecto.Schema,
      extensions: [PowResetPassword, PowEmailConfirmation]

  schema "users" do
    pow_user_fields()

    timestamps()
  end

  def changeset(user_or_changeset, attrs) do
    user_or_changeset
    |> pow_changeset(attrs)
    |> pow_extension_changeset(attrs)
  end
end
```

And of course the routes `WEB_PATH/router.ex` too - at the top of the file add:
so it looks like:
```
defmodule MyAppWeb.Router do
  use MyAppWeb, :router
  use Pow.Phoenix.Router
  use Pow.Extension.Phoenix.Router,
      extensions: [PowResetPassword, PowEmailConfirmation]
```

And in the pow routes config change from:
```
  scope "/" do
    pipe_through :browser

    pow_routes()
  end
```
to:
```
  scope "/" do
    pipe_through :browser

    pow_routes()
    pow_extension_routes()
  end
```

Now finally, we need can update any views needed by POW's new extensions with:
```
mix pow.extension.phoenix.gen.templates --extension PowResetPassword --extension PowEmailConfirmation
```


Now we can update the sign-in page with a reset password button.  We will add the following, to the end of `lib/fare_web/templates/pow/session/new.html.eex`:
```
|
<span>
<%= link "Reset Password", to: Routes.pow_reset_password_reset_password_path(@conn, :new) %>
</span>
```

Now lets be sure we can link to reset password view.

First we will do our migration:
```
mix ecto.migrate
```

now to to sign-in and see if the reset password link works.
Cool it does, but it we try to use it - it complains it needs email back-end setup.

## Email backend

https://dev.to/oliverandrich/learn-elixir-and-phoenix-add-authentication-55kl

First we will create a mailer function in: `lib/my_app/pow_mailer.ex`

```
defmodule FareWeb.Pow.Mailer do
  use Pow.Phoenix.Mailer
  # use Bamboo.Mailer, otp_app: :fare
  require Logger

  # import Bamboo.Email

  @impl true
  def cast(%{user: user, subject: subject, text: text, html: html}) do
    # for use when Bamboo is configured
    # new_email(
    #   to: user.email,
    #   from: "reading-list@example.com",
    #   subject: subject,
    #   html_body: html,
    #   text_body: text
    # )

    # send to logger - disable when
    %{to: user.email, subject: subject, text: text, html: html}
  end

  @impl true
  def process(email) do
    # actually deliver emails - enable when `bamboo` configured
    # deliver_now(email)

    # check email functionality and contents
    Logger.debug("E-mail sent: #{inspect email}")
  end
end
```

now that we have an email template we need to tell pow about the mailer with the config: `mailer_backend: MyAppWeb.Pow.Mailer` in `config/confix.exs` so change from:
```
# config for pow - user authentication
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo,
  web_module: MyAppWeb,
  extensions: [PowPersistentSession, PowResetPassword, PowEmailConfirmation],
  controller_callbacks: Pow.Extension.Phoenix.ControllerCallbacks
```
to:
```

# config for pow - user authentication
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo,
  web_module: MyAppWeb,
  mailer_backend: MyAppWeb.Pow.Mailer,
  extensions: [PowPersistentSession, PowResetPassword, PowEmailConfirmation],
  controller_callbacks: Pow.Extension.Phoenix.ControllerCallbacks
```

Now generate any mail templates needed with:
```
mix pow.extension.phoenix.mailer.gen.templates --extension PowResetPassword --extension PowEmailConfirmation
```

Phoenix also needs to know about the mailer templates we will generate so add to `lib/my_app_web.ex`:
```
  def mailer_view do
    quote do
      use Phoenix.View, root: "lib/my_app_web/templates",
                        namespace: MyAppWeb

      use Phoenix.HTML
    end
  end
```

Now the final config change in `config/config.ex` from:
```
# config for pow - user authentication
config :fare, :pow,
  user: Fare.Users.User,
  repo: Fare.Repo,
  web_module: FareWeb,
  mailer_backend: MyAppWeb.Pow.Mailer,
  extensions: [PowPersistentSession, PowResetPassword, PowEmailConfirmation],
  controller_callbacks: Pow.Extension.Phoenix.ControllerCallbacks
```
to:
```
# config for pow - user authentication
config :fare, :pow,
  user: Fare.Users.User,
  repo: Fare.Repo,
  web_module: FareWeb,
  web_mailer_module: MyAppWeb,
  mailer_backend: MyAppWeb.Pow.Mailer,
  extensions: [PowPersistentSession, PowResetPassword, PowEmailConfirmation],
  controller_callbacks: Pow.Extension.Phoenix.ControllerCallbacks
```

Now if we resart phoenix and test out reset link - we should see in the logs 'a pretend sent email' - something like:
```
[debug] E-mail sent: %{html: "<h3>Hi,</h3>\n<p>Please use the following link to reset your password:</p>\n<p><a href=\"http://localhost:4000/reset-password/SFMyNTY.MTJkNDliZWItZTg2My00ZDM3LTg2YzgtYzE5MDdjMDk5ODgz.kFRCfvdOSeEnupbbujdAKoaCuMXXk91qzZCUMrB43mw\">http://localhost:4000/reset-password/SFMyNTY.MTJkNDliZWItZTg2My00ZDM3LTg2YzgtYzE5MDdjMDk5ODgz.kFRCfvdOSeEnupbbujdAKoaCuMXXk91qzZCUMrB43mw</a></p>\n<p>You can disregard this email if you didn&#39;t request a password reset.</p>", subject: "Reset password link", text: "Hi,\n\nPlease use the following link to reset your password:\n\nhttp://localhost:4000/reset-password/SFMyNTY.MTJkNDliZWItZTg2My00ZDM3LTg2YzgtYzE5MDdjMDk5ODgz.kFRCfvdOSeEnupbbujdAKoaCuMXXk91qzZCUMrB43mw\n\nYou can disregard this email if you didn't request a password reset.\n", to: "btihen@gmail.com"}
```

copy the link out of the email in the log:
```
http://localhost:4000/reset-password/SFMyNTY.MTJkNDliZWItZTg2My00ZDM3LTg2YzgtYzE5MDdjMDk5ODgz.kFRCfvdOSeEnupbbujdAKoaCuMXXk91qzZCUMrB43mw
```
into the browser - type a new password and try to login.

Assuming all works we will snapshot now!

```
git add .
git commit -m "pow configured to send emails - no sender yet"
```

## After Logout - go to Landing Page (After Hook Routing)

https://experimentingwithcode.com/phoenix-authentication-with-pow-part-2/

One little annoying thing is that when we logout we go to the sign-in page instead of the landing page.  We can fix that by adding a call_back_route - you can find all the callback routes at: https://github.com/danschultzer/pow/blob/master/lib/pow/phoenix/routes.ex - we will use: the `after_sign_out_path` callback.

To do this we will make a new `pow.routes` file:
```
touch lib/warehouse_web/pow/routes.ex
```

Add the following contents:
```
cat << EOF> lib/my_app_web/pow/routes.ex
defmodule MyAppWeb.Pow.Routes do
  use Pow.Phoenix.Routes
  alias MyAppWeb.Router.Helpers, as: Routes

  def after_sign_out_path(conn), do: Routes.page_path(conn, :index)
end
EOF
```

Now finally update `config/confix.exs` by adding `routes_backend: MyAppWeb.Pow.Routes` to the `:pow` config so now it would look like:
```
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo,
  web_module: MyAppWeb,
  web_mailer_module: MyAppWeb,
  mailer_backend: MyAppWeb.Pow.Mailer,
  routes_backend: MyAppWeb.Pow.Routes,
  controller_callbacks: Pow.Extension.Phoenix.ControllerCallbacks,
  extensions: [PowPersistentSession, PowResetPassword, PowEmailConfirmation]
```


Assuming all works we will snapshot now!

```
git add .
git commit -m "on logout go to landing page"
```

## Configure Email BAMBOO with POW

https://www.kabisa.nl/tech/real-world-phoenix-lets-send-some-emails/
https://dev.to/oliverandrich/learn-elixir-and-phoenix-add-authentication-55kl

Use Bamboo to do the mailing find the new versions at:
* https://hex.pm/packages/bamboo
* https://hex.pm/packages/bamboo_smtp
Add to `mix.exs`:
```
    {:bamboo, "~> 2.1"},
    {:bamboo_smtp, "~> 4.0"}
```

Also add bamboo to the `applications` in mix - now it should look like:
```
  def application do
    [
      mod: {Fare.Application, []},
      applications: [:bamboo, :bamboo_smtp],  # this was added
      extra_applications: [:logger, :runtime_tools]
    ]
  end
```

Now install and setup up: https://github.com/mailhog/ (on a MacOS) simply install with:
```
brew install mailhog
```

and run mailhog with:
```
mailhog
```
or if you want mailhog running all the time in the background you can type:
```
  brew services start mailhog
```
Or you can use: or https://mailcatcher.me/

These serivices  - listen on `localhost:1025` and you can view the email at: http://localhost:8025

now configure the mail service (in `config/dev.exs`) to use Mailhog or Mailcather with Phoenix by adding:
```
# config/dev.exs
config :my_app, MyAppWeb.Pow.Mailer,
  adapter: Bamboo.SMTPAdapter,
  server: "localhost",
  port: 1025
```


in production it might look like:
```
# config/config.exs
config :my_app, MyApp.Mailer,
  adapter: Bamboo.SMTPAdapter,
  server: "smtp.domain",
  hostname: "your.domain",
  port: 1025,
  username: "your.name@your.domain", # or {:system, "SMTP_USERNAME"}
  password: "pa55word", # or {:system, "SMTP_PASSWORD"}
  tls: :if_available, # can be `:always` or `:never`
  allowed_tls_versions: [:"tlsv1", :"tlsv1.1", :"tlsv1.2"], # or {:system, "ALLOWED_TLS_VERSIONS"} w/ comma seprated values (e.g. "tlsv1.1,tlsv1.2")
  ssl: false, # can be `true`
  retries: 1,
  no_mx_lookups: false, # can be `true`
  auth: :if_available # can be `:always`. If your smtp relay requires authentication set it to `:always`.
```


Update the pow mail file `lib/read_it_later_web/pow/mailer.ex` to use Bamboo - the code will look like:
```
defmodule MyAppWeb.Pow.Mailer do
  use Pow.Phoenix.Mailer
  use Bamboo.Mailer, otp_app: :my_app

  import Bamboo.Email

  @impl true
  def cast(%{user: user, subject: subject, text: text, html: html}) do
    new_email(
      to: user.email,
      from: "reading-list@example.com",
      subject: subject,
      html_body: html,
      text_body: text
    )
  end

  @impl true
  def process(email) do
    deliver_now(email)
  end
end
```

## Adding flash messages to POW

https://experimentingwithcode.com/phoenix-authentication-with-pow-part-2/



## Configure to allow 3rd Parties - Google, Apple, Github, etc.

https://github.com/pow-auth/pow_assent
https://www.youtube.com/watch?v=hnD0Z0LGMIk


First add to the mix file:
```
    # third party auth
    {:pow_assent, "~> 0.4.10"},
    # recommended for SSL validation with :httpc adapter
    {:certifi, "~> 2.4"},
    {:ssl_verify_fun, "~> 1.1"},
```

and of course: `mix deps.get`

and install with: `mix pow_assent.install`

and now configure `lib/fare/users/user.ex` after `use Pow.Ecto.Schema` add `use PowAssent.Ecto.Schema` so now the top of this file should look like:
```
# lib/fare/users/user.ex
defmodule Fare.Users.User do
  use Ecto.Schema
  use Pow.Ecto.Schema
  use PowAssent.Ecto.Schema  # added in this step
  use Pow.Extension.Ecto.Schema,
      extensions: [PowResetPassword, PowEmailConfirmation]
```

At the top of the `lib/fare_web/router.ex` file after `use PowAssent.Phoenix.Router` add `use PowAssent.Phoenix.Router` - now the top of this file should look like:
```
# lib/fare_web/router.ex
defmodule MyAppWeb.Router do
  use MyAppWeb, :router
  use Pow.Phoenix.Router
  use PowAssent.Phoenix.Router
  use Pow.Extension.Phoenix.Router,
      extensions: [PowResetPassword, PowEmailConfirmation]
```

Now after the last pipelines add a new `pipeline` and its `scope` - its a copy of the `:browser` pipeline - without `:protect_from_forgery` since that conflicts with **OAuth** & after `pow_routes()` add `pow_assent_routes()` so now this section of the routes looks like (when Phoenix is configured for LiveView):
```
  pipeline :skip_csrf_protection do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :put_root_layout, {FareWeb.LayoutView, :root}
    # plug :protect_from_forgery     # conflicts with oauth
    plug :put_secure_browser_headers
  end

  scope "/" do
    pipe_through :skip_csrf_protection

    # this adds new pow routes
    pow_assent_authorization_post_callback_routes()
  end

  scope "/" do
    pipe_through :browser

    pow_routes()
    pow_assent_routes()    # newly added
    pow_extension_routes()
  end
```

Remember to run the new migrations with:
```
mix ecto.migrate
```

Generate the PowAssent template too (the page when using this where the user add username and OAuth password from remote site):
```
mix pow_assent.phoenix.gen.templates
```

### Setup remote OAutho providers (Github - for now)

Go to:
https://github.com/settings/applications/new

Enter an **Application name** and enter the **Homepage url** as:
```
http://localhost:4000/
```
and the **Authorization callback** (for our dev environment) as:
```
http://localhost:4000/auth/github/callback
```

**Configure Github Credential Secrets**

* https://hexdocs.pm/elixir/Application.html
* https://devato.com/post/handling-environment-variables-in-phoenix
* https://stackoverflow.com/questions/44510403/phoenix-import-module-into-config
* https://stackoverflow.com/questions/30995743/how-to-get-a-variable-value-from-environment-files-in-phoenix

First update `.gitignore` with the line:
```
**/*.secret.exs
```

then add in our case the `dev.secrets.exs` file:
```
touch config/dev.secret.exs
```
Once you get your **Client ID** and **Client secrets** you can configure  `config/dev.secret.exs` with the following config:
```
import Config

config :my_app, :pow_assent,
  providers: [
    github: [
      client_id: "REPLACE_WITH_CLIENT_ID",
      client_secret: "REPLACE_WITH_CLIENT_SECRET",
      strategy: Assent.Strategy.Github
    ]
  ]
```

Now at the END of `config/dev.exs` add the line:
```
import_config "dev.secret.exs"
```

Now at the end of:

* `lib/my_app_web/templates/pow/registration/edit.html.eex` (edit profile),
* `lib/my_app_web/templates/pow/registration/new.html.eex` (register),
* `lib/fare_web/templates/pow/session/new.html.eex` (sign-in)

add the following comprehension to list all the configured OAuth log-in links:
```
<%=
  for link <- PowAssent.Phoenix.ViewHelpers.provider_links(@conn),
      do: content_tag(:span, link)
%>
```


## Testing

- https://github.com/dashbitco/mox
- https://hex.pm/packages/phoenix_integration
-



## Resources:

* pow readme: https://hexdocs.pm/pow/README.html
* video tutorial: https://www.youtube.com/watch?v=hnD0Z0LGMIk
* add pubsub https://curiosum.dev/blog/elixir-phoenix-liveview-messenger-part-3
* add email to pow: https://dev.to/oliverandrich/learn-elixir-and-phoenix-add-authentication-55kl
* https://experimentingwithcode.com/phoenix-authentication-with-pow-part-1/
* https://experimentingwithcode.com/phoenix-authentication-with-pow-part-2/
