---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails with Mingines (Minimized Engines)"
subtitle: "Rails with minimized engines"
summary: ""
authors: ['btihen']
tags: ['Ruby', 'Rails', 'Engines', 'Organizations']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2023-07-11T01:20:00+02:00
lastmod: 2023-10-20T01:20:00+02:00
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
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`
#   Otherwise, set `projects = []`.
projects: []
---

## Intro

This article was inspired by [Julián Pinzón] and his talk at Ruby Australia 2023 [All you need is Rails (Engines): Compartmentalising your Monolith](https://www.youtube.com/watch?v=StDoHXO8H6E).

Its widely known that Engines are a power way to create a full 'sub-rails application'.  However, I've struggled to enjoy engine usage, in particular with the front-end aspects. Thus, I liked the idea of low overhead modularization - just using modules and an updated rails config [Rails with Protected Modules](https://btihen.dev/posts/ruby/rails_protected_modules/).  However, if you want to have the ability to make a fully independent Engine that can be distributed eventually as a gem - but for mow avoiding a full engine, this article describes doing this.

I like calling these `mingines` - minimized engines.

The code in this article available on [rails_mingine](https://github.com/btihen/rails_mingine).

## create a base rails project

note: I like rspec so - I'll skip the tests

```
rails new rails_mingines --javascript=esbuild --css=tailwind --skip-test
cd classrooms
bin/rails db:create

# lets make a place for our mingines - minimized-engines
mkdir mingines
```

## lets create a rails core engine

```
# we will make a landing folder:
mkdir -p mingines/core

# make necessary 'mingine' folders
mkdir -p mingines/core/app
mkdir -p mingines/core/config
mkdir -p mingines/core/lib

# lets make the engine file - (it's a rails loading-helper)
mkdir -p mingines/core/lib/core
touch mingines/core/lib/core/engine.rb

cat <<EOF > mingines/core/lib/core/engine.rb
module Core
  class Engine < Rails::Engine
    # this engine should not use isolated namespace
    # isolate_namespace Core
  end
end
EOF

# routes
mkdir -p mingines/core/config
touch mingines/core/config/routes.rb
cat <<EOF > mingines/core/config/routes.rb
Core::Engine.routes.draw do
end
EOF
```

now lets move our rails files into this engine
```
mv app/* mingines/core/app/.

# however assets and js are easiest to leave in place
mv app/controllers/assets app/.
mv app/controllers/javascript app/.
```

now we need to update `confg/application.rb` with:
* `require_relative '../mingines/core/lib/core/engine.rb'` to load our engine
* `config.paths['db/migrate'] << 'mingines/*/db/migrate'` to find future migrations

Now it should look like:
```
require_relative "boot"

require "rails"
# Pick the frameworks you want:
require "active_model/railtie"
require "active_job/railtie"
require "active_record/railtie"
require "active_storage/engine"
require "action_controller/railtie"
require "action_mailer/railtie"
require "action_mailbox/engine"
require "action_text/engine"
require "action_view/railtie"
require "action_cable/engine"
# require "rails/test_unit/railtie"

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)

# we need to tell rails where to find and load our engines
# if this is confusing you can add each one individually with:
# require_relative '../mingines/core/lib/core/engine.rb'
# to automate this we can do the following: https://stackoverflow.com/questions/1899072/getting-a-list-of-folders-in-a-directory
Dir.chdir('mingines') do
  Dir.glob('*').select { |f| File.directory? f }.each do |name|
    require_relative "../mingines/#{name}/lib/#{name}/engine.rb"
  end
end

module RailsMingines
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 7.1

    # we need to tell rails where to find the engine migrations
    config.paths['db/migrate'] << 'mingines/*/db/migrate'

    # Don't generate system test files.
    config.generators.system_tests = nil
  end
end
```

lets start rails
```
bin.rails server
```
and be sure it all still works

cool let's commit:
```
git add .
git commit -m 'added a rails - core engine'
```

## Landing-page engine

This is of course overdone for a single page, but for demo purposes we will pretend we have large complex landing-page that needs its own namespace

Instead instead of running: `bin/rails plugin new landing --mountable --skip-git`

we will instead build the engines ourselves (without the gemspecs and other extras to make our engine portable)
```
# we will make a landing folder:
mkdir -p mingines/landing

# make necessary 'mingine' folders
mkdir -p mingines/landing/app
mkdir -p mingines/landing/config
mkdir -p mingines/landing/db/migrate
mkdir -p mingines/landing/lib

# lets make the engine file - (it's a rails loading-helper)
mkdir -p mingines/landing/lib/landing
touch mingines/landing/lib/landing/engine.rb

cat <<EOF > mingines/landing/lib/landing/engine.rb
module Landing
  class Engine < Rails::Engine
    # ensures we have our own namespace within our engine
    isolate_namespace Landing
  end
end
EOF

# routes
mkdir -p mingines/landing/config
touch mingines/landing/config/routes.rb
cat <<EOF > mingines/landing/config/routes.rb
Landing::Engine.routes.draw do
end
EOF
```

So we can build our home page controller:
```
# we can build it with the normal rails generator
bin/rails g controller landing/home index --no-helper

# now we see that we have created: controllers, views and routes
mkdir -p mingines/landing/app/controllers
mkdir -p mingines/landing/app/views

# copy our new files into our 'mingine'
mv app/controllers/landing mingines/landing/app/controllers/.
mv app/views/landing mingines/landing/app/views/.
```

Now we will need to update the 'mingine' route
```
mkdir -p mingines/landing/config
touch mingines/landing/config/routes.rb

cat <<EOF > mingines/landing/config/routes.rb
Rails.application.routes.draw do
  get 'home/index'

  root "home#index"
end
EOF
```

NOW we need to update `config/routes.rb` so it looks like:
```
Rails.application.routes.draw do
  mount Landing::Engine => '/'
  # ...
end
```

Now we should have our landing page when we go to:
`localhost:3000`

assuming it works we can add a commit:
```
git add .
git commit -m 'added landing page minigine'
```

## Let's try an Engine with Data Models

we will build the engines with just what we need
```
# we will make a landing folder:
mkdir -p mingines/blogs

# make necessary 'mingine' folders
mkdir -p mingines/blogs/app
mkdir -p mingines/blogs/config
mkdir -p mingines/blogs/db/migrate
mkdir -p mingines/blogs/lib

# lets make the engine (loading helper)
mkdir -p mingines/blogs/lib/blogs
touch mingines/blogs/lib/blogs/engine.rb

# create a loading engine that keeps 'landing' as an isolated namespace
cat <<EOF > mingines/blogs/lib/blogs/engine.rb
module Blogs
  class Engine < Rails::Engine
    isolate_namespace Blogs
  end
end
EOF

# routes
mkdir -p mingines/blogs/config
touch mingines/blogs/config/routes.rb
cat <<EOF > mingines/blogs/config/routes.rb
Blogs::Engine.routes.draw do
end
EOF
```


Let's create our models (etc) - again using standard generators
```
bin/rails g scaffold blogs/user full_name email --no-helper
bin/rails g scaffold blogs/article title body:text blogs_user:references --no-helper

# create missing folders
mkdir mingines/blogs/app/controllers
mkdir mingines/blogs/app/models
mkdir mingines/blogs/app/views
mkdir mingines/blogs/db
mkdir mingines/blogs/db/migrate

# copy our new code
mv app/controllers/blogs mingines/blogs/app/controllers/.
mv app/models/blogs mingines/blogs/app/models/.
mv app/models/blogs.rb mingines/blogs/app/models/.
mv app/views/blogs mingines/blogs/app/views/.

mv db/migrate/* mingines/blogs/db/migrate/.
```

now we need to update our mingine routes with:
```
Blogs::Engine.routes.draw do
  resources :articles
  resources :users
end
```

now lets update the core routes:
```
Rails.application.routes.draw do
  mount Blogs::Engine, at: 'blogs'
  # ...
end
```

Now we need to make 3 adjustments for the generators (of course if you are experienced you can avoid this and just create the necessary files yourself):

**First** - the models classnames
```
# mingines/blogs/app/models/blogs/articles.rb
class Blogs::Article < ApplicationRecord
  belongs_to :blogs_user, class_name: 'Blogs::User'
end

# and

# mingines/blogs/app/models/blogs/user.rb
class Blogs::User < ApplicationRecord
  has_many :blogs_articles, class_name: 'Blogs::Article', foreign_key: 'blogs_user_id', dependent: :destroy
end
```

**Second** - the paths
```
edit_blogs_article_path -> edit_article_path
new_blogs_article_path -> new_article_path
blogs_article_path -> article_path

edit_blogs_user_path -> edit_user_path
new_blogs_user_path -> new_user_path
blogs_user_path -> user_path

blogs_user_url -> user_url
blogs_users_url -> users_url
blogs_article_url -> article_url
blogs_articles_url -> articles_url
```
Now your paths should match: `bin/rails routes`

**Third** - params in controllers:
`param is missing or the value is empty: blogs_user`
```
# mingines/blogs/app/controllers/blogs/articles_controller.rb
    # Only allow a list of trusted parameters through.
    def blogs_article_params
      params.require(:article).permit(:title, :body, :blogs_user_id)
      # params.require(:blogs_article).permit(:title, :body, :blogs_user_id)
    end
end

# and

# mingines/blogs/app/controllers/blogs/users_controller.rb
    # Only allow a list of trusted parameters through.
    def blogs_user_params
      params.require(:user).permit(:full_name, :email)
	    # params.require(:blogs_user).permit(:full_name, :email)
    end
end
```

run the migrations (given the `config/application.rb` update they run in place)
```
bin/rails db:migrate
```

**NOTE:** I use the trick to configure them to run in-place, since I haven't set up the standard rails way to copy the migrations to the main app `db/migrations` folder - which would normally be done with:
```
bin/rails blogs:install:migrations
bin/rails db:migrate
```

now we can go to:
```
# first create a user
http://localhost:3000/users

# afterwards create an article
http://localhost:3000/articles
```

now we commit
```
git add .
git commit -m "add the blogs mingine"
```


## Experience - Standard vs Modular

My colleague [Jörg Jenni](https://www.linkedin.com/in/jorgjenni/) - created a little app [messy vs modular](https://github.com/Enceradeira/messy_app) - that need needs new features.  It has 2 base Apps `messy` and `modulariyed`.  Its a great way to experience the benefits of modular design.

My solution can be seen at: https://github.com/btihen/coupled_vs_modular


## using JS within an mingine

## using rspec within an mingine

## using factories within an mingine

## protected APIs within an mingine

## using packwerk within an mingine

## Further Ideas

For further improvements (protected namespaces, using packwerk to provide feedback and a list of design ideas from many others) see [Rails with Protected Modules](https://btihen.dev/posts/ruby/rails_protected_modules/)
