---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Passwordless Authentication with Devise"
subtitle: "Passwordless Devise Strategy using Secure Global IDs"
summary: "Devise offers a lot of useful User management, but has no Passwordless Strategy - here's how to do it"
authors: ["btihen"]
tags: ['Ruby', "authentication", "passwordless", "Devise", "Signed GlobalID", "Magic-Link"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-12-18T01:36:55+02:00
lastmod: 2021-12-18T01:36:55+02:00
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

I wrote a little app for a small non-profit group.  Some of them had severe problems with password management - so this article is how I solved that.

The easiest way to approach this is to use Rails built-in Secure Global IDs, in this way no database migrations are needed.


## Overview

1. User enters their email-address in a simple form
2. If account is found - a link with a token is generated and email is sent
3. User is notified that the link is on its way (even if the account is not found and no email is sent)
4. When the user follows the link in the email, a session is generated
5. Session valid until the session expires or the user logs out (deleting the session).

**NOTE:** Since you are using Devise, I will assume you are using it to also manage accounts.  However to keep the code short I will only show what is needed for this one feature.

### Understanding GlobalIDs

Let's start by understanding how Global IDs work
- We start by grabbing a user object and generating a token.
- Next we use the token to retrieve the same user object.
Code to demostrate the usage:

```ruby
bin/rails c
user_orig = User.first
# * the `for: 'user_auth'` must matching on the receiving end.
# * the `expires_in: 1.hour` can be set for any length of time (default is 30 days)
sgid = user_orig.to_sgid(expires_in: 1.hour, for: 'user_auth')

# now that we have a secured global id, we can generate a token
auth_token = sgid.to_s # token from the Global ID

# should retrieve the user since the token is still valid and the `for:` string matches
user_retrieved = GlobalID::Locator.locate_signed(auth_token, for: 'user_auth')
# try this token again in an hour+ and it should fail!
user_retrieved.id == user_orig.id

# should be nil, since the `for:` string didn't match
GlobalID::Locator.locate_signed(auth_token, for: 'admins_access').nil?
```

### Global IDs for login

To use a Global ID will need a user's email to send it to them and generate a token points to a url that can verify the token and create a session.   So we would need code that looks something like:

```ruby
email = params[:email]
user = User.find_by(email: email)
sgid = user.to_sgid(expires_in: 1.hour, for: 'user_auth')
auth_url = Rails.application.routes.url_helpers
                .auth_user_session_url(auth_token: sgid.to_s)
UserAuthMailer.send_link(user, auth_url).deliver_now
```

Of course we don't have the `auth_user_session_url` route and `UserAuthMailer.send_link` yet, but we will build that soon.

To unpack the token and build a session we will need code that looks like:

```ruby
auth_token = params[:auth_token]
user = GlobalID::Locator.locate_signed(auth_token, for: 'user_auth')
if user.present?
  sign_in(user) # a devise method
  flash[:notice] = "Welcome back! #{user.email}"
else
  flash[:alert] = 'invalid token'
end
redirect_to root_path
```

Lets figure out where all this code goes to integrate with both Rails and Devise.

### Simple App

This code-repo is posted at: https://github.com/btihen/ruby_kafi_passwordless_devise_code (actually this has code to demo two methods)

Create a Rails Project

```bash
bin/rails new passwordless_devise_code
cd passwordless_devise_code

bin/rails db:create
bin/rails g controller Landing index
bin/rails g scaffold Pet name species
bin/rails db:migrate

## config/routes.rb
Rails.application.routes.draw do
  resources :pets
  root to: "landing#index"
end
```
The landing page and pets should both be fully available.

### Add the Devise Gem

User-Controller to manage users:
```bash
bundle add devise
bundle install
bin/rails generate devise:install
bin/rails generate devise User
# update the migration to match any added features
bin/rails db:migrate
```

### Devise / Rails Config for email and URLs

Devise and Rails need a few config tweeks to do what we want.  (the example is for development, but when publishing of course production will need the appropriate configs too)
```ruby
# config/environments/development.rb
self.default_url_options = { host: 'http://localhost:3000' }
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }

# app/views/layouts/application.html.erb
<p class="notice"><%= notice %></p>
<p class="alert"><%= alert %></p>
```

### Require Authentication to access Pets

Now lets activate Devise on all pages, except the landing page.

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
end

# app/controllers/landing_controller.rb
class LandingController < ApplicationController
  skip_before_action :authenticate_user!
  def index
  end
end
```

Now the landing page should be available, but not the pets page.

### Setup Session Authentication

We are going to need to generate our own DeviseSession controller and add to it.

```ruby
rails g devise:controllers users -c=sessions

# app/controllers/users/sessions_controller.rb:
class Users::SessionsController < Devise::SessionsController
  def auth_token
    auth_token = params[:auth_token]
    user = GlobalID::Locator.locate_signed(auth_token, for: 'user_auth')
    # if we get a user then we know the secured global ID checked out
    if user.present?
      sign_in(user)
      flash[:notice] = "Welcome back! #{user.email}"
      redirect_to pets_path
    else
      flash[:alert] = 'OOPS - something went wrong.'
      redirect_to root_path
    end
  end
end
```

We need to tell Devise and Rails about our new code in the routes:

```ruby
#config/routes.rb
Rails.application.routes.draw do
  # tell devise about our sessions controller
  devise_for :users, controllers: { sessions: 'users/sessions' }
  devise_scope :user do  # tell rails and devise about our new passwordless authorization route
    get 'users/auth_token/:auth_token', as: :auth_user_session, to: 'users/sessions#auth_token'
  end
  resources :pets
  root to: "landing#index"
end
```

If we try this now we will get a password error, since devise always assumes a user MUST have and has entered a password.

### Allow Devise to ignore passwords

It is tricky to remove the passwords, but easy to ignore them with:
```ruby
class User < ApplicationRecord
  before_validation :set_password, on: :create

  # don't require passwords with user_authenticate!
  def password_required?
    false # because we aren't using passwords
  end
  private
  # set random Devise passwords to keep devise happy
  def set_password
    tmp_passwd = SecureRandom.alphanumeric(30) # the longer the better (more or less)
    self.password = tmp_passwd
    self.password_confirmation = tmp_passwd
  end
end
```

### Testing our new model and controller

```ruby
bin/rails c

# create new devise user (without a known password)
User.create(email: "tester@test.ch", name: "Tester")
user = User.last
sgid = user.to_sgid(expires_in: 1.hour, for: 'user_auth')
auth_url = Rails.application.routes.url_helpers
                .auth_user_session_url(auth_token: sgid.to_s)
#                `auth_user_session_url` - matches our route name we set!
```

Copy this generated url into your browser and you should end up on the `pets` page!

### Emailing our auth_token

Now that we know the Auth Token works, lets learn to email them to the appropriate email

```ruby
# create the Emailer code with:
bin/rails g mailer UserAuth send_link

# app/mailers/user_auth_mailer.rb
class UserAuthMailer < ApplicationMailer
  def send_url(user, auth_url)
    @user = user
    @url  = auth_url
    @host = Rails.application.config.hosts.first
    mail to: @user.email, subject: 'Sign in into #{@host}'
  end
end
```

Now lets create the email contents - let's include a greeting, the sending host and of course the auth_url (we will determine later)
```ruby


# app/views/user_auth_mailer/send_url.html.erb
<p>
  Hi <%= @user.email %>,
  <br><br>
  For access to <%= @host %> <%= link_to "Click here", @url %>
  <br><br>
  or in plain-text: <%= @url %>
</p>

# app/views/user_auth_mailer/send_url.text.erb
Hi <%= @user.email %>,

For access to <%= @host %> follow this link:
<%= @url %>
```

### Test the emailer

Start mailhog and then:

```ruby
bin/rails c
user = User.first
sgid = user.to_sgid(expires_in: 1.hour, for: 'user_auth')
auth_url = Rails.application.routes.url_helpers
                .auth_user_session_url(auth_token: sgid.to_s)
UserAuthMailer.send_link(user, auth_url).deliver_now
```

Now open the mailhog browser tab (or copy the url from the console) and click on the link and pets should open.

### Now the Hard (Devise Strategy) part

Now we have all that we need to update Devise with a new Strategy.

```ruby
mkdir app/lib/devise
mkdir app/lib/devise/models
mkdir app/lib/devise/strategies
touch app/lib/devise/models/token_authenticatable.rb
touch app/lib/devise/strategies/token_authenticatable.rb

# app/lib/devise/models/passwordless_authenticatable.rb
require Rails.root.join('app/lib/devise/strategies/token_authenticatable')
module Devise
  module Models
    module TokenAuthenticatable
      extend ActiveSupport::Concern
    end
  end
end

# app/lib/devise/strategies/token_authenticatable.rb
require 'devise/strategies/authenticatable'
require_relative '../../../mailers/user_mailer'

module Devise::Strategies
  class TokenAuthenticatable < Authenticatable
    def authenticate!
      email = params.dig(:user, :email)
      user = User.find_by(email: email)
      if user.present? && !user.locked_at? # and other restrictions as (depending on what was configured)
        auth_sgid = user.to_sgid(expires_in: 1.hour, for: 'user_auth')
        auth_token = auth_sgid.to_s
        auth_url = Rails.application.routes.url_helpers
                        .auth_user_session_url(login_token: auth_token)
        UserAuthMailer.send_url(user, auth_url).deliver_later
      end
      fail!("An email was sent to you with an authorization link.")s
    end
  end
end
Warden::Strategies.add(:token_authenticatable, Devise::Strategies::TokenAuthenticatable)
```

**NOTE:** this strategy authenticates that the user is allowed to get a token.  The `auth` method in sessions_controller - authenticates the token and creates a session.

### Tell Devise to use the new Strategy

Now we want to move Devise away from its default `database_athenticable` to `token_authenticable`

So now we need to add `:token_authenticatable` to our User model:

```ruby
# app/models/users.rb
class User < ApplicationRecord
  before_validation :set_password, on: :create

  # at this point validatable is basically only checking that the email is valid
  devise :token_authenticatable, :validatable

  def password_required?
    false # because we aren't using passwords
  end
  private
  # since we aren't using passwords
  def set_password
    tmp_passwd = SecureRandom.alphanumeric(20)
    self.password = tmp_passwd
    self.password_confirmation = tmp_passwd
  end
end
```
However, this isn't enough - devise must know to load this strategy at boot - we do this with by adding the following to the VERY TOP of Devise initializer file:

```ruby
# config/initializers/devise.rb
Devise.add_module(:token_authenticatable, {
  strategy:   true,
  route:      :session,
  controller: :sessions,
  model:      'app/lib/devise/models/token_authenticatable'
})
```

NOTE: if all is good the devise routes should be there (plus our extra one):
```ruby
new_user_session      GET     /users/sign_in(.:format)     users/sessions#new
user_session          POST    /users/sign_in(.:format)     users/sessions#create
destroy_user_session  DELETE  /users/sign_out(.:format)    users/sessions#destroy
auth_user_session     GET     /users/auth_token(.:format)  users/sessions#auth_token
```

### Devise Views (remove the passwords)

Now we need to change the password in the views so we will need to generate the devise views (and configure devise to use scoped views is probably best):

```ruby
# config/initializers/devise.rb:
config.scoped_views = true

# generate the devise views (to override them)
bin/rails generate devise:views users
```

Now you will need to remove the password field from all the views.  For this project, I will only show the sign_in page:

```ruby
# app/views/users/sessions/new.html.erb
<h2>Log in</h2>
<%= form_for(resource, as: resource_name, url: user_session_path) do |f| %>
  <div class="field">
    <%= f.label :email %><br />
    <%= f.email_field :email, autofocus: true, autocomplete: "email" %>
  </div>
  <% if devise_mapping.rememberable? %>
    <div class="field">
      <%= f.check_box :remember_me %>
      <%= f.label :remember_me %>
    </div>
  <% end %>
  <div class="actions">
    <%= f.submit "Log in" %>
  </div>
<% end %>
<%= render "users/shared/links" %>
```

### Security Note

NOTE: I prefer a short sgid key life-spans and longer session-lifespans (both are configurable)

By default rails sessions have no expiration (until logout) and sgids are valid for a month. I find both of these settings too long. To change this default behavior, you can set the session length with the setting:

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store, expire_after: 14.days
```

And you saw in the code where the sgid lifespan is defined.


## Test the full flow

With all that completed you should be able to test the full workflow!

- start rails with: `bin/rails s`
- start mailhog with: `mailhog`
- go to: `http://localhost:3000/user/home` (should get redirected to the below URL)
- go to: `http://localhost:3000/user/logins`
- enter the "tester@test.ch" email
- Check mailhog for the link `http://localhost:8025/`
- click the link you should now be on `http://localhost:3000/user/home`


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


**Token Technologies**

* JWT - Common but be careful - https://jwt.io - (everything)
* Branca - Encrypted, simple & secure - https://github.com/thadeu/branca-ruby - (closure, .net, elixir, erlang, go, java, javascript, kotlin, php, python, ruby, rust)
* PASETO - Addresses security problems with JWT - https://paseto.io - (v3/v4: go, node, php, python, rust, swift) & (v1/v2: c, elixir, go, java, javascript, lua, .net, node, php, python, ruby, rust, swift) - https://dev.to/techschoolguru/why-paseto-is-better-than-jwt-for-token-based-authentication-1b0c
* Macaroon - better than cookies - http://macaroons.io & https://research.google/pubs/pub41892/ & https://github.com/localmed/ruby-macaroons & https://github.com/rescrv/libmacaroons - authorization with caveats and 3rd parties (c, .net, elixir, go, java, python, ruby, rust, php)
* Biscuit - has authentication dsl - https://www.biscuitsec.org & https://github.com/biscuit-auth/biscuit - (rust, web assembly, haskell, java, go)
