---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Creating the Rails Age Plugin"
subtitle: "An explanation of somewhat confusing process of creating a rails engine"
summary: ""
authors: ['btihen']
tags: ['Rails', "GraphDB", "Postgres AGE", "Rails Engine", "Rails Plugin", "Rails Gem"]
categories: ["Code", "GraphDB", "Ruby Language", "Rails Framework"]
date: 2024-05-21T01:20:00+02:00
lastmod: 2024-05-22T01:20:00+02:00
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

Creating a simple Rails Engine despite the variety of documentation was confusing, so here is what I learned.

It's important to set up a dummy app from the beginning (to enable testing)

So after several starts I found the following worked to start the project (in my case the database must be `postgresql`):
```
rails plugin new rails_age -d=postgresql -T --mountable \
      --dummy-path=spec/dummy
```

I like to use rspec so I also used the `-T` option.

Now to add rspec I added:
`spec.add_development_dependency 'rspec-rails'`
to the `gemspec` file.

You need to update the rest of the `gemspec` too - so now mine looks like:
```ruby
# rails_age.gemspec
require_relative "lib/rails_age/version"

Gem::Specification.new do |spec|
  spec.name        = "rails_age"
  spec.version     = RailsAge::VERSION
  spec.authors     = ["Bill Tihen"]
  spec.email       = ["btihen@gmail.com"]
  spec.homepage    = "https://github.com/marpori/rails_age"
  spec.summary     = "Apache AGE plugin for Rails 7.1"
  spec.description = spec.summary
  spec.license     = "MIT"

  spec.metadata["homepage_uri"] = spec.homepage
  spec.metadata["source_code_uri"] = spec.homepage
  spec.metadata["changelog_uri"] = "#{spec.homepage}/blob/main/CHANGELOG.md"

  spec.files = Dir.chdir(File.expand_path(__dir__)) do
    Dir["{app,config,db,lib}/**/*", "MIT-LICENSE", "Rakefile", "README.md", "CHANGELOG.md"]
  end

  spec.add_dependency "rails", ">= 7.1.3.2"

  spec.add_development_dependency 'rspec-rails'
end
```

Now run:
```bash
bundle install
rails generate rspec:install
bundle binstubs rspec-core
```

I wanted to create test files in my dummy app and in the gem so the first thing to do is copy the `rails_helper.rb` and `spec_helper.rb` into the dummy app.

```bash
mkdir spec/dummy/spec
cp spec/rails_helper.rb spec/dummy/spec/.
cp spec/spec_helper.rb spec/dummy/spec/.
```

Now it is important to change our original file `spec/rails_helper.rb` to point to the dummy environments (our gem / plugin has no such file), so we change the line:
`require_relative '../config/environment'`
to
`require File.expand_path('../dummy/config/environment', __FILE__)`

In this way our test environment can find an environment - now this file looks like:
```ruby
# spec/rails_helper.rb

require 'spec_helper'
ENV['RAILS_ENV'] ||= 'test'
# require_relative '../config/environment'
require File.expand_path('../dummy/config/environment', __FILE__)
abort("The Rails environment is running in production mode!") if Rails.env.production?
require 'rspec/rails'

# Checks for pending migrations and applies them before tests are run.
# If you are not using ActiveRecord, you can remove these lines.
begin
  ActiveRecord::Migration.maintain_test_schema!
rescue ActiveRecord::PendingMigrationError => e
  abort e.to_s.strip
end
RSpec.configure do |config|
  # Remove this line if you're not using ActiveRecord or ActiveRecord fixtures
  config.fixture_paths = [
    Rails.root.join('spec/fixtures')
  ]

  # If you're not using ActiveRecord, or you'd prefer not to run each of your
  # examples within a transaction, remove the following line or assign false
  config.use_transactional_fixtures = true

  # You can uncomment this line to turn off ActiveRecord support entirely.
  # config.use_active_record = false

  # RSpec Rails can automatically mix in different behaviours to your testsx
  # The different available types are documented in the features, such as in
  # https://rspec.info/features/6-0/rspec-rails
  config.infer_spec_type_from_file_location!

  # Filter lines from Rails gems in backtraces.
  config.filter_rails_from_backtrace!
end
```

now in the `gem` root directory I create a migration to configure the database for ApacheAge.
```
rails generate migration ConfigureApacheAge
```

The migration looks like:
```ruby
class ConfigureApacheAge < ActiveRecord::Migration[7.1]
  def up
    # Allow age extension
    execute('CREATE EXTENSION IF NOT EXISTS age;')

    # Load the age code
    execute("LOAD 'age';")

    # Load the ag_catalog into the search path
    execute('SET search_path = ag_catalog, "$user", public;')

    # Create age_schema graph if it doesn't exist
    execute("SELECT create_graph('age_schema');")
  end

  def down
    execute <<-SQL
      DO $$
      BEGIN
        IF EXISTS (
          SELECT 1
          FROM pg_constraint
          WHERE conname = 'fk_graph_oid'
        ) THEN
          ALTER TABLE ag_catalog.ag_label
          DROP CONSTRAINT fk_graph_oid;
        END IF;
      END $$;
    SQL

    execute("SELECT drop_graph('age_schema', true);")
    execute('DROP SCHEMA IF EXISTS ag_catalog CASCADE;')
    execute('DROP EXTENSION IF EXISTS age;')
  end
end
```

Now I can finally setup the database:
```bash
bin/rails db:create RAILS_ENV=test
bin/rails db:migrate RAILS_ENV=test
```

In my case, I need to write the schema myself as rails doesn't handle the Apache Age extension changes properly. So I needed to rewrite the schema as:
```ruby
ActiveRecord::Schema[7.1].define(version: 2024_05_21_062349) do
  # These are extensions that must be enabled in order to support this database
  enable_extension "plpgsql"

  # Allow age extension
  execute('CREATE EXTENSION IF NOT EXISTS age;')

  # Load the age code
  execute("LOAD 'age';")

  # Load the ag_catalog into the search path
  execute('SET search_path = ag_catalog, "$user", public;')

  # Create age_schema graph if it doesn't exist
  execute("SELECT create_graph('age_schema');")
end
```
**NOTE:** the version number MUST match the migration filename - in my case the file is: `db/migrate/20240521062349_configure_apache_age.rb` so the version needs to be: `2024_05_21_062349`!

I used (for better or worse a different namespace for my code as the gem)
```
mkdir lib/apache_age
# code files:
touch lib/apache_age/class_methods.rb
touch lib/apache_age/common_methods.rb
touch lib/apache_age/edge.rb
touch lib/apache_age/entity.rb
touch lib/apache_age/vertex.rb
```

Now it is important to update the file `lib/rails_age.rb` with all the new files:
```ruby
# lib/rails_age.rb
require "rails_age/version"
require "rails_age/engine"

module RailsAge
  # Additional code goes here...
end

module ApacheAge
  require "apache_age/class_methods"
  require "apache_age/common_methods"
  require "apache_age/edge"
  require "apache_age/entity"
  require "apache_age/vertex"
end
```

Now I created my spec tests for the gem:
```bash
mkdir -p spec/lib/apache_age

touch spec/lib/apache_age/class_methods_spec.rb
touch spec/lib/apache_age/common_methods_spec.rb
touch spec/lib/apache_age/edge_spec.rb
touch spec/lib/apache_age/entity_spec.rb
touch spec/lib/apache_age/vertex_spec.rb
```

Now I can test with:
`bundle exec rspec spec`
or better yet using the binstub:
`bin/rspec spec`

## Optional Dummy App Testing

Now if you want you can go into `spec/dummy` and write / execute tests within the dummy rails app:

Add the Graph App files:
```bash
mkdir -p spec/dummy/app/graphs/edges
mkdir -p spec/dummy/app/graphs/nodes

spec/dummy/app/graphs/edges/works_at.rb
spec/dummy/app/graphs/nodes/company.rb
spec/dummy/app/graphs/nodes/person.rb
```

generate a controller and views:
```bash
bin/rails g scaffold_controller Person
```

Now we can add tests:
```bash
mkdir -p spec/dummy/spec/graphs/edges
mkdir -p spec/dummy/spec/graphs/nodes

spec/dummy/spec/graphs/edges/works_at_spec.rb
spec/dummy/spec/graphs/nodes/company_spec.rb
spec/dummy/spec/graphs/nodes/person_spec.rb
```

We need to run update the dummy app `schema.rb` with the same data as in the gem `db/schema.rb` so do:
```bash
cp db/schema.rb spec/dummy/db/schema.rb
```

now within the dummy app we can run our test:
```bash
spec/dummy/
bundle exec rspec spec
```

cool it works within a dummy app too!

## Gem Usage Testing
assuming the tests are green, then we can build the gem with:
```bash
gem build rails_age.gemspec
```

add `*.gem` to the end of `.gitignore` file.

Now we can build a new project and try out our gem:
```bash
rails new graphdb_age_app -T -d=postgresql
```

now at the end of the `Gemfile` add:
```ruby
# Gemfile
gem 'rails_age', path: '../rails_age'
```
and of course run `bundle` and test the new app with the plugin.

## Gem Repo Publishing & Usage

Assuming this works, we can now push the repo live (first make a repo on github) in my case at:
`https://github.com/marpori/rails_age`

then:

```bash
git add .
git commit -m "initial Rails Engine Plugin Commit"
git remote add origin git@github.com:marpori/rails_age.git
git branch -M main
git push -u origin main
```

Now we can change how we access the gem using:
```ruby
# Gemfile
gem 'rails_age', git: 'https://github.com/marpori/rails_age.git'
```
and of course run `bundle` and test.

## RubyGem Publishing & Usage

Assuming this still works well, we can publish the gem using - assuming you already have an account on [rubygems](https://rubygems.org/)

```bash
# be sure to update the version number before building a new version!
gem build rails_age.gemspec
gem signin
gem push rails_age-0.1.0.gem
```

Now the gem should also be usable with just
`gem 'rails_age'`
in the `Gemfile`:
```ruby
# Gemfile
gem 'rails_age'
```
to the `Gemfile` & of course running: `bundle` all should still work and be published on rubygems at: `https://rubygems.org/gems/rails_age`
