---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails Design: Protected Modules with Injection"
subtitle: "Long-term Manageable Rails through Low Dependency/Entanglement and Loose Coupling"
summary: ""
authors: ['btihen']
tags: ['Ruby', 'Rails', 'Modules', 'Isolation', 'Injection']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2023-08-12T01:20:00+02:00
lastmod: 2023-08-12T01:20:00+02:00
featured: true
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

Rails Design - Modules with Boundaries and Injection

Recently, a colleague [Severin Räz](https://www.linkedin.com/in/severin-r%C3%A4z-567ab81b1/) suggested that we could be using `module` to create namespaces (to organize our code with high cohesion) and `private_constant` to enforce API boundaries (enforce low coupling). Based on that I ran some experiments and wrote about them in [Rails with Protected Modules](https://btihen.dev/posts/ruby/rails_protected_modules/) and [Rails with Minimal Engines)](https://btihen.dev/posts/ruby/rails_minimal_engines/)

Similarly, another colleague [Jörg Jenni](https://www.linkedin.com/in/jorgjenni/) has been encouraging dependency injection to ensure loose coupling, making it easy to replace a service with another and simplifying testing. Dependency-Inversion is an alternative to hardcoding classes into code.

This article builds on the previous articles [Rails with Protected Modules](https://btihen.dev/posts/ruby/rails_protected_modules/) and [Rails with Mingines](https://btihen.dev/posts/ruby/rails_minimal_engines/), but is not required for this article to make sense.

In any case, this article describes what we (after discussions and a few code iterations) are using as a starting point for maintainable design.

## App Description

Let's assume your application creates valuable documents that need to be archived.  We will keep this simple and design a module that can send or retrieve a document to and from an Archive Server.  I will use the open source [Teedy](https://teedy.io/#!/) Server - which can be self-hosted with [docker](https://github.com/sismics/docs#install-with-docker).  The [API](https://demo.teedy.io/apidoc/) is straight forward and easy to use.

Teedy will be our default adapter, however, we want to make it easy for our app to work with any Archive Server.

## Plan our approach

* We will start by building a simple module to act as an API wrapper.
* we will build a basic Teedy adapter (feel free to build other for other archive server)
* we will then integrate it into our Rails app

## Getting Started

The code for this experiment can be found at: <https://github.com/btihen/protected_rails_modules>

Just for fun we will use the newest Rails and Ruby versions, but these techniques should work on any version of Rails.

```bash
# install ruby
rbenv install 3.2.2
# or next preview
rbenv install 3.3.0-preview1

# best for production
rbenv local 3.2.2
# fun for learning
rbenv local 3.3.0-preview1

# `--main` uses the rails main branch instead of a released version (--skip-test - I like rspec)
rails new archiver --main --javascript=esbuild --css=tailwind --database=postgresql --skip-test

cd archiver
git add .
git commit -m "initial commit"

bin/rails db:create
```

Later when Rails 7.1 is released we can set the version in the Gemfile with: gem 'rails', '~> 7.1' instead of the current: gem "rails", github: "rails/rails", branch: "main"

# Create Documents

Let's make an Article model within the Author's namespace (article content will use Action Text so we can capture the person's writing) which we can then display as an HTML page or save as a PDF file.

first we will install the required tools:

```bash
bin/rails action_text:install
bin/rails db:migrate
# prawn needs to be added to load the the matrix gem (for some reason pawn-html does not load it)
bundle add 'matrix'
bundle add 'prawn-html'
# for some reason I had to run this to fix a bundler conflict:
bundle install --gemfile Gemfile
# the error was:
# `You specified: image_processing (~> 1.2) and image_processing (>= 0). Gem already added. Bundler cannot continue.`
```

Let's generate the Article scaffold (with a namespace to keep this separate from other features):

```bash
# content will use Action Text so we can format the writing
bin/rails g scaffold Blog::Article author:string title:string
```

Now we need to add the Action Text field to the Article model:

```ruby
# app/models/blog/article.rb
class Article < ApplicationRecord
  has_rich_text :content
end
```

We need to update our controller to permit the new `content` field:

```ruby
# app/controllers/blog/articles_controller.rb
  ...
  private

  # Use callbacks to share common setup or constraints between actions.
  def set_blog_article
    @blog_article = Blog::Article.find(params[:id])
  end

  # Only allow a list of trusted parameters through.
  def blog_article_params
    params.require(:blog_article).permit(:author, :title, :content)
  end
end
```

We will also need to add `<%= form.rich_text_area :content %>` to the Article form :

```erb
# app/views/blog/articles/_form.html.erb
  ...
  <div>
    <%= form.label :author, style: "display: block" %>
    <%= form.text_field :author %>
  </div>

  <div>
    <%= form.label :title, style: "display: block" %>
    <%= form.text_field :title %>
  </div>

  <div>
    <%= form.label :content, style: "display: block" %>
    <%= form.rich_text_area :content %>
  </div>

  <div>
    <%= form.submit %>
  </div>
<% end %>
```

We also need to update our view to display the content with `<%= article.content %>`:

```erb
# app/views/blog/articles/_article.html.erb
<div id="<%= dom_id article %>">
  <p>
    <strong>Author:</strong>
    <%= article.author %>
  </p>

  <p>
    <strong>Title:</strong>
    <%= article.title %>
  </p>

  <div>
    <%= article.content %>
  </div>

</div>
```

Now let's test by going to:
```bash
# NOTE: with the tradionitional `bin/rails s` the Action Text (Trix) field will not display so use!
# https://stackoverflow.com/questions/73299862/action-text-rails-7-error-vanishing-field
bin/dev

# open our browser and add a new article with some formatting:
open http://localhost:3000/blog/articles
```

cool - let's add this feature to our repo:

```bash
git add .
git commit -m "add Blog::Article with ActionText"
```

## Build Module and API

```
touch
```

## Resources

* [Practical Object-Oriented Design, An Agile Primer Using Ruby (POODR)](https://www.poodr.com) - this book covers dependency injection and dependency inversion in detail (as well as many other Design practices)
* [Practical Object-Oriented Design in Ruby - Notes](https://github.com/serodriguez68/poodr-notes#reversing-dependencies)
* [Exploring dependency injection in Ruby](https://remimercier.com/dependency-injection-in-ruby/)
* [Introduction to dependency injection in Ruby](https://medium.com/@Bakku1505/introduction-to-dependency-injection-in-ruby-dc238655a278)
* [How to implement Dependency Injection Pattern in Ruby On Rails?](https://dev.to/vladhilko/how-to-implement-dependency-injection-pattern-in-ruby-on-rails-28d6)
* [Dependency Injection in Ruby from 0 to hero (Part 1)](https://hanamimastery.com/episodes/14-dependency-injection-in-ruby-from-zero-to-hero-part-1)
* [Dependency Injection in Ruby - GOD Level! Meet dry-system! (Part 2)](https://hanamimastery.com/episodes/15-dependency-injection-god-level-part-2)
* [dry-auto_inject](https://dry-rb.org/gems/dry-auto_inject/1.0/)
* [dry-container](https://dry-rb.org/gems/dry-container/0.11/)
* [dry-system](https://dry-rb.org/gems/dry-system/1.0/)
* [dry-rails](https://dry-rb.org/gems/dry-rails/0.7/)

## Other supporting resources\

* [Matrix is now a separate gem](https://github.com/gdelugre/origami/issues/81)
* [Matrix Gem Error](https://qameta.com/posts/fixing-require-loaderror-as-example-for-matrix-gem)
* [Matrix cannot Load](https://stackoverflow.com/questions/71991514/rails-require-cannot-load-such-file-matrix)
* [Trix Field now displayed](https://stackoverflow.com/questions/73299862/action-text-rails-7-error-vanishing-field)
