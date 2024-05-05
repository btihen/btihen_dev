---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x Base App"
subtitle: "Base App for Blogs"
summary: ""
authors: ['btihen']
tags: ['Rails', 'Base App']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2024-04-01T01:20:00+02:00
lastmod: 2024-04-29T01:20:00+02:00
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

A basic Rails app structure that offers a variety of opportunities for various Rails experiments.

This code can be found at: https://github.com/btihen-dev/rails_base_app

## Quick Summary

without explanations nor step:
```bash
# use other options as needed
rails new base_app --database=postgresql -T
# main is good for testing very new features, but leads to lots of update messages
# rails new base_app -T --main --database=postgresql --javascript=esbuild

# setup and git commit (in case you are experimenting and want to rollback)
cd base_app
bin/rails db:create
git add .
git commit -m "initial commit"
```

use generators to build basic code structure
```bash
bin/rails g scaffold Species species_name
bin/rails g scaffold Character nick_name first_name \
            last_name given_name gender species:references
bin/rails g scaffold Company company_name
bin/rails g scaffold Job role company:references
bin/rails g model PersonJob start_date:date end_date:date \
            person:references job:references
git add .
git commit -m "add generated code"
```

update migrations:
```ruby
class CreateSpecies < ActiveRecord::Migration[7.2]
  def change
    create_table :species do |t|
      t.string :species_name, null: false

      t.timestamps
    end

    add_index :species, :species_name, unique: true
  end
end

class CreateCharacter < ActiveRecord::Migration[7.2]
  def change
    create_table :characters do |t|
      t.string :first_name, null: false
      t.string :last_name, null: false
      t.string :nick_name
      t.string :given_name
      t.string :gender, null: false
      t.references :species, null: false, foreign_key: true, index: true

      t.timestamps
    end

    add_index :characters, %i[first_name last_name gender],
              unique: true
  end
end

class CreateCompanies < ActiveRecord::Migration[7.2]
  def change
    create_table :companies do |t|
      t.string :company_name, null: false

      t.timestamps
    end

    add_index :companies, :company_name, unique: true
  end
end

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

class CreatePersonJobs < ActiveRecord::Migration[7.2]
  def change
    create_table :person_jobs do |t|
	    t.date :start_date, null: false
		  t.date :end_date
      t.references :person, null: false, foreign_key: { to_table: :characters }, index: true
      t.references :job, null: false, foreign_key: true, index: true

      t.timestamps
    end

    add_index :person_jobs, %i[person_id job_id start_date], unique: true
  end
end

git add .
git commit -m "update migrations"
```

update models:
```ruby
cat << EOF > app/models/species.rb
class Species < ApplicationRecord
  normalizes :species_name, with: ->(value) { value.strip }

  validates :species_name, presence: true
  validates :species_name, uniqueness: true
end
EOF

cat << EOF > app/models/character.rb
class Character < ApplicationRecord
  belongs_to :species

  has_many :person_jobs, dependent: :destroy, foreign_key: :person_id
  has_many :jobs, through: :person_jobs
  has_many :companies, through: :jobs

  normalizes :first_name, :nick_name, :last_name, :given_name,
             with: ->(value) { value.strip }

  validates :first_name,
            uniqueness: { scope: :last_name,
                          message: "first_name and last_name already exists" }
  validates :first_name, presence: true
  validates :last_name, presence: true
  validates :species, presence: true
  validates :gender, presence: true
  validates :gender, inclusion: { in: %w[male female] }
end
EOF

cat << EOF > app/models/company.rb
class Company < ApplicationRecord
  has_many :jobs, dependent: :destroy
  has_many :person_jobs, through: :jobs
  has_many :people, through: :person_jobs

  normalizes :company_name,  with: ->(value) { value.strip }

  validates :company_name, presence: true
  validates :company_name, uniqueness: true
end
EOF

cat << EOF > app/models/job.rb
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
EOF

cat << EOF > app/models/person_job.rb
class PersonJob < ApplicationRecord
  belongs_to :person, class_name: 'Character', foreign_key: :person_id
  belongs_to :job

  has_one :company, through: :job

  validates :job, presence: true
  validates :person, presence: true
  validates :start_date, presence: true
  validates :person,
            uniqueness: { scope: [ :job, :start_date ],
                          message: "person and job with start_date already exists" }
end
EOF

git add .
git commit -m "update models"
```

update the routes `config/routes.rb` with a root page using `root "characters#index"`:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :jobs
  resources :companies
  resources :characters
  resources :species
  # Define your application routes per the DSL in https://guides.rubyonrails.org/routing.html

  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  # Render dynamic PWA files from app/views/pwa/*
  get "service-worker" => "rails/pwa#service_worker", as: :pwa_service_worker
  get "manifest" => "rails/pwa#manifest", as: :pwa_manifest

  # Defines the root path route ("/")
  root "characters#index"
end
```

UPDATE the Character Controller to prevent N+1 queries:
```ruby
# app/controllers/characters_controller.rb
  def index
    @characters = Character
                  .includes(species: [], person_jobs: { job: :company })
                  .all
  end
```

Also the index view MUST use full paths to avoid N+1 queries - even with all the includes:
```ruby
# app/views/characters/index.html.erb
<p style="color: green"><%= notice %></p>

<% content_for :title, "Characters" %>
<div class="container text-center">

  <div class="row justify-content-start">
    <div class="col-9">
      <h1>Characters</h1>

      <table class="table table-striped table-hover">
        <thead class="sticky-top">
          <tr class="table-primary">

            <th scope="col">
              ID
            </th>
            <th scope="col">
              First Name
            </th>
            <th scope="col">
              Last Name
            </th>
            <th scope="col">
              Gender
            </th>
            <th scope="col">
              Species
            </th>
            <th scope="col">
              Company-Job
            </th>
          </tr>
        </thead>

        <tbody class="scrollable-table">
          <div id="characters">
          <% @characters.each do |character| %>
            <tr id="<%= dom_id(character) %>">
              <th scope="row"><%= link_to "#{character.id}", edit_character_path(character) %></th>
              <td><%= character.first_name %></td>
              <td><%= character.last_name %></td>
              <td><%= character.gender %></td>
              <td><%= character.species.species_name %></td>
              <td class="text-start">
                <ul class="list-unstyled">
                  <% character.person_jobs.each do |person_job| %>
                    <li>
                      <b><%= person_job.job.company.company_name %></b><br>
                      &nbsp; - <%= person_job.job.role %><br>
                      &nbsp; &nbsp;
                      <em>
                        (<%= person_job.start_date.strftime("%e %b '%y") %> -
                         <%= person_job.end_date&.strftime("%e %b '%y") || 'present' %> )
                      </em>
                    </li>
                  <% end %>
                </ul>
              </td>
            </tr>
          <% end %>
          </div>
        </tbody>
      </table>
    </div>

    <div class="col-3">
      <%= link_to "New", new_character_path, class: "mt-5 sticky-top btn btn-primary" %>
    </div>
  </div>
</div>
```

populate seed file `db/seeds.rb` with the seed data (see Appendix):

```bash
# run the migratons
bin/rails db:migrate

# seed app with test data
bin/rails db:seed
```

Test with:
```ruby
bin/rails c

Character.first
Character.first.jobs
Character.first.companies

Job.first
Job.last.people
Job.first.company

Company.first
Company.last.jobs
Company.first.people
```

assuming this works, you should now be good to go!

```bash
git add .
git commit -m "base app with migration and seed data"
```


## Setup with Explanations

The options may change based on the experiment, but in general the following is my preferred starting point.

```bash
# I use ASDF and generally forget to set the local version
asdf local ruby 3.3.0

# generate the new app
rails new base_app -T --main --database=postgresql --javascript=esbuild

# setup and git commit (in case you are experimenting and want to rollback)
cd base_app
bin/rails db:create

git add .
git commit -m "initial commit"
```

## Base App

Now we will use generators to create some basic models and controllers for our base app quickly.

### Add Species

We have several types of characters, humans, aliens, pets, etc. So every character will be associated with a species.

```bash
bin/rails g scaffold Species species_name
```

update migration to require that the `species_name` and must be unique.
```ruby
class CreateSpecies < ActiveRecord::Migration[7.2]
  def change
    create_table :species do |t|
      t.string :species_name, null: false

      t.timestamps
    end

    add_index :species, :species_name, unique: true
  end
end
```

update the model to reflect the model.
```ruby
class Species < ApplicationRecord
  normalizes :species_name, with: ->(value) { value.strip }

  validates :species_name, presence: true
  validates :species_name, uniqueness: true
end
```


### Add Characters

Add a People model, controller and associated views via scaffold.

```bash
bin/rails g scaffold Character nick_name first_name \
            last_name given_name gender species:references
```

Let's update the migration to make the first & last name required as well as gender & lets ensure we cant add the same person twice with a unique index on `first_name last_name gender`

```ruby
class CreateCharacter < ActiveRecord::Migration[7.2]
  def change
    create_table :characters do |t|
      t.string :first_name, null: false
      t.string :last_name, null: false
      t.string :nick_name
      t.string :given_name
      t.string :gender, null: false
      t.references :species, null: false, foreign_key: true, index: true

      t.timestamps
    end

    add_index :characters, %i[first_name last_name gender],
              unique: true
  end
end
```

Let's update the model to trim extra leading and trailing spaces and enforce the same DB options as at the DB level:

```ruby
class Character < ApplicationRecord
  belongs_to :species

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

Character.first

Character.all
```

Let's snapshot assuming this works and shows reasonable data.
```bash
git add .
git commit -m "Add person model"
```

### Add Companies

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
      t.string :company_name, null: false

      t.timestamps
    end

    add_index :companies, :company_name, unique: true
  end
end
```

let's normalize the name field, and add the validations to our database to reflect the database restrictions
```ruby
class Company < ApplicationRecord
  normalizes :company_name,  with: ->(value) { value.strip }

  validates :company_name, presence: true
  validates :company_name, uniqueness: true
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

  normalizes :job_title,  with: ->(value) { value.strip }

  validates :job_title, presence: true
  validates :job_title, uniqueness: true
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
#  company_name: "San Cemente",
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
#  company_name: "Water Buffalo Lodge",
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
  belongs_to :person, class_name: "Character", foreign_key: :person_id
  belongs_to :job

  has_one :company, through: :job

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
class Character < ApplicationRecord
  belongs_to :species

  has_many :person_jobs, dependent: :destroy, foreign_key: :person_id
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

  normalizes :company_name,  with: ->(value) { value.strip }

  validates :company_name, presence: true
  validates :company_name, uniqueness: true
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

# Test Character Relationships
Character.first.jobs
# [#<Job:0x0000000120351760
#   id: 1,
#   role: "owner",
#   company_id: 1,
#   created_at: Sun, 31 Mar 2024 15:48:33.169108000 UTC +00:00,
#   updated_at: Sun, 31 Mar 2024 15:48:33.169108000 UTC +00:00>]

Character.first.companies
# [#<Company:0x000000011e785158
#   id: 1,
#   company_name: "San Cemente",
#   created_at: Sun, 31 Mar 2024 15:48:33.006684000 UTC +00:00,
#   updated_at: Sun, 31 Mar 2024 15:48:33.006684000 UTC +00:00>]

# Test Jobs Relationships
Job.first.people
# [#<Character:0x00000001015db900
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
#  company_name: "San Cemente",
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
# [#<Character:0x000000011e8ad0d0
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

## APPENDIX (Seeds File)

In rails we can seed sample data using the file: `db/seeds.rb`  The data found for this sample project was found from the following links.
```
# db/seeds.rb

# Family Tree and jobs
# https://www.youtube.com/watch?app=desktop&v=AHWVVm_wd0s

# Flintstone characters and pets
# https://en.wikipedia.org/wiki/The_Flintstones
# https://www.ranker.com/list/all-the-flintstones-characters/reference

```

### Seed File

```ruby
## Planet
#########
# earth = Planet.create!(name: 'Earth')
# zetox = Planet.create!(name: 'Zetox')

## Species
##########
human = Species.create!(species_name: 'human')
alien = Species.create!(species_name: 'alien')
dino = Species.create!(species_name: 'dinosaur')
kanga = Species.create!(species_name: 'kangaroo')
tiger = Species.create!(species_name: 'saber-tooth tiger')

## PEOPLE
#########
## Flintstone family
# agriculture (hillbilly)
# San Cemente Owner
zeke = Character.create!(species: human, first_name: 'Zeke', last_name: 'Flintstone', gender: 'male')
# agriculture (hillbilly)
jed = Character.create!(species: human, first_name: 'Jed', last_name: 'Flintstone', gender: 'male')

# soldier / pilot
rocky = Character.create!(species: human, first_name: 'Rockbottom', nick_name: 'Rocky', last_name: 'Flintstone', gender: 'male')

# rich uncle
giggles = Character.create!(species: human, first_name: 'Jay Giggles', nick_name: 'Uncle Giggles', last_name: 'Flintstone', gender: 'male')

# freeway traffic reporter
pops = Character.create!(species: human, first_name: 'Ed Pops', nick_name: 'Pops', last_name: 'Flintstone', gender: 'male')
# homemaker
edna = Character.create!(species: human, first_name: 'Edna', last_name: 'Flintstone', given_name: 'Hardrock', gender: 'female')

# married to wilma
# son of pops & edna (crane operator at 'Slate Rock & Gravel Company')
fred = Character.create!(species: human, first_name: 'Fredrick Jay', nick_name: 'Fred', last_name: 'Flintstone', gender: 'male')
# married to fred
# reporter & caterer & homemaker
wilma = Character.create!(species: human, first_name: 'Wilma', last_name: 'Flintstone', given_name: 'Slaghoople', gender: 'female')

# daughter of fred & wilma, married to bamm-bamm
# advertising executive
pebbles = Character.create!(species: human, first_name: 'Pebbles Wilma', nick_name: 'Pebbles', last_name: 'Rubble', given_name: 'Flintstone', gender: 'female')
# adopted brother to pebbles
stoney = Character.create!(species: human, first_name: 'Stoney', last_name: 'Flintstone', gender: 'male')


## Hardrock family
# father to Edna, Tex, Jemina (married to Lucile)
james = Character.create!(species: human, first_name: 'James', last_name: 'Hardrock', gender: 'male')
# mother to Edna, Tex, Jemina (married to James)
lucile = Character.create!(species: human, first_name: 'Lucile', last_name: 'Hardrock', given_name: 'von Stone', gender: 'female')

# sister to Tex & Edna
jemina = Character.create!(species: human, first_name: 'Jemina', last_name: 'Hardrock', gender: 'female')

# texrock rangers & rancher (town: texrock)
# brother to Edna
tex = Character.create!(species: human, first_name: 'Tex', last_name: 'Hardrock', gender: 'male')

# daughter of tex
mary = Character.create!(species: human, first_name: 'Mary Lou', last_name: 'Hardrock', gender: 'female')
# son of tex (ranch owner)
tumbleweed = Character.create!(species: human, first_name: 'Tumbleweed', last_name: 'Hardrock', gender: 'male')

## Slaghoople family
# father to Wilma, married to Pearl
ricky = Character.create!(species: human, first_name: 'Richard', nick_name: 'Ricky', last_name: 'Slaghoople', gender: 'male')
pearl = Character.create!(species: human, first_name: 'Pearl', last_name: 'Slaghoople', gender: 'female')

# wilma's sister
mica = Character.create!(species: human, first_name: 'Mica', last_name: 'Slaghoople', gender: 'female')
# wilma's sister
mickey = Character.create!(species: human, first_name: 'Michael', nick_name: 'Mickey', last_name: 'Slaghoople', gender: 'female')
# wilma's brother
michael = Character.create!(species: human, first_name: 'Jerry', last_name: 'Slaghoople', gender: 'male')

## McBricker family
brick = Character.create!(species: human, first_name: 'Brick', last_name: 'McBricker', gender: 'male')
jean = Character.create!(species: human, first_name: 'Jean', last_name: 'McBricker', gender: 'female')

# betty's bother (child of brick & jean)
# HS Basketball player
brad = Character.create!(species: human, first_name: 'Brad', last_name: 'McBricker', gender: 'male')


## Slate family
# flo's brother (lives in granite town)
# manager of 'Bedrock & Gravel Quarry Company'
mr_slate = Character.create!(species: human, first_name: 'George', nick_name: 'Mr.', last_name: 'Slate', gender: 'male')
# married to mr. slate
mrs_slate = Character.create!(species: human, first_name: 'Mrs.', last_name: 'Slate', gender: 'female')

# child of mr. slate & mrs. slate
eugene = Character.create!(species: human, first_name: 'Eugene', last_name: 'Slate', gender: 'male')
# child of mr. slate & mrs. slate
bessie = Character.create!(species: human, first_name: 'Bessie', last_name: 'Slate', gender: 'female')
# bessie's child (son)
eddie = Character.create!(species: human, first_name: 'Edward', nick_name: 'Eddie', last_name: 'Slate', gender: 'male')

## Rubble family
# married to flo
# used car salesman
bob = Character.create!(species: human, first_name: 'Robert', nick_name: 'Bob', last_name: 'Rubble', gender: 'male')
# married to bob (homemaker)
flo = Character.create!(species: human, first_name: 'Florence', nick_name: 'Flo', last_name: 'Rubble', given_name: 'Slate', gender: 'female')

# barney's brother (younger)
dusty = Character.create!(species: human, first_name: 'Dusty', last_name: 'Rubble', gender: 'male')

# married to betty (child of bob & flo)
# police officer & crane operator at 'Slate Rock & Gravel Company'
barney = Character.create!(species: human, first_name: 'Bernard Matthew', nick_name: 'Barney', last_name: 'Rubble', gender: 'male')
# married to barney, child of brick & jean
# reporter & caterer & homemaker
betty = Character.create!(species: human, first_name: 'Elizabeth Jean', nick_name: 'Betty', last_name: 'Rubble', given_name: 'McBricker', gender: 'female')

# adopted son of barney & betty (married to pebbles)
# auto mechanic, then screenwriter
bamm = Character.create!(species: human, first_name: 'Bamm-Bamm', last_name: 'Rubble', gender: 'male')
# son of bamm-bamm & pebbles
chip = Character.create!(species: human, first_name: 'Charleston Frederick', nick_name: 'Chip', last_name: 'Rubble', gender: 'male')
# daughter of bamm-bamm & pebbles
roxy = Character.create!(species: human, first_name: 'Roxann Elisabeth', nick_name: 'Roxy', last_name: 'Rubble', gender: 'female')

## The Gruesomes – A creepy but friendly family, who move in next door to the Flintstones in later seasons.
# Uncle Ghastly – The uncle of Gobby from Creepella's side of the family, who is mostly shown as a large furry hand with claws emerging from a door, a well, or a wall. His shadow was also seen in their debut episode. He wasn't named until his second appearance, which is also the only time he is heard speaking, as he is heard laughing from a well.
ghastly = Character.create!(species: human, first_name: 'Ghastly', last_name: 'Gruesome', gender: 'male')
# Weirdly Gruesome – The patriarch of the Gruesome family, who works as a reality-show host.
# reality host
weirdly = Character.create!(species: human, first_name: 'Weirdly', last_name: 'Gruesome', gender: 'male')
# Creepella Gruesome – Weirdly's tall wife.
creepella = Character.create!(species: human, first_name: 'Creepella', last_name: 'Gruesome', gender: 'female')
# Goblin "Gobby" Gruesome – Weirdly and Creepella's son.
gobby = Character.create!(species: human, first_name: 'Goblin', nick_name: 'Gobby', last_name: 'Gruesome', gender: 'male')


## The Hatrocks
# Granny Hatrock – The mother of Jethro and grandmother of Zack and Slab.
granny = Character.create!(species: human, first_name: 'Granny', last_name: 'Hatrock', gender: 'female')
# Jethro Hatrock – The patriarch of the Hatrock Family. He had brown hair in "The Hatrocks and the Flintstones" and taupe-gray hair in "The Hatrocks and the Gruesomes".
jethro = Character.create!(species: human, first_name: 'Jethro', last_name: 'Hatrock', gender: 'male')
# Gravella Hatrock – Jethro's wife.
gravella = Character.create!(species: human, first_name: 'Gravella', last_name: 'Hatrock', gender: 'female')
# Zack Hatrock – Jethro and Gravella's oldest son.
zack = Character.create!(species: human, first_name: 'Zack', last_name: 'Hatrock', gender: 'male')
# Slab Hatrock – The youngest son of Jethro and Gravella.
slab = Character.create!(species: human, first_name: 'Slab', last_name: 'Hatrock', gender: 'male')
# Benji Hatrock – Jethro's son-in-law.
benji = Character.create!(species: human, first_name: 'Benji', last_name: 'Hatrock', gender: 'male')

## others
# Friend to Barney & Fred (fire chief)
joe = Character.create!(species: human, first_name: 'Joseph', nick_name: 'Joe', last_name: 'Rockhead', gender: 'male')

# paperboy (town: bedrock)
arnold = Character.create!(species: human, first_name: 'Arnold', last_name: 'Granite', gender: 'male')

stoney = Character.create!(species: human, first_name: 'Stoney', last_name: 'Curtis', gender: 'male')
perry = Character.create!(species: human, first_name: 'Perry', last_name: 'Masonry', gender: 'male')

# Sam Slagheap – The Grand Poobah of the Water Buffalo Lodge.
sam = Character.create!(species: human, first_name: 'Samuel', nick_name: 'Sam', last_name: 'Slagheap', gender: 'male')

## Alien
########
# planet = Zetox
gazoo = Character.create!(species: alien, first_name: 'The Great', last_name: 'Gazoo', gender: 'male')
# gazoo = Character.create!(species: alien, first_name: 'The Great', last_name: 'Gazoo', gender: 'male', planet: zetox)

## PETS
#######
dino = Character.create!(species: dino, first_name: 'Dino', last_name: 'Flintstone', gender: 'male')
pussy = Character.create!(species: tiger, first_name: 'Baby Pussy', last_name: 'Flintstone', gender: 'female')
hoppy = Character.create!(species: kanga, first_name: 'Hoppy', last_name: 'Rubble', gender: 'male')


## Companies
############
san_cemente = Company.create!(company_name: 'San Cemente')
bedrock_news = Company.create!(company_name: 'Bedrock Daily News')
bedrock_police = Company.create!(company_name: 'Bedrock Police Department')
bedrock_fire = Company.create!(company_name: 'Bedrock Fire Department')
bedrock_quarry = Company.create!(company_name: 'Bedrock & Gravel Quarry Company')
betty_wilma_catering = Company.create!(company_name: 'Betty & Wilma Catering')
texrock_ranch = Company.create!(company_name: 'Texrock Ranch')
teradactyl = Company.create!(company_name: 'Teradactyl Flights')
auto_repair = Company.create!(company_name: 'Bedrock Auto Repair')
used_cars = Company.create!(company_name: 'Bedrock Used Cars')
bedrock_entetainment = Company.create!(company_name: 'Bedrock Entertainment')
bedrock_army = Company.create!(company_name: 'Bedrock Army')
independent = Company.create!(company_name: 'Independent')
advertising = Company.create!(company_name: 'Bedrock Advertising')
buffalo_lodge = Company.create!(company_name: 'Water Buffalo Lodge')

## Jobs
#######
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

## PersonJobs
##############
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
