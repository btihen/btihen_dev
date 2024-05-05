---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x Dynamic Tables using Turbo Morph"
subtitle: "Simple Dynamic Tables with Turbo 8 Morph"
summary: ""
authors: ['btihen']
tags: ['Rails', 'Tables', 'Turbo', 'Turbo 8', 'Morph', 'Morph Dom']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2024-04-01T01:20:00+02:00
lastmod: 2024-05-03T01:20:00+02:00
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

I recently learned that Rails 7.1+ has some delightful features that make it easy to render dynamic tables without JavaScript.  This is an exploration of using these new features.  This article is an adaptation of the article: https://www.colby.so/posts/turbo-8-refresh-sorting and https://www.colby.so/posts/filtering-tables-with-rails-and-hotwire

This code can be found at: https://github.com/btihen-dev/rails_dynamic_table

## Getting Started

I will create a basic application - using the starting point in the [Basic App](https://btihen.dev/posts/ruby/rails_7_1_base_app/) found at the [repo](https://github.com/btihen-dev/rails_base_app).

In the case I will use the following options:

```bash
rails _7.1.3.2_ new dynamic_tables -T --database=postgresql --css=bootstrap
cd dynamic_tables
```

NOTE: So far this code only works when using `import maps` and not when using `esbuild`.  When I find a solution I will update this article or write a new one about these features with esbuild.

## Dynamic Tables

Let's convert the people `index` view into a table view `app/views/people/index.html.erb`

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

start rails:
```bash
bin/rails s -p 3030
```

go to: `http://localhost:3030/people` and be sure this table looks reasonable.

now let's commit this.
```bash
git add .
git commit -m "basic people table added"
```

## Sortable Columns

We want sortable columns without a page reload (so open your developer window and open the network so you can see how fast and little data is transmitted)

Let's add a sort link helper method to our `app/helpers/people_helper.rb`

```ruby
# app/helpers/characters_helper.rb
module CharactersHelper
  def sort_link(column:, label:)
    link_to(label, characters_path(column: column))
  end
end
```

Let's update the people index controller `app/controllers/people_controller.rb` to use this new helper method.
```ruby
# app/controllers/people_controller.rb
  def index
    query =
      Character
      .includes(:species)
      .includes(person_jobs: [ job: :company ])

    if params[:column].present?
      @characters = query.order("#{params[:column]}").all
    else
      @characters = query.all
    end
end
```

Let's update the people `index` with our new sort methods.

```ruby
# app/views/people/index.html.erb
  <thead class="sticky-top">
    <tr class="table-primary">
      <th scope="col">
        <%= sort_link(column: "id", label: "Id") %>
      </th>
      <th scope="col">
        <%= sort_link(column: "first_name", label: "First Name") %>
      </th>
      <th scope="col">
        <%= sort_link(column: "last_name", label: "Last Name") %>
      </th>
      <th scope="col">
        <%= sort_link(column: "gender", label: "Gender") %>
      </th>
      <th scope="col">
        <%= sort_link(column: "species.species_name", label: "Species") %>
      </th>
      <th scope="col">
        Company-Job
      </th>
    </tr>
  </thead>
```

This is a cool proof of concept, but unfortunately only sorts into ascending and not yet descending.

## Bi-directional Sort

we update the helper with:
```ruby
# app/helpers/characters_helper.rb
module CharactersHelper
  def sort_link(column:, label:)
    direction = column == params[:column] ? next_direction : 'asc'
    link_to(label, characters_path(column: column, direction: direction))
  end

  def next_direction = params[:direction] == 'asc' ? 'desc' : 'asc'

  def sort_indicator = tag.span(class: "sort sort-#{params[:direction]}")

  def show_sort_indicator_for(column)
    sort_indicator if params[:column] == column
  end
end
```

and the controller with:
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

this is pretty cool, but with every refresh we reset our scroll location, which is annoying when navigating a long table.

## Adding Sort Arrows

```ruby
# app/helpers/characters_helper.rb
module CharactersHelper
  def sort_link(column:, label:)
    direction = column == params[:column] ? future_direction : 'asc'
    link_to(label, characters_path(column: column, direction: direction))
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

now let's update the view too:
```ruby
# app/views/people/index.html.erb
  <thead class="sticky-top">
    <tr class="table-primary">
      <th scope="col">
        <%= sort_link(column: "id", label: "Id") %>
        <%= sort_arrow_for("id") %>
      </th>
      <th scope="col">
        <%= sort_link(column: "first_name", label: "First Name") %>
        <%= sort_arrow_for("first_name") %>
      </th>
      <th scope="col">
        <%= sort_link(column: "last_name", label: "Last Name") %>
        <%= sort_arrow_for("last_name") %>
      </th>
      <th scope="col">
        <%= sort_link(column: "gender", label: "Gender") %>
        <%= sort_arrow_for("gender") %>
      </th>
      <th scope="col">
        <%= sort_link(column: "species.species_name", label: "Species") %>
        <%= sort_arrow_for("species.species_name") %>
      </th>
      <th scope="col">
        Company-Job
      </th>
    </tr>
  </thead>
```

## Fix Scroll Reset

To fix scroll reset we need to enable new Turbo 8 features - morph dom and keeping scroll location:

```ruby
# app/views/layouts/application.html.erb
  <head>
    ...
    <!-- adds morph-dom to rails and fixes scroll reset -->
    <meta name="turbo-refresh-method" content="morph">
    <meta name="turbo-refresh-scroll" content="preserve">
    <%= turbo_refreshes_with method: :morph, scroll: :preserve  %>
    <%= yield :head %>
    ...
  </head>
```

you can add this feature more surgically, this enables these features everywhere.

Now we need to inform our helper about the morph-dom by adding `data: { turbo_action: 'replace' }` to our path so the helper now looks
```ruby
# app/helpers/characters_helper.rb
module CharactersHelper
  def sort_link(column:, label:)
    direction = column == params[:column] ? future_direction : 'asc'
    link_to(label, characters_path(column: column, direction: direction), data: { turbo_action: 'replace' })
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

## Add Search Filter

https://www.colby.so/posts/filtering-tables-with-rails-and-hotwire


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
