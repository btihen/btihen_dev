---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 6.x Auth with MagicLink using SecureRandom Token"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ['Ruby', "authentication", "passwordless auth", "MagicLink", "SecureRandom"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-09-22T16:09:57+02:00
lastmod: 2021-09-22T16:09:57+02:00
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

Passwordless Authentication is very convenient for users and generally as secure as passwords (according to many articles as long as the email access-links are short-lived - as email is not very secure).

Therefore, after some reading, it seems like a good approach is to make a short-lived link, and then transfer the security to a session.

I found that there seems to be three simple approaches:

1) Do it yourself: with a Signed-GlobalID from Rails (self-times out & no migration)
2) Do it yourself: with a Stored-Token (adapts to any framework)
3) Other Options: Devise Plugin (when using devise) or other Gems

This article will focus on using Secure Random - since it can work with any Framework (in Rails however, I prefer to use SignedGlobalIDs - see: [https://btihen.me/](https://btihen.me/post_ruby_rails/rails_6_x_auth_w_magic_links_using_signed_global_id/), since it simplifies the user model and the expiration logic)

## Overview

1. User enters their email-address in a simple form
2. If account is found - a link with a token is generated and email is sent
3. User is notified that the link is on its way (even if the account is not found and no email is sent)
4. When the user follows the link in the email, a session is generated
5. Session valid until the session expires or the user logs out (deleting the session).

**NOTE:** I will be assuming that the account must exist, but you could also just create a new account _(consider this option carefully and some limits on account creation per IP address or per hour, etc.  As you could otherwise be flooded with useless, malicious emails!)_

## Do it yourself

This is relatively easy to do with built-in Rails security - and I like not being dependent on external code, I'll show a way to do this.  In this case, assume that the accounts are already created (or not).

If you want to do user registration, confirmation, etc -- then I think it is best to use Devise or some other gem!

### Getting Started

Code repo is posted at: https://github.com/btihen/magic_token

Create a Rails Project:
```bash
bin/rails new magic_token
cd magic_token
bin/rails db:create
```

now start rails and be sure you get the welcome page at:
`http://localhost:3000`

Assuming all works well:
```bash
git add .
git commit -m "initial commit"
```


### Create a Landing Page

We will now make a landing page (it will need to be always available):

```bash
bin/rails g controller landing index --helper false
```

now lets point the root page to that too - make the routes page look like:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  get '/landing', to: 'landing#index', as: :landing
  root to: "landing#index"
end
```

Now lets check all is well with the routes:
```bash
bin/rails routes | grep landing
# should show
  landing   GET   /landing(:format)   landing#index
     root   GET   /                   landing#index
```
(quite likely it will be all spread out)

Start up rails and be sure we can access these pages:

* `http://localhost:3000/`
* `http://localhost:3000/landing`

Feel free to make them look nicer!

assuming all works well:
```bash
git add .
git commit -m "add landing page"
```

### Create Users Management Page

User-Controller to manage users:
```bash
bin/rails g scaffold User email:string token:string token_expires_at:datetime --helper false
bin/rails db:migrate
```

Lets make a few accounts in the seed file (or enter in the console `bin/rails c`):
```ruby
# db/seeds.rb
User.create(email: 'test1@test.ch')
User.create(email: 'test2@test.ch')
```

now run the seed file:
```bash
bin/rails db:seed
```

Lets start Rails
```bash
bin/rails s
```

Go to: `http://localhost:3000/users`

Now you should see the users & be able to create a few more users.

Feel free to make the GUI nicer!

Assuming all is good:
```bash
git add .
git commit -m "user management page"
```


### Add auth restrictions to Application Controller

This will allow us to control access to all urls within our app (we will also allow exceptions for a landing page)

The application controller ensures only authenticated users (with a session) can access pages - with the following code (especially the `users_only`, but current_user is also very helpful generally)
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :users_only

  def current_user
    # `dig` and `find_by` avoid raising an exception w/o a session
    user_id = session.dig(:user_id)
    @current_user ||= User.find_by(id: user_id)
  end

  private

  # code to ensure only logged in users have access to users pages
  def users_only
    if current_user.blank?
      # send to login page to get an access link
      redirect_back(fallback_location: landing_path,
                    :alert => "Login Required")
      # # uncomment to send people access link page (when built)
      # redirect_back(fallback_location: new_login_path,
      #               :alert => "Login Required")
    end
  end
end
```

Now we should NOT be able to reach our previous pages

`http://localhost:3000/`
`http://localhost:3000/users`
`http://localhost:3000/landing`

Now lets allow access to the landing page again - we need to add:

`skip_before_action :users_only`

to `app/controllers/landing_controller.rb` in order to allow unathenticated access.
```ruby
# app/controllers/landing_controller.rb
class LandingController < ApplicationController
  skip_before_action :users_only
  def index
  end
end
```

Assuming that works:
```bash
git add .
git commit -m "restrict access w/exception"
```


### Create a Users Homepage

Now that we have a public home / default page - lets make an authenticed (user) homepage - where we auto-redirect people on login.

```bash
bin/rails g controller home index --helper false
```

update the routes page with the following (I'm not a fan of including the `index` in the url)
```ruby
# config/routes.rb
Rails.application.routes.draw do
  get '/home',    to: 'home#index',     as: :home
  resources :users
  get '/landing', to: 'landing#index',  as: :landing
  root to: "landing#index"
end

```

Now lets check all is well with the routes:
```bash
bin/rails routes | grep home
# should show
  home  GET   /home(:format)    home#index
```

Assuming the routes are correct when we try to go to:

`http://localhost:3000/home`

we should end up at (be redirected to):

`http://localhost:3000/landing`


assuming this works well:
```bash
git add .
git commit -m "restricted user home page"
```

### Optional - setup a mail trap (MailHog)

I like to view the emails in a browser to check the look as well as the content, for this quick blog - just viewing the info in the logs is good enough.  However, in case you are interested a quick mini MailHog tutorial (for a Mac):

Install mailhog:
```bash
brew install mailhog
mailhog
```

now open `config/environments/development.rb`

and add the following mail settings (for development):
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

at this point you will be able to go to: `http://localhost:8025/` and you should see a webpage that looks like a simple mailreader.  In the future, when you send an email from rails it should be available here.


### Create an emailer to send Access-Links

We need a way to send the login link - so we will create a login mailer with:
```bash
bin/rails generate mailer Login send_link --helper false
```

Configure our emailer for our needs to send the login link:
```ruby
# app/mailer/login_mailer.rb
class LoginMailer < ApplicationMailer
  def send_link(user, login_url)
    @user = user
    @login_url  = login_url
    host = Rails.application.config.hosts.first

    mail(to: @user.email, subject: "Access-Link for #{host}")
  end
end
```

Now we need to create the mailer views to send the url with the access token

The HTML view:
```ruby
# app/views/login_mailer/send_link.html.erb
<h1>Hi <%= @user.email %>,</h1>

<p><a href="<%= @login_url %>">Access-Link for <%= @host %></a></p>

<p> <%= @login_url %> </p>

<p>Link is valid for about an hour from <%= DateTime.now %></p>
```

The text view:
```ruby
# app/views/login_mailer/send_link.text.erb
Hi <%= @user.email %>,

Access-Link for <%= @host %> is:

<%= @login_url %>

Link is valid for about an hour from <%= DateTime.now %>.
```

Again feel free to make these pages more beautiful with CSS (Bulma or Tailwind are my favorites)

Lets test our mailer with the rail console:
```ruby
bin/rails c

# for testing we don't care much which user we pick
user = User.first
# we will just send a 'fake url' - we are just testing our mail sending
url  = "http://localhost:3000/landing"

# we should should now be able to send the mail with:
LoginMailer.send_link(user, url).deliver_later
```

Now you should be able to see that the mail was send from the console output or by going to: `http://localhost:8025/` if you are running mailhog and see the email sent.

Assuming that work:
```bash
git add .
git commit -m "Login URL mailer"
```


### Create an Session Authorization Controller

For the session controller we don't need views or anything else so we can just create the controller file directly.

```bash
touch app/controllers/sessions_controller.rb
```

```ruby
class SessionsController < ApplicationController
  skip_before_action :users_only, only: :create

  def create
    token = params[:token].to_s
    # find the user with a matching token and with current-time < token_expired_at
    user = User.where(token: token)
               .where('users.token_expires_at > (?)', DateTime.now)
               .first
    if user
      # create the session id for current_user to access
      session[:user_id] = user.id
      # send the user to their homepage (or where erver you prefer)
      redirect_to(home_path, notice: "Welcome back #{user.name}")
    else
      flash[:alert] = 'Oops - a valid login link is required'
      redirect_to(landing_path)
      # when the login request page is built it might make sense to redirect to:
      # redirect_to(new_login_path)
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

Add to routes (I am using a get for the create instead of a post verb - since I don't know of a way to make a text url embed a post verb) - so we will add:
```ruby
  get '/sessions/:token', to: 'sessions#create',  as: :create_session
  resources :sessions,    only: [:destroy]
```
to the routes file:

Now the routes should look something like:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  # use get to create since I don't think a text url can create a post
  get '/sessions/:token', to: 'sessions#create',  as: :create_session
  resources :sessions,    only: [:destroy]
  get '/home',    to: 'home#index',     as: :home
  resources :users
  get '/landing', to: 'landing#index',  as: :landing
  root to: "landing#index"
end
```

now if we check the session routes we should see something like:
```bash
bin/rails routes | grep session
  create_session    GET     /sessions/:token(.:format)   sessions#create
         session    DELETE  /sessions/:id(.:format)      sessions#destroy
```

**OPTIONAL:** by default rails sessions have no expiration, thus are deleted when the browser closes. To change this default behavior, we need to create the file.
```bash
touch config/initializers/session_store.rb
```

Now you can set the session length (time until a new login is required) with the setting:
```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store, expire_after: 14.days
```
you might want to use 2 weeks or 4 weeks - whatever you and your users are comfortable with before forcing a new login (if unsure - go with a shorter time-frame)

Let's setup a user with a known valid token and test our new session controller:
```ruby
bin/rails c

user = User.first
# the token length isn't so important but should be enough to make guessing very hard
user.token = SecureRandom.hex(50)  # be sure to use a url-safe random-generator
# expiration time should be relatively short - email is generally not encrypted
user.token_expires_at = DateTime.now + 1.hour
user.save
user.reload

# generate the URL for the session path
# (we need to give the full rails path to the url_helpers since we don't have the controller loaded)
url = Rails.application.routes.url_helpers.create_session_url(token: user.token, host: 'localhost:3000')

# copy the above url into the browser
```

Now when we enter the url generated in the email (or click on the link in mailhog), we should be redirected to the "home" page `http://localhost:3000/home`

Assumeing that works:
```bash
git add .
git commit -m "session controller (login)"
```

### Access-Link Generation

We will need to allow the user to request an access-link.

#### Login Controller

Now lets create a user login controller:
```bash
bin/rails g controller Logins new create --helper false
```

Now we need the Logins Controller login AND be sure to add

`skip_before_action :users_only`

so this page is always available!

Also note the code is similar to what we entered previously in the console.
```ruby
# app/controllers/users/logins_controller.rb
class LoginsController < ApplicationController
  # we need to skip the users only check so this pages can be accessed
  skip_before_action :users_only

  def new
    user = User.new
    render :new, locals: {user: user}
  end

  def create
    email = user_params[:email]

    # we may or may not find a user
    user = User.find_by(email: email)

    # always take the time to calculate token info (discourages email fishing)
    token = SecureRandom.hex(50)
    # besure to use NOW and not NEW!
    token_expires_at = DateTime.now + 1.hour
    token_params = {token: token, token_expires_at: token_expires_at}

    # if we have a user and the update is successful
    if user && user.update(token_params)
      access_url = create_session_url(token: user.token)
      LoginMailer.send_link(user, access_url).deliver_later
    end

    # # uncomment to add noise to discourage response time monitoring
    # # in order to mine user emails
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
**NOTE:**  In real projects I tend to put all my business logic in a `command` or `service` class -- I like skinny models and skinny controllers)

We need to add the route to the users login_controller with:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :logins,      only: [:new, :create]
  # use get to create since I don't think a text url can create a post
  get '/sessions/:token', to: 'sessions#create',  as: :create_session
  resources :sessions,    only: [:destroy]
  get '/home',    to: 'home#index',     as: :home
  resources :users
  get '/landing', to: 'landing#index',  as: :landing
  root to: "landing#index"
end
```

now check the routes:
```bash
bin/rails routes | grep logins
# should return
     logins   POST  /logins(.:format)       logins#create
  new_login   GET   /logins/new(.:format)   logins#new
```

Lets go to the new page `http://localhost:3000/logins/new` and be sure that we can get access that page.

Now login into the console and check the user attributes.
```ruby
bin/rails c
user = User.find_by(email: 'test1.test.ch') # or whatever email you used
user.token # be sure it updated with the same key as in the logs
user.token_expires_at # should be an hour into the future
# if the date is the year 0000 - then you used .new (with an `e`) instead of .now
```

Assuming this works:
```bash
git add .
git commit -m "login controller"
```


#### Login Form

Login email form - we only need `app/views/logins/new.html.erb`

We can delete `app/views/logins/create.html.erb` as that just posts to an action and then redirects to our user's home_path

```ruby
# app/views/ogins/new.html.erb
<%= form_for(user, local: true,
             url: logins_path, # NEW MUST BE PLURAL for POST
             id: "login-form", class: "user" ) do |form|  %>

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
PS - I dislike using instance variables (and often use 'input' classes) with my forms - this is why this form looks a little different than standard rails.

Note I often use Bulma - so here is how I like to format my forms (without Bulma installed the form will be ugly).

Let's test this code:

First:
- go to: `http:localhost:3000/home`
- hopefully your are redirected to: `http:localhost:3000/landing`

Now:
- go to: `http:localhost:3000/logins/new`
- enter a user's email address
- find the login url generated in the email
- enter that login_url in the browser - (ideally click on the link in mailhog - much like a 'real user' would do)
- hopefully you are now redirected to `http:localhost:3000/home`

Assuming this works:
```bash
git add .
git commit -m "login form and create action with redirect"
```

## Resources
s
Code Repository is at:

**Rails GlobalID**

The nice thing about these is that the auto expire - simplifying the code and the usermodel.

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
