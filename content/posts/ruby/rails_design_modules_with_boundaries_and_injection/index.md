---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails Design: Protected Modules with Injection"
subtitle: "Long-term Manageable Rails through Low Dependency/Entanglement and Loose Coupling"
summary: ""
authors: ['btihen']
tags: ['Ruby', 'Rails', 'Modules', 'Isolation', 'Injection']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2023-08-12T01:20:00+02:00
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

## Configure for Modules / Packages

Let's start by create a modular structure for our app as described in [Rails with Protected Modules](https://btihen.dev/posts/ruby/rails_protected_modules/) - although this is not required.

I generally put everything in either `app/packages`, `app/modules`, `packages` or `modules` - but you can choose your preference. (I tend to use 'packages; when using packwerk).

Configure Rails to find our modules:

```ruby
# config/application.rb
require_relative "boot"
# ...
module ModulesDesign
  class Application < Rails::Application
    # Initialize configuration defaults for originally generated Rails version.
    config.load_defaults 7.1

    # add packages PATH BEFORE autoload config!
    # otherwise it will not find the modules with the error:
    # ActionController::RoutingError (uninitialized constant Landing)
    config.paths.add 'app/packages', glob: '*/{*,*/concerns}', eager_load: true

    config.autoload_lib(ignore: %w(assets tasks))
    # ...
    # Don't generate system test files.
    config.generators.system_tests = nil
  end
end
```

Configure Rails to find our module views:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  # find views within our modules
  append_view_path(Dir.glob(Rails.root.join('app/packages/*/views')))
end
```

Move the Rails code into its own directory (in this case `app/packages/rails_shim` but you can choose your preference):

```bash
mkdir -p app/packages/rails_shim

# move the rails code into the rails module
mv app/* app/packages/rails_shim/.

# we need to return assets and javascript to the app directory
mv app/packages/rails_shim/assets app/assets
mv app/packages/rails_shim/javascript app/javascript
```

Test with `bin/dev` that it all still works and then commit:

```bash
git add .
git commit -m "move rails code into packages"
```

## Landing Page

Let's create a simple landing page to show that our app is working.

```bash
bin/rails g controller landing/home index --no-helper
```

move into a package
```bash
mkdir app/packages/landing
mv app/views app/packages/landing/.
mv app/controllers app/packages/landing/.
```
configre the homepage route
```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :landing do
    get 'home/index'
  end

  # Defines the root path route ("/")
  root 'landing/home#index'
end
```

add to repo now that we have the basics working & tested:
```bash
git add .
git commit -m "add landing page package"
```

## Create Blog Articles

Now that we have a modularized app working, let's add a simple article (blog post) feature.

Let's make an Article model within the **Articles** namespace (article content will use Action Text so we can capture the person's writing) which we can then display as an HTML page or save as a PDF file.

first we will install the required tools:

```bash
bin/rails action_text:install
bin/rails db:migrate

# move the active storage code into the rails module
mv app/views/active_storage app/packages/rails_shim/views/.
mv app/views/layouts/action_text app/packages/rails_shim/views/layouts/.
```

Let's generate the Articles scaffold (with a namespace to keep this separate from other features):

```bash
# content will use Action Text so we can format the writing
bin/rails g scaffold Articles::Post author:string title:string --no-helper
bin/rails db:migrate
```

move the new code into its module:

```bash
mkdir -p app/packages/articles
mv app/views app/packages/articles/.
mv app/models app/packages/articles/
mv app/controllers app/packages/articles/.
```

Now we need to add the ActionText field to the Article model:

```ruby
# app/packages/articles/models/articles/post.rb
module Articles
  class Post < ApplicationRecord
    has_rich_text :content
  end
end
```

We need to update our controller to permit the new `content` field:

```ruby
# app/packages/articles/controllers/articles/posts_controller.rb
  ...
  private
  ...
  # Only allow a list of trusted parameters through.
  def articles_post_params
    params.require(:articles_post).permit(:author, :title, :content)
  end
end
```

We will also need to add `<%= form.rich_text_area :content %>` to the Article form :

```erb
# app/packages/articles/views/articles/posts/_form.html.erb
  ...
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
# app/packages/articles/views/articles/posts/_post.html.erb
  ...

  <p>
    <strong>Title:</strong>
    <%= post.title %>
  </p>

  <!-- use div since rich text can generate html beyond a p tag-->
  <div>
    <%= post.content %>
  </div>
</div>
```

Now let's test by going to:
```bash
# NOTE: with the tradionitional `bin/rails s` the Action Text (Trix) field will not display so use!
# https://stackoverflow.com/questions/73299862/action-text-rails-7-error-vanishing-field
bin/dev

# open our browser and add a new article with some formatting:
open http://localhost:3000/articles/posts/new
```

cool - let's test and add this to our repo:

```bash
git add .
git commit -m "add Article::Post with ActionText"
```

## Simple PDF Generation

Let's start by installing the necessary tooling:

```bash
# `matrix` was part of ruby, but now its a separate gem (and pawn-html does not load it automatcally)
bundle add 'matrix'
bundle add 'prawn-html'
# for some reason I had to run this to fix a bundler conflict:
bundle install --gemfile Gemfile
# the error was:
# `You specified: image_processing (~> 1.2) and image_processing (>= 0). Gem already added.`
```

we need to extend our Post model to allow for a PDF file to be attached:

```ruby
module Articles
  class Post < ApplicationRecord
    has_rich_text :content
  end
end
```

let's create a simple PDF generator:

```ruby
# app/packages/articles/services/articles/pdf_generator.rb
require 'prawn'
require 'prawn-html'

module Articles
  class PdfGenerator
    def self.generate(model)
      pdf = Prawn::Document.new

      # Add title to the PDF
      pdf.text model.title, size: 24, style: :bold

      # Add author to the PDF
      pdf.text "by: #{model.author}", size: 16, style: :italic

      # Add model content to the PDF using PrawnHtml
      PrawnHtml.append_html(pdf, model.content.to_s)

      # Return the generated PDF
      pdf.render
    end
  end
end
```

Lets add a route to generate the PDF:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # ...
  namespace :articles do
    resources :posts do
      get :download_pdf, on: :member
    end
  end
  # ... other routes
end
```

Lets add the action to the controller (lets also nest PostController under Articles namespace):

```ruby
# app/packages/articles/controllers/articles/posts_controller.rb
module Articles
  class PostsController < ApplicationController
    before_action :set_articles_post, only: %i[show edit update destroy generate_pdf download_pdf]

    def download_pdf
      rendered_pdf = PdfGenerator.generate(@articles_post)

      if rendered_pdf
        formated_time = DateTime.now.strftime('%Y%m%d_%H%M%S')
        filename = "post_#{@articles_post.id}_#{formated_time}.pdf"
        type = 'application/pdf'

        send_data(rendered_pdf, filename:, type:)
      else
        redirect_to articles_post_url(@articles_post), alert: 'OOPS - NO PDF was generation.'
      end
    end
    # ... (other actions and methods)
  end
end
```

Now add the generate PDF link to the show view with:
`<%= link_to 'Generate PDF', generate_pdf_articles_post_path(@articles_post), method: :get %>`

```erb
# app/packages/articles/views/articles/posts/show.html.erb
<p style="color: green"><%= notice %></p>

<%= render @articles_post %>

<div>
  <br>
  <%= link_to 'Download as PDF', download_pdf_articles_post_path(@articles_post), method: :get %>
  <br>
  <%= link_to "Edit this post", edit_articles_post_path(@articles_post) %> |
  <%= link_to "Back to posts", articles_posts_path %>
  <%= button_to "Destroy this post", @articles_post, method: :delete %>
</div>
```

## Create an Archive Package

Let's start by creating a new its folder structure:

```bash
mkdir -p app/packages/archive
```

### Define Wrapper Needs

Probably the module should handle construction _(this allows us to use multiple adapters - perhaps each author is responsbile for their own archive service)_.
And the Adapter wrapper should handle the all other aspects of the API _(the interactions with the remote service)_.

What we need the API to do:

1. **constructor** (we want to be able to use multiple remote archive services)
2. **initialize** (desired Adapter from config & authenticate with remote archive service)
3. **archive_file** (on success report needed information, on error report nil)
4. **retrieve_file** (on success report file location/path, on error report nil)
5. **errors** (report any errors)

### Define Configuration

Before we look at the Constructor, let's consider what configuration would look like.

So if we want to start with a simple mock adapter (good for testing and perhaps a service we can show to unpaid users to see what it might be like), we could do something like the following since we don't need to authenticate, nor any other particular infomation, just the configuration name and the adapter class name:

```
mkdir -p app/packages/archive/config
touch app/packages/archive/config/archive.yml
vim app/packages/archive/config/archive.yml
```


```yaml
---
archive_service:
- adapter_config: Tester
  adapter_klass: TestAdapter
```

Reviewing the [TeedyAPI Docs](https://demo.teedy.io/apidoc/) and with a little experience, we can see that we need to provide the following information:

```yaml
- adapter_config: DemoAuthor
  adapter_klass: TeedyAdapter
  url: http://localhost:8080
  archive_path: /archive
  username: demo
  password: password

- adapter_config: AdminAuthor
  adapter_klass: TeedyAdapter
  url: http://localhost:8080
  archive_path: /archive
  username: admin
  password: admin
```

### Module Constructor

Let's see if we can define the API for the module constructor.

1. we need to find a configuration that matches the name provided
2. we need to find the adapter class that matches the name provided
3. we need to instantiate the adapter class with the configuration provided
4. if needed we need to authenticate with the remote service (& fetch any necessay configs / tokens, etc)
5. we need to return the adapter instance (or nil if there was an error)?

```bash
touch app/packages/archive/archive.rb
```

```ruby
# app/packages/archive/archive.rb
require 'yaml'

module Archive
  def self.adapter_for(config_name, config_file = 'app/packages/archive/config/archive.yml')
    config_file_path = Rails.root.join(config_file)
    adapter_configs = YAML.load_file(config_file_path)['archive']
    vendor_config = adapter_configs.find { |config| config['adapter_config'] == config_name }
    vendor_klass = adapter_config[:adapter_klass].constantize
    vendor_adapter = vendor_klass.new(vendor_config)
    Adapter.new(vendor_adapter)
  end
end
```

### Define Adapter

Let's start by creating a simple mock adapter wrapper:

```bash
mkdir app/packages/archive/adapters
touch app/packages/archive/adapters/adapter.rb
touch app/packages/archive/adapters/test_adapter.rb
```

```ruby
# app/packages/archive/adapters/adapter.rb
module Archive
  module Adapters
    class AdapterError < StandardError; end

    class Adapter
      attr_reader :vendor_adapter
      private     :vendor_adapter

      # we could delegate all the methods to the vendor_adapter - with:
      # `delegate :archive_file, :retrieve_file, :errors, :errors? to: :remote_adapter`
      # but I like to explicity force the API to be the same for all adapters and have
      # one location to trap and handle errors consistently for all adapters

      def initialize(vendor_adapter)
        @vendor_adapter = vendor_adapter

        raise AdapterError, vendor_adapter.errors if vendor_adapter.errors?
      end

      # most document services have need the file_path, mime_type, and some type of metadata to describe the file conext and contents
      def archive_file(model, file_path, mime_type, options: {})
        metadata = {
          model: model.class.name,
          model_id: model.id,
          created_at: model.created_at,
          updated_at: model.updated_at
        }
        response = vendor_adapter.archive_file(file_path, mime_type, metadata)
        ArchiveSend.create!(model:, options:, metadata:,
                            remote_key: response[:remote_key],
                            container_key: response[:container_key])
        return response if response && !response.errors?

        raise AdapterError, remote_adapter.errors
      end

      # most document services have some type of 'container' concept (Teedy calls this a document)
      # to track multiple versions of a given file
      def retrieve_file(file_key, container_key = nil)
        response = vendor_adapter.retrieve_file(remote_keys, container_key)
        return response.to_s if response

        raise AdapterError, vendor_adapter.errors
      end
    end

    private_constant :Adapter
  end
end
```

### Define Mock Adapter

```ruby
# app/packages/archive/adapters/test_adapter.rb
require 'prawn-html'

module Archive
  module Adapters
    class TestAdapter
      attr_reader :errors
      private     :errors

      def initialize(_config)
        @errors = []
      end

      def archive_file(file_path, mime_type, metadata)
        metadata
      end

      def retrieve_file(remote_keys, container_key)
        file_path = Rails.root.join('tmp', 'test.pdf')
        pdf = Prawn::Document.new(page_size: 'A4')
        PrawnHtml.append_html(pdf, '<h1 style="text-align: center">Just a test</h1>')
        pdf.render_file(rails.root.join('tmp', 'test.pdf')
        file_path
      end

      def errors? = errors.any?
    end
  end
end
```


### REAL Implimentations

Now you can make any adapter that implement `archive_file(model, file_path, mime_type, options: {})`,  `retrieve_file(remote_keys, container_key)`, `.errors?` and `errors`

You now have a blox module with a consistent and clearly defined API & error handling and the rest of your app accesses it via the module methods!

More upfront work, but prrevents coupling and makes it easy to add new adapters and enforces consistency.

## Experience - Standard vs Modular

My colleague [Jörg Jenni](https://www.linkedin.com/in/jorgjenni/) - created a little app [messy vs modular](https://github.com/Enceradeira/messy_app) - that need needs new features.  It has 2 base Apps `messy` and `modulariyed`.  Its a great way to experience the benefits of modular design.

My solution can be seen at: https://github.com/btihen/coupled_vs_modular


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
