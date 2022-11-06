---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.0 - Decoupling with Events (or Commands / Service Objects)"
subtitle: "Decoupling for slim Models and slim Controllers"
summary: "It's possible to loosen the Coupling between Rails, Models and Controllers from your Business Logic (Commands/Service Objects) by enabling an Event architecture.  Ruby has several lightweight Event Buses (& full featured and Event Stores)"
authors: ['btihen']
tags: ['Ruby', 'event bus', 'publish/subscribe pattern', 'command class', 'Architecture', 'design', 'high coehsion', 'low coupling']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2022-04-23T01:20:00+02:00
lastmod: 2022-05-26T01:20:00+02:00
featured: true
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

**NOTE:** the concepts work (we use them at work) - but this particular code hasn't yet been tested.  (just recording these ideas for me, but feel free to let me know of any suggestions)

## Organizing Code

In Rails its all too easy to accidentally tightly couple controller activities with follow-up actions - resulting in bloated controllers and / or tight coupling with models.

A relatively simple fix to help with this is to use Events.

Observer & Pub/Sub differences

## Ruby and Rails Environment

Using Rails 7 & Ruby 3.1.2 - I found that it is important to update my ruby environment - so before we start this is what I didn't remove errors:

```bash
# I've had the error several times without updating:
# /Users/btihen/.rbenv/versions/3.1.0/lib/ruby/gems/3.1.0/gems/bundler-2.3.8/lib/bundler/rubygems_ext.rb:18:in `source': uninitialized constant Gem::Source (NameError)
#
#       (defined?(@source) && @source) || Gem::Source::Installed.new
#                                            ^^^^^^^^
# Did you mean?  Gem::SourceList
# this seems to fix it:
# https://bundler.io/guides/bundler_2_upgrade.html
# https://stackoverflow.com/questions/4859600/bundler-throws-uninitialized-constant-gemsilentui-nameerror-error-after-upgr
rbenv local 3.1.2
gem update --system
gem install bundler
gem install rails
rbenv rehash
```

## Rails Project - Simple Blog

Since my other projects are using `esbuild` I use that here too

```ruby
rails new rails_events -T --database=postgresql --css=bootstrap --javascript=esbuild
cd rails_events
bin/rails db:create

# add the helper gem
bundle add wisper_next
```

NOTE: wisper_next works very similar to the other gems (wisper, event_bg_bus, ma).

## Idea

Lets do some activities after a user changes:
* on create: A confirmation email, an invoice email and some setup for the service.
* on change: A confirmation email of change & setup change as needed

Fundamentally, events should be a past tense fact:
*

## Code

First lets create the users model

```bash
bin/rails g scaffold user name config email
```

## User Hooks

Lets do the simplest possible thing - we can use after commits and use them to start activities.

This is ok when very simple, but is highly coupled and requires changes to the user model instead of classes designed to handle the after creation processes.

```ruby
class User < ApplicationRecord
  after_commit :user_created_event, on: :create
  after_commit :user_updated_event, on: :update

  def user_created_event
    # lots of complicated business logic
    puts "UserCreatedConfirmJob: #{email} - send creation confirmation email"
    puts "UserCreatedConfigJob: #{self.changes} - create user setup"
  end

  def user_updated_event
    # lots of complicated business logic
    puts "UserUpdatedConfirmJob: #{email} - send change confirmation email"
    puts "UserUpdatedConfirmJob: #{self.changes} - updated user account config"
  end
end
```

## Controller Alternative

In reality the user isn't responsible for the app so lets try moving the business logic into the Controller where the change is made.  Lets move our Event methods to the controlle, change the create & update actions & leave the model empty.

```ruby
# app/models/user.rb
class User < ApplicationRecord
end

# app/controllers/users_controller.rb
class UsersController < ApplicationController
  # ...
  def create
    @user = User.new(user_params)
    respond_to do |format|
      if @user.save
        # add event call here on success
        user_created_event(@user, @user.changes)
        format.html { redirect_to user_url(@user), notice: "User was successfully created." }
        format.json { render :show, status: :created, location: @user }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @user.errors, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @user.update(user_params)
        # add event call here on success
        user_updated_event(@user, @user.changes)
        format.html { redirect_to user_url(@user), notice: "User was successfully updated." }
        format.json { render :show, status: :ok, location: @user }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @user.errors, status: :unprocessable_entity }
      end
    end
  end

  private

  def user_created_event(user, changes)
    changed_attributes = changes
    # lots of complicated business logic
    puts "UserCreatedConfirmJob: #{user.email} - send creation confirmation email"
    puts "UserCreatedConfigJob: #{changed_attributes} - create user setup"
  end

  def user_updated_event(user, changes)
    changed_attributes = changes
    # lots of complicated business logic
    puts "UserUpdatedConfirmJob #{user.email} - send change confirmation email"
    puts "UserUpdatedConfigJob: #{changed_attributes} - updated user account config"
  end
end
```

this is a bit better, now the business logic isn't embedded in the models.
## Using Commands to Decouple

Ideally, I like having Business Logic in its own class and decoupled from Rails.  Lets move our methods into classes.

```ruby
# app/commands/user_created_command.rb
class UserCreatedCommand
  attr_reader :user, :changes

  def initialize(user:, changes: nil)
    @user = user
    @changes = changes
  end

  def self.call(user:, changes: nil)
    new(user: user, changes: changes).run
  end

  def run
    # lots of complicated business logic
    puts "UserCreatedConfirmJob: #{user.email} - send creation confirmation email"
    puts "UserCreatedConfigJob: #{changes} - create user setup"
  end
end

# app/commands/user_updated_command.rb
class UserUpdatedCommand
  attr_reader :user, :changes

  def initialize(user:, changes:)
    @user = user
    @changes = changes
  end

  def self.call(user:, changes:)
    new(user: user, changes: changes).run
  end

  def run
    puts "UserUpdatedConfirmJob: #{user.email} - send change confirmation email"
    puts "UserUpdatedConfigJob: #{changes} - updated user account config"
  end
end

```

so now we can adjust the controller
```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  # ...
  def create
    @user = User.new(user_params)
    respond_to do |format|
      if @user.save
        # add event call here on success
        UserCreatedCommand.call(@user, @user.changes)
        format.html { redirect_to user_url(@user), notice: "User was successfully created." }
        format.json { render :show, status: :created, location: @user }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @user.errors, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @user.update(user_params)
        # add event call here on success
        UserUpdatedCommand.call(@user, @user.changes)
        format.html { redirect_to user_url(@user), notice: "User was successfully updated." }
        format.json { render :show, status: :ok, location: @user }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @user.errors, status: :unprocessable_entity }
      end
    end
  end
end
```

Commands allow lots of decoupling

## Events

Commands are great, but what if we want well partitioned code, but various parts of the code need to be notified and act on events (if activated)?

This is where events are great!

Lets build our listeners (notice the prefix and async):

```ruby
# app/listeners/user_created_listener.rb
class UserCreatedListener
  include Wisper.subscriber(prefix: true, async: true)

  def on_user_created(user, changes)
    UserCreatedCommand.call(user: user, changes: user.changes)
  end
end

# app/listeners/user_updated_listener.rb
class UserUpdatedListener
  include Wisper.subscriber(prefix: true, async: true)

  def on_user_updated(user, changes)
    UserUpdatedCommand.call(user: user, changes: user.changes)
  end
end

```

We need to define an event_bus and register our listeners (they can be registered on the fly too) - not just at start-up

```ruby
# config/initializers/event_bus.rb
EVENT_BUS = WisperNext::Events.new

EVENT_BUS.subscribe(UserCreatedListener.new)
EVENT_BUS.subscribe(UserUpdatedListener.new)
```

Now we can update the controller again (we need to add the include and broadcast)

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  include WisperNext.publisher
  # ...
  def create
    @user = User.new(user_params)
    respond_to do |format|
      if @user.save
        # add event call here on success - all listers for this event must have :on_user_created
        EVENT_BUS.broadcast(:user_created, user: @user, changes: @user.changes)
        # UserCreatedCommand.call(@user, @user.changes)
        # user_created_event(@user, @user.changes)
        format.html { redirect_to user_url(@user), notice: "User was successfully created." }
        format.json { render :show, status: :created, location: @user }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @user.errors, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @user.update(user_params)
        # add event call here on success - all listers for this event must have :on_user_updated
        EVENT_BUS.broadcast(:user_updated, user: user, changes: @user.changes)
        # UserUpdatedCommand.call(@user, @user.changes)
        # user_updated_event(@user, @user.changes)
        format.html { redirect_to user_url(@user), notice: "User was successfully updated." }
        format.json { render :show, status: :ok, location: @user }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @user.errors, status: :unprocessable_entity }
      end
    end
  end
end
```

Now we are quite flexible - but debugging fully async and decoupled behaviors is difficult - so I generally stay with calling commands directly until I need the configuration flexibility

## summary

Commands are the sweet spot - until config flexibility is needed

## Gems

* ma (active) - https://gitlab.com/kris.leech/ma - objects & async
* wisper_next (active) - https://gitlab.com/kris.leech/wisper_next - (messages with async)
* eventmachine (active) - https://github.com/eventmachine/eventmachine/
* event_bg_bus (active) - https://github.com/indiereign/event_bg_bus (with backgrounding options)
* rails_event_store (active) - https://github.com/RailsEventStore/rails_event_store - (focus on DDD - event bus and event storage)
* wisper (inactive 3 years) - https://github.com/krisleech/wisper
* resugan (inactive 5 years) - https://github.com/jedld/resugan
* event_bus (inactive 8 years) - https://github.com/kevinrutherford/event_bus

## Resources

* https://www.youtube.com/watch?v=q0LtzYTrmMY&t=270s
* https://www.toptal.com/ruby-on-rails/the-publish-subscribe-pattern-on-rails
* https://www.igvita.com/2008/05/27/ruby-eventmachine-the-speed-demon/
* https://www.mikeperham.com/2010/03/30/using-activerecord-with-eventmachine/
* https://www.mikeperham.com/2010/01/27/scalable-ruby-processing-with-eventmachine/


## Going Further with Events & DDD

* https://github.com/RailsEventStore/rails_event_store
* https://medium.com/kontenainc/event-driven-microservices-with-rabbitmq-and-ruby-7a54ae01b285
* https://www.globalapptesting.com/engineering/design-the-unknown-with-the-help-of-event-storming
* Event-Driven Architecture and Messaging Patterns for Ruby Microservices - Kirill Shevchenko - https://www.youtube.com/watch?v=e9AAUy4kkek
