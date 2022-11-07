---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Durable_rails_config"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ['Ruby']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2020-07-10T20:46:07+02:00
lastmod: 2020-07-10T20:46:07+02:00
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
# Rails Setup

Take from:
* https://gist.github.com/alxndr/7569551
* https://www.codewithjason.com/rails-integration-tests-rspec-capybara/
* https://hackernoon.com/how-to-build-awesome-integration-tests-with-capybara-j9333y68

create the project:
```
# -T - skips tests - I like using rspec
# -d postgresql - I like using postgresql best for the db
# Spring & listen speed testing - but can get out of sync and make problems
rails new challenges -T -d postgresql --skip-spring --skip-listen

Excerpt From: David Bryant Copeland. “Sustainable Web Development with Ruby on Rails.” Apple Books.
cd challenges
gem install bundler:2.1.4
rails app:update:bin
```

update the readme with purpose, setup and running tests
```
git add .
git commit -m "intial commit"
git remote add origin git@gitlab.com:btihen/challenges.git
git push -u origin master
```

add rspec, devise & factory_bot to Gemfile
```
cat <<EOF >> Gemfile
# btihen Gems
#################

# BACK END
##########
gem 'devise'

# DEV / TESTS
#############
group :development, :test do
  gem 'awesome_print'        # formats pry (& irb outputs into readable formats)

  gem 'pry-rails'
  gem 'pry-byebug'           # Adds byebug's step debugging and stack navigation
  # gem 'pry-debugger'       # adds step, continue, etc (alternative to pry-byebug)
  gem 'pry-stack_explorer'   # easy stack traces when debugging
  # more pry gems if needed at: https://spin.atomicobject.com/2012/08/06/live-and-let-pry/

  gem 'factory_bot_rails'
  gem 'faker'

  # gem 'rspec-rails'
  gem 'capybara'
  gem 'rspec-rails', '~> 4.0.0'

  # lets spring work with rspec - uncomment if using spring
  # gem 'spring-commands-rspec'
end

group :test do
  # easier tests (inside rspec)
  gem 'shoulda-matchers'

  # cucumber can test emails (rspec too?)
  # gem 'email_spec'

  # code coverage
  gem 'simplecov'
  gem 'simplecov-console'
end
EOF

# update gems with
bundle install

# install rspec with:
bin/rails g rspec:install
```
## CONFIGURE RSPEC

```
# create a spec features folder
mkdir spec/features
# create a place to put commonly used tests code
mkdir spec/support
mkdir spec/support/features

# at the top of `spec/rails_helper.rb below `require 'rspec/rails'` add:
# to enable integration tests with rspec & capybara
require 'capybara/rspec'
# include custom support files
Dir[Rails.root.join("spec/support/**/*.rb")].each { |file| require file }


# just after the ActiveRecord config and before RSpec.configure block add:
Capybara.register_driver :selenium_chrome do |app|
  Capybara::Selenium::Driver.new(app, browser: :chrome)
end
Capybara.javascript_driver = :selenium_chrome


# add FactoryBot config to Rspec in `spec/rails_helper.rb`
RSpec.configure do |config|
  # ...

  # support for Factory Bot
  config.include FactoryBot::Syntax::Methods

  # setup devise login helpers in Rspec
  config.include Devise::Test::IntegrationHelpers, type: :request
  config.include Warden::Test::Helpers
end

# support for shoulda matches
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end

EOF
```
the rspec_helper.rb file now looks like:
```
# This file is copied to spec/ when you run 'rails generate rspec:install'
require 'spec_helper'

ENV['RAILS_ENV'] ||= 'test'
require File.expand_path('../config/environment', __dir__)
# Prevent database truncation if the environment is production
abort("The Rails environment is running in production mode!") if Rails.env.production?
require 'rspec/rails'

# to enable integration tests with rspec
require 'capybara/rspec'

# include custom support files
Dir[Rails.root.join("spec/support/**/*.rb")].each { |file| require file }

# Add additional requires below this line. Rails is not loaded until this point!

# Requires supporting ruby files with custom matchers and macros, etc, in
# spec/support/ and its subdirectories. Files matching `spec/**/*_spec.rb` are
# run as spec files by default. This means that files in spec/support that end
# in _spec.rb will both be required and run as specs, causing the specs to be
# run twice. It is recommended that you do not name files matching this glob to
# end with _spec.rb. You can configure this pattern with the --pattern
# option on the command line or in ~/.rspec, .rspec or `.rspec-local`.
#
# The following line is provided for convenience purposes. It has the downside
# of increasing the boot-up time by auto-requiring all files in the support
# directory. Alternatively, in the individual `*_spec.rb` files, manually
# require only the support files necessary.
#
# Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f }

# Checks for pending migrations and applies them before tests are run.
# If you are not using ActiveRecord, you can remove these lines.
begin
  ActiveRecord::Migration.maintain_test_schema!
rescue ActiveRecord::PendingMigrationError => e
  puts e.to_s.strip
  exit 1
end

# configure capybara integration tests
Capybara.register_driver :selenium_chrome do |app|
  Capybara::Selenium::Driver.new(app, browser: :chrome)
end
Capybara.javascript_driver = :selenium_chrome

RSpec.configure do |config|
  # Remove this line if you're not using ActiveRecord or ActiveRecord fixtures
  config.fixture_path = "#{::Rails.root}/spec/fixtures"

  # If you're not using ActiveRecord, or you'd prefer not to run each of your
  # examples within a transaction, remove the following line or assign false
  # instead of true.
  config.use_transactional_fixtures = true

  # You can uncomment this line to turn off ActiveRecord support entirely.
  # config.use_active_record = false

  # RSpec Rails can automatically mix in different behaviours to your tests
  # based on their file location, for example enabling you to call `get` and
  # `post` in specs under `spec/controllers`.
  #
  # You can disable this behaviour by removing the line below, and instead
  # explicitly tag your specs with their type, e.g.:
  #
  #     RSpec.describe UsersController, type: :controller do
  #       # ...
  #     end
  #
  # The different available types are documented in the features, such as in
  # https://relishapp.com/rspec/rspec-rails/docs
  config.infer_spec_type_from_file_location!

  # Filter lines from Rails gems in backtraces.
  config.filter_rails_from_backtrace!
  # arbitrary gems may also be filtered via:
  # config.filter_gems_from_backtrace("gem name")

  # support for Factory Bot
  config.include FactoryBot::Syntax::Methods

  # setup devise login helpers in Rspec (login helpers)
  config.include Devise::Test::IntegrationHelpers, type: :request
  # allows us for force session logouts (im feature tests)
  config.include Warden::Test::Helpers
end

Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

If you plan to user database_cleaner -- then also see this article to finishy yoru config:

https://medium.com/@amliving/my-rails-rspec-set-up-6451269847f9

a basic login feature test might look like:
```
require 'rails_helper'

RSpec.describe 'Users Login', type: :feature do
  let(:user)  { FactoryBot.create :user }
  after :each do
    Warden.test_reset!
  end
  describe 'user logs in successfully' do
    scenario 'and is redirected to user home page' do
      user_log_in(user)
      expect(current_path).to eql(auth_user_root_path)
    end
  end
end
```

## Configure for ActionText and ActiveStorage

uncomment the following gem for Active Storage variant
```
gem 'image_processing', '~> 1.2'
# of course
bundle install
```

now install the components:
```
bundle exec rails webpacker:install
bundle exec rails webpacker:install:stimulus
bundle exec rails active_storage:install
bundle exec rails action_text:install
```

### Lets create a landing page - before we install and configure devise:
Since I don't use helpers (generally) nor contoller or view spec - I use thie following:
```
rails g controller Landing index --no-helper --no-assets --no-controller-specs --no-view-specs
```

now check or add this the root page to: `config/routes.rb`:
```

  get '/landing', to: 'landing#index', as: :landing
  # get 'landing/index'
  root to: "landing#index"
```

lets add a hidden paragraph as content to the top of the landing page so we can easily test for it.
```
<p hidden id='landing_index'>Landing Index</p>
```

lets create the following feature test (to test the feature setup):
```
# spec/features/landing_page_spec.rb
require 'rails_helper'

RSpec.describe 'Landing Page Works without a login', type: :feature do
  scenario 'Visit landing Page' do
    visit root_path

    page_tag = find('p#landing_index', text: 'Landing Index', visible: false)
    expect(page_tag).to be_truthy
  end
end
```

create a request test to be sure it works as we wish (we dont need a feature test when we arent navigating)
```
RSpec.describe "Landings", type: :request do

  describe "GET /index" do
    it "'/landing' returns http success" do
      get "/landing"
      expect(response).to have_http_status(:success)
      expect(response.body).to include("<p hidden id='landing_index'>Landing Index</p>")
    end
    it "'landing_path' returns http success" do
      get landing_path
      expect(response).to have_http_status(:success)
      expect(response.body).to include("<p hidden id='landing_index'>Landing Index</p>")
    end
    it "'/' returns http success" do
      get "/"
      expect(response).to have_http_status(:success)
      expect(response.body).to include("<p hidden id='landing_index'>Landing Index</p>")
    end
    it "'root_path' returns http success" do
      get root_path
      expect(response).to have_http_status(:success)
      expect(response.body).to include("<p hidden id='landing_index'>Landing Index</p>")
    end
    # it "'/landing/index' returns http success" do
    #   get "/landing/index"
    #   expect(response).to have_http_status(:success)
    #   expect(response.body).to include("<p hidden id='landing_index'>Landing Index</p>")
    # end
    # it "'landing_index_path' returns http success" do
    #   get landing_index_path
    #   expect(response).to have_http_status(:success)
    #   expect(response.body).to include("<p hidden id='landing_index'>Landing Index</p>")
    # end
  end

end
```

Assumign all it green - lets commit:
```
git add .
git commit -m "rspec configured and working"
git push
```
