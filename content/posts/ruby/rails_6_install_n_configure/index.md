---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Install and Configure Rails"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ['Ruby', "configure", "install", "durable", "testing"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2020-09-10T01:46:07+02:00
lastmod: 2021-08-07T01:46:07+02:00
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
# Intro

To document is mostly for me -- at least until I automate my setup defaults. However, I am glad to share and get ideas from others too.  I will build a little calendar app I use with friends (it's focused on being mobile friendly and easy to use -- not a full featured calendar).

# Rails Setup

Taken from:
* https://gist.github.com/alxndr/7569551
* https://www.codewithjason.com/rails-integration-tests-rspec-capybara/
* https://hackernoon.com/how-to-build-awesome-integration-tests-with-capybara-j9333y68

## create the project:
```bash
# -T - skips tests;              I like rspec
# -d postgresql;                 I like postgresql best for the db
# --skip-spring --skip-listen;   Spring caches and doesn't notice all changes (even after rails restart)
#                                I have lost several hours not realizing Spring wasn't seeing my changes

rails new calendar -T -d postgresql --webpack=stimulus --skip-turbolinks --skip-spring

cd calendar

# in some cases you may have serveral bundlers or need to create binstubs
# gem install bundler:2.1.4
# rails app:update:bin
```

## update the README and initialize Git
```bash
git add .
git commit -m "initial commit"
git remote add origin git@gitlab.com:btihen/calendar.git
git push -u origin master
```

## Add extra Gems for this project
add rspec, devise, factory_bot and stimulus_reflex

Execute the following command (or add to the Gemfile)
```ruby
cat <<EOF >> Gemfile
# Project Gems
##############

# FRONT END
###########
gem "hotwire-rails"                # probably not needed as of Rails 7.x
# gem "stimulus_reflex", "~> 3.3"  # probably superseeded by hotwire-rails

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

  # lets spring work with rspec
  gem 'spring-commands-rspec'
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
```

## Now uncomment a few Gems in the Original Gemfile

Uncomment the following to ensure ActionText and Stimulus Refelx (work properly).

`gem 'image_processing', '~> 1.2'`

is needed by Active Storage (ActionText needs Active Storage)

and

`gem 'redis', '~> 4.0'`

is needed by Stimulus Reflex (which uses Action Channels) to manage WebSockets

## Install and configure base gems

now run:

`bundle install`

to install all the new gems and create a `Gemfile.lock`


## Install ActiveStorage and ActionText

run the following commands:
```bash
# bundle exec rails webpacker:install
# bundle exec rails webpacker:install:stimulus
bundle exec rails active_storage:install
bundle exec rails action_text:install
bin/rails hotwire:install
bin/rails g devise:install
bin/rails g rspec:install
```

## Rspec: Config Files

### Create needed folders for our config

```bash
mkdir spec/features

# a place to put test helper code
mkdir spec/support
mkdir spec/support/features
```

### Rspec Config file `spec/rails_helper.rb`

1. To enable integration tests with rspec add: `require 'capybara/rspec'` below `require 'rspec/rails'`
2. To load Test helper code add: `Dir[Rails.root.join("spec/support/**/*.rb")].each { |file| require file }` below `require 'capybara/rspec'`
3. just after the ActiveRecord config and before RSpec.configure block add:
```ruby
Capybara.register_driver :selenium_chrome do |app|
  Capybara::Selenium::Driver.new(app, browser: :chrome)
end
Capybara.javascript_driver = :selenium_chrome
```
4. Add the FactoryBot config in the section with:
```ruby
RSpec.configure do |config|
  # ...

  # support for Factory Bot
  config.include FactoryBot::Syntax::Methods

  # setup devise login helpers in Rspec
  config.include Devise::Test::IntegrationHelpers, type: :request

  # allows us for force session logouts (im feature tests)
  config.include Warden::Test::Helpers
end
```
5. finally at the end of the file add support for shoulda matchers with:
```ruby
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

NOW `spec/rails_helper.rb` should look like (its long, sometimes the full context is clearer):
```ruby
# This file is copied to spec/ when you run 'rails generate rspec:install'
require 'spec_helper'
ENV['RAILS_ENV'] ||= 'test'
require File.expand_path('../config/environment', __dir__)
# Prevent database truncation if the environment is production
abort("The Rails environment is running in production mode!") if Rails.env.production?
require 'rspec/rails'
# Add additional requires below this line. Rails is not loaded until this point!

# enables integration/feature tests using rspec
require 'capybara/rspec'

# loads custom helper test code
Dir[Rails.root.join("spec/support/**/*.rb")].each { |file| require file }
# or you could use:
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

# Create / Test a landing page

A simple config test before we setup devise (authentication).

1. **Generate a page** -- I don't (generally) use helpers nor contoller or view specs - so I'll create the landing page using the following generator:
```bash
rails g controller Landing index --no-helper --no-assets --no-controller-specs --no-view-specs
```
2. **Update Routes** `config/routes.rb` with:
```ruby
  get 'landing/index'
  root to: "landing#index"
```
3. **Add Hidden Test Content** to simplify testing add:
```ruby
<p hidden id='landing_index'>Landing Index</p>
```
4. Request test:
```ruby
# spec/requests/landing_request_spec.rb
require 'rails_helper'

RSpec.describe "Landings", type: :request do

  describe "GET /index" do
    it "returns http success" do
      get "/landing/index"
      expect(response).to have_http_status(:success)

      expect(response.body).to include("<p hidden id='landing_index'>Landing Index</p>")
    end
  end
end
```
5. Feature Test (to be sure they are working too)
```ruby
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

Test and commit
```bash
rake db:migrate
bundle exec rspec
git add .
git commit -m "rspec: unit and feature tests configured and landing page works"
git push
```


### Config Hotwire
Ensure the  In the end the `app/views/layouts/application.html.erb` looks like:
```ruby
<!DOCTYPE html>
<html>

<head>
  <title>Tweets</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>

  <!-- Bootstrap 4 if interested
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.6.0/dist/css/bootstrap.min.css" integrity="sha384-B0vP5xmATw1+K9KRQjQERJvTumQW0nPEzvF6L/Z6nronJ3oUOFUFpCjEUQouq2+l" crossorigin="anonymous">
  -->
  <%= stylesheet_link_tag 'application', media: 'all' %>
  <%= javascript_pack_tag 'application' %>
  <%= yield :head %>
  <%= turbo_include_tags %>
  <%# stimulus_include_tags %>
</head>

<body>
  <p class="notice"><%= notice %></p>
  <p class="alert"><%= alert %></p>
  <%= yield %>
</body>

</html>
```

### Devise / User Config

Configure dev email for devise:
```ruby
# config/environments/development.rb:
  config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
```

Create the user and migration
```bash
rails g devise user
# if you will make a custom login (probably needed to look nice)
# rails g devise:views
```

Adjust the migration:
```ruby
class DeviseCreateUsers < ActiveRecord::Migration[6.1]
  def change
    create_table :users do |t|
      ## Database authenticatable
      t.string :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""

      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Rememberable
      t.datetime :remember_created_at

      ## Trackable
      t.integer  :sign_in_count, default: 0, null: false
      t.datetime :current_sign_in_at
      t.datetime :last_sign_in_at
      t.string   :current_sign_in_ip
      t.string   :last_sign_in_ip

      ## Confirmable
      # t.string   :confirmation_token
      # t.datetime :confirmed_at
      # t.datetime :confirmation_sent_at
      # t.string   :unconfirmed_email # Only if using reconfirmable

      ## Lockable
      # t.integer  :failed_attempts, default: 0, null: false # Only if lock strategy is :failed_attempts
      # t.string   :unlock_token # Only if unlock strategy is :email or :both
      # t.datetime :locked_at

      t.timestamps null: false
    end

    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    # add_index :users, :confirmation_token,   unique: true
    # add_index :users, :unlock_token,         unique: true
  end
end
```


Route file should now look like:
```ruby
Rails.application.routes.draw do
  devise_for :users
  get 'landing/index'
  root to: "landing#index"
end
```

We will update the user model with password complexity validation:
```ruby
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  validate :password_complexity

  def password_complexity
    # Regexp extracted from https://stackoverflow.com/questions/19605150/regex-for-password-must-contain-at-least-eight-characters-at-least-one-number-a
    return if password.blank? || password =~ /^(?=.*?[A-Z])(?=.*?[a-z])(?=.*?[0-9])(?=.*?[#?!@$%^&*-]).{10,70}$/

    errors.add :password, 'Complexity requirement not met. Length should be 10-70 characters and include: 1 uppercase, 1 lowercase, 1 digit and 1 special character'
  end
end
```

Create the user Factory (which also uses Faker):
```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    email       { Faker::Internet.safe_email }  # probably need to add index for uniqueness
    password    { Faker::Internet.password(min_length: 10, max_length: 50, mix_case: true, special_characters: true) }
  end
  trait :invalid do
    email       { Faker::Internet.username }
    password    { "hoi" }
  end
end
```

Create the user spec (uses FactoryBot & Shoulda):
```ruby
# spec/models/user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  describe "Factory with" do

    context "default parameters" do
      it "creates a valid model" do
        user = FactoryBot.build :user
        expect(user.valid?).to be_truthy
      end
    end

    context "invalid parameters" do
      it "fails model validation" do
        user = FactoryBot.build :user, :invalid
        expect(user.valid?).to be_falsey
      end
    end
  end

  context "ActiveRecord / DB Tests" do
    it { should have_db_column(:email) }
    it { should have_db_index(:email).unique }
  end

  context "ActiveModel / Validations" do
    it "detects a bad email" do
      user = FactoryBot.build :user, email: "bill"
      expect(user.valid?).to be_falsey
      expect(user.errors.messages[:email]).to match_array ["is invalid"]
    end
    it "detects a non-compliant password" do
      user = FactoryBot.build :user, password: "hoi"
      expect(user.valid?).to be_falsey
      expect(user.errors.messages[:password]).to match_array ["is too short (minimum is 6 characters)",
                                                              "Complexity requirement not met. Length should be 10-70 characters and include: 1 uppercase, 1 lowercase, 1 digit and 1 special character"]
    end
  end

end
```

### Test setup and commit when green:
```bash
rake db:migrate
bundle exec rspec
git add .
git commit -m "devise configured, FactoryBot, Faker and Shoulda working"
git push
```

## create user landing / profile page (autoredirect)

## Test restricted logins
a basic login feature test might look like:
```ruby
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

## Install Fonts / Icons

### Fontawesome (Good with Bulma)

https://kelishrestha.medium.com/how-to-install-font-awesome-with-yarn-in-rails-6-0-c2506543c13d

```bash
yarn add @fortawesome/fontawesome-free
```

update application.scss
```css
$fa-font-path: '@fortawesome/fontawesome-free/webfonts';
@import '@fortawesome/fontawesome-free/scss/fontawesome';
@import '@fortawesome/fontawesome-free/scss/solid';
@import '@fortawesome/fontawesome-free/scss/regular';
@import '@fortawesome/fontawesome-free/scss/brands';
@import '@fortawesome/fontawesome-free/scss/v4-shims';
```

update application.js
```css
import "@fortawesome/fontawesome-free/js/all";
```
or via cdn: vhttps://fontawesome.com/how-to-use/customizing-wordpress/snippets/setup-cdn-webfont
add to `application.html.erb` ()
```html
<meta name="viewport" content="width=device-width, initial-scale=1">
<link rel="stylesheet"
      href="https://pro.fontawesome.com/releases/v5.10.0/css/all.css"
      integrity="sha384-AYmEC3Yw5cVb3ZcuHtOA93w35dYTsvhLPVnYs9eStHfGJvOvKxVfELGroGkvsg+p"
      crossorigin="anonymous"/>
```

## Install BULMA: a CSS Framework (if desired)

Bulma is a relatively new CSS framework. It feels like a light, streamlined alternative to Bootstrap. Bulma doesn’t include any JavaScript at all. This means some stuff just won’t work out of the box. For example, the burger menu won’t toggle without a little JavaScript help. We’ll get to that later.
```bash
yarn add bulma
```

Open app/javascript/packs/application.js and add the following to the top:
```javascript
import '../styles'
```

Create app/javascript/styles.scss:
```css
@import '~bulma/bulma';
```

customize bulma by adding to the top of `styles.scss` file: https://stackoverflow.com/questions/48809328/bulma-navbar-breakpoint
```css
@import "~bulma/sass/utilities/initial-variables.sass";
$navbar-breakpoint: $tablet;
@import "~bulma/bulma.sass";
@import '~bulma/bulma';
```

choices are: $desktop (default 960px), $tablet (769px), $widescreen (1152px), $fullhd (1344px)
variable defaults: https://bulma.io/documentation/customize/variables/
variables that can be set: https://bulma-customizer.bstash.io

### A sample Bulma navbar

Open app/views/layouts/application.html.erb and add the following just above the yield line:
```ruby
<%= render 'layouts/navbar' %>
```

Create app/views/layouts/_navbar.html.erb:
```ruby
<div class="container">
  <nav class="navbar">
    <div class="navbar-brand">
      <a class="navbar-item" href="#">
        <img src="https://bulma.io/images/bulma-logo.png" width="112" height="28">
      </a>
      <div class="navbar-burger burger" data-target="main-nav">
        <span></span>
        <span></span>
        <span></span>
      </div>
    </div>

    <div id="main-nav" class="navbar-menu">
      <div class="navbar-start">
        <%= link_to root_url, class: 'navbar-item' do %>
          <span class="icon">
            <i class="far fa-gem"></i>
          </span>
          <span>Home</span>
        <% end %>
        <%= link_to home_about_url, class: 'navbar-item' do %>
          <span class="icon">
            <i class="far fa-star"></i>
          </span>
          <span>About</span>
        <% end %>
      </div>
    </div>
  </nav>
</div>
```
This is basically copied from the Bulma examples. It is a basic nav bar with two menu items; Home and About.

We now have all the pieces in place and can start wiring up our Stimulus controllers.

Create a Stimulus controller
To keep this example simple, we’re going to create a single controller which we’ll attach to the body tag in the main layout. This controller will be responsible for rendering the Font Awesome icons (as described in a previous post) as well as handling our Bulma burger menu.

Create app/javascript/controllers/main_controller.js:
```javascript
import fontawesome from '@fortawesome/fontawesome'
import icons from '@fortawesome/fontawesome-free-regular'
import { Controller } from 'stimulus'
export default class extends Controller {
  initialize() {
    fontawesome.library.add(icons)
  }
  connect() {
    fontawesome.dom.i2svg()

    // Get all "navbar-burger" elements
    var $navbarBurgers = Array.prototype.slice.call(document.querySelectorAll('.navbar-burger'), 0);

    // Check if there are any navbar burgers
    if ($navbarBurgers.length > 0) {

      // Add a click event on each of them
      $navbarBurgers.forEach(function ($el) {
        $el.addEventListener('click', function () {

          // Get the target from the "data-target" attribute
          var target = $el.dataset.target;
          var $target = document.getElementById(target);

          // Toggle the class on both the "navbar-burger" and the "navbar-menu"
          $el.classList.toggle('is-active');
          $target.classList.toggle('is-active');

        });
      });
    }
  }
}
```

This controller imports the icons from Font Awesome when initialize is called. Every time connect is called it renders the icons and then searches for navbar burgers to attach the appropriate click events on.

Connect the controller
Now we want to connect the body tag to our controller using an HTML5 data attribute.

Open `app/views/layouts/application.html.erb` and add the following attribute to the `<body>` tag.
```html
<body data-controller="main">
```

Now it should look like:
```ruby
# app/views/layouts/application.html.erb
<!DOCTYPE html>
<html>

<head>
  <title>Tweets</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>

  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.6.0/dist/css/bootstrap.min.css" integrity="sha384-B0vP5xmATw1+K9KRQjQERJvTumQW0nPEzvF6L/Z6nronJ3oUOFUFpCjEUQouq2+l" crossorigin="anonymous">
  <%= stylesheet_link_tag 'application', media: 'all' %>
  <%= javascript_pack_tag 'application' %>
  <%= yield :head %>
  <%= turbo_include_tags %>
  <%# stimulus_include_tags %>
</head>

<body data-controller="main">
  <%= render 'layouts/navbar' %>
  <p class="notice"><%= notice %></p>
  <p class="alert"><%= alert %></p>
  <%= yield %>
</body>

</html>
```
https://blackninjadojo.com/css/bulma/2019/02/27/how-to-create-a-layout-for-your-rails-application-using-bulma.html


### Discourgaged - no longer necessary:

If you plan to user database_cleaner -- then also see this article to finish your config:

https://medium.com/@amliving/my-rails-rspec-set-up-6451269847f9
