---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Using Hotwire with Nested Models"
subtitle: "New SPA technology for Rails from DHH / Basecamp"
summary: "Rails 6.1 makes it easy to create a SPA (with HTML over sockets) with minimal JavaScript"
authors: ["btihen"]
tags: ['Ruby', "Rails 6", "SPA", "HTML", "WebSocket"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-02-25T18:57:00+02:00
lastmod: 2021-08-07T01:57:00+02:00
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

https://www.youtube.com/watch?v=77GvSdc2Pf8

https://www.driftingruby.com/episodes/hotwire

## Setup
```bash
rails new tickets -d postgresql -T
bundle add hotwire-rails

bin/rails hotwire:install
# installs redis if needed
# be sure it removes turbolinks!

bin/rails action_text:install

# we'll use action text for writing
bin/rails g scaffold tickets title

bin/rails g model comment ticket:references

bin/rails db:create
bin/rails db:migrate
```

###
# app/models/ticket.rb
```ruby
class Ticket < ApplicationRecord
  has_rich_text :content
  has_many :comments, dependent: :destroy
end
```

###
# app/models/comment.rb
```ruby
class Comment < ApplicationRecord
  belongs_to :ticket
  has_rich_text :content
end
```

###
# app/controllers/tickets_controller.rb
```ruby
    # Only allow a list of trusted parameters through.
    def ticket_params
      params.require(:ticket).permit(:title, :content)
    end
```

###
# app/views/tickets/_form.html.erb
```ruby
<div class="field">
  <%= form.label :content %>
  <%= form.rich_text_area :content %>
</div>
```

###
# app/views/tickets/show.html.erb
```ruby
<h1>
  <small>Title:</small>
  <strong><%= @ticket.title %></strong>
</h1>
<p><%= @ticket.content %></p>

<%= link_to 'Edit', edit_ticket_path(@ticket) %> |
<%= link_to 'Back', tickets_path %>

<div id='comments'>
  <h2>Comments</h2>
  <%= render @ticket.comments %>
</div>

<%= link_to 'New Comment', new_ticket_comment_path(@ticket) %>
```

###
# app/views/comments/_comment.html.erb
```ruby
<%= content_tag :div do %>
  <em>Written <%= time_ago_in_words(comment.created_at) %> ago</em>
  <%= comment.content %>
  <hr>
<% end %>
```

###
# app/views/comments/new.html.erb
```ruby
<h2>New Comment</h2>
<%= form_with model: [@ticket, @comment] do |form| %>
<div class='field'>
  <%= form.rich_text_area :content %>
  <%= form.submit 'Post' %>
  </div>
<% end %>
```

###
# app/controllers/comments_controller.rb
```ruby
class CommentsController < ApplicationController

  before_action :set_ticket

  def new
    @comment = @ticket.comments.new
  end

  def create
    @comment = @ticket.comments.create(comment_params)
    respond_to do |format|
      format.html { redirect_to @ticket }
    end
  end

  private

  def set_ticket
    @ticket = Ticket.find(params[:ticket_id])
  end

  def comment_params
    params.require(:comment).permit(:content)
  end

end
```

###
# config/routes.rb
```ruby
Rails.application.routes.draw do
  resources :tickets do
    resources :comments, only: [:new, :create]
  end
  root 'tickets#index'
end
```

####
# TEST NOW ADD TURBO


####
# app/views/tickets/show.html.erb
```ruby
<%= turbo_frame_tag 'ticket' do %>
  <h1>
    <small>Title:</small>
    <strong><%= @ticket.title %></strong>
  </h1>
  <p><%= @ticket.content %></p>

  <%= link_to 'Edit', edit_ticket_path(@ticket) %> |
  <%= link_to 'Back', tickets_path %>
<% end %>

<div id='comments'>
  <h2>Comments</h2>
  <%= render @ticket.comments %>
</div>

<%= link_to 'New Comment', new_ticket_comment_path(@ticket) %>
```

####
# app/views/tickets/edit.html.erb
```ruby
<!-- same tage as show - so will replace the block in show -->
<%= turbo_frame_tag 'ticket' do %>
  <h1>Editing Ticket</h1>

  <%= render 'form', ticket: @ticket %>

  <%= link_to 'Show', @ticket %> |
  <%= link_to 'Back', tickets_path %>
<% end %>
```

####
# test again!

####
# ADD BROADCAST

####
# app/views/tickets/show.html.erb
```ruby
<!-- set up broadcast channel -->
<%= turbo_stream_from @ticket %>

<%= turbo_frame_tag 'ticket' do %>
  <!-- need a partial with ID for pub/sub broadcast to work-->
  <%= render @ticket %>

  <%= link_to 'Edit', edit_ticket_path(@ticket) %> |
  <!-- make back path turbo aware -->
  <%= link_to 'Back', tickets_path, 'data-turbo-frame': :_top %>
<% end %>

<div id='comments'>
  <h2>Comments</h2>
  <%= render @ticket.comments %>
</div>

<%= link_to 'New Comment', new_ticket_comment_path(@ticket) %>
```

####
# app/views/tickets/_ticket.html.erb
```ruby
<!-- add div with ID so all members of the channel can update dom with id -->
<%= content_tag :div, id: dom_id(ticket) do %>
  <h1>
    <small>Title:</small>
    <strong><%= ticket.title %></strong>
  </h1>
  <p><%= ticket.content %></p>
<% end %>
```

####
TEST TITLE BEHAVIOR - NOW COMMENTS

###
# app/views/tickets/show.html.erb
```ruby
<!-- set up broadcast channel -->
<%= turbo_stream_from @ticket %>

<%= turbo_frame_tag 'ticket' do %>
<!-- need a partial with ID for pub/sub broadcast to work-->
<%= render @ticket %>

<%= link_to 'Edit', edit_ticket_path(@ticket) %> |
<!-- make back path turbo aware -->
<%= link_to 'Back', tickets_path, 'data-turbo-frame': :_top %>
<% end %>

<div id='comments'>
  <h2>Comments</h2>
  <%= render @ticket.comments %>
</div>

<%# link_to 'New Comment', new_ticket_comment_path(@ticket) %>
<!-- source and target -- 'new_comment' muss gefunden im `app/views/comments/new.html.erb` -->
<%= turbo_frame_tag 'new_comment', src: new_ticket_comment_path(@ticket), target: :_top %>
```


###
# app/views/comments/new.html.erb
```ruby
<%= turbo_frame_tag 'new_comment' do %>
  <h2>New Comment</h2>
  <%= form_with model: [@ticket, @comment] do |form| %>
    <div class='field'>
      <%= form.rich_text_area :content %>
      <%= form.submit 'Post' %>
    </div>
  <% end %>
<% end %>
```

###
# app/views/comments/_comment.html.erb
```ruby
<!-- need dom id to dynamicaly update -->
<%= content_tag :div, id: dom_id(comment) do %>
  <em>Written <%= time_ago_in_words(comment.created_at) %> ago</em>
  <%= comment.content %>
  <hr>
<% end %>
```

###
# app/models/comment.rb
```ruby
class Comment < ApplicationRecord
  belongs_to :ticket
  has_rich_text :content

  # broadcast changes to ticket since we are in a ticket context
  broadcasts_to :ticket
  # what above does is all 3:
  # after_create_commit -> { broadcast_append_to ticket }
  # after_update_commit -> { broadcast_replace_to ticket }
  # after_destroy_commit -> { broadcast_remove_to ticket }
end
```

###
# avoid full refresh - add turbo_stream behavior
# app/controllers/comments_controller.rb
```ruby
class CommentsController < ApplicationController

  before_action :set_ticket

  def new
    @comment = @ticket.comments.new
  end

  def create
    @comment = @ticket.comments.create(comment_params)
    respond_to do |format|
      # looks for a create.turbo_stream.erb
      format.turbo_stream
      format.html { redirect_to @ticket }
    end
  end

  private

  def set_ticket
    @ticket = Ticket.find(params[:ticket_id])
  end

  def comment_params
    params.require(:comment).permit(:content)
  end

end
```

# add view for turbostream
# app/views/comments/create.turbo_stream.erb
```ruby
<!-- nothing needed if using the default otherwise you would add something like -->
<%# turbo_stream.append 'comments', @comment %>
```

# FIX FORM RESET
###
# stimulus to reset form wo reload
# app/javascript/controllers/reset_form_controller.js
```ruby
import { Controller } from "stimulus"

export default class extends Controller {
  // re-activate the submit button wo reload
  static targets = ["button"]

  reset() {
    this.element.reset()
    this.buttonTarget.disabled = false
  }
}
```

####
# app/views/comments/new.html.erb
```ruby
<%= turbo_frame_tag 'new_comment' do %>
  <h2>New Comment</h2>
  <%= form_with model: [@ticket, @comment],
      data: { controller: 'reset_form',
              action: 'turbo:submit-end->reset_form#reset' } do |form| %>
    <div class='field'>
      <%= form.rich_text_area :content %>
      <%= form.submit 'Post', 'data-reset_form-target': 'button' %>
    </div>
  <% end %>
<% end %>
```
