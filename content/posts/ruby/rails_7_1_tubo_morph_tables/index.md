---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x - Dynamic Tables with Turbo Morph"
subtitle: "Very Simple and easy Dynamic Tables"
summary: ""
authors: ['btihen']
tags: ['Rails', 'Tables', 'Turbo', 'Turbo 8', 'Morph', 'Morph Dom']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2024-05-01T01:20:00+02:00
lastmod: 2024-05-03T01:20:00+02:00
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

I recently learned that Rails 7.1+ has some delightful features that make it easy to render dynamic tables without JavaScript.  This is an exploration of using these new features.

The cool thing is that morph updates without a full page reload - so its fast! and very easy to setup.

This code can be found at: https://github.com/btihen-dev/rails_morph_tables

## Quick Summary

By adding this config in our head we enable Morph:

```ruby
# app/views/layouts/application.html.erb
  <head>
    ...

    <meta name="turbo-refresh-method" content="morph">
    <meta name="turbo-refresh-scroll" content="preserve">
    <%= turbo_refreshes_with method: :morph, scroll: :preserve  %>
    <%= yield :head %>

    ...
  </head>
```

Then we can add:
`, data: { turbo_action: 'replace' }`
to links (like sorting links or form submit urls). Then morph will automatically update only changes without a full page reload and it doesn't reset our page location.

The following article shows how to do this with sorting links and a filter/search form - this article is a summary of [Dynamic Table Sorting with Morph](https://btihen.dev/posts/ruby/rails_7_1_dynamic_tables/) and expands on Dynamic Filtering (with a form input) using Turbo morph.

## Getting Started

Initially, I had a little problem with esbuild - partly because I didn't start rails with `bin/dev` procfile.  See the **Appendix**  to be sure `esbuild` works and builds the proper files.

With import-maps you can start rails with just: `bin/rails`, but with esbuild be sure to use `bin/dev`!

```bash
# install rails 7.1.x
rails _7.1.3.2_ new morph_tables -T -j esbuild -d postgresql --css=bootstrap
# rails _7.1.3.2_ new morph_tables -T -d postgresql --css=bootstrap
cd morph_tables

# build the models
bin/rails g scaffold Species species_name
bin/rails g scaffold Character nick_name first_name \
            last_name given_name gender species:references
bin/rails g scaffold Company company_name
bin/rails g scaffold Job role company:references
bin/rails g model PersonJob start_date:date end_date:date \
            character:references job:references

# Add data to the seeds file see:
# https://github.com/btihen-dev/rails_morph_tables/blob/main/db/seeds.rb

bin/rails db:create
bin/rails db:migrate
bin/rails db:seed

git add .
git commit -m "add generated code"

# start rails and test
bin/dev
```

NOTE: So far this code only works when using `import maps` and not when using `esbuild`.  When I find a solution I will update this article or write a new one about these features with esbuild.

## Basic Setup Table Sort

This section is a summary of the article [Dynamic Table Sorting with Morph](https://btihen.dev/posts/ruby/rails_7_1_dynamic_tables/), but repeated hear to build the base app and is an adaptation of the article: https://www.colby.so/posts/turbo-8-refresh-sorting

Lets make the character index page our home (root) page by adding `root "characters#index"` to the end of our routes:
```ruby
#
Rails.application.routes.draw do
  resources :jobs
  resources :companies
  resources :characters
  resources :species
  # Define your application routes per the DSL in https://guides.rubyonrails.org/routing.html

  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  # Defines the root path route ("/")
  root "characters#index"
end
```

tidy models:

**Character**
```ruby
# app/models/character.rb
class Character < ApplicationRecord
  belongs_to :species

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
  validates :species, presence: true
  validates :gender, presence: true
  validates :gender, inclusion: { in: %w[male female] }
end
```

**Company**
```ruby
# app/models/company.rb
class Company < ApplicationRecord
  has_many :jobs, dependent: :destroy
  has_many :person_jobs, through: :jobs
  has_many :characters, through: :person_jobs

  normalizes :company_name,  with: ->(value) { value.strip }

  validates :company_name, presence: true
  validates :company_name, uniqueness: true
end
```

**Job**
```ruby
# app/models/job.rb
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

**PersonJob**
```ruby
# app/models/person_job.rb
class PersonJob < ApplicationRecord
  belongs_to :character
  belongs_to :job

  has_one :company, through: :job

  validates :job, presence: true
  validates :person, presence: true
  validates :start_date, presence: true
  validates :person,
            uniqueness: { scope: [ :job, :start_date ],
                          message: "person and job with start_date already exists" }
end
```

**Species**
```ruby
# app/models/species.rb
class Species < ApplicationRecord
  has_many :characters, dependent: :destroy

  normalizes :species_name, with: ->(value) { value.strip }

  validates :species_name, presence: true
  validates :species_name, uniqueness: true
end
```

update the index of the `Character` controller to avoid an N+1 query for our table:
```ruby
# app/controllers/characters_controller.rb
  def index
    query = Character
            .includes(:species)
            .includes(person_jobs: { job: :company })
    if params[:column].present?
      # @characters = query.order("#{params[:column]}").all
      @characters = query.order("#{params[:column]} #{params[:direction]}").all
    else
      @characters = query.all
    end
  end
```

now let's make some helpers for our dynamic table:
```ruby
# app/helpers/characters_helper.rb
module CharactersHelper
  def sort_link(column:, label:)
    direction = column == params[:column] ? future_direction : 'asc'
    link_to(
      "#{label} #{sort_arrow_for(column)}".html_safe,
      characters_path(column: column, direction: direction)
    )
  end

  def future_direction = params[:direction] == 'asc' ? 'desc' : 'asc'

  def sort_arrow
    case params[:direction]
    when 'asc' then tag.i(class: "bi bi-arrow-up")
    when 'desc' then tag.i(class: "bi bi-arrow-down")
    else tag.i(class: "bi bi-arrow-down-up")
    end
  end

  def sort_arrow_for(column)
    params[:column] == column ? sort_arrow : tag.i(class: "bi bi-arrow-down-up")
  end
end
```

Let's convert the people `index` view into a table view `app/views/people/index.html.erb`

```ruby
# app/views/characters/index.html.erb
<p style="color: green"><%= notice %></p>

<% content_for :title, "Characters" %>
<div class="container text-center">

  <div class="row justify-content-start">
    <div class="col-9">
      <h1>Characters</h1>
      <%= render "characters", characters: @characters %>
    </div>

    <div class="col-3">
      <%= link_to "New", new_character_path, class: "mt-5 sticky-top btn btn-primary" %>
    </div>
  </div>
</div>
```

```ruby
# app/views/characters/_characters.html.erb
<table class="table table-striped table-hover">
  <thead class="sticky-top">
    <tr class="table-primary">
      <th scope="col">
        <%= sort_link(column: "characters.id", label: "Id", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "first_name", label: "First Name", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "last_name", label: "Last Name", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "gender", label: "Gender", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "species.species_name", label: "Species", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        Company
      </th>
    </tr>
  </thead>

  <tbody class="scrollable-table">
    <div id="characters">
    <% characters.each do |character| %>
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
```

start rails:
```bash
bin/dev
```

go to: `http://localhost:3030/people` and be sure this table looks reasonable.

now let's commit this.
```bash
git add .
git commit -m "basic people table added"
```

## Fix Scroll Reset

To fix scroll reset we need to enable new Turbo 8 features - morph dom and keeping scroll location:

```ruby
# app/views/layouts/application.html.erb
  <head>
    <title>EsbuildTables</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <meta name="turbo-refresh-method" content="morph">
    <meta name="turbo-refresh-scroll" content="preserve">
    <%= turbo_refreshes_with method: :morph, scroll: :preserve  %>
    <%= yield :head %>

    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_include_tag "application", "data-turbo-track": "reload", type: "module" %>
  </head>
```

you can add this feature more surgically, this enables these features everywhere.

Now we need to inform our helper about the morph-dom by adding `data: { turbo_action: 'replace' }` to our path so the helper now looks
```ruby
# app/helpers/characters_helper.rb
module CharactersHelper
  def sort_link(column:, label:)
    direction = column == params[:column] ? future_direction : 'asc'
    link_to(
      "#{label} #{sort_arrow_for(column)}".html_safe,
      characters_path(column: column, direction: direction),
      data: { turbo_action: 'replace' }
    )
  end

  def future_direction = params[:direction] == 'asc' ? 'desc' : 'asc'

  def sort_arrow
    case params[:direction]
    when 'asc' then tag.i(class: "bi bi-arrow-up")
    when 'desc' then tag.i(class: "bi bi-arrow-down")
    else tag.i(class: "bi bi-arrow-down-up")
    end
  end

  def sort_arrow_for(column)
    return sort_arrow if params[:column] == column

    tag.i(class: "bi bi-arrow-down-up")
  end
end
```

now when you sort a column it doesn't reset the scroll location.

## Add Filter (submit with button)

This section is built up the article: https://www.colby.so/posts/filtering-tables-with-rails-and-hotwire, but adapted to use the simplicity of Turbo 8 Morph instead of the full hotwire features.

Add a form to submit our filter text in the `company` header with:
```ruby
      <th scope="col">
        Company
        <%= form_with url: characters_path, method: :get do |form| %>
          <%= form.button "Filter", class: "btn btn-light btn-sm" %>
          <%= form.text_field :company_name,
                placeholder: "company name",
                value: params[:company_name],
                class: "form-control form-control-sm" %>
        <% end %>
      </th>
```

now the table can be updated to look like:
```ruby
# app/views/characters/_characters.html.erb
<table class="table table-striped table-hover">
  <thead class="sticky-top">
    <tr class="table-primary">
      <th scope="col">
        <%= sort_link(column: "characters.id", label: "Id", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "first_name", label: "First Name", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "last_name", label: "Last Name", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "gender", label: "Gender", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "species.species_name", label: "Species", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        Company
        <%= form_with url: characters_path, method: :get do |form| %>
          <%= form.button "Filter", class: "btn btn-light btn-sm" %>
          <%= form.text_field :company_filter, # field sent back to controller
                placeholder: "company name",
                value: params[:company_filter], # holds our value for user to view
                class: "form-control form-control-sm" %>
        <% end %>
      </th>
    </tr>
  </thead>

  <tbody class="scrollable-table">
    <div id="characters">
    <% characters.each do |character| %>
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
```

now we need to receive it in our controller - so we update index with an filter:
```ruby
    # filter by company name if present
    @company_filter = params[:company_filter]

    if @company_filter.present?
      query = query
              .joins(person_jobs: { job: :company })
              .where('companies.company_name ilike ?', "%#{@company_filter}%")
    end
```

Now index will look like
```ruby
# app/controllers/characters_controller.rb
  def index
    @company_filter = params[:company_filter]
    query = Character
            .includes(:species)
            .includes(person_jobs: { job: :company })

    # sort column and direction if present
    query = query.order("#{params[:column]} #{params[:direction]}") if params[:column].present?

    # filter by company name if present
    if @company_filter.present?
      query = query
              .joins(person_jobs: { job: :company })
              .where('companies.company_name ilike ?', "%#{@company_filter}%")
    end
    @characters = query.all
  end
```


This works well, but we loose our filter when we sort - so let's update by adding our `company_filter` into our helper that builds the url:
```ruby
# app/helpers/characters_helper.rb
module CharactersHelper
  def sort_link(column:, label:, company_filter:)
    direction = column == params[:column] ? future_direction : 'asc'
    link_to(
      "#{label} #{sort_arrow_for(column)}".html_safe,
      characters_path(column:, direction:, company_name: company_filter),
      data: { turbo_action: 'replace' }
    )
  end

  def future_direction = params[:direction] == 'asc' ? 'desc' : 'asc'

  def sort_arrow
    case params[:direction]
    when 'asc' then tag.i(class: "bi bi-arrow-up")
    when 'desc' then tag.i(class: "bi bi-arrow-down")
    else tag.i(class: "bi bi-arrow-down-up")
    end
  end

  def sort_arrow_for(column)
    params[:column] == column ? sort_arrow : tag.i(class: "bi bi-arrow-down-up")
  end
end

```


## Filter without Button

create a new stimulus controller that will feed our data (after a short delay) automatically

```bash
rails g stimulus filter_form
```
now make it look like (400ms) is teh delay to allow the user to type before it sends a request (adjust it as you wish):

```js
// app/javascript/controllers/filter_form_controller.js
import { Controller } from "@hotwired/stimulus";

export default class extends Controller {
  static targets = ["form"];

  filter() {
    clearTimeout(this.timeout);
    this.timeout = setTimeout(() => {
      this.formTarget.requestSubmit();
    }, 200);
  }
}
```

We can now adjust our form and wire it to stimulus with
`data: { action: "input->filter-form#filter" }`
and remove the button now the form looks like:
```ruby
      <th scope="col">
        Company
        <%= form_with url: characters_path, method: :get,
            data: {
              controller: "filter-form", filter_form_target: "form", turbo_action: 'replace' # adding replace here keeps the scroll location correct
            } do |form| %>
          <%= form.text_field :company_filter,
              placeholder: "partial name",
              value: params[:company_filter],
              class: "form-control form-control-sm",
              autocomplete: "off",
              data: { action: "input->filter-form#filter" } %>
        <% end %>
      </th>
```

now the template looks like:

```ruby
# app/views/characters/_characters.html.erb
<table class="table table-striped table-hover">
  <thead class="sticky-top">
    <tr class="table-primary">
      <th scope="col">
        <%= sort_link(column: "characters.id", label: "Id", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "first_name", label: "First Name", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "last_name", label: "Last Name", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "gender", label: "Gender", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        <%= sort_link(column: "species.species_name", label: "Species", company_filter: @company_filter) %>
      </th>
      <th scope="col">
        Company
        <%= form_with url: characters_path, method: :get,
            data: {controller: "filter-form", filter_form_target: "form", turbo_action: 'replace' # adding replace here keeps the scroll location correct
            } do |form| %>
          <%= form.text_field :company_filter,
              placeholder: "partial name",
              value: params[:company_filter],
              class: "form-control form-control-sm",
              autocomplete: "off",
              data: { action: "input->filter-form#filter" } %> ## connects with the js
        <% end %>
      </th>
    </tr>
  </thead>

  <tbody class="scrollable-table">
    <div id="characters">
    <% characters.each do |character| %>
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
```

## Resources

### Adding Bootstrap to an existing project

* https://www.youtube.com/watch?v=phOUsR0dm5s
* https://medium.com/@gjuliao32/installing-bootstrap-rails-7-a-step-by-step-guide-0fc4a843d94f

### Bootstrap Icond

* https://icons.getbootstrap.com/

### Rails Table Articles

* [Table Sorting Rails 7.1 - 21 Mar 2024](https://www.colby.so/posts/turbo-8-refresh-sorting)
* [Table Filtering Rails 7.0 - 15 Oct 2021](https://www.colby.so/posts/filtering-tables-with-rails-and-hotwire)
* [Table Sorting Rails 7.0 - 19 Sep 2021](https://www.colby.so/posts/sortable-table-with-rails-and-turbo-frames)
* [Table Sorting with Stimulus](https://www.colby.so/posts/a-sortable-table-with-rails-and-stimulusreflex)

### Rails Hotwire

* [Turbo Rails Intro](https://www.colby.so/posts/turbo-rails-101-todo-list)
* [Hotwiring Rails Book](https://book.hotwiringrails.com/)
* [Hot Rails Tutorial - building Turbo Rails](https://www.hotrails.dev/turbo-rails)
* [Rebuilding Turbo Rails - new version](https://www.hotrails.dev/rebuilding-turbo-rails)
* [Rails Hotwire Modals](https://webcrunch.com/posts/hotwire-rails-turbo-modals)
* [Turbo Frame Pages in Ruby on Rails 7](https://www.youtube.com/watch?v=iwZDoz_Ya2k)
* [Digging into Turbo with Ruby on Rails 7](https://www.youtube.com/watch?v=0CSGsHnci2I)
* [Mastering Turbo Frames and Turbo Streams in Rails 7: Build a Journal Entry Tagging Feature](https://www.youtube.com/watch?v=lG5aRBJHDBQ)
* [Odin Probject - Turbo Tutorial](https://www.theodinproject.com/lessons/ruby-on-rails-turbo)
* [Hotwire Turbo Transitions](https://dev.to/nejremeslnici/how-to-use-view-transitions-in-hotwire-turbo-1kdi)

## APPENDIX

when using esbuild you should see the following two files and you should NOT see `import-map` files or config!

David Colby who wrote: https://www.colby.so/posts/turbo-8-refresh-sorting and https://www.colby.so/posts/filtering-tables-with-rails-and-hotwire

writes:

For reference, `Procfile.dev` looks like this for me on a fresh esbuild + bootstrap install:
```bash
web: env RUBY_DEBUG_OPEN=true bin/rails server
js: yarn build --watch
css: yarn watch:css
```

And `package.json` looks like this:
```js
{
  "name": "app",
  "private": true,
  "dependencies": {
    "@hotwired/stimulus": "^3.2.2",
    "@hotwired/turbo-rails": "^8.0.4",
    "@popperjs/core": "^2.11.8",
    "autoprefixer": "^10.4.19",
    "bootstrap": "^5.3.3",
    "bootstrap-icons": "^1.11.3",
    "esbuild": "^0.20.2",
    "nodemon": "^3.1.0",
    "postcss": "^8.4.38",
    "postcss-cli": "^11.0.0",
    "sass": "^1.76.0"
  },
  "scripts": {
    "build": "esbuild app/javascript/*.* --bundle --sourcemap --format=esm --outdir=app/assets/builds --public-path=/assets",
    "build:css:compile": "sass ./app/assets/stylesheets/application.bootstrap.scss:./app/assets/builds/application.css --no-source-map --load-path=node_modules",
    "build:css:prefix": "postcss ./app/assets/builds/application.css --use=autoprefixer --output=./app/assets/builds/application.css",
    "build:css": "yarn build:css:compile && yarn build:css:prefix",
    "watch:css": "nodemon --watch ./app/assets/stylesheets/ --ext scss --exec \"yarn build:css\""
  },
  "browserslist": [
    "defaults"
  ]
}
```
