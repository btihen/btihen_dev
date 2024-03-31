---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x JSON API Introduction with Nested data"
subtitle: "JSON API Versioning in Rails 7.1.x"
summary: ""
authors: ['btihen']
tags: ['Rails API', 'Rails JSON', 'n+1']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2024-03-30T01:20:00+02:00
lastmod: 2024-03-31T01:20:00+02:00
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

I was recently asked about Rails API applications and wanted to explore the options (based on the Flintstones characters).

The end goal is to return one or many records in the following format - without an N+1 query.
```json
{ "id":8,
  "first_name":"Wilma",
  "last_name":"Flintstone",
  "given_name":"Slaghoople",
  "nick_name":null,
  "gender":"female",
  "person_jobs": [
    { "start_date":"1980-01-01",
      "end_date":"1989-12-31",
      "job": {
        "role":"news reporter",
        "company": {
          "id":2,
          "name":"Bedrock Daily News"
    }}},
    { "start_date":"1990-01-01",
      "end_date":null,
      "job":{
        "role":"caterer",
        "company":{
          "id":6,
          "name":"Betty and Wilma Catering"
    }}},
    ...
  ]
}
```

This code can be found at: https://github.com/btihen-dev/rails_api_intro

## Getting Started

I will create a basic application - then start the api exploration.  I will use: `bin/rails new bedrock -T` instead of `bin/rails new bedrock -T --api` because I will also use this same base code to play with hotwire.  I am also going to use the database `postgresql` to then later add the AGE postgres plugin to model the relationships using a graph data-structure.  Likewise, I will be using esbuild for the hotwire features - but this can easily be omitted.

```bash
rails new bedrock -T --main --database=postgresql --javascript=esbuild
cd bedrock
bin/rails db:create

git add .
git commit -m "initial commit"
```

## Base Model - People

I'll add people with scaffolding

```bash
bin/rails g scaffold Person nick_name first_name \
                     last_name given_name gender
```

Let's update the migration to make the first & last name required as well as gender & lets ensure we cant add the same person twice with a unique index on `first_name last_name gender`

```ruby
class CreatePeople < ActiveRecord::Migration[7.1]
  def change
    create_table :people do |t|
      t.string :first_name, null: false
      t.string :last_name, null: false
      t.string :nick_name
      t.string :given_name
      t.string :gender, null: false

      t.timestamps
    end

    add_index :people, %i[first_name last_name gender],
              unique: true
  end
end
```

Let's update the model to trim extra leading and trailing spaces and enforce the same DB options as at the DB level:

```ruby
class Person < ApplicationRecord
  normalizes :first_name, :nick_name, :last_name, :given_name,
             with: ->(value) { value.strip }

  validates :first_name,
            uniqueness: { scope: :last_name,
			                    message: "first- and last-name already exists" }
  validates :first_name, presence: true
  validates :last_name, presence: true
  validates :gender, inclusion: { in: %w[male female non-binary] }
end
```

now migrate and add a people seed (see the appendix for seed data)

```bash
bin/rails db:migrate
bin/rails db:seed
```

let's test:
```ruby
bin/rails c

Person.first

Person.all
```

Let's snapshot assuming this works and shows reasonable data.
```bash
git add .
git commit -m "Add person model"
```

## NAMESPACE JSON API

Now that we have a basic setup that works lets setup the API code.

I like to namespace to allow for easy versioning.

So let's configure the namespace in our routing so it now looks like

```ruby
Rails.application.routes.draw do
	resources :people

	namespace :api do
	  namespace :v0 do
		  resources :people
	  end
	end

  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  # Defines the root path route ("/")
  root "people#index"
end
```

Let's start by building a JSON only api controller with:

```ruby
mkdir app/controllers/api
cat << EOF > app/controllers/api/json_controller.rb
module Api
  class JsonController < ActionController::API
  end
end
EOF
```

Now lets build an api v0 person controller that inherits from the aoi only controller
```ruby
mkdir app/controllers/api/v0

cat << EOF > app/controllers/api/v0/people_controller.rb
module Api
  module V0
    class PeopleController < JsonController
      before_action :set_person, only: %i[ show update destroy ]

      # GET /people
      def index
        @people = Person.all

        render json: @people
      end

      # GET /people/1
      def show
        render json: @person
      end

      # POST /people
      def create
        @person = Person.new(person_params)

        if @person.save
          render json: @person, status: :created, location: @person
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # PATCH/PUT /people/1
      def update
        if @person.update(person_params)
          render json: @person, status: :ok, location: @person
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # DELETE /people/1
      def destroy
        @person.destroy!
      end

      private

      # Use callbacks to share common setup or constraints between actions.
      def set_person
        @person = Person.find(params[:id])
      end

      # Only allow a list of trusted parameters through.
      def person_params
        params.require(:person)
              .permit(:first_name, :last_name, :nick_name, :given_name, :gender)
      end
    end
  end
end
EOF
```


Let's test:

```bash
bin/rails s -p 3030

curl http://localhost:3030/api/v0/people/1.json

{
  "id":1,
  "first_name":"Zeke",
  "last_name":"Flintstone",
  "given_name":null,
  "nick_name":null,
  "gender":"male",
  "created_at":"2024-03-29T21:35:46.032Z",
  "updated_at":"2024-03-29T21:35:46.032Z"
}


curl http://localhost:3030/api/v0/people.json

#this returns way to much info:
[
  { "id":1,
    "first_name":"Zeke",
    "last_name":"Flintstone",
    "given_name":null,
    "nick_name":null,
    "gender":"male",
    "created_at":"2024-03-29T21:35:46.032Z",
    "updated_at":"2024-03-29T21:35:46.032Z" },
  { "id":2,
    "first_name":"Jed",
    "last_name":"Flintstone",
    "given_name":null,
    "nick_name":null,
    "gender":"male",
    "created_at":"2024-03-29T21:35:46.055Z",
    "updated_at":"2024-03-29T21:35:46.055Z" }
    ...
]
```

assuming this works is as expected let's snapshot our work:

```bash
git add .
git commit -m "namespaced a basic person api that returns everything"
```

## JSON V1 (restrict data returned)

Let's make a breaking change to our api and only return fields we wish and not all fields. So we will make a v1 and update the routes with:

```ruby
Rails.application.routes.draw do
	resources :people

	namespace :api do
	  namespace :v0 do
		  resources :people
	  end
	  namespace :v1 do
		  resources :people
	  end
	end

  get "up" => "rails/health#show", as: :rails_health_check

  root "people#index"
end
```

Let's return only: id, first_name, last_name, and gender by changing how we return data to use:
`render json: @people.as_json(only: [:id, :first_name, :last_name, :gender, :given_last_name])`

or we can use:

`render json: @people.as_json(except: [:created_at, :updated_at])`

so now our controller would look like:

```ruby
mkdir app/controllers/api/v1

cat << EOF > app/controllers/api/v1/people_controller.rb
# returns only specific fields
module Api
  module V1
    class PeopleController < JsonController
      before_action :set_person, only: %i[ show update destroy ]

      # GET /people
      def index
        @people = Person.all

        render json: @people.as_json(
          except: [:created_at, :updated_at]
        )
      end

      # GET /people/1
      def show
        render json: @person.as_json(
          only: [:id, :nick_name, :first_name, :last_name, :given_name, :gender]
        )
      end

      # POST /people
      def create
        @person = Person.new(person_params)

        if @person.save
          render json: @person.as_json(
            except: [:created_at, :updated_at]
          ), status: :created, location: @person
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # PATCH/PUT /people/1
      def update
        if @person.update(person_params)
          render json: @person.as_json(
            only: [:id, :nick_name, :first_name, :last_name, :given_name, :gender]
          ), status: :ok, location: @person
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # DELETE /people/1
      def destroy
        @person.destroy!
      end

      private

      # Use callbacks to share common setup or constraints between actions.
      def set_person
        @person = Person.find(params[:id])
      end

      # Only allow a list of trusted parameters through.
      def person_params
        params.require(:person)
              .permit(:first_name, :last_name, :nick_name, :given_name, :gender)
      end
    end
  end
end
EOF
```

Let's test:

```bash
bin/rails s -p 3030

curl http://localhost:3030/api/v1/people/1.json

{
  "id":1,
  "first_name":"Zeke",
  "last_name":"Flintstone",
  "given_name":null,
  "nick_name":null,
  "gender":"male",
}

curl http://localhost:3030/api/v1/people.json

[
  { "id":1,
    "first_name":"Zeke",
    "last_name":"Flintstone",
    "given_name":null,
    "nick_name":null,
    "gender":"male",},
  { "id":2,
    "first_name":"Jed",
    "last_name":"Flintstone",
    "given_name":null,
    "nick_name":null,
    "gender":"male"}
    ...
]
```

Assuming this works, lets snapshot here:
```bash
git add .
git commit -m "added v1 aoi - restricts data fields shared"
```

### Refactor

This is good, but every time we change our return or the model we need to make adjustments everywhere -- lets centralize our data send with a method like:
```ruby
      private

      def render_formatted_people(ar_query, options = {})
        render json: ar_query.as_json(
          only: [:id, :first_name, :last_name, :gender, :given_last_name]
        ), **options
      end
```

or alternatively
```ruby
      private

      def render_formatted_people(ar_query, options = {})
        render json: ar_query.as_json(
          except: [ :updated_at, :created_at ]
        ), **options
      end
```
so now we can rewrite the controller like:

```ruby
module Api
  module V1
    class PeopleController < JsonController
      before_action :set_person, only: %i[ show update destroy ]

      # GET /people
      def index
        @people = Person.all

        render_json_person(@people)
      end

      # GET /people/1
      def show
        render_json_person(@person)
      end

      # POST /people
      def create
        @person = Person.new(person_params)

        if @person.save
          options = { status: :created, location: @person }
          render_json_person(@person, options)
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # PATCH/PUT /people/1
      def update
        if @person.update(person_params)
          options = { status: :ok, location: @person }
          render_json_person(@person, options)
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # DELETE /people/1
      def destroy
        @person.destroy!
      end

      private

      def render_json_person(ar_query, options = {})
        render json: ar_query.as_json(
          except: [ :updated_at, :created_at ]
        ), **options
      end

      # Use callbacks to share common setup or constraints between actions.
      def set_person
        @person = Person.find(params[:id])
      end

      # Only allow a list of trusted parameters through.
      def person_params
        params.require(:person)
              .permit(:first_name, :last_name, :nick_name, :given_name, :gender)
      end
    end
  end
end
```

Let's do a quick test to see that everything still works:

```bash
curl http://localhost:3030/api/v1/people.json
```
Here is are the commands to test all the actions:

index:
```bash
curl http://localhost:3030/api/v1/people.json
# or
curl -X GET http://localhost:3030/api/v1/people \
     -H "Accept: application/json"
```

show:
```bash
curl http://localhost:3030/api/v1/people/1.json
# or
curl -X GET http://localhost:3030/api/v1/people/1 \
     -H "Accept: application/json"
```

delete:
```
curl -X DELETE http://localhost:3030/api/v1/people/2 \
     -H "Accept: application/json"
```

Update (patch / **put**):
```
curl -X PATCH http://localhost:3030/api/v1/people/2 \
     -H "Content-Type: application/json" \
     -d '{"first_name": "NewFirstName", "last_name": "NewLastName"}'
```

**Create** a new person:
```
curl -X POST http://localhost:3030/api/v1/people \
     -H "Content-Type: application/json" \
     -d '{"first_name": "John", "last_name": "Doe"}'
```

now we can snapshot our refactoring:

```bash
git add .
git commit -m "refactored v1 to easily managed return values"
```

## Nested Associations

### add companies

It's nice to list users, but we are also interested in additional information about our people - like their jobs and the associated companies they work for.

lets add these to the code.

first will will add the companies:

```bash
bin/rails g scaffold Company name
```

let's make the company name required and unique in the database:
```ruby
class CreateCompanies < ActiveRecord::Migration[7.2]
  def change
    create_table :companies do |t|
      t.string :name, null: false

      t.timestamps
    end

    add_index :companies, :name, unique: true
  end
end
```

let's normalize the name field, and add the validations to our database to reflect the database restrictions
```ruby
class Company < ApplicationRecord
  normalizes :name,  with: ->(value) { value.strip }

  validates :name, presence: true
  validates :name, uniqueness: true
end
```

be sure the companies act as expected - you can add the company seed data to the database. The easiest was to add more data is just to recreate the DB and reseed your data.

```bash
bin/rails db:drop
bin/rails db:create
bin/rails db:migrate
bin/rails db:seed
```

let's test:
```ruby
bin/rails c

Company.first

Company.all
```

assuming everything works as expected:

```bash
git add .
git commit -m "add company information"
```

### add Jobs

a job is associated with a company and a role so we can generate our code with:

```bash
bin/rails g scaffold Job role company:references
```

now let's update the migration to prevent the same job from being added twice (we will now do a unique index on 2 columns) with `add_index :jobs, %i[role company_id], unique: true` and ensure we have a role with `t.string :role, null: false` so now the migration looks like:

```ruby
class CreateJobs < ActiveRecord::Migration[7.2]
  def change
    create_table :jobs do |t|
      t.string :role, null: false
      t.references :company, null: false, foreign_key: true, index: true

      t.timestamps
    end

    add_index :jobs, %i[role company_id], unique: true
  end
end
```

again let's update the job model to reflect our database structure by adding `belongs_to :company` and removing leading and trailing empty spaces in the role field with `normalizes :role, with: ->(value) { value.strip }` finally adding the `2 column` uniqueness and presence validations.
```ruby
class Job < ApplicationRecord
  belongs_to :company

  normalizes :role, with: ->(value) { value.strip }

  validates :company, presence: true
  validates :role, presence: true
  validates :role,
            uniqueness: { scope: :company_id,
                          message: "role and company already exists" }
end
```

we should now also update `company` to reflect that it can have many jobs with `has_many :jobs` so now the class looks like.
```ruby
class Company < ApplicationRecord
  has_many :jobs

  normalizes :name,  with: ->(value) { value.strip }

  validates :name, presence: true
  validates :name, uniqueness: true
end
```

Let's update the seeds file and run our migrations.

```bash
bin/rails db:drop
bin/rails db:create
bin/rails db:migrate
bin/rails db:seed
```

let's test (especially the associations):
```ruby
bin/rails c

Company.first
# #<Company:0x0000000122e58ad0
#  id: 1,
#  name: "San Cemente",
#  created_at: Sun, 31 Mar 2024 15:32:31.512139000 UTC +00:00,
#  updated_at: Sun, 31 Mar 2024 15:32:31.512139000 UTC +00:00>

Company.first.jobs
# [#<Job:0x0000000122e5b7d0
#   id: 1,
#   role: "owner",
#   company_id: 1,
#   created_at: Sun, 31 Mar 2024 15:32:31.693088000 UTC +00:00,
#   updated_at: Sun, 31 Mar 2024 15:32:31.693088000 UTC +00:00>]
```

Let's test the reverse too:
```ruby
bin/rails c

Job.last
# #<Job:0x0000000104e43720
#  id: 21,
#  role: "The Grand Poobah",
#  company_id: 15,
#  created_at: Sun, 31 Mar 2024 15:32:31.877515000 UTC +00:00,
#  updated_at: Sun, 31 Mar 2024 15:32:31.877515000 UTC +00:00>

Job.last.company
# #<Company:0x0000000131d28480
#  id: 15,
#  name: "Water Buffalo Lodge",
#  created_at: Sun, 31 Mar 2024 15:32:31.637401000 UTC +00:00,
#  updated_at: Sun, 31 Mar 2024 15:32:31.637401000 UTC +00:00>
```

cool, its looking good - let's snapshot before ew

```bash
git add .
git commit -m "added jobs in association with companies"
```

### PersonJobs

now we want to associate people with jobs (and thereby also companies)

so we will add a person_jobs join table (with start and end dates - since some characters take on different jobs) using:

```bash
bin/rails g model PersonJob start_date:date end_date:date \
                  person:references job:references
```

let's ensure that a PersonJob always has a `start_date` and each person can only have one job with given start_date (Barney for example switches back and forth between being a police officer and a dino-crane operator so he may have multiple entries for each job - with differing start dates) so this migration with its 3 column constraint looks like:

```ruby
class CreatePersonJobs < ActiveRecord::Migration[7.2]
  def change
    create_table :person_jobs do |t|
	    t.date :start_date, null: false
		  t.date :end_date
      t.references :person, null: false, foreign_key: true, index: true
      t.references :job, null: false, foreign_key: true, index: true

      t.timestamps
    end

    add_index :person_jobs, %i[person_id job_id start_date], unique: true
  end
end
```

so now let's ensure our new model reflects the database - with the following code.  The logic and code changes should look familiar.
```ruby
class PersonJob < ApplicationRecord
  belongs_to :person
  belongs_to :job

  validates :job, presence: true
  validates :person, presence: true
  validates :start_date, presence: true
  validates :person,
            uniqueness: { scope: [:job, :start_date],
                          message: "person and job with start_date already exists" }
end
```

now let's update the person model so that we can find out who has which jobs at which companies with a bunch of `has_many & has_many through` statements.  Now the model should look like:
```ruby
class Person < ApplicationRecord
  has_many :person_jobs, dependent: :destroy
  has_many :jobs, through: :person_jobs
  has_many :companies, through: :jobs

  normalizes :first_name, :nick_name, :last_name, :given_name,
             with: ->(value) { value.strip }

  validates :first_name,
            uniqueness: { scope: :last_name,
                          message: "first_name and last_name already exists" }
  validates :first_name, presence: true
  validates :last_name, presence: true
  validates :gender, presence: true
  validates :gender, inclusion: { in: %w[male female] }
end
```

if we want to do similar queries with companies to know who works for a company than we need to update the company model with `has_many through` statements as well.

```ruby
class Company < ApplicationRecord
  has_many :jobs, dependent: :destroy
  has_many :person_jobs, through: :jobs
  has_many :people, through: :person_jobs

  normalizes :name,  with: ->(value) { value.strip }

  validates :name, presence: true
  validates :name, uniqueness: true
end
```

and similarly we can add `has_many and has_many_through` in jobs to know about the people working at the company.
```ruby
class Job < ApplicationRecord
  belongs_to :company

  has_many :person_jobs, dependent: :destroy
  has_many :people, through: :person_jobs

  normalizes :role, :title, :company, with: ->(value) { value.strip }

  validates :company, presence: true
  validates :role, presence: true
  validates :role,
            uniqueness: { scope: :company_id,
                          message: "role and company already exists" }
end
```
seed the data for PersonJob and test all the relations.

```bash
bin/rails db:drop
bin/rails db:create
bin/rails db:migrate
bin/rails db:seed
```

let's test (especially the associations):
```ruby
bin/rails c

# Test People Relationships
Person.first.jobs
# [#<Job:0x0000000120351760
#   id: 1,
#   role: "owner",
#   company_id: 1,
#   created_at: Sun, 31 Mar 2024 15:48:33.169108000 UTC +00:00,
#   updated_at: Sun, 31 Mar 2024 15:48:33.169108000 UTC +00:00>]

Person.first.companies
# [#<Company:0x000000011e785158
#   id: 1,
#   name: "San Cemente",
#   created_at: Sun, 31 Mar 2024 15:48:33.006684000 UTC +00:00,
#   updated_at: Sun, 31 Mar 2024 15:48:33.006684000 UTC +00:00>]

# Test Jobs Relationships
Job.first.people
# [#<Person:0x00000001015db900
#   id: 1,
#   first_name: "Zeke",
#   last_name: "Flintstone",
#   nick_name: nil,
#   given_name: nil,
#   gender: "male",
#   created_at: Sun, 31 Mar 2024 15:48:32.620435000 UTC +00:00,
#   updated_at: Sun, 31 Mar 2024 15:48:32.620435000 UTC +00:00>]

Job.first.company
# #<Company:0x000000012086dfc8
#  id: 1,
#  name: "San Cemente",
#  created_at: Sun, 31 Mar 2024 15:48:33.006684000 UTC +00:00,
#  updated_at: Sun, 31 Mar 2024 15:48:33.006684000 UTC +00:00>

# Test Company Relationships
Company.first.jobs
# [#<Job:0x0000000120844ec0
#   id: 1,
#   role: "owner",
#   company_id: 1,
#   created_at: Sun, 31 Mar 2024 15:48:33.169108000 UTC +00:00,
#   updated_at: Sun, 31 Mar 2024 15:48:33.169108000 UTC +00:00>]

Company.first.people
# [#<Person:0x000000011e8ad0d0
#   id: 1,
#   first_name: "Zeke",
#   last_name: "Flintstone",
#   nick_name: nil,
#   given_name: nil,
#   gender: "male",
#   created_at: Sun, 31 Mar 2024 15:48:32.620435000 UTC +00:00,
#   updated_at: Sun, 31 Mar 2024 15:48:32.620435000 UTC +00:00>]
```

Cool now that everything works let's snapshot again.

```bash
git add .
git commit -m "add PersonJob and associations"
```


## JSON V2 - nested data

now that we have our base application and relationships - lets make another breaking change and update the data structure and return nested Job and Company data.

Let's add our new route:

```ruby
Rails.application.routes.draw do
  resources :jobs
  resources :companies
	resources :people

	namespace :api do
	  namespace :v0 do
		  resources :people
	  end
	  namespace :v1 do
		  resources :people
	  end
	  namespace :v2 do
		  resources :people
	  end
	end

  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  # Defines the root path route ("/")
  root "people#index"
end
```

now of course we will need our new controller - to do this we need to adjust our method `render_json_person(ar_query, options = {})` so that it returns our nested data - we do this with the `include` statement so now we will change and a nice mix of `only` and `except` to control what data is actually returned.
```ruby
      def render_json_person(ar_query, options = {})
        render json: ar_query.as_json(
          except: [ :updated_at, :created_at ]
        ), **options
      end
```
to (now we have some nice examples of only, include and except):
```ruby
      def render_json_person(ar_query, options = {})
        render json: model.as_json(
          include: {
            person_jobs: {
              only: [ :start_date, :end_date ],
              include: {
                job: {
                  only: [ :role ],
                  include: {
                    company: { only: [ :id, :name ] }
                  }
                }
              }
            }
          },
          except: [ :updated_at, :created_at ]
        ), **options
      end
```
notice we have nested includes - this is perfectly valid.

NOTE: we haven't adjusted our queries yet (we will refactor shortly), first let's ensure we return the wanted data.

Now our newest controller should look like:
```ruby
mkdir app/controllers/api/v2

cat << EOF > app/controllers/api/v2/people_controller.rb
# returns nested jobs and companies for each person.
module Api
  module V2
    class PeopleController < JsonController
      before_action :set_person, only: %i[ show update destroy ]

      # GET /people
      def index
        @people = Person.all

        render_json_person(@people)
      end

      # GET /people/1
      def show
        render_json_person(@person)
      end

      # POST /people
      def create
        @person = Person.new(person_params)

        if @person.save
          options = { status: :created, location: @person }
          render_json_person(@person, options)
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # PATCH/PUT /people/1
      def update
        if @person.update(person_params)
          options = { status: :ok, location: @person }
          render_json_person(@person, options)
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # DELETE /people/1
      def destroy
        @person.destroy!
      end

      private

      # Use callbacks to share common setup or constraints between actions.
      def set_person
        @person = Person.find(params[:id])
      end

      # Only allow a list of trusted parameters through.
      def person_params
        params.require(:person)
              .permit(:first_name, :last_name, :nick_name, :given_name, :gender)
      end

      def render_json_person(ar_query, options = {})
        render json: ar_query.as_json(
          include: {
            person_jobs: {
              only: [ :start_date, :end_date ],
              include: {
                job: {
                  only: [ :role ],
                  include: {
                    company: { only: [ :id, :name ] }
                  }
                }
              }
            }
          },
          except: [ :updated_at, :created_at ]
        ), **options
      end
    end
  end
end
EOF
```
now when we test show with:

`curl http://localhost:3030/api/v2/people/8.json`

we should get:
```json
{ "id":8,
  "first_name":"Wilma",
  "last_name":"Flintstone",
  "given_name":"Slaghoople",
  "nick_name":null,
  "gender":"female",
  "person_jobs": [
    { "start_date":"1980-01-01",
      "end_date":"1989-12-31",
      "job": {
        "role":"news reporter",
        "company": {
          "id":2,
          "name":"Bedrock Daily News"
    }}},
    { "start_date":"1990-01-01",
      "end_date":null,
      "job":{
        "role":"caterer",
        "company":{
          "id":6,
          "name":"Betty and Wilma Catering"
    }}},
    ...
  ]
}
```

cool let's snapshot this:
```bash
git add .
git commit -m "add JSON v2 people with nested job and company data"
```

### Refactor: fix N+1 Queries

when we query `index` with:

`curl http://localhost:3030/api/v2/people.json`

we notice we have a problem we need many queries to return all the people, their jobs and workplaces.  Here is an abbreviated log from such a request (with over 100 queries to process our results).

```log
Started GET "/api/v3/people.json" for ::1 at 2024-03-31 13:11:12 +0200
Processing by Api::V3::PeopleController#index as JSON
  Person Load (3.4ms)  SELECT "people".* FROM "people"
  ↳ app/controllers/api/v3/people_controller.rb:15:in `index'
  PersonJob Load (1.4ms)  SELECT "person_jobs".* FROM "person_jobs" WHERE "person_jobs"."person_id" = $1  [["person_id", 1]]
  ↳ app/controllers/api/v3/people_controller.rb:15:in `index'
  Job Load (1.1ms)  SELECT "jobs".* FROM "jobs" WHERE "jobs"."id" = $1 LIMIT $2  [["id", 1], ["LIMIT", 1]]
  ↳ app/controllers/api/v3/people_controller.rb:15:in `index'
  Company Load (1.1ms)  SELECT "companies".* FROM "companies" WHERE "companies"."id" = $1 LIMIT $2  [["id", 1], ["LIMIT", 1]]
  ...
  PersonJob Load (1.2ms)  SELECT "person_jobs".* FROM "person_jobs" WHERE "person_jobs"."person_id" = $1  [["person_id", 52]]
  ↳ app/controllers/api/v3/people_controller.rb:15:in `index'
  Job Load (1.2ms)  SELECT "jobs".* FROM "jobs" WHERE "jobs"."id" = $1 LIMIT $2  [["id", 21], ["LIMIT", 1]]
  ↳ app/controllers/api/v3/people_controller.rb:15:in `index'
  Company Load (1.5ms)  SELECT "companies".* FROM "companies" WHERE "companies"."id" = $1 LIMIT $2  [["id", 15], ["LIMIT", 1]]
  ↳ app/controllers/api/v3/people_controller.rb:15:in `index'
Completed 200 OK in 248ms (Views: 0.4ms | ActiveRecord: 146.3ms | Allocations: 105119)
```
When the number of queries grows for each record called - this is called an N+1 query and are very inefficient and slow with large amounts of data.  Notice this request took `250ms` to return to us.

We can fix (refactor our queries with `includes`) this will let the database return associated data with a fixed number of queries - that doesn't grow as we return more data.  To do this lets change:

`@people = Person.all`
to
`@people = Person.includes(person_jobs: [ job: :company ]).all`
this is a concise way of writing:
```ruby
@people = Person.includes(:person_jobs) # return nested `person_jobs` data for `has_many :person_jobs`
                .includes(person_jobs: :job) # return nested `jobs` for `has_many :jobs, through: :person_jobs
                .includes(person_jobs: [ job: :company ]) # return nested `companies` for `has_many :companies, through: :jobs
                .all
```

less critical but we can also rewrite:
`@person = Person.find_by(params[:id])`
to
```ruby
@person = Person.includes(person_jobs: [ job: :company ])
                .where(id: params[:id])
                .limit(1).first
```

In fact, I like to make this into a method so that the base query is always the same I do this by adding `person_query` and rewriting `index` and `set_person` to use the person query:

```ruby
      # GET /people
      def index
        @people = person_query.all

        render_json_person(@people)
      end

      private

      def person_query
        Person.includes(person_jobs: [ job: :company ])
      end

      # Use callbacks to share common setup or constraints between actions.
      def set_person
        @person = person_query.where(id: params[:id])
                              .limit(1).first
      end
      # ...
```


so refactored the controller (that avoids n+1) now looks like:
```ruby
module Api
  module V2
    class PeopleController < JsonController
      before_action :set_person, only: %i[ show update destroy ]

      # GET /people
      def index
        @people = person_query.all

        render_json_person(@people)
      end

      # GET /people/1
      def show
        render_json_person(@person)
      end

      # POST /people
      def create
        @person = Person.new(person_params)

        if @person.save
          options = { status: :created, location: @person }
          render_json_person(@person, options)
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # PATCH/PUT /people/1
      def update
        if @person.update(person_params)
          options = { status: :ok, location: @person }
          render_json_person(@person, options)
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # DELETE /people/1
      def destroy
        @person.destroy!
      end

      private

      def person_query
        Person.includes(person_jobs: [ job: :company ])
      end

      # Use callbacks to share common setup or constraints between actions.
      def set_person
        @person = person_query.where(id: params[:id])
                              .limit(1).first
      end

      # Only allow a list of trusted parameters through.
      def person_params
        params.require(:person)
              .permit(:first_name, :last_name, :nick_name, :given_name, :gender)
      end

      def render_json_person(ar_query, options = {})
        render json: ar_query.as_json(
          include: {
            person_jobs: {
              only: [ :start_date, :end_date ],
              include: {
                job: {
                  only: [ :role ],
                  include: {
                    company: { only: [ :id, :name ] }
                  }
                }
              }
            }
          },
          except: [ :updated_at, :created_at ]
        ), **options
      end
    end
  end
end
```

now when we test index again with:

we see significant improvements:
- queries from over 100 down to 4 (that won't grow)
- response time from 260ms down to 60ms (that won't grow much)

Here is the log/proof.
```log
Started GET "/api/v2/people.json" for ::1 at 2024-03-31 13:07:50 +0200
Processing by Api::V2::PeopleController#index as JSON
  Person Load (4.1ms)  SELECT "people".* FROM "people"
  ↳ app/controllers/api/v3/people_controller.rb:14:in `index'
  PersonJob Load (0.9ms)  SELECT "person_jobs".* FROM "person_jobs" WHERE "person_jobs"."person_id" IN ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16, $17, $18, $19, $20, $21, $22, $23, $24, $25, $26, $27, $28, $29, $30, $31, $32, $33, $34, $35, $36, $37, $38, $39, $40, $41, $42, $43, $44, $45, $46, $47, $48, $49, $50, $51, $52)  [["person_id", 1], ["person_id", 2], ["person_id", 3], ["person_id", 4], ["person_id", 5], ["person_id", 6], ["person_id", 7], ["person_id", 8], ["person_id", 9], ["person_id", 10], ["person_id", 11], ["person_id", 12], ["person_id", 13], ["person_id", 14], ["person_id", 15], ["person_id", 16], ["person_id", 17], ["person_id", 18], ["person_id", 19], ["person_id", 20], ["person_id", 21], ["person_id", 22], ["person_id", 23], ["person_id", 24], ["person_id", 25], ["person_id", 26], ["person_id", 27], ["person_id", 28], ["person_id", 29], ["person_id", 30], ["person_id", 31], ["person_id", 32], ["person_id", 33], ["person_id", 34], ["person_id", 35], ["person_id", 36], ["person_id", 37], ["person_id", 38], ["person_id", 39], ["person_id", 40], ["person_id", 41], ["person_id", 42], ["person_id", 43], ["person_id", 44], ["person_id", 45], ["person_id", 46], ["person_id", 47], ["person_id", 48], ["person_id", 49], ["person_id", 50], ["person_id", 51], ["person_id", 52]]
  ↳ app/controllers/api/v2/people_controller.rb:14:in `index'
  Job Load (1.4ms)  SELECT "jobs".* FROM "jobs" WHERE "jobs"."id" IN ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16, $17, $18, $19, $20, $21)  [["id", 1], ["id", 2], ["id", 4], ["id", 3], ["id", 5], ["id", 6], ["id", 8], ["id", 10], ["id", 7], ["id", 12], ["id", 11], ["id", 16], ["id", 9], ["id", 17], ["id", 15], ["id", 13], ["id", 14], ["id", 18], ["id", 19], ["id", 20], ["id", 21]]
  ↳ app/controllers/api/v2/people_controller.rb:14:in `index'
  Company Load (1.3ms)  SELECT "companies".* FROM "companies" WHERE "companies"."id" IN ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15)  [["id", 1], ["id", 13], ["id", 12], ["id", 8], ["id", 2], ["id", 5], ["id", 6], ["id", 14], ["id", 7], ["id", 10], ["id", 3], ["id", 9], ["id", 11], ["id", 4], ["id", 15]]
  ↳ app/controllers/api/v2/people_controller.rb:14:in `index'
Completed 200 OK in 103ms (Views: 1.0ms | ActiveRecord: 57.2ms | Allocations: 36896)
```

cool lets wrap this up:
```bash
git add .
git commit -m "fixed n+1 query when returning nested person data"
```

Note: you can follow the same logic to write controllers for Jobs and Companies.

## Resources

* https://guides.rubyonrails.org/api_app.html
* https://www.youtube.com/watch?v=5pr1zCYqK00
* https://medium.com/swlh/beginners-guide-to-building-a-rails-api-7b22aa7ec2fb


## APPENDIX

In rails we can seed sample data using the file: `db/seeds.rb`  The data found for this sample project was found from the following links.
```
# db/seeds.rb

# Family Tree and jobs
# https://www.youtube.com/watch?app=desktop&v=AHWVVm_wd0s
#
# Flintstone characters and pets
# https://en.wikipedia.org/wiki/The_Flintstones
# https://www.ranker.com/list/all-the-flintstones-characters/reference

```

### People Seed

```ruby
## PEOPLE
#########
## Flintstone family
# agriculture (hillbilly)
# San Cemente Owner
zeke = Person.create!(first_name: 'Zeke', last_name: 'Flintstone', gender: 'male')
# agriculture (hillbilly)
jed = Person.create!(first_name: 'Jed', last_name: 'Flintstone', gender: 'male')

# soldier / pilot
rocky = Person.create!(first_name: 'Rockbottom', nick_name: 'Rocky', last_name: 'Flintstone', gender: 'male')

# rich uncle
giggles = Person.create!(first_name: 'Jay Giggles', nick_name: 'Uncle Giggles', last_name: 'Flintstone', gender: 'male')

# freeway traffic reporter
pops = Person.create!(first_name: 'Ed Pops', nick_name: 'Pops', last_name: 'Flintstone', gender: 'male')
# homemaker
edna = Person.create!(first_name: 'Edna', last_name: 'Flintstone', given_name: 'Hardrock', gender: 'female')

# married to wilma
# son of pops & edna (crane operator at 'Slate Rock & Gravel Company')
fred = Person.create!(first_name: 'Fredrick Jay', nick_name: 'Fred', last_name: 'Flintstone', gender: 'male')
# married to fred
# reporter & caterer & homemaker
wilma = Person.create!(first_name: 'Wilma', last_name: 'Flintstone', given_name: 'Slaghoople', gender: 'female')

# daughter of fred & wilma, married to bamm-bamm
# advertising executive
pebbles = Person.create!(first_name: 'Pebbles Wilma', nick_name: 'Pebbles', last_name: 'Rubble', given_name: 'Flintstone', gender: 'female')
# adopted brother to pebbles
stoney = Person.create!(first_name: 'Stoney', last_name: 'Flintstone', gender: 'male')


## Hardrock family
# father to Edna, Tex, Jemina (married to Lucile)
james = Person.create!(first_name: 'James', last_name: 'Hardrock', gender: 'male')
# mother to Edna, Tex, Jemina (married to James)
lucile = Person.create!(first_name: 'Lucile', last_name: 'Hardrock', given_name: 'von Stone', gender: 'female')

# sister to Tex & Edna
jemina = Person.create!(first_name: 'Jemina', last_name: 'Hardrock', gender: 'female')

# texrock rangers & rancher (town: texrock)
# brother to Edna
tex = Person.create!(first_name: 'Tex', last_name: 'Hardrock', gender: 'male')

# daughter of tex
mary = Person.create!(first_name: 'Mary Lou', last_name: 'Hardrock', gender: 'female')
# son of tex (ranch owner)
tumbleweed = Person.create!(first_name: 'Tumbleweed', last_name: 'Hardrock', gender: 'male')

## Slaghoople family
# father to Wilma, married to Pearl
ricky = Person.create!(first_name: 'Richard', nick_name: 'Ricky', last_name: 'Slaghoople', gender: 'male')
pearl = Person.create!(first_name: 'Pearl', last_name: 'Slaghoople', gender: 'female')

# wilma's sister
mica = Person.create!(first_name: 'Mica', last_name: 'Slaghoople', gender: 'female')
# wilma's sister
mickey = Person.create!(first_name: 'Michael', nick_name: 'Mickey', last_name: 'Slaghoople', gender: 'female')
# wilma's brother
michael = Person.create!(first_name: 'Jerry', last_name: 'Slaghoople', gender: 'male')

## McBricker family
brick = Person.create!(first_name: 'Brick', last_name: 'McBricker', gender: 'male')
jean = Person.create!(first_name: 'Jean', last_name: 'McBricker', gender: 'female')

# betty's bother (child of brick & jean)
# HS Basketball player
brad = Person.create!(first_name: 'Brad', last_name: 'McBricker', gender: 'male')


## Slate family
# flo's brother (lives in granite town)
# manager of 'Bedrock & Gravel Quarry Company'
mr_slate = Person.create!(first_name: 'George', nick_name: 'Mr.', last_name: 'Slate', gender: 'male')
# married to mr. slate
mrs_slate = Person.create!(first_name: 'Mrs.', last_name: 'Slate', gender: 'female')

# child of mr. slate & mrs. slate
eugene = Person.create!(first_name: 'Eugene', last_name: 'Slate', gender: 'male')
# child of mr. slate & mrs. slate
bessie = Person.create!(first_name: 'Bessie', last_name: 'Slate', gender: 'female')
# bessie's child (son)
eddie = Person.create!(first_name: 'Edward', nick_name: 'Eddie', last_name: 'Slate', gender: 'male')


## Rubble family
# married to flo
# used car salesman
bob = Person.create!(first_name: 'Robert', nick_name: 'Bob', last_name: 'Rubble', gender: 'male')
# married to bob (homemaker)
flo = Person.create!(first_name: 'Florence', nick_name: 'Flo', last_name: 'Rubble', given_name: 'Slate', gender: 'female')

# barney's brother (younger)
dusty = Person.create!(first_name: 'Dusty', last_name: 'Rubble', gender: 'male')

# married to betty (child of bob & flo)
# police officer & crane operator at 'Slate Rock & Gravel Company'
barney = Person.create!(first_name: 'Bernard Matthew', nick_name: 'Barney', last_name: 'Rubble', gender: 'male')
# married to barney, child of brick & jean
# reporter & caterer & homemaker
betty = Person.create!(first_name: 'Elizabeth Jean', nick_name: 'Betty', last_name: 'Rubble', given_name: 'McBricker', gender: 'female')

# adopted son of barney & betty (married to pebbles)
# auto mechanic, then screenwriter
bamm = Person.create!(first_name: 'Bamm-Bamm', last_name: 'Rubble', gender: 'male')
# son of bamm-bamm & pebbles
chip = Person.create!(first_name: 'Charleston Frederick', nick_name: 'Chip', last_name: 'Rubble', gender: 'male')
# daughter of bamm-bamm & pebbles
roxy = Person.create!(first_name: 'Roxann Elisabeth', nick_name: 'Roxy', last_name: 'Rubble', gender: 'female')


## The Gruesomes – A creepy but friendly family, who move in next door to the Flintstones in later seasons.
# Uncle Ghastly – The uncle of Gobby from Creepella's side of the family, who is mostly shown as a large furry hand with claws emerging from a door, a well, or a wall. His shadow was also seen in their debut episode. He wasn't named until his second appearance, which is also the only time he is heard speaking, as he is heard laughing from a well.
ghastly = Person.create!(first_name: 'Ghastly', last_name: 'Gruesome', gender: 'male')
# Weirdly Gruesome – The patriarch of the Gruesome family, who works as a reality-show host.
# reality host
weirdly = Person.create!(first_name: 'Weirdly', last_name: 'Gruesome', gender: 'male')
# Creepella Gruesome – Weirdly's tall wife.
creepella = Person.create!(first_name: 'Creepella', last_name: 'Gruesome', gender: 'female')
# Goblin "Gobby" Gruesome – Weirdly and Creepella's son.
gobby = Person.create!(first_name: 'Goblin', nick_name: 'Gobby', last_name: 'Gruesome', gender: 'male')


## The Hatrocks – A family of hillbillies, who feuded with the Flintstones' Arkanstone branch similarly to the Hatfield–McCoy feud. Fred and Barney reignite a feud with them in "The Bedrock Hillbillies", when Fred inherits San Cemente from his late great-great-uncle Zeke Flintstone and they fight over who made Zeke's portrait. The Hatrocks later return in "The Hatrocks and the Gruesomes", where they bunk with the Flintstones during their trip to Bedrock World's Fair and their antics start to annoy them as they guilt-trip Fred into extending their stay. It is also revealed that they dislike bug music. and the Flintstones, the Rubbles, and the Gruesomes are able to drive them away by performing the Four Insects song "She Said Yeah Yeah Yeah".[a] After learning that the Bedrock World's Fair would feature the Four Insects performing, they fled back to Arkanstone.
# Granny Hatrock – The mother of Jethro and grandmother of Zack and Slab.
granny = Person.create!(first_name: 'Granny', last_name: 'Hatrock', gender: 'female')
# Jethro Hatrock – The patriarch of the Hatrock Family. He had brown hair in "The Hatrocks and the Flintstones" and taupe-gray hair in "The Hatrocks and the Gruesomes".
jethro = Person.create!(first_name: 'Jethro', last_name: 'Hatrock', gender: 'male')
# Gravella Hatrock – Jethro's wife.
gravella = Person.create!(first_name: 'Gravella', last_name: 'Hatrock', gender: 'female')
# Zack Hatrock – Jethro and Gravella's oldest son.
zack = Person.create!(first_name: 'Zack', last_name: 'Hatrock', gender: 'male')
# Slab Hatrock – The youngest son of Jethro and Gravella.
slab = Person.create!(first_name: 'Slab', last_name: 'Hatrock', gender: 'male')
# Benji Hatrock – Jethro's son-in-law.
benji = Person.create!(first_name: 'Benji', last_name: 'Hatrock', gender: 'male')

## others
# Friend to Barney & Fred (fire chief)
joe = Person.create!(first_name: 'Joseph', nick_name: 'Joe', last_name: 'Rockhead', gender: 'male')

# paperboy (town: bedrock)
arnold = Person.create!(first_name: 'Arnold', last_name: 'Granite', gender: 'male')

stoney = Person.create!(first_name: 'Stoney', last_name: 'Curtis', gender: 'male')
perry = Person.create!(first_name: 'Perry', last_name: 'Masonry', gender: 'male')

# Sam Slagheap – The Grand Poobah of the Water Buffalo Lodge.
sam = Person.create!(first_name: 'Samuel', nick_name: 'Sam', last_name: 'Slagheap', gender: 'male')
```


### Company Seeds

```ruby
## Companies
san_cemente = Company.create!(name: 'San Cemente')
bedrock_news = Company.create!(name: 'Bedrock Daily News')
bedrock_police = Company.create!(name: 'Bedrock Police Department')
bedrock_fire = Company.create!(name: 'Bedrock Fire Department')
bedrock_quarry = Company.create!(name: 'Bedrock & Gravel Quarry Company')
betty_wilma_catering = Company.create!(name: 'Betty & Wilma Catering')
texrock_ranch = Company.create!(name: 'Texrock Ranch')
teradactyl = Company.create!(name: 'Teradactyl Flights')
auto_repair = Company.create!(name: 'Bedrock Auto Repair')
used_cars = Company.create!(name: 'Bedrock Used Cars')
bedrock_entetainment = Company.create!(name: 'Bedrock Entertainment')
bedrock_army = Company.create!(name: 'Bedrock Army')
independent = Company.create!(name: 'Independent')
advertising = Company.create!(name: 'Bedrock Advertising')
buffalo_lodge = Company.create!(name: 'Water Buffalo Lodge')
```

### Jobs Seeds

```ruby
## Jobs
## San Cemente Owner
cemente = Job.create!(role: 'owner', company: san_cemente)
# agriculture
farmer = Job.create!(role: 'farmer', company: independent)
# pilot
pilot = Job.create!(role: 'pilot', company: teradactyl)
# soldier
soldier = Job.create!(role: 'soldier', company: bedrock_army)
# wealthy
wealth = Job.create!(role: 'independently wealthy', company: independent)
# reporter
traffic = Job.create!(role: 'traffice reporter', company: bedrock_news)
reporter = Job.create!(role: 'news reporter', company: bedrock_news)
# homemaker
homemaker = Job.create!(role: 'homemaker', company: independent)
# mining company manager
manager = Job.create!(role: 'manager', company: bedrock_quarry)
# crane operator
crane = Job.create!(role: 'crane operator', company: bedrock_quarry)
# advertising executive
advertising = Job.create!(role: 'advertising executive', company: advertising)
# caterer
caterer = Job.create!(role: 'caterer', company: betty_wilma_catering)
# auto mechanic
mechanic = Job.create!(role: 'auto mechanic', company: auto_repair)
# screenwriter
screenwriter = Job.create!(role: 'screenwriter', company: bedrock_entetainment)
# police officer
police = Job.create!(role: 'police officer', company: bedrock_police)
# rancher
rancher = Job.create!(role: 'rancher', company: texrock_ranch)
# used car salesman
salesman = Job.create!(role: 'used car salesman', company: used_cars)
# reality show host
host = Job.create!(role: 'reality show host', company: bedrock_entetainment)
# fire chief
fire_chief = Job.create!(role: 'fire chief', company: bedrock_fire)
# paperboy
paper_delivery = Job.create!(role: 'paperboy', company: bedrock_news)
# Grand Poobah
grand_poobah = Job.create!(role: 'The Grand Poobah', company: buffalo_lodge)
```


### PersonJobs Seeds

```ruby
## Person Jobs
# zeke - San Cemente Owner
PersonJob.create!(person: zeke, job: cemente, start_date: Date.new(1980, 1, 1))
# jed - farmer
PersonJob.create!(person: jed, job: farmer, start_date: Date.new(1980, 1, 1))
# rocky - ww1 soldier
PersonJob.create!(person: rocky, job: soldier, start_date: Date.new(1980, 1, 1), end_date: Date.new(1985, 12, 31))
# rocy - pilot after war
PersonJob.create!(person: rocky, job: pilot, start_date: Date.new(1986, 1, 1))

# giggles rich uncle
PersonJob.create!(person: giggles, job: wealth, start_date: Date.new(1980, 1, 1))

# pops - freeway traffic reporter
PersonJob.create!(person: pops, job: traffic, start_date: Date.new(1980, 1, 1))
# edna - homemaker
PersonJob.create!(person: edna, job: homemaker, start_date: Date.new(1980, 1, 1))

# fred - crane operator
PersonJob.create!(person: fred, job: crane, start_date: Date.new(1980, 1, 1))
# married to fred
# wilma - reporter & caterer & homemaker
PersonJob.create!(person: wilma, job: reporter, start_date: Date.new(1980, 1, 1), end_date: Date.new(1989, 12, 31))
PersonJob.create!(person: wilma, job: caterer, start_date: Date.new(1990, 1, 1))
PersonJob.create!(person: wilma, job: homemaker, start_date: Date.new(1980, 1, 1))

# pebbles - advertising executive
PersonJob.create!(person: pebbles, job: advertising, start_date: Date.new(1995, 1, 1))

# texrock rangers & rancher (town: texrock)
PersonJob.create!(person: tex, job: rancher, start_date: Date.new(2080, 1, 1))

# mr_slate - manager
PersonJob.create!(person: mr_slate, job: manager, start_date: Date.new(1980, 1, 1))


## Rubble family
# bob - used car salesman
PersonJob.create!(person: bob, job: salesman, start_date: Date.new(1980, 1, 1))
# flo - (homemaker)
PersonJob.create!(person: flo, job: homemaker, start_date: Date.new(1980, 1, 1))

# police officer & crane operator at 'Slate Rock & Gravel Company'
PersonJob.create!(person: barney, job: police, start_date: Date.new(1980, 1, 1), end_date: Date.new(1989, 12, 31))
PersonJob.create!(person: barney, job: crane, start_date: Date.new(1990, 1, 1))

# betty - reporter & caterer & homemaker
PersonJob.create!(person: betty, job: reporter, start_date: Date.new(1980, 1, 1), end_date: Date.new(1989, 12, 31))
PersonJob.create!(person: betty, job: caterer, start_date: Date.new(1990, 1, 1))
PersonJob.create!(person: betty, job: homemaker, start_date: Date.new(1980, 1, 1))

# bamm-bamm - auto mechanic, then screenwriter
PersonJob.create!(person: bamm, job: mechanic, start_date: Date.new(1995, 1, 1), end_date: Date.new(1999, 12, 31))
PersonJob.create!(person: bamm, job: screenwriter, start_date: Date.new(2000, 1, 1))


## The Gruesomes
# weirdly - reality host
PersonJob.create!(person: weirdly, job: host, start_date: Date.new(1990, 1, 1))

## others
# joe - fire chief)
PersonJob.create!(person: joe, job: fire_chief, start_date: Date.new(1980, 1, 1))

# paperboy (town: bedrock)
PersonJob.create!(person: arnold, job: paper_delivery, start_date: Date.new(1980, 1, 1))

# Sam Slagheap – The Grand Poobah of the Water Buffalo Lodge.
PersonJob.create!(person: sam, job: grand_poobah, start_date: Date.new(1985, 1, 1))
```
