---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 6.x - Framework Agnostic Associations - part 1"
subtitle: "Basic Associations: belongs_to and has_many"
summary: "Framework Agnostic Associations - Data models that work across many frameworks"
authors: ["btihen"]
tags: ['Ruby', 'Databases', 'Data models', 'Framework Agnostic', 'belongs_to', 'has_many']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-05-19T01:57:00+02:00
lastmod: 2021-08-07T01:57:00+02:00
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
## Purpose

In the interest of coding Rails in a way to work well with other code bases, I looking at ways to do complex database relations in a framework agnostic way.  In particular, this article will primarily explore Polymorphic Relationships.

This is the second article in the series.  This article is followed up with (part 2)[post_ruby_rails/rails_6_x_agnostic_associations_2/]

## Overview

In this case, I want to model a contact list of businesses and people.  Some people will be associated with a company.  Additionally, we will track transactions with each person and business.

The basic model will then look something like:
```
                       ┌───────────┐           ┌───────────┐
                       │           │╲         ╱│           │
      ┌──────────────○┼│  Contact  │───────────│UserContact│
      │                │           │╱         ╲│           │
      │                └───────────┘           └───────────┘
      │                      ┼                      ╲│╱
      │                      ○                       │
      │                      │                       │
      │                      │                       │
     ╱│╲                    ╱│╲                      │
┌───────────┐          ┌───────────┐                 │
│           │╲         │           │                 │
│ Business  │─○───────┼│  Person   │                 │
│           │╱         │           │                 │
└───────────┘          └───────────┘                 │
     ╲│╱                    ╲│╱                      │
      │                      │                       │
      │                      │                       │
      │                      ○                       │
      │                      ┼                      ╱│╲
      │                ┌───────────┐           ┌───────────┐
      │                │           │          ╱│           │
      └──────────────○┼│  Remark   │┼──────────│   User    │
                       │           │          ╲│           │
                       └───────────┘           └───────────┘
                    Created with Monodraw
```

## Create a default Rails app

```bash
rails new rails_poly
cd rails_poly
bin/rails db:create
bin/rails db:migrate
git add .
git commit -m "initial commit"
```

## Starting Simple - optional relations

### Build Businesses

Lets start with the simple relationship between businesses and people:
```
┌────────────┐             ┌───────────┐
│            │╲          1 │           │
│  Business  │─○──────────┼│  Person   │
│-legal_name │╱0..*        │-full_name │
└────────────┘             └───────────┘
```

For expedience, I'll use scaffolds:

Generating a simple business model.
```bash
rails g scaffold Business legal_name
```

Lets adjust the migration to require the business' legal name, by adding `null: false` to the name:
```ruby
# db/migrate/20210516080420_create_businesses.rb
class CreateBusinesses < ActiveRecord::Migration[6.1]
  def change
    create_table :businesses do |t|
      t.string :legal_name, null: false

      t.timestamps
    end
  end
end
```


Now we will validate the business' name in the model:
```ruby
# app/models/business.rb
class Business < ApplicationRecord
  validates :legal_name, presence: true
end
```

Now lets be sure we can migrate:
```bash
bin/rails db:migrate
```

lets use seed to quickly check our models and relations (& get an idea of how to use them):
```ruby
# db/seeds.rb
business = Business.create(legal_name: "Business")
```

Lets check the seed with:
```bash
bin/rails db:seed
```

Assuming this works, let's see the "/businesses" page:
```bash
bin/rails s
open localhost:3000/businesses/
```

Great - lets snapshot:
```bash
git add .
git commit -m "created business model"
```

### Build People

Now let's build the person model and its relations to businesses.
```bash
rails g scaffold Person full_name business:references
```

In this case we want the person to optionally be a member of a business, so lets update the both the models and the migration.  Starting with the migration, we need to remove `null: false` in the foreign key, and add that to the name - so it should now look like:
```ruby
# db/migrate/20210516080414_create_people.rb
class CreatePeople < ActiveRecord::Migration[6.1]
  def change
    create_table :people do |t|
      t.string :full_name, null: false
      t.references :company, foreign_key: true

      t.timestamps
    end
  end
end
```

Now lets adjust the person model - we'll make the relation optional with `optional: true` and require the name with the validation `validates :full_name, presence: true`, so it should now look like:
```ruby
# app/models/person.rb
class Person < ApplicationRecord
  belongs_to :company, optional: true

  validates :full_name, presence: true
end
```

And lets let the Business know it can have lots of people with `has_many :people` - now the model will look like:
```ruby
# app/models/business.rb
class Business < ApplicationRecord
  has_many :people

  validates :legal_name, presence: true
end
```

Lets check the migrations work:
```bash
bin/rails db:migrate
```


lets use seed a couple of people too - so it now looks like:
```ruby
# db/seed.rb
business = Business.create(legal_name: "Business")
company = Business.create(legal_name: "Company")

company.build_person(full_name: "Company Man")
company.save

person = Person.create(full_name: "Own Person")
```

Lets check the seed with:
```bash
bin/rails db:seed
```

Now lets check our pages again:
```bash
bin/rails s
open localhost:3000
```

Lets check the index pages

On the business page it would be nice to see how many employees - so we can update the model with:
```ruby
# app/models/business.rb
class Business < ApplicationRecord
  has_many :people

  validates :legal_name, presence: true

  def people_count
    people.count
  end
end
```

And now `people_count` is added as a virtual attribute (as well as all other business fields because of `'businesses.*`) - now we can use in our view using = `<td><%= business.people_count %></td>` so now it would look something like:
```ruby
# app/views/businesses/index.html.erb
<h1>Businesses</h1>

<table>
  <thead>
    <tr>
      <th>Legal name</th>
      <th>Employee Count</th>
      <th colspan="3"></th>
    </tr>
  </thead>

  <tbody>
    <% @businesses.each do |business| %>
    <tr>
      <td><%= business.legal_name %></td>
      <td><%= business.people_count %></td>
      <td><%= link_to 'Show', business %></td>
      <td><%= link_to 'Edit', edit_business_path(business) %></td>
      <td><%= link_to 'Destroy', business, method: :delete, data: { confirm: 'Are you sure?' } %></td>
    </tr>
    <% end %>
  </tbody>
</table>
```

and on the '/people' page it would be nice to see there business name instead of id.

so in the model:
```ruby
# app/model/person.rb
class Person < ApplicationRecord
  belongs_to :business, optional: true

  validates :full_name, presence: true

  def associated_business_name
    business&.legal_name
  end
end
```

and in the index view:
```ruby
# app/views/people/index.html.erb
<table>
  <thead>
    <tr>
      <th>Full name</th>
      <th>Business</th>
      <th colspan="3"></th>
    </tr>
  </thead>

  <tbody>
    <% @people.each do |person| %>
    <tr>
      <td><%= person.full_name %></td>
      <td><%= person.associated_business_name %></td>
      <td><%= link_to 'Show', person %></td>
      <td><%= link_to 'Edit', edit_person_path(person) %></td>
      <td><%= link_to 'Destroy', person, method: :delete, data: { confirm: 'Are you sure?' } %></td>
    </tr>
    <% end %>
  </tbody>
</table>
```

to show all employees on the business show page we can do:
```ruby
# app/views/businesses/show.html.erb
<p>
  <strong>Legal name:</strong>
  <%= @business.legal_name %>
</p>

<table>
  <thead>
    <tr>
      <th>Employee</th>
    </tr>
  </thead>
  <tbody>
    <% @business.people.each do |person| %>
    <tr><td>person.full_name</td></tr>
    <% end %>
  </tbody>

</table>
```

And now lets look for n+1 queries - to do that we will create many records in the seeds file:
```ruby
# db/seeds.rb
business = Business.create(legal_name: "Business")
company  = Business.create(legal_name: "Company")
boss_man = Person.create(full_name: "Company Man", business: company)
person = Person.create(full_name: "Own Person")

# larger numbers (look for n+1 lookups)
50.times do |business_number|
  company  = Business.create(legal_name: "Company #{business_number}")
  business_number.times do |employee_number|
    Person.create(full_name: "Employee #{employee_number}",
                  business: company)
  end
end
```

Now when we visit '/people' we see an n+1 (to look up the business to get the business name) - this is an easy fix with a pre-load in the controller - just add `.include(:business)` to the query - now the index method will look like
```ruby
# app/controllers/people_controller.rb
class PeopleController < ApplicationController

  def index
    @people = Person.include(:business).all
  end
```



Fix n+1 lookups - for the business employee count is a bit trickier - to avoid lots of look ups we need the db to do the count and add the count as a virtual attribute - this is done with the following query:
```ruby
# app/controllers/people_controller.rb
class BusinessController < ApplicationController

  def index
    # businesses = Business.all  # (N+1 when using referring to people)
    # select must go last or it gets lost / overwritten
    @businesses = Business.joins(:people)
                          .group('businesses.id')
                          .select('businesses.*, count(people.id) as people_count')
  end
```

to avoid confusion - lets rename the method in the class to `employee_count`:
```ruby
# app/models/business.rb
class Business < ApplicationRecord
  has_many :people

  validates :legal_name, presence: true

  def employee_count
    people.count
  end
end
```


lets run the seeds again:
```bash
bin/rails db:seed
```

cool now when we look at the log we just have one query instead of many!

Now let's make the people form to associate a business by name instead of the id!
```ruby
# app/views/people/_form.html.erb
<%= form_with(model: person) do |form| %>
  <% if person.errors.any? %>
    <div id="error_explanation">
      <h2><%= pluralize(person.errors.count, "error") %> prohibited this person from being saved:</h2>

      <ul>
        <% person.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="field">
    <%= form.label :full_name %>
    <%= form.text_field :full_name %>
  </div>

  <div class="field">
    <%= form.label :business %>
    <%= form.select :business_id,
                    Business.all.collect { |b| [ b.legal_name, b.id ] },
                    prompt: "Select One", include_blank: true %>
  </div>

  <div class="actions">
    <%= form.submit %>
  </div>
<% end %>
```

Great - lets snapshot:
```bash
git add .
git commit -m "created person related to businesses - w/o n+1"
```

## Polymorphic (STI) - sometime called inverse polymorphic

              ┌───────────────┐
              │    Contact    │
              │  (functions)+ │ + supplier, reseller, customer, sales-rep
              │(display_name)*│ * virtual attribute
              └───────────────┘
                      ┼ 1
         ┌────────────┴─────────────┐
        ╱│╲ *                    * ╱│╲
┌───────────────┐          ┌───────────────┐
│    Business   │╲       1 │    Person     │
│  (legal_name) │ ○ ─ ─ ─ ┼│  (full_name)  │
│(display_name)*│╱ 0..*    │(display_name)*│
└───────────────┘          └───────────────┘

A contact could be either a person or a business - but must be one or the other.

This is implemented in (part 2)[post_ruby_rails/rails_6_x_agnostic_associations_2/]



## Polymorphic

a model associated with several different models - serving a similar purpose in both cases

      ┌────────────┐          ┌───────────┐
      │            │╲       1 │           │
      │  Business  │ ○ ─ ─ ─ ┼│  Person   │
      │            │╱ 0..*    │           │
      └────────────┘          └───────────┘
            ╲│╱ *                * ╲│╱
             └───────────┬──────────┘
                         ┼ 1
                 ┌──────────────┐
                 │              │
                 │  Transaction │
                 │              │
                 └──────────────┘

A contact could be either a person or a business - but must be one or the other.

```bash
bin/rails g Contact roles:array business:references person:references
```

update the migration to ensure we have a role provided & relations:
```ruby
#
```

update the Contact model with the validations & flexible relations:
```ruby
# contact.rb
```

update the Person model and relations:
```ruby
# person.rb

```

update the Business model and relations:
```ruby
# business.rb

```


lets use seed a couple of people too - so it now looks like:
```ruby
# db/seed.rb
business = Business.create(legal_name: "Business")
company = Business.create(legal_name: "Company")

company.build_person(full_name: "Company Man")
company.save

person = Person.create(full_name: "Own Person")
```

Lets check the seed with:
```bash
bin/rails db:seed
```

Assuming this works, let's see the "/people" page:
```bash
bin/rails s
open localhost:3000/businesses/
```

Great - lets snapshot:
```bash
git add .
git commit -m "created person possibly related to the model"
```
