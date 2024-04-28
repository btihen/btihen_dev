---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x Secured Namespaced Controllers"
subtitle: "Organizing and Securing Controllers by Usage"
summary: ""
authors: ['btihen']
tags: ['Rails', 'Security', 'Design', 'Organization','Controllers', 'Scope', 'Namespace']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2024-04-27T01:20:00+02:00
lastmod: 2024-04-28T01:20:00+02:00
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

I have found in large code bases Modules and namespaces are critical.

Here is a simple way to build a secured admin panel using namespaces.

This code can be found at: https://github.com/btihen-dev/rails_secured_namespace

## Getting Started

I will create a basic application using:
```bash
rails new secured_namespace -d=postgresql -T
cd secured_namespace

bin/rails g scaffold Contact email first_name last_name message:text

bin/rails db:create
bin/rails db:migrate
```

for simplicity update the `root` route with `root "contacts#new"` contacts as your landing page i.e.:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :contacts

  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  # Defines the root path route ("/")
  root "contacts#new"
end
```

be sure this all works and commit:
```bash
git add .
git commit -m "add new contact to home page"
```

In fact, we only only want users to submit a contact request and noting else (via HTML) - so lets simplify the controller and views to:
```ruby
# app/controllers/contacts_controller.rb
class ContactsController < ApplicationController
  def new
    @contact = Contact.new
  end

  def create
    @contact = Contact.new(contact_params)

    if @contact.save
      redirect_to root_path, notice: "Contact was successfully created."
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  # Only allow a list of trusted parameters through.
  def contact_params
    params.require(:contact).permit(:email, :first_name, :last_name, :message)
  end
end
```

now we can also delete all the unreferenced views
```bash
├── Dockerfile
├── Gemfile
├── Gemfile.lock
├── README.md
├── Rakefile
├── app
│   ├── controllers
│   │   ├── application_controller.rb
│   │   ├── concerns
│   │   └── contacts_controller.rb
│   ...
│   └── views
│       ├── contacts
│       │   ├── _form.html.erb
│       │   └── new.html.erb
│       └── layouts
│           ├── application.html.erb
│           ├── mailer.html.erb
│           └── mailer.text.erb
```

test again and be sure all still works and commit:
```bash
git add .
git commit -m "restrict user to submitting a contact request"
```

## Admin Panel

Now of course we need to access these contacts - and we will restrict this to logged in admins.  Let's build a namespaced controller,

```bash
rails g scaffold_controller admin/contact
```

Now we need to make some adjustments, let's start with thee controller. we need to replace: `Admin::Contacts.` with `Contact.` and
```ruby
# app/controllers/admin/contacts_controller.rb
...

  # Only allow a list of trusted parameters through.
  def admin_contact_params
    params.fetch(:admin_contact, {})
  end
end
```
with:
```ruby
# app/controllers/admin/contacts_controller.rb
...

  # Only allow a list of trusted parameters through.
  def admin_contact_params
    params.require(:contact).permit(:email, :first_name, :last_name, :message)
  end
end
```

so the controller now looks like:
```ruby
# app/controllers/admin/contacts_controller.rb
class Admin::ContactsController < ApplicationController
  before_action :set_admin_contact, only: %i[ show edit update destroy ]

  # GET /admin/contacts or /admin/contacts.json
  def index
    @admin_contacts = Contact.all
  end

  # GET /admin/contacts/1 or /admin/contacts/1.json
  def show
  end

  # GET /admin/contacts/new
  def new
    @admin_contact = Contact.new
  end

  # GET /admin/contacts/1/edit
  def edit
  end

  # POST /admin/contacts or /admin/contacts.json
  def create
    @admin_contact = Contact.new(admin_contact_params)

    respond_to do |format|
      if @admin_contact.save
        format.html { redirect_to admin_contact_url(@admin_contact), notice: "Contact was successfully created." }
        format.json { render :show, status: :created, location: @admin_contact }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @admin_contact.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /admin/contacts/1 or /admin/contacts/1.json
  def update
    respond_to do |format|
      if @admin_contact.update(admin_contact_params)
        format.html { redirect_to admin_contact_url(@admin_contact), notice: "Contact was successfully updated." }
        format.json { render :show, status: :ok, location: @admin_contact }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @admin_contact.errors, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /admin/contacts/1 or /admin/contacts/1.json
  def destroy
    @admin_contact.destroy!

    respond_to do |format|
      format.html { redirect_to admin_contacts_url, notice: "Contact was successfully destroyed." }
      format.json { head :no_content }
    end
  end

  private

  # Use callbacks to share common setup or constraints between actions.
  def set_admin_contact
    @admin_contact = Contact.find(params[:id])
  end

  # Only allow a list of trusted parameters through.
  def admin_contact_params
    params.require(:contact).permit(:email, :first_name, :last_name, :message)
  end
end
```

Now in the views we need to rename:
`app/views/admin/contacts/_contact.html.erb`
to
`app/views/admin/contacts/_admin_contact.html.erb`
and fill it with:
```ruby
# app/views/admin/contacts/_admin_contact.html.erb
<div id="<%= dom_id admin_contact %>">
  <p>
    <strong>Email:</strong>
    <%= admin_contact.email %>
  </p>

  <p>
    <strong>First name:</strong>
    <%= admin_contact.first_name %>
  </p>

  <p>
    <strong>Last name:</strong>
    <%= admin_contact.last_name %>
  </p>

  <p>
    <strong>Message:</strong>
    <%= admin_contact.message %>
  </p>
</div>
```

now we can adjust `index` to use this partial - we need to change:
`<%= render admin_contact %>`
to
`<%= render 'admin_contact', admin_contact: admin_contact %>`
and
`<%= link_to "Show", admin_contact %>`
to
`<%= link_to "Show", admin_contact_path(admin_contact) %>`
I also like to use edit instead of show - so now the index page looks like:
```ruby
# app/views/admin/contacts/index.html.erb
<p style="color: green"><%= notice %></p>

<h1>Contacts</h1>

<div id="admin_contacts">
  <% @admin_contacts.each do |admin_contact| %>
    <%= render 'admin_contact', admin_contact: admin_contact %>
    <p>
    <%= link_to "Show", admin_contact_path(admin_contact) %>
    <%= link_to "Edit", edit_admin_contact_path(admin_contact) %>
    </p>
  <% end %>
</div>

<%= link_to "New contact", new_admin_contact_path %>
```

let's check `show` too - again we need to change
`<%= render admin_contact %>`
to
`<%= render 'admin_contact', admin_contact: admin_contact %>`


now we need to update the admin forms - since rails assumes our variable will also be scoped (even though its not) so we pass the form the action's URL so we changeL
`<%= form_with(model: admin_contact) do |form| %>`
to
`<%= form_with(model: admin_contact, url: form_url) do |form| %>`

thus it now looks like:

```ruby
# app/views/admin/contacts/_form.html.erb
<%= form_with(model: admin_contact, url: form_url) do |form| %>
  <% if admin_contact.errors.any? %>
    <div style="color: red">
      <h2><%= pluralize(admin_contact.errors.count, "error") %> prohibited this admin_contact from being saved:</h2>

      <ul>
        <% admin_contact.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= form.submit %>
  </div>
<% end %>
```

now we can pass the url to the form - in `edit` and `new` so the now call to the for would look like - to figure out the route path type:
`bin.rails routes`
and math the action to the form.
```ruby
<%= render(
      "form",
      admin_contact: @admin_contact,
      form_url: admin_contact_path(@admin_contact) # PATCH /admin_contact => admin_contacts#update
    ) %>
```
now all-to gether im the form:
```ruby
# app/views/admin/contacts/_form.html.erb

<%= form_with(model: admin_contact, url: form_url) do |form| %>

<% if admin_contact.errors.any? %>
    <div style="color: red">
      <h2><%= pluralize(admin_contact.errors.count, "error") %> prohibited this contact from being saved:</h2>

      <ul>
        <% admin_contact.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= form.label :email, style: "display: block" %>
    <%= form.text_field :email %>
  </div>

  <div>
    <%= form.label :first_name, style: "display: block" %>
    <%= form.text_field :first_name %>
  </div>

  <div>
    <%= form.label :last_name, style: "display: block" %>
    <%= form.text_field :last_name %>
  </div>

  <div>
    <%= form.label :message, style: "display: block" %>
    <%= form.text_area :message %>
  </div>

  <div>
    <%= form.submit %>
  </div>
<% end %>
```

Now the `edit` view:
```ruby
# app/views/admin/contacts/edit.html.erb
<h1>Editing contact</h1>

<%= render(
      "form",
      admin_contact: @admin_contact,
      # PATCH /admin_contact => admin_contacts#update
      form_url: admin_contact_path(@admin_contact)
    ) %>

<br>

<div>
  <%= link_to "Show this contact", admin_contacts_path(@admin_contact) %> |
  <%= link_to "Back to contacts", admin_contacts_path %>
</div>
```

and now in `new`

```ruby
# app/views/admin/contacts/new.html.erb
<h1>New contact</h1>

<%= render(
            "form",
            admin_contact: @admin_contact,
            # POST /admin_contacts => admin_contacts#create
            form_url: admin_contacts_path(@admin_contact)
          )%>

<br>

<div>
  <%= link_to "Back to contacts", admin_contacts_path %>
</div>
```

and show also needs updating to all allow deletions:
```ruby
#
<p style="color: green"><%= notice %></p>

<%= render 'admin_contact', admin_contact: @admin_contact %>

<div>
  <%= link_to "Edit", edit_admin_contact_path(@admin_contact) %> |
  <%= link_to "List contacts", admin_contacts_path %>

  <%= button_to "Destroy", admin_contacts_path(@admin_contact), method: :delete %>
</div>
```

Now if we go to: `localhost:3000/admin/contacts` we should be able to see and edit out contacts

let's commit:
```bash
git add .
git commit -m "admin section with full management
```

## Simple Authentication & Authorization
Let's restrict the admin page to logged in users.

We can easily do this with Devise: `https://github.com/heartcombo/devise` which has great instructions which I will summarize here.

```bash
bundle add devise
rails generate devise:install

# in config/environments/development.rb:
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }

rails generate devise User
bin/rails db:migrate

# in app/views/layouts/application.html.erb.
<p class="notice"><%= notice %></p>
<p class="alert"><%= alert %></p>
# to this area:
<body>
  <p class="notice"><%= notice %></p>
  <p class="alert"><%= alert %></p>
  <%= yield %>
</body>

# we need to generate the views to remove the signup links
rails g devise:views
```

Now you can adjust the how devise works with the user in the User model in this case we do NOT want people to register on their own so we will remove `registerable` in:
```ruby
# app/models/user.rb
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, # :registerable,
         :recoverable, :rememberable, :validatable

  normalizes :email, with: -> email { email.downcase.strip }
end
```
Devise without sign-up article:
https://medium.com/@tdaniel/removing-user-registration-from-devise-3baa8d1738be

Now we want to remove the registerable links and forms so we change the line:
`devise_for :users`
to
`devise_for :users, :skip => [:registrations]`

now routes should look like
```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users, :skip => [:registrations]

  namespace :admin do
    resources :contacts
  end
  resources :contacts

  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  # Defines the root path route ("/")
  root "contacts#new"
end
```

Now we need to remove the `sign-up` / `new_registration` link in `app/views/devise/shared/_links.html.erb` as the following code will now error:
```ruby
<%- if devise_mapping.registerable? && controller_name != 'registrations' %>
  <%= link_to "Sign up", new_registration_path(resource_name) %><br />
<% end %>
```

so now this file should look like:
```ruby
# app/views/devise/shared/_links.html.erb
<%- if controller_name != 'sessions' %>
  <%= link_to "Log in", new_session_path(resource_name) %><br />
<% end %>

<%- if devise_mapping.recoverable? && controller_name != 'passwords' && controller_name != 'registrations' %>
  <%= link_to "Forgot your password?", new_password_path(resource_name) %><br />
<% end %>

<%- if devise_mapping.confirmable? && controller_name != 'confirmations' %>
  <%= link_to "Didn't receive confirmation instructions?", new_confirmation_path(resource_name) %><br />
<% end %>

<%- if devise_mapping.lockable? && resource_class.unlock_strategy_enabled?(:email) && controller_name != 'unlocks' %>
  <%= link_to "Didn't receive unlock instructions?", new_unlock_path(resource_name) %><br />
<% end %>

<%- if devise_mapping.omniauthable? %>
  <%- resource_class.omniauth_providers.each do |provider| %>
    <%= button_to "Sign in with #{OmniAuth::Utils.camelize(provider)}", omniauth_authorize_path(resource_name, provider), data: { turbo: false } %><br />
  <% end %>
<% end %>

```

let's check that there is no sign-up route with:
```bash
bin/rails routes

Prefix               Verb   URI Pattern                    Controller#Action
-------------------------------------------------------------------
new_user_session     GET    /users/sign_in(.:format)       devise/sessions#new
user_session         POST   /users/sign_in(.:format)       devise/sessions#create
destroy_user_session DELETE /users/sign_out(.:format)      devise/sessions#destroy
new_user_password    GET    /users/password/new(.:format)  devise/passwords#new
edit_user_password   GET    /users/password/edit(.:format) devise/passwords#edit
user_password        PATCH  /users/password(.:format)      devise/passwords#update
                     PUT    /users/password(.:format)      devise/passwords#update
                     POST   /users/password(.:format)      devise/passwords#create
```

now it is important to restart rails if already running!

Let's create a new user from the Command line (since we can no longer register via a web interface):
```ruby
User.create!({:email => "test@example.com", :password => "1123456789-asdfg", :password_confirmation => "1123456789-asdfg" })
```
If you have `confirmable` module enabled for devise, make sure you are setting the `confirmed_at` value to something like Time.now while creating. In our case this should not be the case.

The easiest way to secure the entire application is to now add:
 `before_action :authenticate_user!` to
our application_controller.
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ApplicationController
  before_action :authenticate_user!
end
```

And the add ` skip_before_action :authenticate_user!` to our root page so that it can be accessed without a login, so it now looks like:

```ruby
# app/controllers/contacts_controller.rb
class ContactsController < ApplicationController
  skip_before_action :authenticate_user!

  def new
    @contact = Contact.new
  end

  def create
    @contact = Contact.new(contact_params)

    if @contact.save
      redirect_to root_path, notice: "Contact was successfully created."
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  # Only allow a list of trusted parameters through.
  def contact_params
    params.require(:contact).permit(:email, :first_name, :last_name, :message)
  end
end
```

one last item to add is a logout link - this must be done since it is a delete and not a get - so just going to a link doesn't work.  So we can add:
` <%= current_user ? button_to("Logout: #{current_user.email}", destroy_user_session_path, method: :delete) : button_to("Login", new_user_session_path) %>`
or if you don't want a sign-in just:
` <%= button_to("Logout: #{current_user.email}", destroy_user_session_path, method: :delete) if current_user %>`

NOTE: most devise redirects are broken due to the way Turbo catches them.
One fix is to use:
`<%= link_to "Sign out", destroy_user_session_path, data: { turbo_method: :delete } %>`
or with the following format:
`<%= link_to "Log Out", destroy_user_session_path, 'data-turbo-method': :delete %>`
instead of:
`<%= link_to "Sign out", destroy_user_session_path, method: delete %>`
Another fix is to use `button_to` i.e.
`button_to("Logout: #{current_user.email}", destroy_user_session_path, method: :delete)`
instead of `link_to` - which just sends a GET verb instead of a DELETE.
`link_to("Logout: #{current_user.email}", destroy_user_session_path, method: :delete)`

NOTE: if any Devise or other link is getting messed up, adding:
`, data: { turbo_method: :delete } `
or
`, 'data-turbo-method': :delete`
to you link is likely to fix the problem.

```ruby
# app/views/layouts/application.html.erb
<!DOCTYPE html>
<html>
  <head>
    <title>SecuredNamespace</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>

  <body>
    <p class="notice"><%= notice %></p>
    <p class="alert"><%= alert %></p>
    <%= current_user ? button_to("Logout: #{current_user.email}", destroy_user_session_path, method: :delete) : link_to("Signin", new_user_session_path) %>
    <%= yield %>
  </body>
</html>
```

now test and be sure that when you login you can see the admin pages and when you are not logged in you cannot see them (& that you still have access to the landing page/root page)

```bash
git add .
git commit -m "admin page is restricted"
```

I'll add this code to the repo and let the reader try this independently.


## Separating Users and Admins

If you would like multiple levels of access we can add a roles attribute to our users.

let's make a user migration to add an array of roles to our user with the default rule as a user and admin as an option.

```bash
rails g migration add_roles_to_user roles:text
```

now we need to adjust our migration a bit to tell postgressql to accept an array - change:
`add_column :users, :roles, :text`
to
`add_column :users, :roles, :text, array: true, default:  ['user']`
now the migration should look like:
```ruby
class AddRolesToUser < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :roles, :text, array: true, default: ['user']
  end
end
```
this will make sure our existing users are a `user` and we can now make a new `admin` account.

now let's migrate:
```
bin/rails db:migrate
```

let's sanitize the user roles by adding
`normalizes :roles, with: -> roles { roles.map { |r| r.blank? ? 'user' : r }.uniq }`
to the user model so that in now looks like

```ruby
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, # :registerable,
         :recoverable, :rememberable, :validatable

  normalizes :email, with: -> email { email.downcase.strip }
  normalizes :roles, with: -> roles { roles.map { |r| r.blank? ? 'user' : r }.uniq }
end
```

now we can create a new admin user from the Command line using:
```ruby
User.create!({:email => "admin@example.com", :roles => ["admin"], :password => "987654321-lkjhg", :password_confirmation => "987654321-lkjhg" })
```

now lets create a new admin application controller to user only admins have access:
```ruby
# app/controllers/admin/application_controller.rb
class Admin::ApplicationController < ApplicationController
  before_action :authenticate_admin!

  private

  def authenticate_admin!
    # if not logged will be redirected to login page
    authenticate_user!
    # if not an admin will be redirected to root page
    redirect_to root_path, alert: 'not authorized' unless current_user.roles.include?('admin')
  end
end
```


Now any controller that inherits from `Admin::ApplicationController` will requite authentication!

so now we can adjust existing admin controller from
`class Admin::ContactsController < ApplicationController`
to:
`class Admin::ContactsController <  Admin::ApplicationController`

thus the controller now now looks like:

```ruby
# app/controllers/admin/contacts_controller.rb
class Admin::ContactsController <  Admin::ApplicationController
  before_action :set_admin_contact, only: %i[ show edit update destroy ]

  def index
    @admin_contacts = Contact.all
  end

  def show
  end

  def new
    @admin_contact = Contact.new
  end

  def edit
  end

  def create
    @admin_contact = Contact.new(admin_contact_params)

    respond_to do |format|
      if @admin_contact.save
        format.html { redirect_to admin_contact_url(@admin_contact), notice: "Contact was successfully created." }
        format.json { render :show, status: :created, location: @admin_contact }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @admin_contact.errors, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @admin_contact.update(admin_contact_params)
        format.html { redirect_to admin_contact_url(@admin_contact), notice: "Contact was successfully updated." }
        format.json { render :show, status: :ok, location: @admin_contact }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @admin_contact.errors, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @admin_contact.destroy!

    respond_to do |format|
      format.html { redirect_to admin_contacts_url, notice: "Contact was successfully destroyed." }
      format.json { head :no_content }
    end
  end

  private

  # Use callbacks to share common setup or constraints between actions.
  def set_admin_contact
    @admin_contact = Contact.find(params[:id])
  end

  # Only allow a list of trusted parameters through.
  def admin_contact_params
    params.require(:contact).permit(:email, :first_name, :last_name, :message)
  end
end
```

be sure you test that all this works as expected and commcontactsit:
```bash
git add .
git commit -m "add roles to allow for other namespaces"
```

now you can also make controllers & resources available within additional namespaces beyond just `admin` and provide different levels of access.

## USER ADMIN PANEL

Now that we have `admin` and `user` roles - we can follow most of the same steps we used for contacts to build an admin page for users. We will start with:
```bash
rails g scaffold_controller admin/user
```

Here I will only provide a few notes for the differences that are a little tricky:

1. roles is an array so we need to use strong params differently in this case we need to tell rails to expect an array by using `roles: []` so the file looks like:
```ruby
# app/controllers/admin/users_controller.rb
  def admin_user_params
    params.require(:user).permit(:email, roles: [])
  end
```

2. the form field for `roles` is a multi-select so `:multiple => true` is important - so the `_form` needs to use something like:
```ruby
# app/views/admin/users/_form.html.erb
  <div>
    <%= form.label :roles, style: "display: block" %>
    <%= form.select :roles,
                    ['admin', 'user'],
                    :prompt => "Select Roles",
                    :multiple => true
    %>
  </div>
```

If any of this is still confusing you can refer to the repo with a full working solution at: https://github.com/btihen-dev/rails_secured_namespace

## RESOURCES

* https://stackoverflow.com/questions/4316940/create-a-devise-user-from-ruby-console
* https://medium.com/@tdaniel/removing-user-registration-from-devise-3baa8d1738be
* https://stackoverflow.com/questions/34908931/devise-and-rails-how-to-secure-login-url
* https://stackoverflow.com/questions/36128818/devise-skip-authentication-based-on-route
* https://stackoverflow.com/questions/32409820/add-an-array-column-in-rails
* https://dev.to/spaquet/rails-7-devise-log-out-d1n
* https://chimkan.com/rails-7-and-devise-issue-with-sign-out/
* https://www.ruby-forum.com/t/how-do-i-select-multiple-things-in-a-form/70186/3
* https://bhartee-tech-ror.medium.com/creating-a-rails-7-application-with-devise-and-roles-15fdac91ebd4
