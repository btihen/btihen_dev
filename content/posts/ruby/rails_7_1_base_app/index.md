---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x Base App"
subtitle: "Base App for Blogs"
summary: ""
authors: ['btihen']
tags: ['Rails', 'Base App']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2024-04-01T01:20:00+02:00
lastmod: 2024-04-01T01:20:00+02:00
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

A basic Rails app structure that offers a variety of opportunities for various Rails experiments.

This code can be found at: https://github.com/btihen-dev/rails_base_app

## Quick Summary

without explanations nor step:
```bash
# use other options as needed
rails new base_app -T --main --database=postgresql --javascript=esbuild

# setup and git commit (in case you are experimenting and want to rollback)
cd base_app
bin/rails db:create

# use generators to build basic code structure
bin/rails g scaffold Person nick_name first_name \
            last_name given_name gender
bin/rails g scaffold Company name
bin/rails g scaffold Job role company:references
bin/rails g model PersonJob start_date:date end_date:date \
            person:references job:references
```

update migrations:
```ruby
class CreatePeople < ActiveRecord::Migration[7.2]
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

class CreateCompanies < ActiveRecord::Migration[7.2]
  def change
    create_table :companies do |t|
      t.string :name, null: false

      t.timestamps
    end

    add_index :companies, :name, unique: true
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
      t.references :person, null: false, foreign_key: true, index: true
      t.references :job, null: false, foreign_key: true, index: true

      t.timestamps
    end

    add_index :person_jobs, %i[person_id job_id start_date], unique: true
  end
end
```

update models:
```ruby
cat << EOF > app/models/person.rb
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
EOF

cat << EOF > app/models/company.rb
class Company < ApplicationRecord
  has_many :jobs, dependent: :destroy
  has_many :person_jobs, through: :jobs
  has_many :people, through: :person_jobs

  normalizes :name,  with: ->(value) { value.strip }

  validates :name, presence: true
  validates :name, uniqueness: true
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
  belongs_to :person
  belongs_to :job

  validates :job, presence: true
  validates :person, presence: true
  validates :start_date, presence: true
  validates :person,
            uniqueness: { scope: [:job, :start_date],
                          message: "person and job with start_date already exists" }
end
EOF
```

run migrations
```bash
bin/rails db:migrate
```

populate seed file `db/seeds.rb` with the seed data (see Appendix):
```bash
bin/rails db:seed
```

you should now be good to go!


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

### Add People

Add a People model, controller and associated views via scaffold.

```bash
bin/rails g scaffold Person nick_name first_name \
                     last_name given_name gender
```

Let's update the migration to make the first & last name required as well as gender & lets ensure we cant add the same person twice with a unique index on `first_name last_name gender`

```ruby
class CreatePeople < ActiveRecord::Migration[7.2]
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

## Companies
############
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

## Person Jobs
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
