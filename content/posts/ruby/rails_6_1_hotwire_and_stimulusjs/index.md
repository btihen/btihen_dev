---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 6.1 - Hotwire with StimulusJS"
subtitle: "A simple Rails App that works off one page using flash messages"
summary: "A simple Single Page App using Rails and flash messages with Hotwire"
authors: ["btihen"]
tags: ['Ruby', 'Hotwire', 'SPA', 'WebSocket', 'Realtime', 'Flash Message']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-03-14T18:57:00+02:00
lastmod: 2021-08-07T01:57:00+02:00
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
## Overview

Hotwire only updates dom_ids (usually only within a partial) - so other Frontend needs still need to be met with Javascript.  Rails uses StimulusJS to augment Hotwire.  I [Using Hotwire with Flash Messages](/post_ruby_rails/rails_6_1_hotwire_flash_messages/) we created a new instance of Tweet in the turbo_template and sent that to the form.  (Pretty non-standard) - we can do this even more simply by using JS to clear the form without instantiating a new object.

## Basic Setup

Start with the code at the end of: [Using Hotwire with Flash Messages](/post_ruby_rails/rails_6_1_hotwire_flash_messages/)

## StimulusJS to clear forms

To enable Flash Messages our create/controller looked like - which seems a little messy - in `create` (happy-path) we handle all the updates via the create.turbo_stream.erb template and with validation errors we explicity (in the controller - handle the validation errors)

So lets start by disabling the code we no longer need in the template:
```ruby
# app/views/tweets/create.turbo_stream.erb
<%# turbo_stream.replace "tweet-form", partial: "tweets/form", locals: { tweet: Tweet.new } %>
<%# to send a message to the notice partial %>
<%= turbo_stream.append "notice", partial: "shared/notice", locals: {notice: "Tweet was successfully created."} %>
```
So we are only leaving the turbo_stream.append active.

Let's test here and be sure the new form doesn't clear after making a new tweet.

## Add a StimulusJS controller

We don't need to add / install or configure StimulusJS since Hotwire already handles this.

So let's create the JS file to clear the form - its quite simple we will just use:
```javascript
// app/javascript/controllers/reset_form_controller.js
import { Controller } from "stimulus"

export default class extends Controller {
  reset() {
    this.element.reset()
  }
}
```
In order to tie this to the form we need to go into the form and add the `data:` info -- so now our form should start with:
```ruby
# app/views/tweets/_form.html.erb
<%= form_with(model: tweet, id: dom_id(tweet),
              data: {controller: "reset-form", action: "turbo:submit-end->reset-form#reset"}
            ) do |form| %>
```

This `data` tag ties the stimulus **controller** `reset_form_controller.js` with the `reset-form` setting -- notice the html uses a `-` when ruby uses `_`. On the form **action** `submit-end` then execute ('->') in the controller `reset-form` the function ('#') `reset`

Fairly straight-forward, but it helps to be aware of the syntax and the differences between Ruby and Javascript.

## Resources

The repo where you can find this code in the branch:
https://github.com/btihen/ruby_kafi_hotwire_tweets/commits/hotwire_with_stimulus
