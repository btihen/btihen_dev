---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.5 Event Bus"
subtitle: "Phoenix, Elixir, Event Bus (Decoupling)"
summary: "Create a Flexible Decoupled Systems"
authors: ["btihen"]
tags: ["Elixir", "Decoupling", "Event Bus"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2021-05-07T01:01:53+02:00
lastmod: 2021-05-07T01:01:53+02:00
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

## Event Bus (1.6.2)

use optimistic locking!

- [EventSourcing - 09/2018 - James Smith](https://www.youtube.com/watch?v=ESRjkG5f7wg)
- [EventStoreDesign - 03/2019 - Ben Smith, with code! - 40m](https://www.youtube.com/watch?v=3R3qeMr_ANE)
- [EventDesign - 08/2019 - Ben Smith - 20m](https://www.youtube.com/watch?v=TdGxvekg6xM)
- [EventArchitechture - 05/2018](https://www.youtube.com/watch?v=8qDXG7tnl9w)
- [EventBus - Repo](https://github.com/otobus/event_bus)
- [EventBus - Docs](https://hexdocs.pm/event_bus/EventBus.html)
- [MessageBus/EventBus-ThinkAddict - 12/2018](https://www.thinkaddict.com/articles/eventbus/)
- [Decoupled Modules with Elixir EventBus - 06/2018](https://medium.com/elixirlabs/decoupled-modules-with-elixir-eventbus-a709b1479411)
- [Event Bus Implementation - 07/2017](https://medium.com/elixirlabs/event-bus-implementation-s-d2854a9fafd5)


## Commanded Resources - very full featured (most common)

- [Commanded1.0 Install](https://10consulting.com/2019/11/25/commanded-and-eventstore-v1/)
- [CommandedRoadMap](https://10consulting.com/2019/01/27/commanded-latest-release/)
- [CommandedEventStore](https://10consulting.com/2019/11/25/commanded-and-eventstore-v1/s)
- [CommandedTelemetry](https://10consulting.com/2019/05/03/telemetry-for-elixir-applications/)
- [CommandedArchitechture](https://10consulting.com/2021/03/18/commanded-application-architecture/)
- [MicroServicesArchitechture](https://10consulting.com/2019/04/12/microservice-integration-patterns/)


## Maestro Resources - multi-node capable

- [Maestro / PG Store](https://hexdocs.pm/maestro/readme.html)


## Event Bus -- simplest (PG Storage is an addon)

- [EventBus](https://hexdocs.pm/event_bus/readme.html#addons)


## Resources

- [Event Storming](https://www.eventstorming.com/book/) - https://leanpub.com/introducing_eventstorming
- [Building Conduit](https://leanpub.com/buildingconduit/)
- [ElixirEventImplementations](https://github.com/slashdotdash/awesome-elixir-cqrs)
- [The Many Meanings of Event-Driven Architecture • Martin Fowler - 05/2017](https://www.youtube.com/watch?v=STKCRSUsyP0&t=1251s)
- [Core Decisions in Event-Driven Architecture - Duana Stanley - 10/2019](https://www.youtube.com/watch?v=SKXS2h3MdPM)
- [Event Driven Architecture - 04/2015](https://www.youtube.com/watch?v=XohG9yQe3Ps)
- [Opportunities and Pitfalls of Event-driven Utopia - 10/2019](https://www.youtube.com/watch?v=jjYAZ0DPLNM)
- [Designing Event-Driven Systems Links](https://gist.github.com/giampaolotrapasso/71221f378770e21e6270ffed76b181d7)


## Integrate with Phoenix App

## Integrate within Umbrella App
