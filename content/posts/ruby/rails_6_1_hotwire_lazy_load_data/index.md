---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 6.1 - Hotwire with Lazy Loading"
subtitle: "A simple Rails App that works off one page using flash messages"
summary: "A simple Single Page App using Rails and flash messages with Hotwire"
authors: ["btihen"]
tags: ['Ruby', 'Hotwire', 'SPA', 'WebSocket', 'Realtime', 'Lazy-Loading']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-03-28T01:57:00+02:00
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

As was seen in [Using Hotwire with Flash Messages](/post_ruby_rails/rails_6_1_hotwire_flash_messages/) Hotwire can easily load data - let's do this in a lazy loaded way (after the html is loaded we add data).

## Basic Setup

Start with the code at the end of: [Using Hotwire in Rails](/post_ruby_rails/rails_6_1_hotwire_simple_realtime/)


## Prepare our code

Let's remove the extra Tweet.new load in the controller's index method:
```ruby
# app/controllers/tweets_controller.rb
  def index
    @tweets = Tweet.all.order(created_at: :desc)
    # @tweet = Tweet.new # no longer needed
  end
```

now if we try our code we get a null value error (for course).

So to fix this we need to load the data back in (and restructure our index page a bit).

Turbo works well if you use the normal templates - so in this case we will use the `new` template on the home page to call the new form and get its own data:

```ruby
# app/views/tweets/index.html.erb
<h1>Tweets</h1>
<%= turbo_stream_from "tweets" %>

<h2 class="mt-3 h4 text-muted">New Tweet</h2>
<div class="card card-body">
  <%= turbo_frame_tag "new-tweet", src: new_tweet_path, target: "_top" %>
</div>

<h2 class="mt-3 h4 text-muted">Tweet Feed</h2>
<%= turbo_frame_tag "tweets" do %>
  <%= render @tweets %>
<% end %>
```
Notice the new template is using the dom_id "new-tweet" and not "new_tweet". Also note that this tag has a `src:` - that is where it is getting its data source (& view to use) - in this case the `new_tweet_path` routes to `tweets_controller#new` and that calls the `veiw` template.  The final thing to note is the `target` - this tells the turbo_tag to look / act outside the contraints of its frame (otherwise we couldn't reach the controller).

Currently this won't work yet - we need to create a **matching tag** -- including the `target` in the `new` template.  So our updated `new` template now looks like:
```ruby
 app/views/tweets/new.html.erb
<h1>New Tweet</h1>

<%= turbo_frame_tag "new-tweet", target: "_top" do %>
  <%= render 'form', tweet: @tweet %>
<% end %>
```

Now we should have a new form that uses the standard rails data flow within the index - just like the display and edit of individual tweets also uses show and edit templates too.

## Resources

The repo where you can find this code in the branch:
https://github.com/btihen/ruby_kafi_hotwire_tweets/commits/hotwire_lazy_load_data
