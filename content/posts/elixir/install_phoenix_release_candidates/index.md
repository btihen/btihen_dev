---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Install Phoenix Release Candidate"
subtitle: "Installing Phoenix Release Candidate"
# Summary for listings and search engines
summary: "Installing Phoenix Release Candidate"
authors: ["btihen"]
tags: ["Elixir", "Phoenix"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2022-11-03T01:01:53+02:00
lastmod: 2022-11-12T01:01:53+02:00
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

I had vacation and Phoenix 1.7 wasn't yet released and I wanted to try it out.  I discovered its simple to install without a hex package.

## Install Phoenix Release Candidate

First I let's install the unreleased version of [Phoenix 1.7 RC](
https://www.phoenixframework.org/blog/phoenix-1.7-released) - **Note** For the rc period, you’ll need to explicitly install the phx.new generator from hex to try out a fresh project:

```bash
mix archive.uninstall phx_new
mix archive.install hex phx_new helpdesk
cd helpdesk
mix ecto.create
git init
git add .
git commit -m "initial phoenix commit"
iex -S mix phx.server
```

Now we have a fully functional Phoenix site - with the new Phoenix Tailwind CSS design.

![Phoenix 1.7 Start Page](phoenix_1_7_default.png)
