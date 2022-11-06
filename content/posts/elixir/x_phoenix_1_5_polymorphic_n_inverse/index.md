---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.5 - Polymorphic and Inverse Polymorphic Relations"
subtitle: "Phoenix, Elixir, TailwindCSS, AlpineJS, LiveView - PETAL Stack"
summary: "Create a modern Phoenix SPA with tremendous flexibility"
authors: ["btihen"]
tags: ["Elixir", "Authentication", "POW", "Email", "HogMail"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2021-05-06T01:01:53+02:00
lastmod: 2021-05-06T01:01:53+02:00
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

## Multi-Tenancy

* https://hexdocs.pm/ecto/multi-tenancy-with-foreign-keys.html#content
* https://hexdocs.pm/ecto/multi-tenancy-with-query-prefixes.html#content


## Multi-Tenancy with POW (authentication)

* https://hexdocs.pm/pow/multitenancy.html
* https://fullstackphoenix.com/tutorials/multi-tenancy-and-authentication-with-pow

## Polymorphic

* https://hexdocs.pm/ecto/polymorphic-associations-with-many-to-many.html
* https://hexdocs.pm/ecto/Ecto.Schema.html#belongs_to/3-polymorphic-associations
* https://elixirforum.com/t/how-to-handle-schemas-polymorphism-in-phoenix/13269/24
* https://stackoverflow.com/questions/49328342/how-to-handle-schemas-polymorphism-in-phoenix
* https://medium.com/pragmatic-programmers/chapter-14-creating-polymorphic-associations-1345df527c24

## Inverse

* https://stackoverflow.com/questions/33184593/inverse-polymorphic-with-ecto

## STI / Inverse

* Thinking Ecto - https://www.youtube.com/watch?v=YQxopjai0CU
* http://codeloveandboards.com/blog/2017/10/03/migrating-activerecord-sti-to-ecto/
After doing some research and watching Darin Wilson's talk on ElixirConf 2017 called Thinking in Ecto, I have decided to use Ecto's embedded_schema. An embedded schema is essentially a schema which doesn't point to any particular data source. It can't be queried or persisted, but, on the other hand, it lets you define its structure, types and validation rules, which is very suitable for our needs. Let's create the embedded schemas for both the Book and Event modules:

## Embeds Many

(order with many items, Parent with many children)
It is recommended to declare your embeds_many/3 field with type :map in your migrations, instead of using {:array, :map}

* https://hexdocs.pm/ecto/Ecto.Schema.html#embeds_many/3


## Simple Inline Embeds many

* https://hexdocs.pm/ecto/Ecto.Schema.html#embeds_many/3-inline-embedded-schema
*

## OOP Inheritance

* https://programmersought.com/article/20071274201/


## ManyToMany

* https://hexdocs.pm/ecto/Ecto.Schema.html#many_to_many/3
* https://medium.com/coletiv-stories/ecto-elixir-many-to-many-relationships-66403933f8c1
