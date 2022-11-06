---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails Devise User with a Site Role"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ['Ruby']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2020-07-10T20:45:51+02:00
lastmod: 2021-03-28T01:45:51+02:00
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
It's common to create need a simple role system - here is a simple way to do this with devise

## Configure devise (for multiple types of accounts)

install the devise engine:
```
bin/rails generate devise:install
```

now follow the basic setup config -- add to `config/environments/development.rb`
```
config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
```

add notifications to the layout for devise in `app/views/layouts/application.html.erb` just above `<%= yeild %>`

```
<p class="notice"><%= notice %></p>
<p class="alert"><%= alert %></p>
```

now create one or more models for devise:
```
rails generate devise user
```

turn on scoped views (since login forms can be different) in `config/initializers/devise.rb`
```
config.scoped_views = true
```
Create the views (to make them look better and add language translations):
```
rails g devise:views
```

now we should open these migrations and uncomment any added fields we use - I generally like to use most of the fields:
```
# frozen_string_literal: true

class DeviseCreateAdmins < ActiveRecord::Migration[6.0]
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
      t.inet     :current_sign_in_ip
      t.inet     :last_sign_in_ip

      ## Confirmable
      t.string   :confirmation_token
      t.datetime :confirmed_at
      t.datetime :confirmation_sent_at
      t.string   :unconfirmed_email # Only if using reconfirmable

      ## Lockable
      t.integer  :failed_attempts, default: 0, null: false # Only if lock strategy is :failed_attempts
      t.string   :unlock_token # Only if unlock strategy is :email or :both
      t.datetime :locked_at

      t.timestamps null: false
    end

    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    add_index :users, :confirmation_token,   unique: true
    add_index :users, :unlock_token,         unique: true
  end
end
```

and adjust the `user` model to turn on the features we want / need. We will go into detail later, for now I will just add trackable to the models:

```
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable, :trackable,
         :recoverable, :rememberable, :validatable
end
```


and of course migrate too.
```
bin/rails db:migrate
```

## enforce authentication

```
# app/controllers/application_controller.rb
class ApplicationController
  before_action :authenticate_user!
end
```

## skip authentication for the landing page

```
# app/controllers/landing_controller.rb
class LandingController < ApplicationController
  skip_before_action :authenticate_user!, only: [:index]

  def index
  end
end
```

Now the landing page should still be accessible.


## lets make logged in home pages and profile pages
```
rails g controller users/home index --no-helper --no-assets --no-controller-specs --no-view-specs
rails g controller users/profile edit --no-helper --no-assets --no-controller-specs --no-view-specs
```

## update user to have a profile

Let's assume the we want to know who the user is and allow a site_role
```
bin/rails g migration AddProfileFieldsToUsers first_name last_name site_role
```

Lets update the migration to dissallow null
```
class AddProfileFieldsToUsers < ActiveRecord::Migration[6.1]
  def change
    add_column :users, :first_name, :string, null: false
    add_column :users, :last_name, :string, null: false
    add_column :users, :site_role, :string, null: false, default: 'user'
  end
end
```

Finally lets update the user model
```
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  validates :email, presence: true, uniqueness: true
  validates :first_name, presence: true
  validates :last_name, presence: true
  validates :site_role, presence: true
  validate :password_complexity

  def admin?
    site_role.eql? "admin"
  end

  def user?
    site_role.eql?("user") || site_role.eql?("admin")
  end

  private

  def password_complexity
    # Regexp extracted from https://stackoverflow.com/questions/19605150/regex-for-password-must-contain-at-least-eight-characters-at-least-one-number-a
    return if password.blank? || password =~ /^(?=.*?[A-Z])(?=.*?[a-z])(?=.*?[0-9])(?=.*?[#?!@$%^&*-]).{10,70}$/

    errors.add :password, 'Complexity requirement not met. Length should be 10-70 characters and include: 1 uppercase, 1 lowercase, 1 digit and 1 special character'
  end
end
```

## Update controllers (with select scopes)

Now we can update the profile controller to configure the user's profile - not the scoping in the set_user is set to current_user - we ignore the params[:id] and always use the current_user info to fetch info to prevent people from changing / accessing other profiles
```
# app/controllers/users/profiles_controller.rb
class Users::ProfilesController < ApplicationController
  # scoped to current_user only!
  before_action :set_user, only: %i[ edit update destroy ]

  def edit
    respond_to do |format|
      format.html { render :edit }
    end
  end

  def update
    respond_to do |format|
      if @user.update(firm_params)
        format.html { redirect_to home_url, notice: "Firm was successfully updated." }
        format.json { render :show, status: :ok, location: @user }
      else
        format.html { render :edit }
        format.json { render json: @user.errors, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @user.destroy
    respond_to do |format|
      format.html { redirect_to root_url, notice: "User was successfully destroyed." }
      format.json { head :no_content }
    end
  end

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_user
      @user = current_user # scope to user (ignore params[:id])
    end

    # Only allow a list of trusted parameters through (add our new fields)
    def user_params
      params.require(:profile)
            .permit(:first_name, :last_name, :username,
                    :email, :password, :password_confirmation)
    end
end
```
I'll let you make your own forms.


Now we can make a user's individual home page
```
class Users::HomeController < ApplicationController
  # anything we select to display later will be scoped to the current_user
  def index
    respond_to do |format|
      @user = current_user
      format.html { render :index }
    end
  end
end
```
We don't have much to display yet, but you can make a navbar and display name, etc.

## update Routes to enable the new pages and user home page as root

now lets update our routes to point to these pages if the user is logged in add the following the below the devise_for commands
```
Rails.application.routes.draw do
  # / (root when logged in)
  authenticated :user do
    root 'user/home#index', as: :auth_user_root
  end

  namespace :users do
    # /users/profile
    resource  :profile, only: [:edit, :update, :destroy]
    # /users/home
    get 'home', to: 'home#show'
  end

  # /home
  get '/home', to: 'users/home#show', as: :home

  # /landing
  get '/landing', to: 'landing#index', as: :landing

  # /  (root when not logged in)
  root to: "landing#index"
end
```

## update the user factory

Now we need to configure the factories so all is working:
```
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    # if you need to generate MANY users in the tests
    # sequence(:email) { |n| "#{Faker::Internet.email}".split('@').join("#{n}@") }
    # using .unique is good enough for most situations
    email       { Faker::Internet.unique.safe_email }
    first_name  { Faker::Name.unique.first_name }
    last_name   { Faker::Name.unique.last_name }
    password    { 'Let-M3-In_N0w!' }
  end
  trait :admin do
    site_role   { 'admin' }
  end
  trait :user do
    site_role   { 'user' }
  end
  trait :user_invalid do
    first_name  { "" }
    last_name   { "" }
    password    { "hoi" }
    email       { Faker::Internet.username }
  end
end
```


## Tests:
```
# spec/models/user.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  describe "Factory with" do

    context "default parameters" do
      it "creates a valid model" do
        model = FactoryBot.build :user
        expect(model.valid?).to be_truthy
      end
    end

    context "invalid parameters" do
      it "fails model validation" do
        user = FactoryBot.build :user, :user_invalid
        expect(user.valid?).to be_falsey
      end
    end
  end

  context "Model Methods / Behaviors" do
    it "test site_role methods" do
      model = FactoryBot.create :user
      expect(model.user?).to be_truthy
    end
    it "test site_role methods" do
      model = FactoryBot.create :user
      expect(model.admin?).to be_falsey
    end
    it "test site_role methods" do
      model = FactoryBot.create :user, :admin
      expect(model.user?).to be_truthy
    end
    it "test site_role methods" do
      model = FactoryBot.create :user, :admin
      expect(model.admin?).to be_truthy
    end
  end

  context "relationships" do
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
    it "detects a bad username, first_name, last_name" do
      user = FactoryBot.build :user, first_name: "", last_name: ""
      expect(user.valid?).to be_falsey
      expect(user.errors.messages[:last_name]).to match_array ["can't be blank"]
      expect(user.errors.messages[:first_name]).to match_array ["can't be blank"]
    end
  end

end
```

Now let's test our profile page:
(note: I add a hidden paragraph tag to help with testing since root and other pages need to be identified:  `"<p hidden id='user_profile'>Profile</p>"`)
```
# spec/requests/users/profile_request_spec.rb
require 'rails_helper'

RSpec.describe "Users::Profile", type: :request do

  describe "GET /edit" do
    context "not logged in" do
      it "'/users/profile' - returns http success" do
        get "/users/profile/edit"
        expect(response).to have_http_status(:redirect)
        expect(response.body).to include 'users/sign_in'
      end
      it "user_profile_path - returns http success" do
        get edit_users_profile_path
        expect(response).to have_http_status(:redirect)
        expect(response.body).to include 'users/sign_in'
      end
    end

    context "logged in" do
      let(:user)   { FactoryBot.create :user }
      before do
        sign_in user
      end
      after do
        sign_out user
      end
      it "'/users/profile' - returns http success" do
        get "/users/profile/edit"
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='user_profile'>Profile</p>"
      end
      it "user_profile_path - returns http success" do
        get edit_users_profile_path
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='user_profile'>Profile</p>"
      end
    end
  end

  # test update

  # test delete

end
```

Now lets test our home page
```
# spec/requests/users/home_request_spec.rb
require 'rails_helper'

RSpec.describe "User::Home", type: :request do

  describe "GET /index" do
    context "not logged in" do
      it "'/home' - redirects to user_login" do
        get "/home"
        expect(response).to have_http_status(:redirect)
        expect(response.body).to include 'users/sign_in'
      end
      it "home_path - redirects to user_login" do
        get home_path
        expect(response).to have_http_status(:redirect)
        expect(response.body).to include 'users/sign_in'
      end
      it "'/users/home' - redirects to user_login" do
        get "/users/home"
        expect(response).to have_http_status(:redirect)
        expect(response.body).to include 'users/sign_in'
      end
      it "users_home_path - redirects to user_login" do
        get users_home_path
        expect(response).to have_http_status(:redirect)
        expect(response.body).to include 'users/sign_in'
      end
      it "'/' - goes to landing page" do
        get "/"
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='landing_index'>Landing</p>"
      end
      it "user_root_path - goes to landing page" do
        get user_root_path
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='landing_index'>Landing</p>"
      end
    end

    context "logged in" do
      let(:user)   { FactoryBot.create :user }
      before do
        sign_in user
      end
      after do
        sign_out user
      end
      it "'/home' - returns http success" do
        get "/home"
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='user_home'>Home</p>"
      end
      it "home_path - returns http success" do
        get home_path
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='user_home'>Home</p>"
      end
      it "'/users/home' - returns http success" do
        get "/users/home"
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='user_home'>Home</p>"
      end
      it "users_home_path - returns http success" do
        get users_home_path
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='user_home'>Home</p>"
      end
      it "'/ - returns http success" do
        get "/"
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='user_home'>Home</p>"
      end
      it "root_path - returns http success" do
        get root_path
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='user_home'>Home</p>"
      end
      it "users_root_path - returns http success" do
        get user_root_path
        expect(response).to have_http_status(:success)
        expect(response.body).to include "<p hidden id='user_home'>Home</p>"
      end
    end
  end
end
```
