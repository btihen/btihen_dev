---
title:  "Rails Packwerk Refactor"
date:   2022-08-01 01:59:53 +0200
lastmod:   2022-08-01 01:59:53 +0200
summary: Packwerk enables lightweight modular Rails projects.  It is a flexible, low overhead alterntive to Rails Engines. Crucially, for established code it allows a gradual migration and to only enforce module boundaries when the code is sufficiently refactored with low coupling.
authors: ['btihen']
tags: ['Ruby', 'Packwerk', 'Architecture', 'Design']
categories: ["Code", "Ruby Language", "Rails Framework"]
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

I have been interested in building Rails with much less accidental coupling.  Until recently, Engines have been the best way to do that, but the setup effort is rather heavy (to integrate the migrations, tests, namespaces, ...).  In fact, heavy enough that most people do without modules.

Packwerk however, makes it easy to use modules and critcially migrate toward modules overtime.  Packwerk allows you to organize into modules, without enforcing boundaries - until you are ready to fully refactor and disentagle your code.

To demonstrate Packwerk's usage lets start with a Standard Rails Monolith and iteratively transform it into discrete packages with boundaries.  The file structure of a fully packaged project is easy to visually understand.

The routes file often will give us a clue to or application boundaries (espescially if name spaces or scopes are in use), alternatively a well structured engine architechture may clarify the application boundaries.

For example I have packaged a standard rails application - here are the before and after for the file system (compared to the scoped routes file).

![File Structure and Routes](/images/rails_packwerk_refactor/file_strucutures_n_routes.png)

Here is the packaged rails dependency graph:

![Packaged Dependency Graph](/images/rails_packwerk_refactor/packwerk_structure_07_rails.png)

**NOW IDEALLY**, by looking within `app/packages` and the `packwerk.png` the next person will have a decent idea about what the code does and where.

An out line of how to create this basic app is in the appendix - for those who want to follow along.

------------

## Setup Packwerk

Packwerk its self is a simple gem install.  Crucially, Packwerk has a companion tool `graphwerk` to visualize your modules and their dependencies. We will install and demostrate both.

### Prequisits

In oder to use graphwerk you must have graphviz installed:
```bash
brew install graphviz
```
does the trick on MacOS

You will also need to use at least `Ruby 2.6+` and `Rails 6.0+` with `Zeitwerk` as the loader (which is the default for new Projects).

### Install Packwerk

This will be a very simple projects (too simple to need modules), but that also makes the examples easier to grock.

We will start with a fresh rails projects to keep the complexity low and make it straight-forward to follow along before using in your own established projects.

Now add the packwerk and graphwerk gems to `./Gemfile`
```ruby
# Gemfile
...
# Packwerk (modular Rails) - dependency tooling and mapping
gem "packwerk", "~> 2.2"
gem 'graphwerk', group: %i[development test]
```
of course normally, I would install rspec and a slew of other tools, but the focus here is simply the usage of Packwerk and its associated tools.

Now of course we need to finalize the install and config:
```bash
bundle install
# make it easy to
bundle binstub packwerk
# create intial packwerk config files
bin/packwerk init
```

now we should see a file `./packwerk.yml` with all the configs commented out.  We will learn to configure as we go.

----------

Now lets be sure dependency visualization tool works:
```bash
bin/rails graphwerk:update
```

Now we should see an intial dependency map named `./packwerk.png` looking like:

![Structural Image](/images/rails_packwerk_refactor/rails_standard_01_packwerk.png)

### Configure Packages

First, we need a location to place your packages - let's make a folder:
```bash
mkdir app/packages
```

Now Tell rails how to find & load the code in your packages (remember this is dependent on Zeitwerk) `config.paths.add 'app/packages', glob: '*/{*,*/concerns}', eager_load: true` to `config/application.rb`. Now it will look something like:
```ruby
# config/application.rb
require_relative "boot"

require "rails/all"

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)

module Packaged
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 7.0

    # config packages fur packwerk
    config.paths.add 'app/packages', glob: '*/{*,*/concerns}', eager_load: true
  end
end
```

Finally, let the controllers know how to find the views within packages  `app/controllers/application_controller.rb` to:
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  append_view_path(Dir.glob(Rails.root.join('app/packages/*/views')))
end
```


------------

## Using Packwerk

From doing this a few times - I've learned its easiest to work from the outside in.  In other words extract actions first and organize them first.  Then move the models into the appropriate packages.  Then package (most of rails - assets is trickier - so I jut leave it alone

### Marketing Package

We can add our proposed packages with:
```bash
mkdir app/packages/marketing
mkdir app/packages/marketing/public
mkdir -p app/packages/marketing/views
mkdir app/packages/marketing/controllers
cp package.yml app/packages/marketing/package.yml
```

if you have helpers, etc make those folders too.

now we can copy the code (controllers & views):
```bash
mv app/controllers/landing_controller.rb app/packages/marketing/controllers/.
mv app/views/landings app/packages/marketing/views/landings
```

----------

now start rails and test that the base config is good (then we will start with enforcing package cofig)

if you get the error:
```
LandingController#index is missing a template for request formats: text/html
```

check that `app/controllers/application_controller.rb` looks like
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  append_view_path(Dir.glob(Rails.root.join('app/packages/*/views')))
end
```

if you get the error `uninitialized constant LandingController`

check that `config/application.rb` has the following code:
```ruby
...
module Packwerk
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 7.0
    ...
    # load the packages
    config.paths.add 'app/packages', glob: '*/{*,*/concerns}', eager_load: true
  end
end
```

If you forgot this config - you will need to restart rails!

-----

Now that we have confirmed our config - we need to config boundary enforcement in the package's `packages.yml` file config.  So lets configure the package in the file `app/packages/marketing/package.yml` - we will use the simplest possible config.
```yml
# app/packages/marketing/package.yml
# Turn on dependency checks for this package
enforce_dependencies: true

# Turn on privacy checks for this package
enforce_privacy: true

# this allows you to modify what your package's public path is within the package
# code that this package publicly shares with other packages
public_path: public/

# A list of this package's dependencies
# Note that packages in this list require their own `package.yml` file
# '.' - we are dependent on the root application
dependencies:
- '.'
```

----------

Now that the code works, lets generate a new diagram of our app using:
```bash
bin/rails graphwerk:update
```

if your graph looks the same as before, check that `app/packages/marketing/packages.yml` is there and configured.

If all is well the new graph in `packwerk.png` will look like:

![Marketing Structure](/src/images/rails_packwerk_refactor/packwerk_structure_02_managers.png)

Now that the package is recognized, lets if packwerk finds any problems
```bash
bin/packwerk check
```

Ideally, packwerk finds no problems and we get:
```bash
No offenses detected
No stale violations detected
```

In a full production system this command would then be integrated into the CI to only allow clean packages to commit.

---------

### Package Managers

basically the same as for marketing

lets now make our new `manage` package structure:
```bash
mkdir app/packages/managers
mkdir app/packages/managers/public
mkdir app/packages/managers/views
mkdir app/packages/managers/controller
cp app/packages/marketing/package.yml app/packages/managers/package.yml
```

move the appropriate files:
```bash
mv app/controllers/users_controller.rb app/packages/manage/controller/users_controller.rb
mv app/controllers/blogs_controller.rb app/packages/manage/controller/blogs_controller.rb

mv app/packages/manage/views/users app/packages/manage/views/users
mv app/packages/manage/views/blogs app/packages/manage/views/blogs
```

Let's be sure things still work by going to: `localhost:3000/manage/users`

Let's check that the dependencies look like we expect:

`bin/rails graphwerk:update`

and hopefully we see:

![Manage Package](/images/rails_packwerk_refactor/packwerk_stucture_03_managers.png)

now let's check for violations using: `bin/packwerk check` and hopefully we see:
```
No offenses detected
No stale violations detected
```

### Writers Package

### Guests Package

---------
### Core Package


```bash
mkdir -p app/packages/core
mkdir -p app/packages/core/public
mkdir -p app/packages/core/models
cp app/packages/managers/package.yml app/packages/core/package.yml
```

lets move the models
```bash
mv app/models/blog.rb app/packages/core/models/blog.rb
mv app/models/user.rb app/packages/core/models/user.rb
```

Let's see if the code still works like before.


Now finally, let's check our enforcement with:

`bin/packwerk check`

oops - now we have multiple `dependency` and `privacy` violations:
```log
app/packages/writers/controllers/posts_controller.rb:21:12
Dependency violation: ::Blog belongs to 'app/packages/core', but 'app/packages/writers' does not specify a dependency on 'app/packages/core'.
Are we missing an abstraction?
Is the code making the reference, and the referenced constant, in the right packages?

Inference details: this is a reference to ::Blog which seems to be defined in app/packages/core/models/blog.rb.
To receive help interpreting or resolving this error message, see: https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md#Troubleshooting-violations
```

problem is we haven't told our packages they depend on core - to fix we will update package.yml files in managers, writers, guests to depend on core.  Marketing and rails application don't need to change. we will make them look like:
```yml
# app/packages/marketing/package.yml
# Turn on dependency checks for this package
enforce_dependencies: true

# Turn on privacy checks for this package
enforce_privacy: true

# this allows you to modify what your package's public path is within the package
# code that this package publicly shares with other packages
public_path: public/

# A list of this package's dependencies
# Note that packages in this list require their own `package.yml` file
# '.' - we are dependent on the root application
dependencies:
- 'app/packages/core'
- '.'
```

If you are still getting the above error check your tests.  Unfortunately, packwerk doesn't tell zou the offending file, but you can search for the model reference with excluding `app/packages`

check again:

```log
app/packages/writers/controllers/profiles_controller.rb:5:16
Privacy violation: '::User' is private to 'app/packages/core' but referenced from 'app/packages/writers'.
Is there a public entrypoint in 'app/packages/core/public/' that you can use instead?
Inference details: this is a reference to ::User which seems to be defined in app/packages/core/models/user.rb.
To receive help interpreting or resolving this error message, see: https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md#Troubleshooting-violations
```

now we have a privacy error - we need to move our models to public - as a way to declare we intend to share them

lets put our models in public (if zou want them in public/models - zou will need to make more tweaks to rails paths)

Let's be sure our dependency map is now updated

`bin/rails graphwerk:update`

and we should see:
![Core Package](/images/rails_modules_using_packwerk/with_core_package.png)


### Rails Package

pretty good but folders still messy - lets make rails in a packagr too

```bash
mkdir app/packages/rails
```

Move everything remaining (our rails files)

- app/assets
- app/packages

into `app/packages/rails`


------------

## Overview

Hanami 2.x will use a structure like Packwerk and is built on dry-rb, which also has many intresting features and useful patterns.  When Hanami 2.x is released, a similar dependency checker like packwerk would be valuable.

Currently with Rails one can achieve most of what Hanami 2.x framework offers with Packwerk and [dry-rails](https://github.com/dry-rb/dry-rails).

### Benefits

### Drawbacks

-------------

## Resources

For on organizing Ruby code via packages / modules

### Hanami 2.x

* [Hanami 2.x Code](https://github.com/hanami/hanami)
* [Hanami 2.x Blog](https://hanamirb.org/blog/) - currently [Hanami 2.0 is in Beta](https://hanamirb.org/blog/2022/07/20/announcing-hanami-200beta1/) and is targeted for API usage (if you are willing to assemble all the parts Hanami 2.0 will also offers front-end services),  Hanami 2.1 will be focused on delivering a fully integrated full stack.
* [dry-rb code](https://github.com/dry-rb)
* [dry-rb Website](https://dry-rb.org/)
* [rom-rb code](https://github.com/rom-rb/rom)
* [rom-rb website](https://rom-rb.org/) - speed focus alternative to Rail's ActiveRecord.  While many ORMs focus on objects and state tracking, ROM focuses on data and transformations (a bit like Elixir's Ecto).  This project is heavily influenced by
* [Hanami Mastery](https://hanamimastery.com/) - tutorials about dry-rb gems and Hanami 2.x concepts in order to help people coming from Rails to understand and use Hanami 2.x effectively.

### Modular Rails using Packwerk

* [Shopify - Packwerk Code](https://github.com/Shopify/packwerk)
* [Shopify - Packwerk Docs](https://github.com/Shopify/packwerk/blob/main/USAGE.md)
* [Shopify - Packwerk Debug Help](https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md)
* [Shopify - Video introducing Packwerk](https://www.youtube.com/watch?v=olEA157z7kU)
* [Shopify - on Monoliths](https://www.shopify.com/partners/blog/monolith-software)
* [Shopify - enforcing-modularity-rails-apps-packwerk](https://shopify.engineering/enforcing-modularity-rails-apps-packwerk)
* [Package-Based-Rails-Applications Book](https://leanpub.com/package-based-rails-applications), by Stephan Hagemann
* [modularization-with-packwerk](https://thecodest.co/blog/ruby-on-rails-modularization-with-packwerk-episode-i/)
* [packwerk-to-delimit-bounded-contexts](https://www.globalapptesting.com/engineering/implementing-packwerk-to-delimit-bounded-contexts)

### Modular Rails using Engines

* [Rails Engines Docs](https://edgeguides.rubyonrails.org/engines.html)
* [Component-Based-Rails-Applications Website](https://cbra.info/resources/), Stephan Hagemann - many links and articles on using Engines and enforcing boundaries
* [Component-Based Rails Applications Book, 2018](https://www.pearson.com/en-us/subject-catalog/p/Hagemann-Component-Based-Rails-Applications-Large-Domains-Under-Control/P200000009490/9780134774589), by Stephan Hagemann
* [Modular-Rails Book / Website](https://devblast.com/c/modular-rails), by Thibault Denizet


## Appendix - Create the App

### Quick Start

Lets make a this simple Blog App as our starting point.

NOTE: Packwerk require Ruby 2.6 or newer and Rails must be configured with Zeitwerk.

```bash
rails new packaged --javascript=esbuild --css=tailwind --skip-active-storage
cd packaged
bin/rails db:create
# marketing
bin/rails g controller landing index --no-helper
# manager
bin/rails g scaffold user full_name email --no-helper
bin/rails g scaffold blog content user:references --no-helper
# bin/rails g scaffold comment note blog:references --no-helper
bin/rails db:migrate
# writers
bin/rails g controller profiles index show edit update --no-helper
bin/rails g controller posts index show new create edit update delete --no-helper
# bin/rails g controller discussion index new create edit update delete --no-helper
# patrons
bin/rails g controller authors index show --no-helper
bin/rails g controller articles index show --no-helper
```

### Routes

now lets update the routes with:
```ruby
# config/routes.rb
  scope 'guests' do
    resources :authors, only: %i[index show] do
      resources :articles, only: %i[index show]
    end
  end
  scope 'writers' do
    resources :profiles, only: %i[index show edit update] do
      resources :posts
    end
  end
  scope 'managers' do
    resources :users do
      resources :blogs
    end
  end
  get '/landing', to: 'landing#index'
  root 'landing#index' # root path route ("/")
end
```

### Fix the nested Controllers & Views

#### Managers Code



#### Writers
full starting code can be found here:
[]()
