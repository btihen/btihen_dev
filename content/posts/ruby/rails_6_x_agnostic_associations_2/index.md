---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 6.x - Framework Agnostic Associations - part 2"
subtitle: "Aggregating different, but related Data models (Rails STI alternative)"
summary: "Framework Agnostic Associations - Data models that work across many frameworks"
authors: ["btihen"]
tags: ['Ruby', 'Databases', 'Data models', 'Framework Agnostic', 'belongs_to', 'has_one']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-05-29T01:57:00+02:00
lastmod: 2021-08-07T01:57:00+02:00
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

                 Created with Monodraw
```

## Rails app and first Models

```
    ┌────────────┐             ┌───────────┐
    │            │╲          1 │           │
    │  Business  │─○──────────┼│  Person   │
    │-legal_name │╱0..*        │-full_name │
    └────────────┘             └───────────┘
```

We discussed / explained in (part 1)[post_ruby_rails/rails_6_x_agnostic_associations_1/]

## Polymorphic (STI) - sometime called inverse polymorphic

In this article we will build this structure (a replacement for Rails STI).  Many frameworks will only use columns that can be identified as foreign keys to ensure DB integrity - therefore, we will build this using DB structures that are supported by Rails, Lucky and Phoenix and probably most frameworks.
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

A contact could be either a person or a business - but must be one or the other.

### Migration and Relationships

Rails doesn't have a built-in array migration, so we use string and then we change the migration:
```bash
bin/rails g scaffold Contact functions:string business:references person:references
```

Now update the migration to ensure we have a functions as an array & relations as Foreign keys (but optional). Since there we only want/need one of the two foreign_keys at a time they must be nullable and we need to change roles to an array - so now:
```ruby
# db/migrate/20210519205042_create_contacts.rb
class CreateContacts < ActiveRecord::Migration[6.1]
  def change
    create_table :contacts do |t|
      t.string :functions, array: true, null: false, default: []
      t.references :business, foreign_key: true
      t.references :person, foreign_key: true

      t.timestamps
    end
  end
end
```

update the Contact model with the validations & flexible relations - we also want to be able to refer to the sub-model by one name we will call that `contactable` - so now the model will look like:
```ruby
# app/models/contact.rb
class Contact < ApplicationRecord
  belongs_to :business, optional: true
  belongs_to :person, optional: true

  VALID_FUNCTIONS_LIST = %w(supplier reseller customer sales-rep)

  validate :validate_relationship_functions
  validate :validate_belongs_to_one_and_only_one_foreign_key

  def contactable
    business || person
  end

  private

  # be sure we have the variable, it is an Array & all elements are in the valid list
  def validate_relationship_functions
    return if functions.present? && functions.is_a?(Array)
              functions.all? { |role| VALID_FUNCTIONS_LIST.include?(role.to_s) }

    errors.add :functions, "must be ONE or MORE of the following options: #{VALID_FUNCTIONS_LIST.join(',')}"
  end

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

update the Person model and relations and enforce every person is a member of the contact list - with a contact role:
```ruby
# app/models/person.rb
class Person < ApplicationRecord
  has_one :contact
  belongs_to :business, optional: true

  validates :contact, presence: true
  validates :full_name, presence: true
end
```

update the business model and relations and enforce every business is a member of the contact list - with a contact role:
```ruby
# # app/models/business.rb
class Business < ApplicationRecord
  has_one :contact
  has_many :people

  validates :contact, presence: true
  validates :legal_name, presence: true
end
```


Lets check the seed with:
```bash
bin/rails db:migrate
```

If we go to a person or business we can no longer make changes - they need to have an associated Contact.
We'll start by rolling back the last migration and fixing it with (we can use the logic in the seeds to guide us in the Business/Person creation controller):

```bash
bin/rails db:rollback
```

we need to fix the old relations in the migration (or simply drop the database and reseed it) - but given this is to article is find cross-framework -- 'real-world' techniques - let's be sure the existing records stay useful.  We will assume a business is a supplier, a person associated with a business is a sales-rep, and unassociated people are customers.
```ruby
# db/migrate/20210519205042_create_contacts.rb
class CreateContacts < ActiveRecord::Migration[6.1]
  def change
    create_table :contacts do |t|
      t.string :functions, array: true, null: false, default: []
      t.references :business, foreign_key: true
      t.references :person, foreign_key: true

      t.timestamps
    end

    # add a contact for each existing company
    businesses = Business.joins(:people)
                         .group('businesses.id')
                         .select('businesses.*, count(people.id) as people_count')
    businesses.each do |business|
      functions = if business.people_count < 10
                    ['supplier']
                  elsif business.people_count < 20
                    ['reseller']
                  elsif business.people_count < 30
                    ['supplier', 'reseller']
                  end
      Contact.create!(functions: functions, business: business)
    end

    # add a contact for each existing person
    Person.all.each do |person|
      functions = if person.business
                    ['sales_rep']
                  else
                    ['customer']
                  end
      Contact.create!(functions: functions, person: person)
    end
  end
end
```

Lets the existing models now:
```bash
bin/rails db:migrate
```

OK - we are in business lets update our seed file too:
```ruby
# db/seed.rb
# create small business w/o employees
20.times do |num|
  business = Business.create(legal_name: "Business #{num}",
                             contact: Contact.new(functions: ['supplier']))
end

# create individuals
20.times do |num|
  person = Person.create(full_name: "Individual #{num}",
                            contact: Contact.new(functions: ['customer']))
end

# create big companies with employees
20.times do |bus_num|
  functions = if bus_num < 3
                ['supplier']
              elsif bus_num< 5
                ['reseller']
              elsif bus_num < 8
                ['supplier', 'reseller']
              else
                %w[supplier reseller customer]
              end
  company  = Business.create(legal_name: "Company #{bus_num}",
                             contact: Contact.new(functions: functions))

  bus_num.times do |emp_num|
    Person.create(full_name: "Employee #{bus_num}-#{emp_num}",
                  business: company,
                  contact: Contact.new(functions: ['sales-rep']))
  end
end
```

Lets check the seed with:
```bash
bin/rails db:seed
```

Great all works!

### Lets make the index page more useful

When we visit the contacts page we would like more than the ids - but we need a unified way to present that info so let's add a display_name so we can show the name of the primary model, if a person we would like to know the associated business if present and if a company we would like the employee_count so we will delegate these to the sub-models.

Let's update contact first by adding:
```ruby
  # this references our existing contactable
  delegate :display_name, :associated_business_name, :employee_count,
           to: :contactable

  def contactable
    business || person
  end
```

So now the contact model will look like (with validations)
```ruby
# app/models/contact.rb
class Contact < ApplicationRecord
  belongs_to :business, optional: true
  belongs_to :person, optional: true

  VALID_FUNCTIONS_LIST = %w(supplier reseller customer sales-rep)

  validate :validate_relationship_functions
  validate :validate_belongs_to_one_and_only_one_foreign_key

  delegate :display_name, :associated_business_name, :employee_count,
           to: :contactable

  def contactable
    business || person
    # would memoize be valuable here?
    # @contactable ||= (business || person)
  end

  private

  # be sure we have the variable, it is an Array & all elements are in the valid list
  def validate_relationship_functions
    return if functions.present? && functions.is_a?(Array)
              functions.all? { |role| VALID_FUNCTIONS_LIST.include?(role.to_s) }

    errors.add :functions, "must be ONE or MORE of the following options: #{VALID_FUNCTIONS_LIST.join(',')}"
  end

  # exclusive or (XOR) is true if one or the other is true, but not when both are true
  # we could get a model (or possibly an id)
  def validate_belongs_to_one_and_only_one_foreign_key
    return if business.present? ^ person.present? ^ business_id.present? ^ person_id.present?

    # add to base since, some forms may not have the person/business fields
    errors.add :base, 'must belong to ONE business or person, but not both'
    # errors.add :contactable, 'must belong to a business or a person'
  end
end
```

Lets update the models to provide the needed info

Business now will look like:
```ruby
# app/models/business.rb
class Business < ApplicationRecord
  has_one :contact
  has_many :people

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

And person will look like:
```ruby
# app/models/person.rb
class Person < ApplicationRecord
  has_one :contact
  belongs_to :business, optional: true

  validates :contact, presence: true
  validates :full_name, presence: true

  def display_name
    full_name
  end

  def employee_count
    nil  # person count has no meaning under person
  end

  def associated_business_name
    business&.display_name
  end
end
```

Now lets update the index view to show our new info:
```ruby
<h1>Contacts</h1>

<table>
  <thead>
    <tr>
      <th>Person/Business</th>
      <th>Employee Count</th>
      <th>Contact Name</th>
      <th>Business Name</th>
      <th>Relationships</th>
      <th colspan="3"></th>
    </tr>
  </thead>

  <tbody>
    <% @contacts.each do |contact| %>
    <tr>
      <td><%= contact.contactable.class.name %></td>
      <td><%= contact.employee_count %></td>
      <td><%= contact.display_name %></td>
      <td><%= contact.associated_business_name %></td>
      <td><%= contact.functions %></td>
      <td><%= link_to 'Show', contact %></td>
      <td><%= link_to 'Edit', edit_contact_path(contact) %></td>
      <td><%= link_to 'Destroy', contact, method: :delete, data: { confirm: 'Are you sure?' } %></td>
    </tr>
    <% end %>
  </tbody>
</table>
```

Now we see another n+1 query - we will fix the main part - but not the employee count this time:
```ruby
class ContactsController < ApplicationController
  def index
    # @contacts = Contact.all
    @contacts = Contact.includes(:business).includes(:person).all
  end
```

Cool now the page is usable (a bit long but we will ignore that)

### Lets be sure we can create new contacts

I usually use an input model (for more flexibility), but for now I will use nested_params.
A few articles on nested params and nested fields:

* https://www.youtube.com/watch?v=PYYwjTlcoa4
* https://www.pluralsight.com/guides/ruby-on-rails-nested-attributes
* https://levelup.gitconnected.com/rails-nested-forms-in-three-steps-5580f0ad0e
* https://levelup.gitconnected.com/handling-nested-attributes-with-a-has-many-through-association-with-rails-api-f91729547ea5

To start we will tell the contacts model that it can create nested models with do by adding:
```ruby
  accepts_nested_attributes_for :business
  accepts_nested_attributes_for :person
```

so now now the contact model looks like:
```ruby
# app/models/contact.rb
class Contact < ApplicationRecord
  belongs_to :business, optional: true
  belongs_to :person, optional: true

  accepts_nested_attributes_for :business
  accepts_nested_attributes_for :person

  VALID_FUNCTIONS_LIST = %w(supplier reseller customer sales-rep)

  validate :validate_relationship_functions
  validate :validate_belongs_to_one_and_only_one_foreign_key

  delegate :display_name, :associated_business_name, :employee_count,
           to: :contactable

  def contactable
    business || person
    # would memoize be valuable here?
    # @contactable ||= (business || person)
  end

  private

  # be sure we have the variable, it is an Array & all elements are in the valid list
  def validate_relationship_functions
    return if functions.present? && functions.is_a?(Array)
              functions.all? { |role| VALID_FUNCTIONS_LIST.include?(role.to_s) }

    errors.add :functions, "must be ONE or MORE of the following options: #{VALID_FUNCTIONS_LIST.join(',')}"
  end

  # exclusive or (XOR) is true if one or the other is true, but not when both are true
  # we could get a model (or possibly an id)
  def validate_belongs_to_one_and_only_one_foreign_key
    return if business.present? ^ person.present? ^ business_id.present? ^ person_id.present?

    # add to base since, some forms may not have the person/business fields
    errors.add :base, 'must belong to ONE business or person, but not both'
    # errors.add :contactable, 'must belong to a business or a person'
  end
end
```

In the controller we need to create models as part of @contact to allow nested-fields - which feed the nested attributes. to allow the new information in via strong params:
```ruby
# app/controllers/contacts_controller.rb
  def new
    @contact = Contact.new
    # add empty sub-models for our form
    @contact.person = Person.new
    @contact.business = Business.new
  end

  # update strong params to accept the sub-model attributes
  # sub-models from nested-forms feeding nested_atttributes in the model
  # take the form <model_name>_attributes
  # `functions` is an empty array since it is taking a list of values
  # person_attributes & business_attributes - need to include the list of attributes to accept!
  # so in our case:
  def contact_params
    contact_attribs = params.require(:contact)
                            .permit(functions: [],  # is empty - takes a list of values
                                    person_attributes: [:full_name],  # needs to include the list of attributes to accept
                                    business_attributes: [:legal_name])
  end
```

update the contact form to tie this all together by adding our nested forms:
```ruby
  <div class="field-group">
    <h2>Create your Contact: a Person or a Business</h2>

    <h3>Business</h3>
    <%= form.fields_for :business, Business.new do |f| %>
      <%= f.label :legal_name %>
      <%= f.text_field :legal_name %>
    <% end %>

    <h3>Person</h3>
    <%= form.fields_for :person, Person.new do |f| %>
      <%= f.label :full_name %>
      <%= f.text_field :full_name %>
    <% end %>
  </div>
```

We will also need to make the list of possible relationship functions a multi-select - I always forget the format -- so remember BOTH {} are required when using multi-select!!  The first one is for normal drop-down select options -- like include_blank, the second one is where the multi-select must go!

This looks like:
```ruby
  <div class="field">
    <%= form.label :functions %>
    <%= form.select :functions,
                    options_for_select(Contact::VALID_FUNCTIONS_LIST,
                                      selected: Contact::VALID_FUNCTIONS_LIST.second),
                                      {}, #{:include_blank => 'None'},
                                      {:multiple => true, size: 3} %>
  </div>
```

so now the template looks like:
```ruby
# app/views/contacts/_form.html.erb
<%= form_with(model: contact) do |form| %>
  <% if contact.errors.any? %>
  <div id="error_explanation">
    <h2><%= pluralize(contact.errors.count, "error") %> prohibited this contact from being saved:</h2>

    <ul>
      <% contact.errors.each do |error| %>
      <li><%= error.full_message %></li>
      <% end %>
    </ul>
  </div>
  <% end %>

  <div class="field">
    <%= form.label :functions %>
    <%= form.select :functions,
                    options_for_select(Contact::VALID_FUNCTIONS_LIST,
                                      selected: Contact::VALID_FUNCTIONS_LIST.second),
                                      {}, #{:include_blank => 'None'},
                                      {:multiple => true, size: 3} %>
  </div>

  <div class="field-group">
    <h2>Create your Contact: a Person or a Business</h2>

    <h3>Business</h3>
    <%= form.fields_for :business, Business.new do |f| %>
      <%= f.label :legal_name %>
      <%= f.text_field :legal_name %>
    <% end %>

    <h3>Person</h3>
    <%= form.fields_for :person, Person.new do |f| %>
      <%= f.label :full_name %>
      <%= f.text_field :full_name %>
    <% end %>
  </div>

  <div class="actions">
    <%= form.submit %>
  </div>
<% end %>
```

Now when we try `/contacts` we notice one more problem - it is always invalid - rails automatically add a leading "" in an array input list :( - so we will have to clean this up in the strong params.  In this case we are working with param objects not a hash so we will do an in-place update (removal of "") using:
```ruby
  contact_attribs["functions"].reject! {|f| f.blank? }
  contact_attribs
```

 we also need to be sure in our case we only send the params of the business or the person, but not both - since we are only creating one.  So we will remove whichever one is empty - also with an in-place update - using:
```ruby
    # find and set to nil the model without params
    if contact_attribs["person_attributes"]
      # since we only have one param we can do
      contact_attribs["person_attributes"] = nil if contact_attribs["person_attributes"]["full_name"].blank?
    end

    if contact_attribs["business_attributes"]
      # assuming we had multiple params the test is easier with:
      contact_attribs["business_attributes"] = nil if contact_attribs["business_attributes"].to_h.all? {|key,value| value.blank?}
    end

    # remove the nested attributes set to nil so contact will only create the desired associated model
    contact_attribs.reject! {|key, value| value.blank? }
    contact_attribs
```

So now the full controller looks like:
```ruby
class ContactsController < ApplicationController
  before_action :set_contact, only: %i[ show edit update destroy ]

  def index
    # @contacts = Contact.all
    @contacts = Contact.includes(:business).includes(:person).all
  end

  def show
  end

  def new
    @contact = Contact.new
    @contact.person = Person.new
    @contact.business = Business.new
  end

  def edit
  end

  def create
    @contact = Contact.new(contact_params)

    respond_to do |format|
      if @contact.save
        format.html { redirect_to @contact, notice: "Contact was successfully created." }
        format.json { render :show, status: :created, location: @contact }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @contact.errors, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @contact.update(contact_params)
        format.html { redirect_to @contact, notice: "Contact was successfully updated." }
        format.json { render :show, status: :ok, location: @contact }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @contact.errors, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @contact.destroy
    respond_to do |format|
      format.html { redirect_to contacts_url, notice: "Contact was successfully destroyed." }
      format.json { head :no_content }
    end
  end

  private

  # Use callbacks to share common setup or constraints between actions.
  def set_contact
    @contact = Contact.find(params[:id])
  end

  # Only allow a list of trusted parameters through.
  def contact_params
    # update strong params to accept the sub-model attributes
    # sub-models from nested-forms feeding nested_atttributes in the model
    # take the form <model_name>_attributes
    # `functions` is an empty array since it is taking a list of values
    # person_attributes & business_attributes - need to include the list of attributes to accept!
    # so in our case:
    contact_attribs = params.require(:contact)
                            .permit(functions: [],
                                    person_attributes: [:full_name],
                                    business_attributes: [:legal_name])
    # cleanup array - always delivers with [''] - :(
    # https://stackoverflow.com/questions/51341912/empty-array-value-being-input-with-simple-form-entries

    # easiest way in in-place replacement (given that params is now objects and not a hash), but that always makes me a bit nervous
    # https://stackoverflow.com/questions/20164354/rails-strong-parameters-with-empty-arrays
    # reject and replace in place
    contact_attribs["functions"].reject! {|f| f.blank? }

    # remove empty model attributes
    # contact_attribs["person_attributes"].reject {|key,value| value.blank?}
    if contact_attribs["person_attributes"]
      contact_attribs["person_attributes"] = nil if contact_attribs["person_attributes"]["full_name"].blank?
    end

    if contact_attribs["business_attributes"]
      contact_attribs["business_attributes"] = nil if contact_attribs["business_attributes"].to_h.all? {|key,value| value.blank?}
    end

    # have to remove nil attributes for models so nested attributes works correctly
    contact_attribs.reject! {|key, value| value.blank? }

    # return the attributes with the tidied array
    contact_attribs
  end
end
```

now when we try again:
```bash
bin/rails s
open localhost:3000/contacts/new
```

Cool - it works.  We could now do the same for the `/business/new` and `/people/new`, but we won't do that here in the article. Lets snapshot:
```bash
git add .
git commit -m "created person possibly related to the model"
```

## Polymorphic

In the next article we will explore the following in (part 3)[post_ruby_rails/rails_6_x_agnostic_associations_3/]

    ┌───────────┐             ┌───────────┐
    │           │╲            │           │
    │ Business  │─○──────────┼│  Person   │
    │           │╱            │           │
    └───────────┘             └───────────┘
          ┼                         ┼
          │                         │
          └────────────┬────────────┘
                       │
                      ╱│╲
                 ┌───────────┐
                 │           │
                 │  Remark   │
                 │           │
                 └───────────┘
