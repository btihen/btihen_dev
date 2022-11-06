---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 6.x - Framework Agnostic Associations - part 2"
subtitle: "Aggregating different, but related Data models (Rails STI alternative)"
summary: "Framework Agnostic Associations - Data models that work across many frameworks"
authors: ["btihen"]
tags: ['Ruby', 'Relationships', 'Databases', 'Data models', 'Framework Agnostic', 'belongs_to', 'has_one']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-05-30T01:57:00+02:00
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

This is the second article in the series.  This article builds on (part 1)[post_ruby_rails/rails_6_x_agnostic_associations_1/]

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
```

## Rails app and first Models

```
    ┌────────────┐             ┌───────────┐
    │            │╲          1 │           │
    │  Business  │─○──────────┼│  Person   │
    │-legal_name │╱0..*        │-full_name │
    └────────────┘             └───────────┘
```

We discussed/explained in (part 1)[post_ruby_rails/rails_6_x_agnostic_associations_1/]

## Polymorphic (STI) - sometime called inverse polymorphic

```
                   ┌─────────────┐
                   │   Contact   │
                   │  relations* │
                   │+display_name│
                   └─────────────┘
                          ┼
                          │
          ┌───────────────┴────────────┐
          │                            │
         ╱│╲                          ╱│╲
    ┌─────────────┐             ┌─────────────┐
    │  Business   │╲            │    Person   │
    │ -legal_name │─○──────────┼│ -full_name  │
    │+display_name│╱            │+display_name│
    └─────────────┘             └─────────────┘
  + array: supplier, reseller, customer, sales-rep
  * virtual attribute (public method)
```

We disucssed/explained this in (part 2)[post_ruby_rails/rails_6_x_agnostic_associations_2/]

## Polymorphic Modeling

Is a model that can be associated with several different models - serving a similar purpose in all cases.  For example perhaps we would like to leave remarks on our interactions with various other business partners as shown below.
```
┌───────────┐          ┌───────────┐
│           │╲         │           │
│ Business  │─○───────┼│  Person   │
│           │╱         │           │
└───────────┘          └───────────┘
     ╲│╱                    ╲│╱
      │                      │
      │                      │
      │                      ○
      │                      ┼
      │                ┌───────────┐           ┌───────────┐
      │                │           │          ╱│           │
      └──────────────○┼│  Remark   │┼──────────│   User    │
                       │           │          ╲│           │
                       └───────────┘           └───────────┘
```

A Remark could be either associated with either a person or a business - this is called polymorphism.  For ubiquitous things like comments, pictures, etc. this is a common approach.

The standard rails way - is convenient (only uses 2 columns for any number of models), but lacks a foreign key so the DB can't ensure Data integrity.  For this reason, many other frameworks do not encourage this approach.  So we will use an approach accepted by all frameworks.

### Models and relationships

Lets build the user model first so we have all the models needed by Remark.

```ruby
bin/rails g model User email:string:uniq
```

Let's add an email validation to match the DB settings (and case insensitive):
```ruby
# app/models/user.rb
class User < ApplicationRecord
  validates :name, presence: true,
                   uniqueness: { case_sensitive: false }
end

```

Given the simplicity of this model we can just continue. lets build Remark now.
```ruby
bin/rails g model Remark note:text user:references business:references person:references
```

Like contact we will need to update the migration to allow null in the Business and Person foreign keys, but not for user.  Then we will update the models too.

update the migration to ensure we have a note and user, and allow either a business or person associated with each remark:
```ruby
# db/migrate/20210530104742_create_remarks.rb
class CreateRemarks < ActiveRecord::Migration[6.1]
  def change
    create_table :remarks do |t|
      t.text :note, null: false

      t.references :business, foreign_key: true
      t.references :person, foreign_key: true
      t.references :user, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

Now we will update the User, Business and Person models to know they could have many remarks with `has_many :remarks`:
```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_many :remarks
  validates :email, presence: true,
                    uniqueness: { case_sensitive: false }
end
```

```ruby
# app/models/person.rb
class Person < ApplicationRecord
  has_one :contact
  has_many :remarks
  belongs_to :business, optional: true

  validates :contact, presence: true
  validates :full_name, presence: true

  def display_name
    full_name
  end

  def employee_count
    nil
  end

  def associated_business_name
    business&.display_name
  end
end
```

```ruby
# app/models/business.rb
class Business < ApplicationRecord
  has_one :contact
  has_many :people
  has_many :remarks

  accepts_nested_attributes_for :contact

  validates :contact, presence: true
  validates :legal_name, presence: true

  def display_name
    legal_name
  end

  def employee_count
    people.count
  end

  def associated_business_name
    ""
  end
end
```

update the Remark model with the validations to enforce relations:
```ruby
# app/models/remark.rb
class Remark < ApplicationRecord
  belongs_to :user
  belongs_to :person, optional: true
  belongs_to :business, optional: true

  validates :user, presence: true
  validates :note, presence: true
  # validate :validate_remarkable_belongs_to_one_and_only_one_foreign_key

  def remarkable
    business || person
  end

  private

  # exclusive or (XOR) is true if one or the other is true, but both
  # if un-persisted we could get a model w/o an id
  # if persisted we could have a model and an id
  def validate_remarkable_belongs_to_one_and_only_one_foreign_key
    return if (business_id.present? ^ person_id.present?) ||
              (business.present? ^ person.present?)

    # add to base since, some forms may not have the person/business fields
    errors.add :base, 'must belong to ONE business or person, but not both'
    # errors.add :remarkable, 'must belong to a business or a person'
  end
end
```

Lets be sure this migrates:
```bash
bin/rails db:migrate
```

Now lets add the following to the end of our seed file:
```ruby
# db/seed.rb
# We will create a few users
require 'securerandom'

10.times do
  username = SecureRandom.alphanumeric(10)  # or use SecureRandom.uuid
  User.create!(email: "#{username}@example.ch")
end

# Lets add a remark to most People and Business (using a random user)
users = User.all

Person.all.each_with_index do |person, index|
  next if rand(1..3) == 1  # skip one in 3 people

  user = users.sample
  Remark.create!(person: person, user: user,
                note: "some note about #{person.display_name}, by user: #{user.email}")
end

Business.all.each_with_index do |business, index|
  next if rand(1..4) == 1  # skip one in 4 businesses

  user = users.sample
  Remark.create!(business: business, user: user,
                note: "some note about #{business.display_name}, by user: #{user.email}")
end
```

Lets check the seed with:
```bash
bin/rails db:seed
```

Cool that works!

lets snapshot:
```bash
git add .
git commit -m "polymorphic remark relations created"
```

### Views

**Comming soon ...**

Assuming this works, let's see the "/people" page:
```bash
bin/rails s
open localhost:3000/businesses/
```

### N+1 checks



Great - lets snapshot:
```bash
git add .
git commit -m "created an agnostic polymorphic model with data integrity enforced"
```

### Input Forms (& Building new Info)
