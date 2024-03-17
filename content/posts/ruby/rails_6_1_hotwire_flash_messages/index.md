---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 6.1 - Hotwire with Flash Messages"
subtitle: "A simple Rails App that works off one page using flash messages"
summary: "A simple Single Page App using Rails and flash messages with Hotwire"
authors: ["btihen"]
tags: ['Ruby', 'Hotwire', 'SPA', 'WebSocket', 'Realtime', 'Flash Message']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-03-06T18:57:00+02:00
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

Now that you have the basics of using Hotwire in Rails [Using Hotwire in Rails](/post_ruby_rails/rails_6_1_hotwire_simple_realtime/) - its interesting to try using it in other contexts, inparticular **modals** are very useful for inputs in Single Page Apps.  So in this Blog we will make the new input form a modal and leave the edit as an in-place form.

## Basic Setup

Start with the code at the end of: [Using Hotwire in Rails](/post_ruby_rails/rails_6_1_hotwire_simple_realtime/)


## Flash Messages in Partial

Remember, turbo_streams requires a dom_id and a partial in order to know where to send / update the HTML it generates -- so let's prepare `application.html.erb` so that flash messages use partials.

```ruby
# app/views/layouts/application.html.erb
<body>
  <%= render "shared/notice", notice: notice %>
  <%= yield %>
</body>
```

and of course we need a partials for notices now (we will keep it very simple):
```ruby
# app/views/shared/_notice.html.erb
<p id="notice"><%= notice %></p>
```

now we will create a turbo template to handle the flash on create:
```ruby
# app/views/tweets/create.turbo_stream.erb
<%# to send a message to the notice partial %>
<!--            action   dom_id         partial with dom_id   data to send in the notice -->
<%= turbo_stream.append "notice", partial: "shared/notice", locals: {notice: "Tweet created."} %>
```

In order for the controller and turbo_stream to handle this non-standard action we need to update the create method in the controller with the instructions `format.turbo_stream` on a successful create:
```ruby
# app/controllers/tweets_controller.rb
def create
    @tweet = Tweet.new(tweet_params)

    respond_to do |format|
      if @tweet.save
        format.turbo_stream  # enables flash message on create - via the create template
        format.html { redirect_to tweets_url, notice: "Tweet was successfully created." }
        format.json { render :show, status: :created, location: @tweet }
      else
        format.turbo_stream { # route turbo validation errors
                      render turbo_stream: turbo_stream.replace(
                              @tweet, partial: "tweets/form",
                              locals: { tweet: @tweet}) }
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @tweet.errors, status: :unprocessable_entity }
      end
    end
  end
```

Now when we test everything works great, except our form no longer clears. We can fix that by adding a second action to the create template (we will send a Tweet.new - there are other approaches too - covered in [Hotwire and StimulusJS](/post_ruby_rails/rails_6_1_hotwire_and_stimulusjs))

```ruby
# app/views/tweets/create.turbo_stream.erb
<%# clear form on create - without using JavaScript - by replacing the old Tweet info with Tweet.new %>
<%= turbo_stream.replace "tweet-form", partial: "tweets/form", locals: { tweet: Tweet.new } %>
<%# to send a message to the notice partial %>
<%= turbo_stream.append "notice", partial: "shared/notice", locals: {notice: "Tweet was successfully created."} %>
```

## Refactor

You might have noticed, that we have moved most of our turbo_steam template to the template file, but not the replace for validation errors -- since we already have a `replace` command in our template - we will need to leave our specific instructions in the errors as is -- until we clear the form with JS.

NOTE: now that we are consolidating our template info it might be tempting to add the following:
```ruby
<!-- to prepend on create - disabled to avoid double vision when broadcasting -->
<%#%   stream_action   dom_id_target, render_partial,       send_local_variables   %>
<%= turbo_stream.prepend "tweets", partial: "tweets/tweet", locals: { tweet: @tweet } %>
```
but don't add the default happy path instructions to the template when a model already has a broadcast after hook - if you add this instruction the person creating a new tweet will see two!


## Flash after we update

This is now very straight forward we simply add `format.turbo_stream` to our save and create an `update.turbo_stream.erb` template

```ruby
# app/views/tweets/update.turbo_stream.erb
<%# to send a message to the notice partial %>
<%= turbo_stream.append "notice", partial: "shared/notice", locals: {notice: "Tweet was successfully created."} %>
```

And now we can tell the controller to use that:
```ruby
#  app/controllers/tweets_controller.rb
  def update
    respond_to do |format|
      if @tweet.update(tweet_params)
        format.turbo_stream
        format.html { redirect_to @tweet, notice: "Tweet was successfully updated." }
        format.json { render :show, status: :ok, location: @tweet }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @tweet.errors, status: :unprocessable_entity }
      end
    end
  end
```

We don't have to clear the form on update since the `edit` template is replaced with the `show` template already.  So we are done.

## Resources

The repo where you can find this code in the branch:
https://github.com/btihen/ruby_kafi_hotwire_tweets/commits/hotwire_flash_messages
