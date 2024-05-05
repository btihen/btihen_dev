---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.0 - Citadel Architecture with Packwerk"
subtitle: "Creating a Rails 7.0 App"
summary: ""
authors: ['btihen']
tags: ['Ruby', 'Citadel Design', 'Architecture', 'Packwerk', 'Shopify']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2022-03-12T01:20:00+02:00
lastmod: 2022-03-12T01:20:00+02:00
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

**Packwerk** is a gem that helps define and enforce boundaries - thus preventing tight coupling.

You might already be using many or all of the following,

* Command Objects
* Input Objects
* View-Models (or View Components)
* Service Objects
* Engines
* ...

but still accidentally end up with coupling and all the fragility it brings. You could of course use multiple services (or even micro-services), but like the ease of deploy with a well organized monolith.

**Packwerk** - maybe the solution you are looking for and it allows incremental adoption and enforcement!

## Starting point

We will build a mini-slack clone using the standard rails approach and the reorganize using shopify's packwerk into a Citadel Architecture.  Citadels are isolated towers with only one (or two) access points (defined boundaries), but still connect to the rest of the fort (monolith).  This can alow people to focus on the the logic associated with the citadel and are protected from tight coupling.

## Getting Started

We need Rails 6.0 or newer to use packwerk - we will use rails 7.0

```bash
gem install rails --version 7.0.0
# rails new slacker_pack -j esbuild -T --css bulma
rails new slacker_pack -j esbuild -T --css tailwind
cd slacker
```
The options:
* `-T` - does not install mini-tests (I like rspec)
* `-j esbuild` - is the middle ground between using webpacker and import-maps
* `--css bulma` - install a css framwork (bulma, tailwind, bootstrap, sass, postcss)

**NOTE:** if you use `esbuild` be sure you have `yarn`, `npm` and `npx` already installed

## install richtext editor

This will also install StimulusJS and ActiveStorage and ActionText altogether
```bash
./bin/rails action_text:install
```

## App Structure

```bash
bin/rails g controller landing index
bin/rails g scaffold Topic title:string
bin/rails g scaffold User name:string email:string
bin/rails g scaffold Note subject user:references # content is rich_text
# **polymorphic join table for messages**
rails g migration AddNotableToNotes notable:references{polymorphic}
```

## Routes
```ruby
# config/routes.rb
Rails.application.routes.draw do
  get 'landing/index'
  resources :notes
  resources :users
  resources :topics

  # Defines the root path route ("/")
  root "landing#index"
end
```

## update Notes with Notable
```ruby
class AddNotableToNotes < ActiveRecord::Migration[7.0]
  def change
    add_reference :notes, :notable, polymorphic: true, null: false, index: true
  end
end
```

## Note Model
```ruby
# app/models/note.rb
class Post < ActiveRecord::Base
  belongs_to :user
  belongs_to :notable, polymorphic: true

  validates :subject, presence: true
  validates :content, presence: true

  has_rich_text :content
end
```

## Topic Model
```ruby
# app/models/topic.rb
class Topic < ActiveRecord::Base
  validates :title, presence: true

  has_many :notes, as: :notable
end
```

## Users
```ruby
# app/models/user.rb
class User < ActiveRecord::Base
  has_many :notes
  has_many :received_messages, as: :notable
end
```

## seeds
```ruby
# db/seeds.rb
users = User.create([{ name: "Bill",
                       email: "bill@example.ch" },
                     { name: "Nyima",
                       email: "nyima.example.ch"}])

topics = Topic.create([{ title: "Ruby" },
                       { title: "Rails" },
                       { title: "JavaScript"}])

messages = Note.create([# Post to a discussion topic
                        { subject: "Hello #{topics.first.title} - from #{users.first.name}",
                          content: "Packwerk post 0", user: users.first,
                          notable: topics.first },
                        { subject: "Hello #{topics.first.title} - from #{users.last.name}",
                          content: "Packwerk post 1", user: users.last,
                          notable: topics.first },
                        { subject: "Hello #{topics.last.title} - from #{users.first.name}",
                          content: "Packwerk post 2", user: users.first,
                          notable: topics.first },
                        # messages to users
                        { subject: "Hello #{users.first.name} - from #{users.last.name}",
                          content: "Packwerk post 3", user: users.last,
                          notable: users.first },
                        { subject: "Hello #{users.last.name} - from #{users.first.name}",
                          content: "Packwerk post 4", user: users.first,
                          notable: users.last },
                        { subject: "Hello to self - from #{users.first.name}",
                          content: "Packwerk post 5", user: users.first,
                          notable: users.first }])
```

now lets seed our DB:
```bash
bin/rails db:drop
bin/rails db:create
bin/rails db:migrate
bin/rails db:seed
```

**start rails and test**

`./bin/dev`

## Usable App

since we don't dont have user management with the scope - we will just build the integration on the landing page

## Reorg with Packwerk

### packwerk
bundle add packwerk
bundle add pocky
bundle binstub packwerk
mkdir packages
bin/packwerk init
bin/rake pocky:generate

### view components
bundle add view-components

## Start up

**In watchmode - like we are accustom**

`./bin/dev`

other options are:
* `foreman start -f Procfile.dev`
* `bin/rails s` and `yarn build --watch` in separate windows
