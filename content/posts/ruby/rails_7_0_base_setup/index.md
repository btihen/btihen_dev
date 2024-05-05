---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.0 Base Setup"
subtitle: "Creating a Rails 7.0 App"
summary: ""
authors: ['btihen']
tags: ['Ruby', 'rails install', 'rails setup']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2022-03-12T01:20:00+02:00
lastmod: 2022-03-12T01:20:00+02:00
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

## update GEMS library
gem update --system

## be sure bundler is installed
gem install bundler

## update Bundler library
bundle update --bundler

## update Ruby
rbenv install 3.1.1
rbenv global 3.1.1

**or**
asdf install ruby 3.1.1
asdf global ruby 3.1.1

## be sure you have rails for your ruby version
```bash
gem install rails
# or ensure version 7.0 using
gem install rails --version 7.0.0
```

## Rails with esbuild & css

The options:
* `-T` - does not install mini-tests (I like rspec)
* `-j esbuild` - is the middle ground between using webpacker and import-maps
* `--css bulma` - install a css framwork (bulma, tailwind, bootstrap, sass, postcss)
*
**NOTE:** if you use `esbuild` be sure you have `yarn`, `npm` and `npx` already installed


```bash
# default rails (using importmaps)
# rails new slacker_base
# or Bulma
rails new slacker_bulma -j esbuild -T --css bulma
# with Tailwind
# rails new slacker_tail -j esbuild -T --css tailwind
cd slacker
```

## install richtext editor

This will also install StimulusJS and ActiveStorage and ActionText altogether
```bash
./bin/rails action_text:install
```

otherwise you can install them separately:

**ActiveStorage**
```
bin/rails active_storage:install
```

**Stimulus**
```
./bin/bundle add stimulus-rails
./bin/bundle install
./bin/rails stimulus:install
```

**Hotwire**
if you initially used `--skip-hotwire` and now want it - type:
```bash
./bin/bundle add turbo-rails
./bin/bundle install
./bin/rails turbo:install
./bin/rails turbo:install:redis
```

## Additional Tooling

copy the following into your Gemfile:

```ruby
cat <<EOF >>Gemfile
group :test do
  gem 'shoulda-matchers' #, '~> 5.0'
end

# allow rspec with feature tests and
group :development, :test do
  gem 'factory_bot_rails'
  gem 'rspec-rails'
  gem 'capybara'
  gem 'launchy'
  gem 'faker'

  # coverage
  gem 'simplecov'

  # security checks
  gem 'brakeman'

  # debugging
  # gem 'pry'
  gem 'pry-rails'
  # stack, up, down, frame n
  gem 'pry-stack_explorer' #, '~> 0.6.0'

  # standards
  # gem 'standard', require: false
  gem 'rubocop-rails', require: false

  # sw quality checks
  # gem 'skunk'
  # gem 'circle-cli'
  # gem 'rubycritic', require: false # uses virtus - discontinued !
end
EOF
```

## Configure Testing

```bash
bin/rails g rspec:install
```

**Setup Configuation**
at the top of the spec/rails_helper.rb be sure to have:
```ruby
require 'capybara/rails'
require 'simplecov'
SimpleCov.start
```

at the bottom of spec/rails helper add:
```ruby
cat <<EOF >>spec/rails_helper.rb
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
EOF
```

## Setup the Database
```bash
bin/rails db:drop
bin/rails db:create
bin/rails db:migrate
bin/rails db:seed
```

## Login Management (as needed):

Devise is commonly used for login  user management

```
bundle add devise
bundle install
rails g devise:install
```

**Follow the instruction on printed on the screen or use (https://guides.railsgirls.com/devise)

## Setup App Structure

Lets build a landing page
```bash
bin/rails g controller landing index
```

**update the routes**
```ruby
# config/routes.rb
Rails.application.routes.draw do
  get 'landing/index'

  # Defines the root path route ("/")
  root "landing#index"
end
```

## Start Rails (dev-mode)

**In watchmode - like we are accustom**

`./bin/dev`

other options are:
* `foreman start -f Procfile.dev`
* `bin/rails s` and `yarn build --watch` in separate windows


## Experiments to Explore

### Packwerk
### StimulusJS
### view components
### HotWire (Turbo)
