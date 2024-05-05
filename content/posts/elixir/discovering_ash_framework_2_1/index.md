---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Discovering the Ash Framework"
subtitle: "The Ash Framework now it has a stable API!"
# Summary for listings and search engines
summary: "Discovering the new Ash Framework with its new stable API"
authors: ["btihen"]
tags: ["Elixir", "Ash", "Phoenix", "GraphQL", "JSON API"]
categories: ["Code", "Elixir Language", "Ash Framework"]
date: 2022-11-03T01:01:53+02:00
lastmod: 2022-11-03T01:01:53+02:00
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

I'm always on the lookout for good ways to organize large and complex codebases.

I've been curious about the Elixir Ash Framework and with the current 'stable' release, I decided to spend part of my vacation to explore and hopefully learn Ash.  To explore and remember what I learn I am writing myself an [Ash Tutorial](/tutorials/ash_2_1/) - example based more than full explanations.  Perhaps this will interest you too.

## Purpose

This tutorial tries to build on existing tutorials and documentation on Ash and present enough code examples and explanations, that a beginner (like me), can successfully build an application.

The idea is to build a relatively simple Ash app (a Support Ticket system), and integrate it with a Phoenix web application.  And then provide alternative API (GraphQL / Json) - to make the app available to say a mobile App.

## Overview

Having played with Ash a bit now, it nicely separates the Data Layer and Business Logic, the Access APIs AND facilitates common needs / patterns, ie:

* Validations & Constraints
* Queries (aggregates, calculations)
* Authentication (not yet authorization)
* ...
Without excluding standard elixir tooling.  I haven't tried this, but Ash claims to be easily extensible.

Given the flexibility of Ash's uses of resources, we will start with a very simple organization (similar to rails - resources will reflect database tables that are directly acted upon.  Once the App gets a bit more complicated (and our resources get a annoyingly large), we will restructure the app to reflect a more modular approach.

NOTE: A resource based app that separates the Data Layer, Business Logic and Access Layers is a new fresh approach, but takes a bit of rethinking.

In the [Thinking Elixir Podcast # 123](https://podcast.thinkingelixir.com/123) Zach describes the design of Ash to be **Declarative Design** designed to preserve functional mindset and splits applications into two aspects.

1. **Resources** - a description of attributes and what actions are allowed (what is required and what should happen)
2. **Engine** - follow the instructions of the resource

However, I like to think of Ash as having four Layers:

* **Engines** - The Ash engine handles the parallelization/running of requests to Ash.
* **Application APIs** - external access to data and actions (AshActions, AshJsonAPI, AshGraphQL, etc)
* **Resources** - a description of what should happen (actions allowed and the data required)
* **Data Layer** - data persistence (in memory, ETS, Mnesia, PostgreSQL, etc)

## Ash Structure

Long-term our system will need users, tickets and comments.

For now we need to create the minimal infrastructure needed by Ash, this includes:

* An Ash **Resource** - the core workable aspects of our data and its associated information.
* An Ash **Registry** - of the Resources and extension the API has access to (basically binds the application together).
* An Ash **Api** - ways to access and manipulate our Resources


# Resources

**Documentation**

* https://www.ash-hq.org/
* https://hexdocs.pm/ash/get-started.html
* https://www.ash-hq.org/docs/guides/ash/2.4.1/tutorials/get-started.md
* https://github.com/phoenixframework/phoenix/blob/master/installer/README.md

**Ash Framework 1.5** - Video and Slides

* https://www.youtube.com/watch?v=2U3vQHXCF0s
* https://speakerdeck.com/zachsdaniel1/introduction-to-the-ash-framework-elixir-conf-2020?slide=6
