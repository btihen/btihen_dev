---
title:  "Modular Rails using Packwerk"
subtitle: "Organizing Rails Apps with Packages"
date:   2022-07-31 01:59:53 +0200
lastmod:   2022-08-01 01:59:53 +0200
authors: ['btihen']
tags: ['Ruby', 'Packwerk', 'Architecture', 'Citadel Design']
categories: ["Code", "Ruby Language", "Rails Framework"]
summary: Packwerk enables lightweight modular Rails projects.  It is a flexible, low overhead alternative to Rails Engines. Crucially, for established code it allows a gradual migration and to only enforce module boundaries when the code is sufficiently refactored with low coupling.
featured: false
draft: false

image:
  caption: ""
  focal_point: ""
  preview_only: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder

# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight

# Projects (optional)
# Associate this post with one or more of your projects
# Simply enter your project's folder or file name without extension
# E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`

# Otherwise, set `projects = []`

projects: []
---

---

I have been interested in building Rails with much less accidental coupling.  Until recently, Engines have been the best way to do that, but the setup effort is rather heavy (to integrate the migrations, tests, namespaces, ...).  In fact, heavy enough that most people do without modules.

Packwerk however, makes it easy to use modules and critcially migrate toward modules overtime.  Packwerk allows you to organize into modules, without enforcing boundaries - until you are ready to fully refactor and disentagle your code.

------------

## Setup

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

```bash
rails new packwerk --javascript=esbuild --css=tailwind
cd packwerk
bin/rails db:create
```

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

now we should see a file `./packwerk.yml` with all the configs commented out.
```ruby
# packwerk.yml

# See: Setting up the configuration file
# https://github.com/Shopify/packwerk/blob/main/USAGE.md#setting-up-the-configuration-file

# List of patterns for folder paths to include
# include:
# - "**/*.{rb,rake,erb}"

# List of patterns for folder paths to exclude
# exclude:
# - "{bin,node_modules,script,tmp,vendor}/**/*"

# Patterns to find package configuration files
# package_paths: "**/"

# List of custom associations, if any
# custom_associations:
# - "cache_belongs_to"

# Whether or not you want the cache enabled (disabled by default)
# cache: true

# Where you want the cache to be stored (default below)
# cache_directory: 'tmp/cache/packwerk'
```
We will update the configs as we progress.

----------

Now lets be sure dependency visualization tool works:
```bash
bin/rails graphwerk:update
```

Now we should see an intial dependency map named `./packwerk.png` looking like:

![Intial Structure](/images/rails_modules_using_packwerk/initial_dependency_map.png)

### Configure Packages

First, we need a location to place your packages - let's make a folder:
```bash
mkdir app/packages
```

Now Tell rails how to find & load the code in your packages (remember this is dependent on Zeitwerk) `config.paths.add 'app/packages', glob: '*/{*,*/concerns}', eager_load: true` to `config/application.rb`. Now it will look something like:
```ruby
# config/application.rb
...
module RailsPack
  class Application < Rails::Application
    ...
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

### Package Structure

Let's assume we are buildig a manageable blog app for multiple users.

Let's decide what packages (Domains) needed (there are several options depending on archichture and other fixed needs) but that istn't the focus here.

- **published** - publicly available, landing page and access to completed blog artciles
- **compose** - where authors compse and manage their blog articles
- **manage** - manage site admins manager users, and possibly moderate blog articles
- **core** - aspects of code common to all (multiple) aspects of the code basis

The focus of this article is on impletementing packages according to the architectural design.

We will put our packages in a folder called `app/packages` with:
```bash
mkdir app/packages
```

We can add our proposed packages with:
```bash
mkdir app/packages/marketing
mkdir app/packages/published
mkdir app/packages/compose
mkdir app/packages/manage
mkdir app/packages/core
```

An important aspect of a package is the `packages.yml` file so we will need one in EACH pagage!  we can do this by using the one in the rails core as a template

```bash
cp package.yml app/packages/marketing/package.yml
cp package.yml app/packages/published/package.yml
cp package.yml app/packages/compose/package.yml
cp package.yml app/packages/manage/package.yml
cp package.yml app/packages/core/package.yml
```

**IDEALLY, by looking within `app/packages` the next person will have a decent idea about what the code does and what happens where.**

### Marketing Package

Lets generate a landing page:

```bash
bin/rails g controller landing index --no-helper
```

now lets update the routes with:
```ruby
# config/routes.rb
Rails.application.routes.draw do
  get '/landing', to: 'landing#index'
  root 'landing#index' # root path route ("/")
end
```

lets see that this worked: `bin/rails start` and go to:

* `http://localhost:3000`
* `http://localhost:3000/landing`

you should see the landing page.

Now lets move this to the marketing package. To do this we will recreate the code structure in the package and then copy the code into the markiting package.

Creating the package structure:
```bash
mkdir app/packages/marketing/public
mkdir app/packages/marketing/controllers
mkdir -p app/packages/marketing/views/landings
# if you created a helper file then also:
mkdir app/packages/marketing/helpers
```

now we can copy the code:
```bash
mv app/controllers/landing_controller.rb app/packages/marketing/controllers/.
mv app/views/landings app/packages/marketing/views/landings
# if helper created
mv app/helpers/landing_helper.rb app/packages/marketing/helpers/.
```

Finally, lets configure the package in the file `app/packages/marketing/package.yml` - we will use the simplest possible config.
```yml
# app/packages/marketing/package.yml
# Turn on dependency checks for this package
enforce_dependencies: true

# Turn on privacy checks for this package
enforce_privacy: true

# this allows you to modify what your package's public path is within the package
# code that this package publicly shares with other packages
# public_path: public/

# A list of this package's dependencies
# Note that packages in this list require their own `package.yml` file
# '.' - we are dependent on the root application
dependencies:
- '.'
```

----------

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

--------

Now that the code works, lets generate a new diagram of our app using:
```bash
bin/rails graphwerk:update
```

if your graph looks the same as before, check that `app/packages/marketing/packages.yml` is there and configured.

If all is well the new graph in `packwerk.png` will look like:

![Marketing Structure](/images/rails_modules_using_packwerk/with_marketing_package.png)

Now that the package is recognized, lets if packwerk finds any problems
```bash
bin/packwerk check
```

Ideally, packwerk finds no problems and we get:
```bash
No offenses detected
No stale violations detected
```

---------

### Manage Package

Let's generate the user management code:

```bash
bin/rails g scaffold user full_name email --no-helper
bin/rails db:migrate
```

As you can see this generate a log more code.  Models, Controllers, views, etc.  This also generates code that is used for different purposes.

Let's test that this works and consider the code implications.

We should be able to go to `localhost:3000/users` and create and view new users (much like a manager will need to do to manage users). The user model itself will need to be available within the composition area to associati with an article.

Now that we want to setup the `manage` package where admins can manage users.  To do this let's start by creating a routing scope in the routes file `config/routes.rb` by changeing `resources :users` to:
```ruby
# config/routes.rb
  scope 'manage' do
    resources :users
  end
```

lets first be sure we can now access users with: `localhost:3000/manage/users`

lets now make our new `manage` package structure:
```bash
mkdir app/packages/manage
mkdir app/packages/manage/controller
mkdir app/packages/manage/public
mkdir app/packages/manage/views
```

and add the appropriate files:
```bash
cp package.yml app/packages/manage/package.yml
mv app/controllers/users_controller.rb app/packages/manage/controller/users_controller.rb
mv app/packages/manage/views/users app/packages/manage/views/users
```

Let's be sure things still work by going to: `localhost:3000/manage/users`

Let's check that the dependencies look like we expect:

`bin/rails graphwerk:update`

and hopefully we see:

![Manage Package](/images/rails_modules_using_packwerk/with_manage_package.png)

now let's check for violations using: `bin/packwerk check` and hopefully we see:
```
No offenses detected
No stale violations detected
```



---------
### Core Package

Lets decide what all aspects of the 'blog' application will be central to blog `composition` and site `management`

To start we will need to allow users to login and manage their own articles - however the site admins will need to be able to block and otherwise manage users as needed.  Thus users are a good canditate for the `core` package.

Lets see how we can do this.

```bash
bin/rails g scaffold user full_name email --no-helper
bin/rails db:migrate
```

As you can see this generate a log more code.  Models, Controllers, views, etc.  This also generates code that is used for different purposes.

Let's test that this works and consider the code implications.

We should be able to go to `localhost:3000/users` and create and view new users (much like a manager will need to do to manage users). The user model itself will need to be available within the composition area to associati with an article.

Thus it seems like `users` might be appropriate in the `core` package and the controller and view belong in the `manage` package.

To do this lets move the `user` model into the `core` package.

```bash
mkdir -p app/packages/core
mkdir -p app/packages/core/public
mkdir -p app/packages/core/models
touch app/packages/core/package.yml
mv app/models/user.rb app/packages/core/models/user.rb
```

Lets use the same very basic package.yml as before (enforcement and dependent on the `rails` infrastructure):
```yml
# Turn on dependency checks for this package
enforce_dependencies: true

# Turn on privacy checks for this package
enforce_privacy: true

# this allows you to modify what your package's public path is within the package
public_path: public/

# A list of this package's dependencies
# Note that packages in this list require their own `package.yml` file
dependencies:
- '.'
```

Let's see if the code still works like before.

Let's be sure our dependency map is now updated

`bin/rails graphwerk:update`

and we should see:
![Core Package](/images/rails_modules_using_packwerk/with_core_package.png)

You will see that everything still relies on `application` (the rails framework).  Some people will move that into a package too, but we know we are using rails and are dependent upon it, so I don't see any reason to package rails.  (in my mind it may not be necessary to show that we are dependent upon rails, but whatever)

Now finally, let's check our enforcement with:

`bin/packwerk check`

oops - now we have multiple `dependency` and `privacy` violations:
```log
app/packages/manage/controller/users_controller.rb:6:13
Dependency violation: ::User belongs to 'app/packages/core', but 'app/packages/manage' does not specify a dependency on 'app/packages/core'.
Are we missing an abstraction?
Is the code making the reference, and the referenced constant, in the right packages?

Inference details: this is a reference to ::User which seems to be defined in app/packages/core/public/user.rb.
To receive help interpreting or resolving this error message, see: https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md#Troubleshooting-violations


app/packages/manage/controller/users_controller.rb:15:12
Dependency violation: ::User belongs to 'app/packages/core', but 'app/packages/manage' does not specify a dependency on 'app/packages/core'.
Are we missing an abstraction?
Is the code making the reference, and the referenced constant, in the right packages?

Inference details: this is a reference to ::User which seems to be defined in app/packages/core/public/user.rb.
To receive help interpreting or resolving this error message, see: https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md#Troubleshooting-violations


app/packages/manage/controller/users_controller.rb:24:12
Dependency violation: ::User belongs to 'app/packages/core', but 'app/packages/manage' does not specify a dependency on 'app/packages/core'.
Are we missing an abstraction?
Is the code making the reference, and the referenced constant, in the right packages?

Inference details: this is a reference to ::User which seems to be defined in app/packages/core/public/user.rb.
To receive help interpreting or resolving this error message, see: https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md#Troubleshooting-violations


app/packages/manage/controller/users_controller.rb:63:14
Dependency violation: ::User belongs to 'app/packages/core', but 'app/packages/manage' does not specify a dependency on 'app/packages/core'.
Are we missing an abstraction?
Is the code making the reference, and the referenced constant, in the right packages?

Inference details: this is a reference to ::User which seems to be defined in app/packages/core/public/user.rb.
To receive help interpreting or resolving this error message, see: https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md#Troubleshooting-violations
```

These messages state that we need to declare our dependency on `core` within `manage`, we can do this by adding `'app/packages/core'` into the `app/packages/manage/package.yml` so that it would now look like:
```yml
# Turn on dependency checks for this package
enforce_dependencies: true

# Turn on privacy checks for this package
enforce_privacy: true

# this allows you to modify what your package's public path is within the package
public_path: public/

# A list of this package's dependencies
# Note that packages in this list require their own `package.yml` file
dependencies:
- '.'
- 'app/packages/core'
```

Now that we have declared our dependencies, let's see if all is well now:


`bin/packwerk check`

oops now we have a privacy violation:
```log
app/packages/manage/controller/users_controller.rb:6:13
Privacy violation: '::User' is private to 'app/packages/core' but referenced from 'app/packages/manage'.
Is there a public entrypoint in 'app/packages/core/public/' that you can use instead?
Inference details: this is a reference to ::User which seems to be defined in app/packages/core/models/user.rb.
To receive help interpreting or resolving this error message, see: https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md#Troubleshooting-violations


app/packages/manage/controller/users_controller.rb:15:12
Privacy violation: '::User' is private to 'app/packages/core' but referenced from 'app/packages/manage'.
Is there a public entrypoint in 'app/packages/core/public/' that you can use instead?
Inference details: this is a reference to ::User which seems to be defined in app/packages/core/models/user.rb.
To receive help interpreting or resolving this error message, see: https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md#Troubleshooting-violations


app/packages/manage/controller/users_controller.rb:24:12
Privacy violation: '::User' is private to 'app/packages/core' but referenced from 'app/packages/manage'.
Is there a public entrypoint in 'app/packages/core/public/' that you can use instead?
Inference details: this is a reference to ::User which seems to be defined in app/packages/core/models/user.rb.
To receive help interpreting or resolving this error message, see: https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md#Troubleshooting-violations


app/packages/manage/controller/users_controller.rb:63:14
Privacy violation: '::User' is private to 'app/packages/core' but referenced from 'app/packages/manage'.
Is there a public entrypoint in 'app/packages/core/public/' that you can use instead?
Inference details: this is a reference to ::User which seems to be defined in app/packages/core/models/user.rb.
To receive help interpreting or resolving this error message, see: https://github.com/Shopify/packwerk/blob/main/TROUBLESHOOT.md#Troubleshooting-violations
```

So lets follow the suggestion and put `user` into core's public folder:
``bash
mkdir app/packages/core/public
mv app/packages/core/models/users.rb app/packages/core/public/users.rb
```

Now when we check again

`bin/packwerk check`

Now we finally get a clean report:
```log
No offenses detected
No stale violations detected
```

Cool, let's run our tests again and be sure all is still working!


### Author Package

Let's now make a space for authors to work with their articles

```bash
bin/rails g scaffold post content user:references --no-helper
```

lets give authors a scope - in `config/routes.rb` replace `resources :posts` with:
```ruby
# config/routes.rb
...
  scope 'author' do
    resources :posts
  end
...
```

Now lets setup the `author` package.

```bash
mkdir -p app/packages/author
mkdir -p app/packages/author/public
mkdir -p app/packages/author/models
touch app/packages/author/package.yml
mkdir -p app/packages/author/controllers
mv app/views/posts app/packages/author/views/posts
mv app/models/blog.rb app/packages/core/public/blog.rb
cp app/controllers/posts_controller.rb app/packages/author/controllers/posts_controller.rb
```

Lets make `app/packages/author/package.yml` the same as `app/packages/manage/package.yml`
```yml
# Turn on dependency checks for this package
enforce_dependencies: true

# Turn on privacy checks for this package
enforce_privacy: true

# this allows you to modify what your package's public path is within the package
public_path: public/

# A list of this package's dependencies
# Note that packages in this list require their own `package.yml` file
dependencies:
- '.'
- 'app/packages/core'
```

lets now check the dependency graph
```bash
bin/rails graphwerk:update
```

![Author Package](/images/rails_modules_using_packwerk/with_author_package.png)

Lets check dependencies:
```bash
bin/packwerk check
```

We are now good to go
```bash
No offenses detected
No stale violations detected
```

------------

### Rails Package

Now lets clean things up and put rails into a package:

![Rails Package](/images/rails_modules_using_packwerk/packaged_structure.png)


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
