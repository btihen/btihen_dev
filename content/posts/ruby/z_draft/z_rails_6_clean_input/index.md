---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails_6_clean_input"
subtitle: ""
summary: "Rails 6 makes it easy to format inputs"
authors: ["btihen"]
tags: ["Rails 5", "Rails 6", "Types", "Virtual Attributes", "Clean Input"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2020-04-14T18:57:00+02:00
lastmod: 2020-04-14T18:57:00+02:00
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

We can make a new data type that strips leading and trailing spaces from input fields.

```bash {linenos=table,hl_lines=1}
mkdir app/types
touch app/types/string_stripped_type.rb
```

The content of the file to remove leading and trailing spaces from input strings.

{{< highlight ruby "linenos=table" >}}
class StringSquishedType < ActiveRecord::Type::String

  def cast(value)
    value.to_s.squish
  end

end
{{< / highlight >}}

Now we can make a `form_object` to use this type:

```bash {linenos=table,hl_lines=1}
mkdir app/forms
touch app/forms/user_form.rb
```

The content of the file is a standard ruby object with `ActiveModel` included:

{{< highlight ruby "linenos=table" >}}
class UserForm

  # ActiveModel provides: initialization with a hash, validations, errors, interaction with view helpers
  include ActiveModel::Model
  # provides the ability to check for changed attributes
  include ActiveModel::Dirty
  # allows casting of specific attribute types
  include ActiveModel::Attributes

  # these two methods are needed if backed by an `active_record` model
  delegate :id, :persisted?, to: :user,  allow_nil: true
  # if there is no active record_model then ActiveModel needs:
  # def id
  #   nil
  # end
  # def persisted?
  #   false
  # end

  # should no longer be necessary, but if form_for and url_paths are broken try:
  # def self.model_name
  #   ActiveModel::Name.new(self, nil, 'User')
  # end
  # default initiator is usually enough, if not build your own
  # def initialize(attribs={})
  #   super
  # end

  # any attributs backed by `active_model - then add:
  delegate :first_name, :last_name, to: :user

  # allows id to be assigned by the form - attributes here use the db backed type (or are not cast at all)
  attr_accessor :id

  # now you can tell rails how to cast these values
  attribute :first_name, :string_squished
  attribute :last_name,  string_squished

  # now we can add our form validations
  validates :first_name, presence: true
  validates :last_name, presence: true

  validates :user_validation

  # this is helpful for the controller
  def self.new_from(user)
    attribs = {}
    if user.present?
      attribs = { id:         user.id,
                  first_name: user.first_name,
                  last_name:  user.last_name,
                }
    end
    new(attribs)
  end

  def user
    @user     ||= assign_user_attribs
  end

  private

  def assign_user_attribs
    # get / create instance
    user = User.find_by(id: id) || User.new

    # update user attributes from form
    user.first_name = first_name
    user.last_name  = last_name
    user
  end

  # convenient if root model has validations
  def user_validation
    return if user.valid?

    user.errors.each do |attribute_name, error_message|
      # add the user model errors to our form attributes
      errors.add(attribute_sym, error_message)
    end
  end

end
{{< / highlight >}}
