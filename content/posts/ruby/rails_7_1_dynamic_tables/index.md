---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x Dynamic Tables"
subtitle: "Simple Dynamic Tables without Javascript"
summary: ""
authors: ['btihen']
tags: ['Rails', 'Tables']
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

I recently learned that Rails 7.1+ has some delightful features that make it easy to render dynamic tables without JavaScript.  This is an exploration of using these new features.

This code can be found at: https://github.com/btihen-dev/rails_dynamic_tables

## Getting Started

I will create a basic application - using the starting point in the [Basic App](https://btihen.dev/posts/ruby/rails_7_1_base_app/) found at the [repo](https://github.com/btihen-dev/rails_base_app).

In the case I will use the following options:

```bash
rails new dynamic_tables -T --main --database=postgresql --javascript=esbuild --css=bootstrap
cd dynamic_tables
bin/rails db:create

git add .
git commit -m "initial commit"
```

Follow the instructions in: https://btihen.dev/posts/ruby/rails_7_1_base_app/

## Dynamic Tables




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
