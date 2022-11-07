---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Lucky Framework - Associations"
subtitle: "Polymorphic Relationships - Trials & Successes"
summary: "A simple project to test complex relationships in Lucky"
authors: ["btihen"]
tags: ["crystal", "Relationships", "Polymorphic", "Web Development"]
categories: ["Code", "Crystal Language", "Lucky Framework"]
date: 2021-05-02T01:01:53+02:00
lastmod: 2021-05-08T01:01:53+02:00
featured: false
draft: true

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

Frequently, I have found it is convenient to make lists of Items of related, but different & these differences often lead to differing attributes that we need to store.  To do this Polymorphic Relationships can be helpful.

## Overview

In this case, I want to model a contact list of businesses and people.  The people can be customers or business reps (so some people will have a relationship to a business too).  Businesses will have their the VAT and business registrations, etc.

The basic model will then look something like:
```
                 ┌──────────────┐
                 │              │
                 │    Contact   │
                 │              │
                 └──────────────┘
                         ┼ 1
             ┌───────────┴──────────┐
            ╱│╲ *                * ╱│╲
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
               Created with Monodraw
```
Each person can belong to zero or 1 business, but a business can have many people associated with it.  Similarly, each business or person must be  on the Contact List once, but of course the contact list has many people and businesses on it.

We will also add another Model - just to have and compare working with it.

## Start a new Lucky Project

We'll build a new lucky project. I'll sort of follow the new Lucky tutorial: https://luckyframework.org/guides/tutorial/overview

I'll call the project `lucky_poly` (without authorization) - we are just testing (I'll assume you are fully ready to go if nott see - ):

```bash
lucky init
cd lucky_poly
script/setup
```

lets be sure all is go - start up and be sure you see the homepage:
```bash
lucky dev
```

great - we'll snapshot:
```bash
git add .
git commit "initial commit on setup"
```

## Add a Non-Relational Model

Lets test our building a model and the Lucky mechanisms before we get fancy with relationships and in particular polymorphism.

https://luckyframework.org/guides/tutorial/new-resource

So we will generate an animal resource - using a full stack generator:
```
lucky gen.resource.browser Animal nick_name:String species:String
lucky db.migrate
```

Now let's create some sample data in `tasks/db/seed/sample_data.cr` - via the seed task - from these instructions: https://luckyframework.org/guides/database/database-setup#seeding-data as our base.

We will start by using what's used to save when we create new records with incomming data. `SaveAnimal.create!(nick_name: "racky coon")` so now our file will look like:
```ruby
# tasks/db/seed/sample_data.cr
require "../../../spec/support/factories/**"

class Db::Seed::SampleData < LuckyTask::Task
  summary "Add sample database records helpful for development"

  def call
    SaveAnimal.create!(nick_name: "racky coon")

    puts "Done adding sample data"
  end
end
```

We test this with:
```bash
lucky db.seed.sample_data
```

assuming this runs we should be able to view this data in our db (I often use the cli - but you might also want to use: `dbgate` https://dbgate.org/):
```
psql
\l
\c lucky_poly_development
select * from animals;
```

cool - lets try a factory too - these are especially help when complex and building relationships, etc:

```crystal
# spec/support/factories/animal_factory.cr
class AnimalFactory < Avram::Factory
  def initialize
    nick_name "Nick Name"
  end
end
```

now lets try using our factory in the seed file:
```ruby
# tasks/db/seed/sample_data.cr
require "../../../spec/support/factories/**"

class Db::Seed::SampleData < LuckyTask::Task
  summary "Add sample database records helpful for development"

  def call
    SaveAnimal.create!(nick_name: "racky coon", species: "racoon")

    # using a factory: https://luckyframework.org/guides/testing/creating-test-data#factory-create
    AnimalFactory.create do |factory|
      factory.nick_name("Dyno")
      factory.species("Dog")
    end

    # a shortcut way to write a block in crystal, see: https://crystal-lang.org/reference/syntax_and_semantics/blocks_and_procs.html#short-one-argument-syntax
    AnimalFactory.create &.nick_name("Shiné").species("cat")

    puts "Done adding sample data"
  end
end
```

test again with:
```bash
lucky db.seed.sample_data
```

Sweet, let's snapshot and try more complex stuff!
```bash
git add .
git commit -m "add a simple model and seed data in it"
```

## Lets build the Business / Person relationship

we will build this piece by piece - I found if you make a little error - it can hard to track - since the error message just tell you in some cases you have an error.


```bash
lucky gen.resource.browser Company legal_name:String
lucky db.migrate
```

Lets build our factory:
```ruby
# spec/support/factories/company_factory.cr
class CompanyFactory < Avram::Factory
  def initialize
    legal_name sequence("Company Name")
  end
end
```

now lets build the people:
```bash
lucky gen.resource.browser Person full_name:String
lucky db.migrate
```

Cool lets build the relationship between the people and company:
```
lucky gen.migration AddBelongsToCompanyForPerson
```

This is a good tutorial for the basics: https://luckyframework.org/guides/tutorial/assocations of building relationships.

First we will update the migration to look like:
```

```

Then fix ...

Then update: ...

Now make the factory:


Now we can migrate:
```
lucky db.migrate
```

Now build more seed data:


And we can test our seed data:
```
```


## Polymorphic Relationship


Now we need to update several files before we migrate!  I used this URL as a starting point: https://luckyframework.org/guides/database/models#polymorphic-associations


```bash
# lucky Avram supports Arrays, but not the migration generator
# lucky gen.resource.browser Contact roles:Array(String)
# So I did this and will fix the files!
lucky gen.resource.browser Contact roles:String
```

fixes for roles
```
lucky db.migrate
```
Let's update the seed file:
```ruby
# tasks/db/seed/sample_data.cr
require "../../../spec/support/factories/**"

class Db::Seed::SampleData < LuckyTask::Task
  summary "Add sample database records helpful for development"

  def call
    SaveAnimal.create!(nick_name: "racky coon", species: "racoon")

    # ...

    cust_contact = ContactFactory.create &.legal_name

    puts "Done adding sample data"
  end
end
```
test again with:
```bash
lucky db.seed.sample_data
```

  527  lucky gen.resource.browser Person full_name:String
  528  lucky db.migrate
  530  lucky gen.migration AddBelongsToCompanyForPerson
  531  vscode .
  532  lucky db.migrate
  533  lucky gen.resource.browser Contact roles:Array(String)
  536  lucky gen.resource.browser Contact roles:String
  537  lucky db.migrate
  538  lucky gen.migration AddPolymorphicBelongsToForContacts
  539  lucky db.migrate
  540  lucky db.seed.sample_data
  541  lucky db.seed.sample_data
  542  lucky db.seed.sample_data
  543  lucky db.seed.sample_data
  544  irc
  545  lucky irc
  546  lucky db.seed.sample_data
  547  lucky irc
  548  lucky irc
  549  lucky irc
  550  lucky irc
  552  lucky db.migrate

```
