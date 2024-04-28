---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x Dynamic Tables"
subtitle: "Simple Dynamic Tables without Javascript"
summary: ""
authors: ['btihen']
tags: ['Rails', 'Tables']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2024-04-01T01:20:00+02:00
lastmod: 2024-04-27T01:20:00+02:00
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

This code can be found at: https://github.com/btihen-dev/rails_dynamic_tables

## Getting Started

I will create a basic application - using the starting point in the [Basic App](https://btihen.dev/posts/ruby/rails_7_1_base_app/) found at the [repo](https://github.com/btihen-dev/rails_base_app).

In the case I will use the following options:

```bash
rails new dynamic_tables -T --main --database=postgresql --javascript=esbuild --css=bootstrap
cd dynamic_tables
```

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
              <% jobs = character.jobs %>
              <% companies = character.companies %>
              <% companies_jobs = companies.zip(jobs) %>
              <tr id="<%= dom_id character %>">
                <th scope="row"><%= link_to "#{character.id}", edit_character_path(character) %></th>
                <td><%= character.first_name %></td>
                <td><%= character.last_name %></td>
                <td><%= character.gender %></td>
                <td><%= character.species.kind %></td>
                <td class="text-start">
                  <ul class="list-unstyled">
                    <% character.person_jobs.each do |person_job| %>
                      <li>
                        <b><%= person_job.company.name %></b><br>
                        &nbsp; &nbsp;- <%= person_job.job.role %><br>
                        &nbsp; &nbsp;- <%= person_job.start_date.strftime("%e %b '%y") %> -
                                       <%= person_job.end_date&.strftime("%e %b '%y") %>
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
bin/rails s
```

go to: `http://localhost:3030/people` and be sure this table looks reasonable.

now let's commit this.
```bash
git add .
git commit -m "basic people table added"
```




## Resources

### Adding Bootstrap to an existing project

* https://www.youtube.com/watch?v=phOUsR0dm5s
* https://medium.com/@gjuliao32/installing-bootstrap-rails-7-a-step-by-step-guide-0fc4a843d94f


### Rails Table Articles

* [Table Filtering Rails 7.0](https://www.colby.so/posts/filtering-tables-with-rails-and-hotwire)
* [Table Sorting Rails 7.1](https://www.colby.so/posts/turbo-8-refresh-sorting)
* [Table Sorting Rails 7.0](https://www.colby.so/posts/sortable-table-with-rails-and-turbo-frames)
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
