---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 6.x+ Auth with MagicLinks using Rails Signed GlobalIDs"
subtitle: "Password-less Authentication"
summary: "Several simple approaches to password-less rails authentication"
authors: ["btihen"]
tags: ['Ruby', "authentication", "passwordless auth", "MagicLink", "Signed GlobalID"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-09-19T01:36:55+02:00
lastmod: 2021-09-21T01:36:55+02:00
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

Passwordless Authentication is very convenient for users and generally as secure as passwords (a good security authentication discussion can be found at: ).

A good approach is to make a short-lived link, and then transfer the security to a session.

I found that there seems to be three simple approaches:

1) Do it yourself: with a Stored-Token (with an expiration date-time)
2) Do it yourself: with a Signed-GlobalID from Rails (self-times out & no stored tokens)
3) Other Options: other Gems for Devise, Sorcery, or independent gems


## Overview

1. User enters their email-address in a simple form
2. If account is found - a link with a token is generated and email is sent
3. User is notified that the link is on its way (even if the account is not found and no email is sent)
4. When the user follows the link in the email, a session is generated
5. Session valid until the session expires or the user logs out (deleting the session).

**NOTE:** User accounts are assumed to be previously created, and verified.  If you need a full set of features - then your best option is probably to use devise, sorcery or authenticate and use an extension or build your own extension to one of these libraries.  Either way, this article will clarify the basics of passwordless authentication.


### Getting Started

Code Repo is posted at: https://github.com/btihen/magic_sgid

Create a Rails Project

```bash
bin/rails new magic_id
cd magic_id
git add .
git commit -m "initial commit on creation"
```

User-Controller to manage users:
```bash
bin/rails db:create
bin/rails g scaffold User name:string email:string
bin/rails db:migrate
```

Lets start Rails
```bash
bin/rails s
```

Go to: `http://localhost:3000/users` and create a few users - feel free to make the GUI nicer!

Assuming all is good:
```bash
git add .
git commit "user management scaffold"
```

### Create a Landing Page

we need a landing / root page to send users when they are not logged in:

```bash
bin/rails g controller landing index
```

now lets point the root page to that too - make the routes page look like:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  get '/landing', to: 'landing#index', as: :landing
  root to: "landing#index"
end
```

I will also remove: `app/helpers/landing_helper.rb` with:
```bash
rm app/helpers/landing_helper.rb
```

Now lets check all is well with the routes:
```bash
bin/rails routes | grep landing
# should show
  landing GET /landing(:format) landing#index
     root GET /                 landing#index
```
(quite likely it will be all spread out)

Now the following pages should be available:

* `http://localhost:3000/`
* `http://localhost:3000/landing`

again feel free to make them look nice.

assuming all works well:
```bash
git add .
git commit -m "create landing page and landing & root route"
```

### Create a User Application Controller (restrict access)

This will allow us to control access to all urls in the /users paths within our app

```bash
mkdir app/controllers/users
touch app/controllers/users/application_controller.rb
```

The application controller ensures only authenticated users (with a session) can access pages in the `users` area:
```ruby
# app/controllers/users/application_controller.rb
class Users::ApplicationController < ApplicationController
  before_action :users_only

  def current_user(user_id = session[:user_id])
    # `try` and `find_by` avoid raising an exception w/o a session
    @current_user ||= User.find_by(id: user_id)
  end

  private

  # code to ensure only logged in users have access to users pages
  def users_only
    # send person to a safe page if not logged in
    if current_user.blank?
      # send to login page to get an access link
      redirect_back(fallback_location: landing_path,
                    :alert => "Login Required")
      # once the below page is created we can redirect to here instead
      # redirect_back(fallback_location: new_users_login_path,
      #               :alert => "Login Required")
    end
  end
end
```

NOTE: if you want all pages protected then put this code in:
`app/controllers/application_controller.rb`
and adjust the routes (remove the namespace)!


### Restricted User Home Page


we need a landing / root page to send users when they are not logged in:

```bash
bin/rails g controller users/landing index
```

now lets point the root page to that too - make the routes page look like:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :users do
    get '/',                as: 'root',         to: 'home#index'
    get '/home',            as: 'home',         to: 'home#index'
  end
  get '/landing', to: 'landing#index', as: :landing
  root to: "landing#index"
end
```

Update the controller to use the Users::ApplicationController:
```ruby
# app/controllers/users/home_controller.rb
class Users::HomeController < Users::ApplicationController
  def index
  end
end
```

I will also remove: `app/helpers/users/home_helper.rb` with:
```bash
rm app/helpers/users/home_helper.rb
```

Now lets check all is well with the routes:
```bash
bin/rails routes | grep user
# should show
  users_home GET /users/home(:format) home#index
```
(quite likely it will be all spread out)

Now the following pages should NOT be available:

* `http://localhost:3000/users`
* `http://localhost:3000/users/home`

and we should be redirected to the landing page.  If you access the home page - then probably the first line is wrong it should be:

`Users::HomeController < Users::ApplicationController`

assuming all works well:
```bash
git add .
git commit -m "create restricted user home page"
```


### Create an Session Authorization Controller

```bash
touch app/controllers/users/sessions_controller.rb
```

```ruby
class Users::SessionsController < Users::ApplicationController
  # before_action :users_only, only: :destroy
  skip_before_action :users_only, only: :create

  def create
    sgid_token = params[:token].to_s
    user = GlobalID::Locator.locate_signed(sgid_token, for: 'user_access')
    if user
      # create the session id for current_user to access
      session[:user_id] = user.id
      redirect_to(users_home_path, notice: "Welcome back #{user.name}")
    else
      flash[:alert] = 'Oops - you need a new login link'
      redirect_to(landing_path)
      # later when created we will redirect to login access link page
      # redirect_to(new_users_login)
    end
  end

  # allow a user to logout / destroy session if desired
  def destroy
    user = current_user
    if user
      session[:user_id] = nil
      flash[:notice] = "logout successful"
    else
      falsh[:alert] = "Oops, there was a problem"
    end
    redirect_to(landing_path)
  end
end
```

Add to routes:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :users do
    get '/',                as: 'root',         to: 'home#index'
    get '/home',            as: 'home',         to: 'home#index'
    # use get to create since I don't think a text url can create a post
    get '/sessions/:token', as: 'session_create', to: 'sessions#create'
    # allow logout / destroy the session
    resources :sessions,    only: [:destroy]
  end
  get '/landing', to: 'landing#index', as: :landing
  root to: "landing#index"
end
```

To test this go to rails console:
```ruby
bin/rails c
# create our own token
user = User.first
global_id = User.find(user.id).to_sgid(expires_in: 1.hour, for: 'user_access')
access_token = global_id.to_s

# check that this global_id works (we should get the same user)
GlobalID::Locator.locate_signed(access_token, for: 'user_access')

# add url helpers to console
include ActionView::Helpers
include ActionView::Helpers::UrlHelper
# generate the URL for the session
Rails.application.routes.url_helpers.users_session_create_url(token: global_id.to_s, host: 'locahost:3000')
# should get something like:
# "http://locahost:3000/users/sessions/BAh7CEkiCGdpZAY6BkVUSSItZ2lkOi8vbWFnaWMtbGlua3MvVXNlci8xP2V4cGlyZXNfaW49MzYwMAY7AFRJIgxwdXJwb3NlBjsAVEkiEHVzZXJfYWNjZXNzBjsAVEkiD2V4cGlyZXNfYXQGOwBUSSIdMjAyMS0wOS0xOVQxNTozNzo0MS4wNjdaBjsAVA==--c948a0a5ccbae391c7ab9c808677fe41da4cbc28"

# copy this url into the browser
# now we be on: `http://localhost:3000/user/` & `http://localhost:3000/user/home`

# if you try again in an hour it should not work!
```

**Note:** by default rails sessions have no expiration, thus are deleted when the browser closes. To change this default behavior, you can set the session length with the setting:
```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store, expire_after: 14.days
```

Assuming all works:
```bash
git add .
git commit -m "session controller gives access to users_home"
```


### Access-Link Generation

We will need to allow the user to request an access-link.  We will do this with (we won't be generating any models just a controller and a submission form - scaffold_controller does this for us):

#### Create a Mailer to send Access-Links

We need a way to send the login link - so we will create a login mailer with:
```bash
bin/rails generate mailer Login send_link
```

Configure our emailer for our needs with:
```ruby
class LoginMailer < ApplicationMailer
  def send_link(user, login_url)
    @user = user
    @login_url  = login_url
    host = Rails.application.config.hosts.first

    mail(to: @user.email, subject: "Access-Link for #{host}")
  end
end
```

And of course set up the views that contain the contents of the email:
The HTML Version
```ruby
# app/views/login_mailer/send_link.html.erb
<h1>Hi <%= @user.name %>,</h1>

<p><a href="<%= @login_url %>">Access-Link for <%= @host %></a></p>

<p> <%= @login_url %> </p>

<p>Link is valid for about an hour from <%= DateTime.now %></p>
```

The text version:
```ruby
# app/views/login_mailer/send_link.text.erb
Hi <%= @user.name %>,

Access-Link for <%= @host %> is:

<%= @login_url %>

Link is valid for about an hour from <%= DateTime.now %>.
```

#### Setup Mailhog (OPTIONAL)

This is optional - technical testing can be done from the log file - but to see what the email formatting looks like this is VERY HELPFUL.

```bash
brew install mailhog
mailhog
```

Configure Rails to send emails to port 1025 in development (where mailhog listens)
```ruby
# config/environments/development.rb
Rails.application.configure do
  # Settings specified here will take precedence over those in config/application.rb.
  # ...
  # mailhog config
  config.action_mailer.perform_deliveries = true
  config.action_mailer.smtp_settings = { address: 'localhost', port: 1025 }
  # ...
end
```

now open - to view:
```bash
open localhost:8025
```


#### Controller

Now lets create a user login controller:
```bash
touch app/controllers/users/logins_controller.rb
```

Add the contents of the controller:
```ruby
# app/controllers/users/logins_controller.rb
class Users::LoginsController < Users::ApplicationController
skip_before_action :users_only

  def new
    user = User.new
    render :new, locals: {user: user}
  end

  def create
    email = user_params[:email]
    ip_address = request.remote_ip
    # the participant might already exist in our db or possimagic_link_url = participants_session_auth_url(token: participant.login_token)bly a new participant
    user = User.find_by(email: email)

    if user
      # create a signed expiring Rails Global ID - this makes LONG tokens, but browswers can handle it
      # all browsers should handle up to 2000 characters.
      # https://stackoverflow.com/questions/417142/what-is-the-maximum-length-of-a-url-in-different-browsers
      # https://www.geeksforgeeks.org/maximum-length-of-a-url-in-different-browsers/
      global_id = user.to_sgid(expires_in: 1.hour, for: 'user_access')
      access_url = users_session_create_url(token: global_id.to_s)
      LoginMailer.send_link(user, access_url).deliver_later
    else
      # if user isn't found then grab a user and compute the global_id and url (but don't send an email)
      # in order to make the time of both paths similar - so people can't find user emails checking the response times
      # see: https://abevoelker.com/skipping-the-database-with-stateless-tokens-a-hidden-rails-gem-and-a-useful-web-technique/

      global_id = User.first.to_sgid(expires_in: 1.hour, for: 'user_access')
      access_url = user_auth_url(token: global_id.to_s)
    end

    # uncomment to add noise to further make email fishing difficult to time
    # mini_wait = Random.new.rand(10..20) / 1000
    # wait(mini_wait)

    # true or not we state we have sent an access link and redirect to the landing page
    # also prevent email fishing by always returning the same answer
    redirect_to(landing_path, notice: "Access-Link has been sent")
  end

  private

    # Only allow a list of trusted parameters through.
    def user_params
      params.require(:user).permit(:email)
    end
end
```


**NOTE:** You can let the GlobalID be valid for however long you want (in hours), but since email isn't very secure, it seems wise to keep this short lived.  The default time is 30.days


We need to add the route to the users login_controller with:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  # restricted area (protected by user login - Users::ApplicationController)
  namespace :users do
    get '/',                as: 'root',         to: 'home#index'
    get '/home',            as: 'home',         to: 'home#index'
    # use get to create since I don't think a text url can create a post
    get '/sessions/:token', as: 'session_create', to: 'sessions#create'
    # allow logout / destroy the session
    resources :sessions,    only: [:destroy]
    # login (generates link and emails to the user)
    resources :logins,      only: [:new, :create]
  end
  get '/landing', to: 'landing#index', as: :landing
  root to: "landing#index"
end
```

now check the routes:
```bash
bin/rails routes | grep users
# should return
     users_logins POST  /users/logins(.:format)     users/logins#create
  new_users_login GET   /users/logins/new(.:format) users/logins#new
```

#### Email / Access Link form

Login email form:

```bash
mkdir app/views/users/logins
touch app/views/users/logins/new.html.erb
```

Note I often use Bulma - so here is how I like to format my forms (without Bulma installed the form will be ugly).
Also note, I dislike using instance variables in my form - so this is why the form looks a little extra complicated.
```ruby
# app/views/users/logins/new.html.erb
<%= form_for(user, local: true,
             url: users_logins_path,   # NEW MUST BE PLURAL for POST
             id: "user-login-form", class: "user" ) do |form|  %>

  <div class="field">
    <label class="label">Email for Access-Link</label>
    <div class="control">
      <%= form.email_field :email,
                            placeholder: "Email",
                            class: 'input' %>
    </div>
    <p class="help"></p>
  </div>

  <div class="control">
    <%= form.submit("Get Access-Link", class: "button is-success") %>
  </div>

<% end %>
```

## Test the full flow

using the command line create some users:
```ruby
bin/rails c
User.create(email: "tester@test.ch", name: "Tester")
```

start rails with: `bin/rails s`
start mailhog with: `mailhog`
go to: `http://localhost:3000/user/home` (should get redirected to the below URL)
go to: `http://localhost:3000/user/logins`
enter the "tester@test.ch" email
Check mailhog for the link `http://localhost:8025/`
Click the link you should now be on `http://localhost:3000/user/home`

Assuming everything works:
```
git add .
git commit -m "working magic links using Rails Global ID"
```

**NOTE:** obviously automated tests are important (both spec and feature tests).

## Resources

**Rails GlobalID**

The nice thing about these is that the auto expire - simplifying the code a lot.

* https://github.com/rails/globalid
* https://www.magicalruby.com/implementing-magic-links-in-rails/

**Token using SecureRandom**

With these you need to create your own expiration and lookup system (more code add a migration), but will work with any framework.

* (using uuids) - https://oozou.com/blog/how-to-implement-passwordless-authentication-in-ruby-on-rails-154
* (using SecureRandom) - https://abevoelker.com/skipping-the-database-with-stateless-tokens-a-hidden-rails-gem-and-a-useful-web-technique/


**Devise Options**

* Devise Plugin - https://github.com/abevoelker/devise-passwordless
* Do it Yourself Devise - https://dev.to/matiascarpintini/magic-links-with-ruby-on-rails-and-devise-4e3o
* Do it yourself Devise - https://www.mintbit.com/blog/passwordless-authentication-in-ruby-on-rails-with-devise


**Other Options**

* passwordless gem - https://github.com/mikker/passwordless#token-and-session-expiry
* magic-link gem - https://github.com/dvanderbeek/magic-link
* Using Sorcery - https://fullstackheroes.com/rails/sorcery-passwordless-authentication/
* Using Sourcery - https://www.sitepoint.com/magical-authentication-sorcery/
* (using JWTs) - https://blog.kiprosh.com/implement-passwordless-authentication-via-magic-link-in-rails-api/
**Passwordless Security Overview**

* https://abevoelker.com/skipping-the-database-with-stateless-tokens-a-hidden-rails-gem-and-a-useful-web-technique/

**Sessions**

* https://api.rubyonrails.org/classes/ActionDispatch/Session/CookieStore.html
* https://blog.saeloun.com/2019/09/12/rails-6-adds-dig-to-actiondispatch-request-session.html
