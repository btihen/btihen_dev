---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Lucky Framework Upgrade"
subtitle: "Upgrading from Lucky 0.27.2 to 0.28.0"
summary: "Exploring how to upgrade crystal projects (Lucky)"
authors: ["btihen"]
tags: ["crystal", "upgrade", "shards", "Lucky", "Crystal"]
categories: ["Code", "Crystal Language", "Lucky Framework"]
date: 2021-05-10T01:01:53+02:00
lastmod: 2021-08-13T01:01:53+02:00
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


## Purpose

I learned about Lucky improvements (fixes from the minor bugs after my first article) and wanted to test them out.

The Lucky repo describes the changes to the shards file and the code base too.
https://github.com/luckyframework/lucky/blob/master/UPGRADE_NOTES.md

## Upgrading

First lets be sure we have a recent crystal version:
```bash
asdf install crystal 1.1.1
asdf global crystal 1.1.1
```

Second, Upgrade the lucky-cli:
```bash
# if you don't already have this
git clone https://github.com/luckyframework/lucky_cli
cd lucky_cli
git fetch
git checkout v0.28.0  # note this does not match the lucky-framework version (0.27.2)!
shards install
crystal build src/lucky.cr
cp lucky /usr/local/bin
cd ..
lucky -v  # hopefully responds with: 0.28.0
```

Now lets be sure we update `.tools-available` in the lucky project folder:

Then (in the project folder - type:
```bash
cd project_name  # my lucky-project
asdf local crystal 1.1.1
```

Lets lets update the `shards` file to -- according to the upgrade guide we should use the following settings:
```ruby
# shard.yml
name: animals
version: 0.1.0

targets:
  animals:
    main: src/animals.cr

crystal: >= 1.0.0

dependencies:
  lucky:
    github: luckyframework/lucky
    version: ~> 0.28.0
  authentic:
    github: luckyframework/authentic
    version: ~> 0.8.0
  carbon:
    github: luckyframework/carbon
    version: ~> 0.2.0
  # this should be removed
  # dotenv:
  #   github: gdotdesign/cr-dotenv
  #   version: ~> 0.8.0
  # use this instead - be shure to follow the instructions at (global search and replace is your friend):
  # https://github.com/luckyframework/lucky/blob/master/UPGRADE_NOTES.md
  lucky_env:
    github: luckyframework/lucky_env
    version: ~> 0.1.3
  lucky_task:
    github: luckyframework/lucky_task
    # version: ~> 0.8.0
  jwt:
    github: crystal-community/jwt
    # version: ~> 0.7.3
development_dependencies:
  lucky_flow:
    github: luckyframework/lucky_flow
    # version: ~> 0.7.3
```

Now type:
```bash
shards update
shards list
```

Cool - it worked!

Now lets see if your site still works
```bash
crystal spec
```

and start up the project with:
```bash
lucky dev
```

while we are at it we should update the yarn / node packages too.
```bash
yarn upgrade
```
