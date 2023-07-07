---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails with Protected Modules"
subtitle: "Rails with sub-modules that can only access the modules public API"
summary: ""
authors: ['btihen']
tags: ['Ruby', 'Rails', 'Modules', 'Isolation']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2023-07-06T01:20:00+02:00
lastmod: 2023-07-07T01:20:00+02:00
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

Rails with Strong Boundaries between the Modules

## Introduction

While writing Rails applications it is very easy to create highly coupled classes.  And possibly low cohesion within features.

Organizing the code with Modules encourages one to consider cohesion and strong Boundaries with Public APIs encourages coupling that can be easily managed.

One could even go one step further and dependency inversion.

In this quick example we will demonstrate using modules with strong boundaries and an enforced public API.

PS - I've read about companies using Modules and API to create a 'citadel' architecture to simply collaboration between groups, such that groups only need to negotiate the public APIs and otherwise groups can work independently without worrying about breaking each others code.

## Getting Started

We will build a VERY simple blog - just to show the concepts.  Obviously, this app is way to simple to benefit from Modules.  However, this keeps it simple enough show how it could be implemented.

Just for fun we will use the newest Rails and Ruby versions.

```
# just for fun
rbenv install 3.3.0-preview1
rbenv local 3.3.0-preview1

# `--main` uses the rails main branch instead of a released version
rails new modules --main --javascript=esbuild --css=tailwind --database=postgresql

cd modules
git init
git add .
git commit -m "initial commit"

bin/rails db:create
```

Later when Rails 7.1 is released we can set the version in the Gemfile with:
`gem 'rails', '~> 7.1'`
instead of the current:
`gem "rails", github: "rails/rails", branch: "main"`

## Config for Modules

make a directory for the modules
```
mkdir app/modules
```

config Rails to find the modules
```
# config/application.rb
module RailsPack
  class Application < Rails::Application
    ...
    # find the code in our modules
    config.paths.add 'app/modules', glob: '*/{*,*/concerns}', eager_load: true
  end
end
```

Finally, let the controllers know how to find the views within packages `app/controllers/application_controller.rb` to:
```
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # find views within our modules
  append_view_path(Dir.glob(Rails.root.join('app/modules/*/views')))
end
```


core compontents in module
```
mkdir app/modules/rails

mv app/views app/modules/rails/.
mv app/models app/modules/rails/.
mv app/helpers app/modules/rails/.
mv app/controllers app/modules/rails/.
```

be sure rails still works  (restart rails if already running)
```
bin/rails s
```

update git
```
git add .
git commit -m "modularize rails"
```


## landing page

```
bin/rails g controller landing/home index --no-helper

mkdir app/modules/landing

mv app/views app/modules/landing/.
mv app/controllers app/modules/landing/.
```


Now update the routes file to look like:
```
# config/routes.rb
Rails.application.routes.draw do
  namespace :landing do
    get 'home/index'
  end

  # Defines the root path route ("/")
  root 'landing/home#index'
end
```

start rails new with `bin/rails s`

the new landing page should appear!

update git
```
git add .
git commit -m "add landing page module"
```

## lets create an authors module

```
bin/rails g scaffold authors/user full_name email --no-helper
bin/rails g scaffold authors/article title body:text authors_user:references --no-helper
bin/rails db:migrate
```

Let's build into an isolated module
```
mkdir app/modules/authors

mv app/views app/modules/authors/.
mv app/models app/modules/authors/.
mv app/controllers app/modules/authors/.
```

rails overlooks namespace relations so we need to change:
```
# app/modules/authors/models/authors/article.rb
class Authors::Article < ApplicationRecord
  belongs_to :authors_user, class_name: 'Authors::User'
end
```

for convience you probably also want to update users to be:
```
# app/modules/authors/models/authors/user.rb
class Authors::User < ApplicationRecord
  has_many :authors_articles, class_name: 'Authors::Article', foreign_key: 'authors_user_id', dependent: :destroy
end
```

so that the following works :
```
bin/rails c
user = Authors::User.first
user.authors_articles
```

**start rails fresh!**

test that `authors/users` & `authors/articles` work as expected


update git
```
git add .
git commit -m "add authors module"
```

## Protected Modules

protect the models using:
`private_constant :Article`
allowing us only access to the models within the `Authors` module - this will require multiple changes.

```
# app/modules/authors/models/authors/article.rb
module Authors
  class Article < ApplicationRecord
    # class_name: 'Authors::User' is needed otherwise Rails assumes the class AuthorsUser
    belongs_to :authors_user, class_name: 'Authors::User'
  end
  private_constant :Article
end
```


```
# app/modules/authors/models/authors/user.rb
module Authors
  class User < ApplicationRecord
  # class_name: 'Authors::Article' is needed otherwise Rails assumes the class AuthorsArticle
    has_many :authors_articles, class_name: 'Authors::Article', foreign_key: 'authors_user_id', dependent: :destroy
  end
  private_constant :User
end
```

Now we need to adjust our `controllers` to avoid

```
# app/modules/authors/controllers/authors/articles_controller.rb
module Authors
  class ArticlesController < ApplicationController
    before_action :set_authors_article, only: %i[ show edit update destroy ]

    # GET /authors/articles or /authors/articles.json
    def index
      @authors_articles = Article.all
      # @authors_articles = Authors::Article.all
    end

    # GET /authors/articles/1 or /authors/articles/1.json
    def show
    end

    # GET /authors/articles/new
    def new
      @authors_article = Article.new
      # @authors_article = Authors::Article.new
    end

    # GET /authors/articles/1/edit
    def edit
    end

    # POST /authors/articles or /authors/articles.json
    def create
      # binding.irb
      # author = Authors::User.find authors_article_params[:authors_user_id]
      # article_params = authors_article_params.except(:authors_user_id).merge(authors_user: author)
      # @authors_article = Authors::Article.new(article_params)
      @authors_article = Article.new(authors_article_params)
      # @authors_article = Authors::Article.new(authors_article_params)

      respond_to do |format|
        if @authors_article.save
          format.html { redirect_to authors_article_url(@authors_article), notice: "Article was successfully created." }
          format.json { render :show, status: :created, location: @authors_article }
        else
          format.html { render :new, status: :unprocessable_entity }
          format.json { render json: @authors_article.errors, status: :unprocessable_entity }
        end
      end
    end

    # PATCH/PUT /authors/articles/1 or /authors/articles/1.json
    def update
      respond_to do |format|
        if @authors_article.update(authors_article_params)
          format.html { redirect_to authors_article_url(@authors_article), notice: "Article was successfully updated." }
          format.json { render :show, status: :ok, location: @authors_article }
        else
          format.html { render :edit, status: :unprocessable_entity }
          format.json { render json: @authors_article.errors, status: :unprocessable_entity }
        end
      end
    end

    # DELETE /authors/articles/1 or /authors/articles/1.json
    def destroy
      @authors_article.destroy!

      respond_to do |format|
        format.html { redirect_to authors_articles_url, notice: "Article was successfully destroyed." }
        format.json { head :no_content }
      end
    end

    private
      # Use callbacks to share common setup or constraints between actions.
      def set_authors_article
        @authors_article = Article.find(params[:id])
        # @authors_article = Authors::Article.find(params[:id])
      end

      # Only allow a list of trusted parameters through.
      def authors_article_params
        params.require(:authors_article).permit(:title, :body, :authors_user_id)
      end
  end
end
```

```
# app/modules/authors/controllers/authors/users_controller.rb
module Authors
  class UsersController < ApplicationController
    before_action :set_authors_user, only: %i[ show edit update destroy ]

    # GET /authors/users or /authors/users.json
    def index
      @authors_users = User.all
      # @authors_users = Authors::User.all
    end

    # GET /authors/users/1 or /authors/users/1.json
    def show
    end

    # GET /authors/users/new
    def new
      @authors_user = User.new
      # @authors_user = Authors::User.new
    end

    # GET /authors/users/1/edit
    def edit
    end

    # POST /authors/users or /authors/users.json
    def create
      @authors_user = User.new(authors_user_params)
      # @authors_user = Authors::User.new(authors_user_params)

      respond_to do |format|
        if @authors_user.save
          format.html { redirect_to authors_user_url(@authors_user), notice: "User was successfully created." }
          format.json { render :show, status: :created, location: @authors_user }
        else
          format.html { render :new, status: :unprocessable_entity }
          format.json { render json: @authors_user.errors, status: :unprocessable_entity }
        end
      end
    end

    # PATCH/PUT /authors/users/1 or /authors/users/1.json
    def update
      respond_to do |format|
        if @authors_user.update(authors_user_params)
          format.html { redirect_to authors_user_url(@authors_user), notice: "User was successfully updated." }
          format.json { render :show, status: :ok, location: @authors_user }
        else
          format.html { render :edit, status: :unprocessable_entity }
          format.json { render json: @authors_user.errors, status: :unprocessable_entity }
        end
      end
    end

    # DELETE /authors/users/1 or /authors/users/1.json
    def destroy
      @authors_user.destroy!

      respond_to do |format|
        format.html { redirect_to authors_users_url, notice: "User was successfully destroyed." }
        format.json { head :no_content }
      end
    end

    private
      # Use callbacks to share common setup or constraints between actions.
      def set_authors_user
        @authors_user = User.find(params[:id])
        # @authors_user = Authors::User.find(params[:id])
      end

      # Only allow a list of trusted parameters through.
      def authors_user_params
        params.require(:authors_user).permit(:full_name, :email)
      end
  end
end
```

Now lets create an API for other aspects of our APP.

create public directory for it
```
mkdir app/modules/authors/public/authors
touch app/modules/authors/public/authors/article_entity.rb
touch app/modules/authors/public/authors/user_entity.rb
```

now create the API files

```
# app/modules/authors/public/authors/article_entity.rb
module Authors
  class ArticleEntity
    include ActiveModel::Model
    attr_accessor :id, :title, :body, :authors_user_id, :created_at, :updated_at

    validates :title, presence: true
    validates :body, presence: true
    validates :authors_user_id, presence: true

    def self.find(id)
      article = Article.find(id)
      new(article.attributes)
    end

    def self.all
      Article.all.map { |article| new(article.attributes) }
    end

    def self.create(params)
      article = Article.new(params)
      article.save
      new(article.attributes)
    end

    def update(params)
      article = Article.find(id)
      article.update(params)
      article.save
      self.attributes = article.attributes
      self
    end

    def destroy
      article = Article.find(id)
      article.destroy
    end
  end
end
```

```
# app/modules/authors/public/authors/user_entity.rb
module Authors
  class UserEntity
    include ActiveModel::Model
    attr_accessor :id, :full_name, :email, :created_at, :updated_at

    validates :full_name, presence: true
    validates :email, presence: true

    def self.find(id)
      user = User.find(id)
      new(user.attributes)
    end

    def self.all
      User.all.map { |user| new(user.attributes) }
    end

    def self.create(params)
      user = User.new(params)
      user.save
      new(user.attributes)
    end

    def update(params)
      user = User.find(id)
      user.update(params)
      user.save
      self.attributes = user.attributes
      self
    end

    def destroy
      user = User.find(id)
      user.destroy
    end
  end
end
```


now we cat test:

be sure we can still create new users and articles in our controllers and that we have access outside with the new public entities
```
bin/rails c

Authors::ArticleEntity.find(1)
Authors::UserEntity.find(1)
# etc
```


update repo
```
git add .
git commit -m "protected module"
```


## JSON API

cgreate the api module
```
mkdir -p app/modules/api/controllers/api/v1
touch app/modules/api/controllers/api/v1/articles_controller.rb
touch app/modules/api/controllers/api/v1/users_controller.rb
```

```
# app/modules/api/controllers/api/v1/articles_controller.rb
module Api
  module V1
    class ArticlesController < ApplicationController
      def index
        articles = Authors::ArticleEntity.all
        render json: { status: 'SUCCESS', message: 'Loaded articles', data: articles }, status: :ok
      end

      # Define other CRUD actions...
    end
  end
end

```


```
# app/modules/api/controllers/api/v1/users_controller.rb
module Api
  module V1
    class UsersController < ApplicationController
      def index
        users = Authors::UserEntity.all
        render json: { status: 'SUCCESS', message: 'Loaded users', data: users }, status: :ok
      end

      # Define other CRUD actions...
    end
  end
end
```

update te route add:
```
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :articles
      resources :users
    end
  end
  # ...
 end
```

now when we go to
`http://localhost:3000/api/v1/articles`
and
`http://localhost:3000/api/v1/users`

we should get the expected results
