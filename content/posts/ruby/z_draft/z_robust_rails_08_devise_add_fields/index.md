---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Robust_rails_04_devise_add_fields"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ['Ruby']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2020-07-20T21:05:32+02:00
lastmod: 2020-07-20T21:05:32+02:00
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

### let's now update the admin model (add username, first & last name)

We want to add to the admimn model the title, username, role, first and last name - so we will create a test:
```
# spec/models/admin_spec.rb
require 'rails_helper'

RSpec.describe Admin, type: :model do

  describe "factory functions" do
    it "generates a valid admin" do
      model = FactoryBot.build :admin
      expect(model.valid?).to be true
    end
    it "saves a valid admin" do
      model = FactoryBot.build :admin
      expect(model.save).to be_truthy
    end
  end

  describe "DB settings" do
    it { have_db_index(:email) }
    it { is_expected.to have_db_column(:encrypted_password) }
    it { is_expected.to have_db_column(:admin_title) }
    it { is_expected.to have_db_column(:first_name) }
    it { is_expected.to have_db_column(:last_name) }
    it { is_expected.to have_db_column(:admin_role) }
  end

  describe "validations" do
    it { is_expected.to validate_presence_of(:email) }
    it { is_expected.to validate_presence_of(:password) }
    it { is_expected.to validate_presence_of(:last_name) }
    it { is_expected.to validate_presence_of(:first_name) }
    it { is_expected.to validate_presence_of(:admin_title) }
    it { is_expected.to validate_presence_of(:admin_role) }
    it { is_expected.to validate_inclusion_of(:admin_role)
                        .in_array(ApplicationHelper::ADMIN_ROLES)
                        .with_message("must include: #{ApplicationHelper::ADMIN_ROLES.join(', ')}") }
    it { is_expected.to validate_presence_of(:username) }
    it { is_expected.to allow_value("cool-name").for(:username) }
    it 'saves a url friendly version of username' do
      model = FactoryBot.build :user, username: 'not ürl friendly'
      expect(model.save).to be_truthy
      model.reload
      expect(model.username).to eq 'not-url-friendly'
      expect(model.username).to eq 'not ürl friendly'.parameterize
    end
    # it { is_expected.to validate_presence_of(:tenant) }
  end
end
```

Now of course `rspec` the admin model is red - lets do a migration and add our missing fields:
```
bin/rails g migration AddFieldsToAdmins username:string:uniq admin_title first_name last_name admin_role
```

Now lets adjust the migration to require these fields
lets follow the advices for adding null restratint to existing tables:
https://www.viget.com/articles/adding-a-not-null-column-to-an-existing-table/
```
class AddFieldsToAdmins < ActiveRecord::Migration[6.0]
  def change
    add_column :admins, :username,    :string
    add_column :admins, :last_name,   :string
    add_column :admins, :first_name,  :string
    add_column :admins, :admin_role,  :string
    add_column :admins, :admin_title, :string

    # a safer way to change a column to an existing table
    # https://www.viget.com/articles/adding-a-not-null-column-to-an-existing-table/
    change_column_null :admins, :username,    false
    change_column_null :admins, :last_name,   false
    change_column_null :admins, :first_name,  false
    change_column_null :admins, :admin_role,  false
    change_column_null :admins, :admin_title, false

    # index username
    add_index :admins, :username, unique: true
  end
end
```

lets be sure all is well withoour migration:
```
bin/rails db:migrate
```

Now lets run our tests and see what broken `rspec`

It looks like almost evertyhing with admin is broken.  Lets start witht the admin factory:
```
class Admin < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable, # :confirmable,
         :recoverable, :rememberable, :validatable, :trackable

  before_validation :username_parameratize
  validates :username,    :presence => true, uniqueness: true
  validates :email,       :presence => true, uniqueness: true
  validates :password,    :presence => true
  validates :last_name,   :presence => true
  validates :first_name,  :presence => true
  validates :admin_title, :presence => true
  validates :admin_role,  :presence => true,
                          inclusion: {in: ApplicationHelper::ADMIN_ROLES,
                                      message: "must include: #{ApplicationHelper::ADMIN_ROLES.join(', ')}"}
  private
  def username_parameratize
    self.username = attributes["username"].to_s.parameterize
  end
end
```
