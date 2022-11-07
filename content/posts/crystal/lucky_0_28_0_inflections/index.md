---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Lucky Language Inflections - 0.28.0"
subtitle: "Customizing how Lucky works with languages"
summary: ""
authors: ["btihen"]
tags: ["crystal", "language", "inflections", "configuration", "Lucky"]
categories: ["Code", "Crystal Language", "Lucky Framework"]
date: 2021-05-12T01:01:53+02:00
lastmod: 2021-08-12T01:01:53+02:00
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

## Motivation

It is helpful to be able to configure your cli-tasks to work the same as lucky.
In lucky you can configure the application's behavior / settings in the `config` folder.

For example, Lucky by default will pluralize `staff` as `staffs` - which is ok if you are talking about shepherd staffs - but not employees.

## Configuration

As of Lucky 0.28.0 `wordsmith` works well with configuration - adding your own settings.  You simple create an `inflector.cr` file in the `config` folder.  Here is an example config:
```ruby
# config/inflector.cr
# `staff` as in employees - not walking sticks:
Wordsmith::Inflector.inflections.uncountable("staff")
Wordsmith::Inflector.inflections.irregular("person", "persons")
```

when we test our new lucky config with `lucky exec`:
```ruby
lucky exec
# then when vim or nano opens you can enter something like:
require "../../src/app.cr"
include Lucky::TextHelpers
pp pluralize(2, "staff")
pp pluralize(2, "person")
```

you will get the expected:
```
"2 staff"
"2 persons"
```

## Problem

However, currently, the settings is only used by the lucky application and not the lucky generators (gen.tasks) - which are pre-compiled.  They are pre-compiled on install - BEFORE you even create a config file. Thus the generators generate files that are incompatible with Lucky's configured behavior.

This is what happens - it generates `staffs` files and routes :(
```ruby
lucky gen.resource.browser Staff name:String

Created CreateStaffs::V20210811201213 in ./db/migrations/20210811201213_create_staffs.cr
Generated Staff in ./src/models/staff.cr
Generated SaveStaff in ./src/operations/save_staff.cr
Generated DeleteStaff in ./src/operations/delete_staff.cr
Generated StaffQuery in ./src/queries/staff_query.cr
Generated Staffs::Index in ./src/actions/staffs/index.cr
Generated Staffs::Show in ./src/actions/staffs/show.cr
Generated Staffs::New in ./src/actions/staffs/new.cr
Generated Staffs::Create in ./src/actions/staffs/create.cr
Generated Staffs::Edit in ./src/actions/staffs/edit.cr
Generated Staffs::Update in ./src/actions/staffs/update.cr
Generated Staffs::Delete in ./src/actions/staffs/delete.cr
Generated Staffs::IndexPage in ./src/pages/staffs/index_page.cr
Generated Staffs::ShowPage in ./src/pages/staffs/show_page.cr
Generated Staffs::NewPage in ./src/pages/staffs/new_page.cr
Generated Staffs::EditPage in ./src/pages/staffs/edit_page.cr
Generated Staffs::FormFields in ./src/components/staffs/form_fields.cr
```

## Solution

(Lucky contributors are considering more elegant solutions)

With guidance from the lucky team we found a clumsy solution.  Once we understood that the tasks were pre-compiled automatically.  I was able to read how the script worked and noticed it responds to a `skip pre-compile` env_var and so we were able to solve it with the following procedure:
```bash
# clean up repo of gen.tasks that were problematic
# git clean -fd

# remove previously compiled shards
rm -rf lib && rm -rf bin

# after trashing all the shard - safest to be sure they are intact (or even updated)
SKIP_LUCKY_TASK_PRECOMPILATION=true shards install # or shards update

# re-run the setup
SKIP_LUCKY_TASK_PRECOMPILATION=true ./script/setup
```

Now with the first task it will compile the task (a bit slow), but it uses your config file!
```
lucky gen.resource.browser Staff name:String

compiling ...
```

Now we finally get the expected results when we run the task!
```bash
Created CreateStaff::V20210812185142 in ./db/migrations/20210812185142_create_staff.cr
Generated Staff in ./src/models/staff.cr
Generated SaveStaff in ./src/operations/save_staff.cr
Generated DeleteStaff in ./src/operations/delete_staff.cr
Generated StaffQuery in ./src/queries/staff_query.cr
Generated Staff::Index in ./src/actions/staff/index.cr
Generated Staff::Show in ./src/actions/staff/show.cr
Generated Staff::New in ./src/actions/staff/new.cr
Generated Staff::Create in ./src/actions/staff/create.cr
Generated Staff::Edit in ./src/actions/staff/edit.cr
Generated Staff::Update in ./src/actions/staff/update.cr
Generated Staff::Delete in ./src/actions/staff/delete.cr
Generated Staff::IndexPage in ./src/pages/staff/index_page.cr
Generated Staff::ShowPage in ./src/pages/staff/show_page.cr
Generated Staff::NewPage in ./src/pages/staff/new_page.cr
Generated Staff::EditPage in ./src/pages/staff/edit_page.cr
Generated Staff::FormFields in ./src/components/staff/form_fields.cr
```
