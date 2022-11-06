---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Robust_rails_04_add_username"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ['Ruby']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2020-07-20T21:04:41+02:00
lastmod: 2020-07-20T21:04:41+02:00
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


## configure devise with username
https://github.com/heartcombo/devise/wiki/How-To:-Allow-users-to-sign-in-with-something-other-than-their-email-address
http://alexvpopov.github.io/blog/2013/10/31/allow-users-to-authenticate-with-username-only-using-devise-activeadmin-rails-4-and-ruby-2/

now we will update our user models and active the model shoulda tests

Lets change user email for username and lets add a user_role field (for now with ut enum):

We will need to add model tests and enforce what's required and test that tool

Lets start by updating our `user` model tests to match our goals:
```
require 'rails_helper'

RSpec.describe User, type: :model do

  describe "factory functions" do
    it "generates a valid user" do
      model = FactoryBot.build :user
      expect(model.valid?).to be true
    end
    it "saves a valid user" do
      model = FactoryBot.build :user
      expect(model.save).to be_truthy
    end
  end

  describe "validations" do
    it { is_expected.to validate_presence_of(:password) }
    it { is_expected.to validate_presence_of(:username) }
    it { is_expected.to allow_value("cool-name").for(:username) }
    it 'saves a url friendly version of username' do
      model = FactoryBot.build :user, username: 'not ürl friendly'
      expect(model.save).to be_truthy
      model.reload
      expect(model.username).to eq 'not-url-friendly'
      expect(model.username).to eq 'not ürl friendly'.parameterize
    end
    it { is_expected.to validate_presence_of(:user_role) }
    it { is_expected.to validate_inclusion_of(:user_role)
                        .in_array(ApplicationHelper::USER_ROLES)
                        .with_message("must include: #{ApplicationHelper::USER_ROLES.join(', ')}") }
  end

  # tests DB field settings that aren't in other tests
  describe "DB settings" do
    it { is_expected.to have_db_column(:encrypted_password) }
    it { is_expected.to have_db_index(:username) }
  end
end
```

Lets start with the user's Migration of changing email to username:
```
# bin/rails g migration ChangeEmailIntoUsernameAndAddUserRoleInUsers

# update the migration so it looks liek:
class ChangeEmailToUsernameInUsers < ActiveRecord::Migration[6.0]
  def change
    rename_column :users, :email, :username

    # a safer way to change a column to an existing table
    # https://www.viget.com/articles/adding-a-not-null-column-to-an-existing-table/
    add_column    :users, :user_role, :string, default: 'student_user'
    change_column_null :users, :user_role,    false
  end
end

# and of course migrate:
bin/rails db:migrate
```

now update Devise config in `config/initalizers/devise.rb` (add `username` to these to settings):
```
config.case_insensitive_keys = [ :email, :username ]
config.strip_whitespace_keys = [ :email, :username ]
```

run the model tests -- oops the user factory is broken:
```
rspec spec/models
```

lets fix that by changin `email` to `username` (we will add sequence too to avoid problems with uniquness) and of course add the user roles which will will store in the Application Helper:
```
# spec/models/factories/users.rb
FactoryBot.define do
  factory :user do
    # avoid uniq conflicts with sequence
    # adding an invalid space for the model to fix with before_validation
    sequence(:username)   { |n| "user-#{Faker::Internet.username}-#{n}" }
    password              { 'LetM3-InNow' }
    password_confirmation { 'LetM3-InNow' }
    user_role             { ApplicationHelper::USER_ROLES.sample }
    # enable this if using confirmable
    # confirmed_at { Date.today }
  end

  trait :student do
    user_role             { 'student_user' }
  end

  trait :teacher do
    user_role             { 'teacher_user' }
  end
end
```


lets also add the list of user roles to the application helper:

first a test
```
require 'rails_helper'

RSpec.describe ApplicationHelper, type: :helper do
  it 'has the correct list of student roles' do
    expect(ApplicationHelper::USER_ROLES).to eq ['student_user', 'teacher_user']
  end
end
```

be sure that is broken:
```
rspec spec/helpers
```

now lets fix the helper test with code:
```
# app/helpers/application_helper.rb
module ApplicationHelper
  USER_ROLES = ['student_user', 'teacher_user']
end
```

run the model test again - the factoryshioudl now be ok, but the model is still broken:
```
rspec spec/models
```

lets fix the user model with (you must add the email methods to ignore the email field):
```
# models/users.rb
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :trackable, :registerable,
         :rememberable, :validatable, # :recoverable, # - can't be recoverable without email
         :authentication_keys => [:username]

  before_validation :username_parameratize
  validates :username, :presence => true, uniqueness: true
  validates :password, :presence => true
  validates :user_role, :presence => true,
                        inclusion: {in: ApplicationHelper::USER_ROLES,
                                    message: "must include: #{ApplicationHelper::USER_ROLES.join(', ')}"}

  def email_required?
    false
  end

  def email_changed?
    false
  end
  # use this instead of email_changed? for Rails = 5.1.x
  alias_method :will_save_change_to_email?, :email_changed?

  private
  def username_parameratize
    self.username = attributes["username"].to_s.parameterize
  end
end
```

lets run all the test now
```
rspec spec
```

Now lets see what else is broken:
```
rspec
```

Now lets update our `user` feature tests too - changing `email` to `username`:
```
# spec/support/features/session_helper_spec.rb
# spec/support/features/session_helpers.rb
module Features
  module SessionHelpers
    def user_sign_up(username:, password:)
      visit new_user_registration_path
      expect(page).to have_button('Sign up')
      fill_in 'Username', with: username
      fill_in 'Password', with: password
      click_button 'Sign up'
    end
    def user_log_in(user = nil)
      user = FactoryBot.create :user if user.nil?
      visit new_user_session_path
      expect(page).to have_button('Log in')
      fill_in 'Username', with: user.username
      fill_in 'Password', with: user.password
      click_on 'Log in'
    end
    # ...
  end
end
```

now that teh tests are fixed lets test again:
```
spec spec/features
```

opps now our devise forms are failing - all the `email` fields in useres/devise must be changed to `username`
```
# app/views/users/registrations/edit.html.erb
# app/views/users/registrations/new.html.erb
# app/views/users/sessions/new.html.erb
# from:
  <div class="field">
    <%= f.label :email %><br />
    <%= f.email_field :email, autofocus: true, autocomplete: "email" %>
  </div>
# to:
  <div class="field">
    <%= f.label :username %><br />
    <%= f.text_field :username, autofocus: true, autocomplete: "username" %>
  </div>
```

now all tests should be green when `rspec` runs

lets commit:
```
git add .
git commit -m "user model switched from email to username"
```
