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
    params.fetch(:admin_contact, {})
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
  <%= link_to "Show this contact", @admin_contact %> |
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

Now if we go to: `localhost:3000/admin/contacts` we should be able to see and edit out contacts

let's commit:
```bash
git add .
git commit -m "admin section with full management
```

## Authentication & Athorization
Lets restrict the admin page if this area an admin - we will use
``
instead.
