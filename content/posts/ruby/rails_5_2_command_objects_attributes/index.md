---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 5.2 - Command Objects and PORO Attributes"
subtitle: "Adding Attribute Validations to specific actions"
summary: "In a complex application validations, models, controllers can quickly get complex - command objects can simplify this - especially as of Rails 5.2"
authors: ["btihen"]
tags: ['Ruby', 'Command-Object', 'Validations', 'Attributes', 'Simplify Complexity']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2022-03-26T01:57:00+02:00
lastmod: 2022-05-22T01:57:00+02:00
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
## Overview

Sometimes within an work flow different validations and business logic is needed.  It can be very complex if each modification uses the same controller and model logic!  I've tried and ist crazy difficult.  The best solution I have found is to use Commands and separate Controllers.  It became practical to treat a command object as a Rails Model as of Rails 5.2 when the Attribute API allows Plain Old Ruby Objects (POROs) to use Attributes.

## Scenario

We have a workflow (at a school) where a young student submits a request.
* Student creates a request
* This is reviewed by the 'assistant to the dean' (approved, conditional, declined)
* When approved it is forwarded to the student's guardian for guardian_approval (approved, conditional, declined)
* When approved it is forwarded to the  'Dean of Students' for final_approval (approved, conditional, declined)

This example will omit much of the real-world complexities especially the relationships between users and the related security and scoping.  But the focus here is to build validations and logic outside the controllers and base model.  (the base model will only contains logic that is always true)

## create a project

We will ensure we are using Rails 5.2 (but it can be later and there are be more features in later versions of Rails)

```bash
rbenv install 2.7.5
rbenv local 2.7.5

gem install rails -v 5.2.7
rails _5.2.7_ new commands_n_attributes --skip-spring

cd commands_n_attributes

bin/rails db:drop
bin/rails db:create
bin/rails db:migrate
```

Let's create a landing page using:

`bin/rails g controller landing index`

And now update routes:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  get 'landing/index'
  root 'landing#index'
end
```

If you start rails with `bin/rails s` '/' should be the new landing-page.

We will need some users and their roles (but we will keep this simple!) using:

`rails g scaffold User name role email:uniq`

And we will update the migration to require our new fields using:
```ruby
class CreateUsers < ActiveRecord::Migration[5.2]
  def change
    create_table :users do |t|
      t.string :name, null: false
      t.string :email, null: false
      t.string :role, default: 'student', null: false

      t.timestamps
    end
    add_index :users, :email, unique: true
  end
end
```

Since all fields are required all the time we will put the user validations in the model - so we can now update the model using:
```ruby
class User < ApplicationRecord
  validates :name, presence: true
  validates :email, presence: true, uniqueness: true
  validates :role, presence: true,
                   inclusion: { in: %w[student assistant parent dean],
                                message: '%{value} is not a valid role' }
end
```

Let's migrate `bin/rails db:migrate` and make some users with `bin/rails db:migrate` or using `/users`
```ruby
# db/seeds.rb
student = User.create(name: 'student', email: 'student@example.ch', role: 'student')
reviewer = User.create(name: 'reviewer', email: 'reviewer@example.ch', role: 'reviewer')
bill = User.create(name: 'bill', email: 'bill@example.ch', role: 'guardian')
dean = User.create(name: 'dean', email: 'dean@example.ch', role: 'dean')
```

## Student Request

allow the student to make a request (this stuff will always be required so we will use all rails standards) -- in real-life this would usually be scoped and the only area students can access, but that is beyond the scope of this article.

`rails g scaffold Request category description:text student:references`

Now lets open this migration and require all fields using:
```ruby
class CreateRequests < ActiveRecord::Migration[5.2]
  def change
    create_table :travel_requests do |t|
      t.string :category, null: false
      t.text :description, null: false
      t.references :student, foreign_key: {to_table: :users}, null: false

      t.timestamps
    end
  end
end
```

Note we are using the student as an alias for the User class -- so we can eventually allow all the various users to interact and 'student' is much clearer than `user` the model name.  And we can enforce these required fields in the model too:

```ruby
class Request < ApplicationRecord
  belongs_to :student, class_name: 'User'

  validates :student, presence: true
  validates :description, presence: true
  validates :category, presence: true,
                       inclusion: { in: %w(travel money),
                                    message: "%{value} is not a valid category" }
end
```

Let's migrate again and create some requests:
```ruby
# db/seeds.rb
student = User.create(name: 'student', email: 'student@example.ch', role: 'student')
reviewer = User.create(name: 'reviewer', email: 'reviewer@example.ch', role: 'reviewer')
bill = User.create(name: 'bill', email: 'bill@example.ch', role: 'guardian')
dean = User.create(name: 'dean', email: 'dean@example.ch', role: 'dean')

request_good = Request.create(category: 'travel', description: 'reasonable', student: student)
request_hmmm = Request.create(category: 'travel', description: 'questionable', student: student)
request_nope = Request.create(category: 'travel', description: 'unreasonable', student: student)
```

## Review Command

Let's add the more complex logic outside the rails-standard way.

First let's add the new fields needed by the reviewer:

```
bin/rails g migration AddReviewFieldsToRequests review_decision review_notes:texr \
                      review_decision_at:timestamp reviewer:references
```
Update the migration:
```ruby
class AddReviewFieldsToRequests < ActiveRecord::Migration[5.2]
  def change
    add_column :requests, :review_notes, :string
    add_column :requests, :review_decision, :string
    add_column :requests, :review_decision_at, :timestamp
    add_reference :requests, :reviewer, foreign_key: {to_table: :users}
  end
end
```

Now we need to update the Review Model too to add the relationship - its critical to add `optional: true` in rails 5.2 - otherwise rails thinks it should always be present!  Otherwise, we are not changing the model nor its validations - since these fields are only required by the reviewer while reviewing a request.
```ruby
class Request < ApplicationRecord
  belongs_to :student, class_name: 'User', foreign_key: 'student_id'
  belongs_to :reviewer, class_name: 'User', foreign_key: 'reviewer_id', optional: true

  validates :student, presence: true
  validates :description, presence: true
  validates :category, presence: true,
                       inclusion: { in: %w(travel money),
                                    message: "%{value} is not a valid category" }
end
```

`bin/rails db:migrate`

Let's create a new controller to reviews the requests using (note - will will only accept the fields: review_decision review_notes - the others we will set in code) - so we can use the command:

`bin/rails g scaffold_controller ReviewRequests review_decision review_notes`

We don't actually want the assistant to create or delete requests so the routes will look like:
```ruby
Rails.application.routes.draw do
  resources :review_requests, only: [:index, :show, :edit, :update]
  resources :requests
  resources :users
  get 'landing/index'
  root 'landing#index'
end
```

Now let's create out command (since we will have many commands we will make a folder called commands):
```ruby
mkdir -p app/commands
touch app/commands/review_request_command.rb

class ReviewRequestCommand
  # Model and Attributes are BOTH needed to user Rails attributes in a Plain Ruby class
  include ActiveModel::Model
  include ActiveModel::Attributes
  include ActiveModel::AttributeAssignment # allows direct assignment `.assign_attributes`
  include ActiveModel::AttributeMethods # allows attribute prefixing, etc
  include ActiveModel::Conversion # provides: #to_model, #to_key, #to_param, and to_partial_path
  include ActiveModel::Dirty # needed to track changes
  include ActiveModel::Validations

  # allow at least request to be seen outside this model (to display above the form)
  attr_reader :params, :request

  # unless all attributes are in the accessor you will get
  # `undefined method `write_from_user' for nil:NilClass` with normal attribute assignment
  # although you can use @id = params[:id] instead
  attr_accessor :id, :reviewer_id, :review_decision, :review_notes,
                :review_decision_at, :reviewer

  # our attributes (these are the only things the form can access/submit)
  attribute :id, :integer
  attribute :reviewer, User
  attribute :reviewer_id, :integer
  attribute :review_notes, :string
  attribute :review_decision, :string
  attribute :review_decision_at, :datetime

  # our rewiewer validations
  validates :id, presence: true
  validates :reviewer_id, presence: true
  validates :review_decision, presence: true,
                              inclusion: { in: %w[approved conditional declined],
                                           message: '%{value} is not a valid decision' }
  # a complex validation with logic
  validate :validate_review_notes

  # not needed but convenient
  def self.call(params = {})
    new(params).run
  end

  def initialize(params = {})
    @params = params # helps with debugging

    # get the request and pre-populate our attributes from there
    @request = Request.find(params[:id])
    self.review_notes = @request.review_notes
    self.review_decision = @request.review_decision

    # set the attributes (incoming attributes overwrite those from the request)
    assign_attributes(params)
  end

  # what does the action
  def run
    request.reviewer_id = reviewer_id
    request.review_notes = review_notes
    request.review_decision = review_decision
    request.review_decision_at = review_decision_at

    # return errors if not valid
    return self unless valid? && request.valid? && request.save

    # if success updating the original request - then do our additional logic
    case request.review_decision
    when 'approved'
      puts 'notify parent of request'
      puts 'notify student of approval'
    else
      puts "notify student that the request is #{review_decision} because #{review_notes}"
    end

    self
  end

  # without this method path(object) doesn't work or you can just use path(id: object.id) instead
  def to_param
    id
  end

  # this must answer true for the form to use patch instead of post!
  def persisted?
    true
  end

  # rails 5.2 requires this to work with forms and dirty tracking
  # unfortunately, rails 5.2 can't self-discover its own attributes
  # without this you get: `undefined method `to_hash' for nil:NilClass` in the form
  def attributes
    {
      id: id,
      reviewer_id: reviewer_id,
      review_notes: review_notes,
      review_decision: review_decision,
      review_decision_at: review_decision_at,
      request: request
    }
  end
  alias_method :to_hash, :attributes

  private

  def validate_review_notes
    # don't report an error when the decision is 'approved' or decision is invalid
    return unless %w[conditional declined].include? review_decision
    return unless review_notes.blank?

    errors.add(:review_notes, 'is required when request is not approved')
  end
end
```

Now you can test this Command in console:

```ruby
student = User.create(name: 'student', email: 'student@example.ch', role: 'student')
reviewer = User.create(name: 'reviewer', email: 'reviewer@example.ch', role: 'reviewer')
bill = User.create(name: 'bill', email: 'bill@example.ch', role: 'guardian')
dean = User.create(name: 'dean', email: 'dean@example.ch', role: 'dean')

request_good = Request.create(category: 'travel', description: 'reasonable', student: student)
request_hmmm = Request.create(category: 'travel', description: 'questionable', student: student)
request_nope = Request.create(category: 'travel', description: 'unreasonable', student: student)

review = ReviewRequestCommand.new(id: request_good.id, reviewer_id: reviewer.id)
review.valid?
review.errors.messages
request_good.reload # should be unchanged

review = ReviewRequestCommand.call(id: request_good.id, reviewer_id: reviewer.id)
review.valid?
review.errors.messages
request_good.reload # should be unchanged

review = ReviewRequestCommand.call(id: request_good.id, reviewer_id: reviewer.id, review_decision: 'accepted')
review.valid?
review.errors.messages
request_good.reload # SHOULD BE CHANGED
```

## Adapt the Review Controller

Basically we need to treat the command as the model and send in the reviewer and the proper params.

```ruby
class ReviewRequestsController < ApplicationController
  def index
    @review_requests = Request.all
  end

  def edit
    @review_request = ReviewRequestCommand.new(command_params)
  end

  def update
    @review_request = ReviewRequestCommand.call review_request_params.merge(command_params)
    respond_to do |format|
      if @review_request.valid?
        format.html { redirect_to review_requests_url, notice: 'Review request was successfully updated.' }
        format.json { render :show, status: :ok, location: @review_request }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @review_request.errors, status: :unprocessable_entity }
      end
    end
  end

  private

  def command_params
    # {id: params[:id].to_i, reviewer_id: current_user.id}
    {id: params[:id].to_i, reviewer_id: current_user.id, reviewer: current_user}
  end

  # Only allow a list of trusted parameters through.
  def review_request_params
    params.require(:review_request_command) # must be the model name
          .permit(:review_decision, :review_notes)
  end

  def current_user
    User.where(role: %w[reviewer admin]).first
  end
end
```

## Adapting the View Form

This next part is always tricky (rails likes working with Default models) -- the important thing is the `form` command needs the URL since the model is called `review_request_command`, but the route is called `review_request_path`

* if you get a url error that review_request_command_path doesn't exist - check the form url
* if this form sends a POST be sure `review_request.persisted?` returns `true`!
* if the form returns the error: `undefined method `to_hash' for nil:NilClass` in the form - be sure your attribute is returned in `review_request.attributes`
* if you get the error `review_request` wasn't included, be sure you check the controller's review_request_params method requires the `review_request_command` not `review_request`

```html
<!-- app/views/review_requests/_form.html.erb -->
<!-- add the url to the form and all should work :) -->
<%= form_with(model: review_request, local: true,
              url: review_request_path(id: review_request.id)) do |form| %>

  <% if review_request.errors.any? %>
    <div id="error_explanation">
      <h2><%= pluralize(review_request.errors.count, "error") %> prohibited this review_request from being saved:</h2>

      <ul>
      <% review_request.errors.full_messages.each do |message| %>
        <li><%= message %></li>
      <% end %>
      </ul>
    </div>
  <% end %>

  <div class="field">
    <%= form.label :review_decision %>
    <%= form.text_field :review_decision %>
  </div>

  <div class="field">
    <%= form.label :review_notes %>
    <%= form.text_field :review_notes %>
  </div>

  <div class="actions">
    <%= form.submit %>
  </div>
<% end %>
```

Now you can have complex process logic isolated in your commands and all your models, controllers and views can be focused on their simple tasks.

## Resources

### attributes allowed in non-ar models
* https://github.com/rails/rails/issues/28020

### Attribute Assignment methods
* https://scottbartell.com/2020/01/30/set-attributes-in-active-record-rails-6/

### Rails 5.2 Docs
* https://api.rubyonrails.org/v5.2/
* https://api.rubyonrails.org/v5.2.3/classes/ActiveModel/Type.html
* https://api.rubyonrails.org/v5.2.3/classes/ActiveRecord/Type.html
* https://api.rubyonrails.org/v5.2.3/classes/ActiveModel/Type/Value.html
* https://api.rubyonrails.org/v5.2.3/classes/ActiveRecord/AttributeMethods/Read.html#method-i-read_attribute
* https://api.rubyonrails.org/v5.2.3/classes/ActiveModel/Conversion.html
* https://api.rubyonrails.org/v5.2.3/classes/ActiveModel/AttributeMethods.html
* https://api.rubyonrails.org/v5.2.3/classes/ActiveModel/AttributeAssignment.html
* https://api.rubyonrails.org/v5.2.3/classes/ActiveModel/Attributes/ClassMethods.html
* https://api.rubyonrails.org/v5.2.3/classes/ActiveRecord/Aggregations/ClassMethods.html
* https://api.rubyonrails.org/v5.2.3/classes/ActiveModel/Attributes/ClassMethods.html#method-i-attribute

### default attribute types:
* https://api.rubyonrails.org/v5.2.3/classes/ActiveModel/Type.html
* https://github.com/rails/rails/tree/v6.0.2.1/activemodel/lib/active_model/type

### book using PORO with attributes
* https://products.arkency.com/domain-driven-rails

### helpful blogs
* https://boringrails.com/tips/rails-attributes-api
* https://codeclimate.com/blog/7-ways-to-decompose-fat-activerecord-models/
* https://api.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html
* https://boringrails.com/tips/rails-attributes-api
* https://blog.dario-hamidi.de/a/rails-hidden-type-system
* https://stuff-things.net/2015/07/21/validating-rails-forms-without-a-model/

### custom Attribute types
* https://metova.com/rails-5-attributes-api/
* https://boringrails.com/tips/rails-attributes-api
* https://jakeyesbeck.com/2015/12/20/rails-5-attributes/
* https://oozou.com/blog/custom-attribute-types-in-rails-5-77
* https://api.rubyonrails.org/v5.2.3/classes/ActiveModel/Type.html
* https://api.rubyonrails.org/v5.2.3/classes/ActiveRecord/Type.html
* https://api.rubyonrails.org/v5.2.3/classes/ActiveModel/Type/Value.html
* https://jetrockets.com/blog/rails-5-attributes-api-value-objects-and-jsonb
* https://dev.to/swanson/automatically-cast-params-with-the-rails-attributes-api-446a
* https://api.rubyonrails.org/v5.2.3/classes/ActiveRecord/Aggregations/ClassMethods.html
* https://edgeapi.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html#method-i-attribute
* https://metova.com/rails-5-attributes-api/
* https://dev.to/swanson/automatically-cast-params-with-the-rails-attributes-api-446a

* https://stackoverflow.com/questions/52711754/casting-to-custom-type-a-postgresql-array

### Form_with

* https://apidock.com/rails/ActionView/Helpers/FormHelper/form_with
* https://stackoverflow.com/questions/43868976/rails-5-form-for-vs-form-with
* https://stackoverflow.com/questions/5160733/ror-using-form-tag-without-an-ar-model
* https://rubyinrails.com/2018/02/19/rails-form-with-alternative-to-form-for-and-form-tag/

## Custom Attributes (when inherited from AcitiveModel)

This is good with a 'normal' active record model you can add 'virtual' attributes - just for the form pages
```ruby
# app/attribute_types/squished_string.rb
class SquishedString < ActiveRecord::Type::Value
  include ActiveModel::Type::Helpers::Mutable
  def type
    :squished_string
  end

  def cast(value)
    value.to_s.squish.strip
  end

  # def cast_value(value)
  # end

  # def deserialize(value)
  # end

  # def serialize(value)
  # end
end
```

```ruby
# config/initializers/attribute_types.rb
# Not needed unless truely a new Datbase type
# ActiveRecord::Type.register(:squished_string, SquishedString)
# critical to define ActiveModel to use within the model
ActiveModel::Type.register(:squished_string, SquishedString)
```


## Custom Attributes (in PORO - not yet in Rails 5.2)

Without access to attribute_type it seems this only works with Manual casting - without explict casting.

@review_notes = TrimmedText.new.cast(' lots to     say ')
"lots to say"

ActiveModels have: #attribute_types & #attribute_names & #columns & #reflections (associations) - when a model has attribute types all works well as documented.
```ruby
[26] pry(Request):2> attribute_types
=> {"id"=>
  #<ActiveModel::Type::Integer:0x00007f8382cf5b18 @limit=8, @precision=nil, @range=-9223372036854775808...9223372036854775808, @scale=nil>,
 "category"=>#<ActiveModel::Type::String:0x00007f8383c36be0 @limit=nil, @precision=nil, @scale=nil>,
 "description"=>#<ActiveRecord::Type::Text:0x00007f8382cf5708 @limit=nil, @precision=nil, @scale=nil>,
 "student_id"=>
  #<ActiveModel::Type::Integer:0x00007f8382cf5b18 @limit=8, @precision=nil, @range=-9223372036854775808...9223372036854775808, @scale=nil>,
 "created_at"=>#<ActiveRecord::ConnectionAdapters::PostgreSQL::OID::DateTime:0x00007f8383c35c68 @limit=nil, @precision=nil, @scale=nil>,
 "updated_at"=>#<ActiveRecord::ConnectionAdapters::PostgreSQL::OID::DateTime:0x00007f8383c35c68 @limit=nil, @precision=nil, @scale=nil>,
 "review_notes"=>#<ActiveRecord::Type::Text:0x00007f8382cf5708 @limit=nil, @precision=nil, @scale=nil>,
 "review_decision"=>#<ActiveModel::Type::String:0x00007f8383c36be0 @limit=nil, @precision=nil, @scale=nil>,
 "review_decision_at"=>
  #<ActiveRecord::ConnectionAdapters::PostgreSQL::OID::DateTime:0x00007f8383c35c68 @limit=nil, @precision=nil, @scale=nil>,
 "reviewer_id"=>
  #<ActiveModel::Type::Integer:0x00007f8382cf5b18 @limit=8, @precision=nil, @range=-9223372036854775808...9223372036854775808, @scale=nil>}
```
