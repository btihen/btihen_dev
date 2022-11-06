---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Squished String Type (Rails 5/6)"
subtitle: "String inputs that strip away leading, trailing and double spaces using typed virtual attributes"
summary: "String inputs that strip away leading, trailing and double spaces using typed virtual attributes"
authors: ["btihen"]
tags: ['Ruby', "Types", "Virtual Attributes"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-08-12T01:11:22+02:00
lastmod: 2021-08-12T01:11:22+02:00
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

* https://discuss.rubyonrails.org/t/database-fields-are-polluted-with-both-nil-and-empty-values-when-submitting-forms/74877/3
* https://bitbucket.org/tiu/rails-application-template/src/master/rails/lib/types/type/string.rb
* https://stackoverflow.com/questions/3879680/how-can-i-make-rails-3-localize-my-date-formats/45743846#45743846

Lets start by making a sample project (without tests -T):

```bash {linenos=table,hl_lines=1}
rails new task_cards -T
cd task_cards
```

We will use a generator to create the structures we need and just focus clean inputs.
```bash {linenos=table,hl_lines=1}
rails g scaffold Card title:string description:text
```

Now lets make our attribute types that cleanup string inputs (we will put them in the a new folder we will call `types`):
```bash
mkdir app/types
touch app/types/string_stripped_type.rb
touch app/types/text_trimmed_type.rb
```

To make a string type that removes leading, trailing and duplicate spaces we will use the squish method.
```ruby
# app/types/string_squished_type.rb
class StringSquishedType < ActiveRecord::Type::String
  # cast the incomming value for Rails
  def cast(value)
    value.to_s.squish
  end

  # convert the data to what the Database expects
  def serialize(value)
    value.to_s
  end

end
```

Since text may want to have newlines and other `double` spaces we will only remove (`trim`) leading and trailing spaces:
```ruby
# app/types/text_trimmed_type.rb
class TextTrimmedType < ActiveRecord::Type::String
  # cast the incomming value for Rails
  def cast(value)
    value.to_s.strip
  end

  # convert the data to what the Database expects
  def serialize(value)
    value.to_s
  end

end
```

To simplify our code we will define short names for our new types -- in the `config/initializers` folder we will make a new file called `types.rb`:
```bash
touch config/initializers/attribute_types.rb
```

To make a string type that removes leading, trailing and duplicate spaces we will use the squish method.
```ruby
# config/initializers/attribute_types.rb
ActiveRecord::Type.register(:string_stripped, StringSquishedType)
ActiveRecord::Type.register(:text_trimmed,    TextTrimmedType)
```

Now lets add our new virutual data types that we will use in our forms to our model:
```ruby
# app/models/card.rb
class Card < ApplicationRecord
  attribute :title_in,       :string_squished
  # attribute :title_in,       StringSquishedType.new
  attribute :description_in. :text_trimmed,   default: '--'
  # attribute :description_in, TextTrimmedType.new, default: '--'

  validate :title_in, presence: true
end
```


https://bitbucket.org/tiu/rails-application-template/src/master/rails/lib/types/type/string.rb
```ruby
# lib/types/type/string.rb
# frozen_string_literal: true

module Type
  # * +squish+ if true, squish value when casting
  # * +strip+ if true, strip value when casting
  # * +nilify_blank+ if true, set blank value to nil when casting
  class String < ActiveModel::Type::String

    def initialize(precision: nil, limit: nil, scale: nil, strip: false, squish: false, nilify_blank: false)
      @strip = strip
      @squish = squish
      @nilify_blank = nilify_blank
      super(precision: precision, limit: limit, scale: scale)
    end

    def serialize(value)
      super(value)
      apply_options(value)
    end

    private

      def cast_value(value)
        value = super(value)
        apply_options(value)
      end

      def apply_options(value)
        return unless value

        if @squish
          value = value.squish
        elsif @strip
          value = value.strip
        end

        value = nil if @nilify_blank && value.blank?

        value
      end
  end
end
```

https://bitbucket.org/tiu/rails-application-template/src/master/rails/lib/types/type/editor_text.rb
```ruby
# lib/types/type/editor_text.rb
# frozen_string_literal: true

# Strips out empty spaces that are default on Ckeditor 5
module Type
  class EditorText < ActiveModel::Type::String

    EMPTY_P_TAG_REGEX = %r{\A(<p[^>]*>(\s|&nbsp;|</?\s?br\s?/?>)*</?p>)\1*\z}.freeze

    def serialize(value)
      super(value)
      apply_options(value)
    end

    private

      def cast_value(value)
        value = super(value)
        apply_options(value)
      end

      def apply_options(value)
        return if value.nil?

        remove_empty_p_tags(value)
      end

      def remove_empty_p_tags(value)
        value.match?(EMPTY_P_TAG_REGEX) ? nil : value
      end
  end
end
```

https://bitbucket.org/tiu/rails-application-template/src/master/rails/lib/types/type/token.rb
```ruby
# lib/types/type/token.rb
# frozen_string_literal: true

# Represents a user-entered code, like a coupon code, discount code or confirmation number.
# Avoids ambiguous characters that could cause user confusion or apprehension.
module Type
  class Token < ActiveRecord::Type::String

    AMBIGUITIES = [
      %w[B 8],
      %w[D O 0],
      %w[G 6],
      %w[I 1 l],
      %w[S 5],
      %w[Z 2]
    ].flatten.freeze
    CHARACTERS = ([*('A'..'Z'), *('0'..'9')] - AMBIGUITIES).freeze
    LENGTH = 6

    def initialize(precision: nil, limit: nil, scale: nil, length: LENGTH)
      @length = length
      super(precision: precision, limit: limit, scale: scale)
    end

    private

      def cast_value(value)
        if value == :random
          random_number
        elsif value.is_a? ::String
          value = value.upcase
          value if value.chars.all? { |c| c.in? CHARACTERS }
        end
      end

      def random_number
        Array.new(@length) { CHARACTERS.sample }.join
      end
  end
end
```

```ruby
# lib/types/localized_date.rb
# frozen_string_literal: true

# Convert localized date string to Date object. This takes I18n formatted date strings
# (e.g. in form text inputs) and casts them back to Date objects when writing the attribute.
#
# See ActiveModel::Type::Date for original, which attempts to parse the Date string, causing
# the months and days swap if input is in "%m/%d/%Y" format.
#
class LocalizedDate < ActiveRecord::Type::Date

  # Full specifier is: %<flag><width><modifier><conversion>
  FORMAT_STRING_EXPR = /(?<=%)(?<flag>[-_0^#])?(?<width>\d)?/.freeze

  def initialize(format: default_format)
    @format_string = safe_format_string(format)
  end

  # Deserialize db value using Date::DATE_FORMATS[:db]
  def deserialize(value)
    cast_value(value, format: Date::DATE_FORMATS[:db]) unless value.nil?
  end

  private

    def cast_value(value, format: @format_string)
      if value.is_a?(::String)
        return if value.empty?

        Date.strptime(value, format)
      elsif value.respond_to?(:to_date)
        value.to_date
      else
        value
      end
    rescue ArgumentError
      nil
    end

    def default_format
      I18n.translate("date.formats.default")
    end

    # Date.strptime doesn't support flags and width, so remove them.
    # See https://ruby-doc.org/stdlib/libdoc/date/rdoc/Date.html#method-c-strptime
    def safe_format_string(value)
      value.gsub FORMAT_STRING_EXPR, ''
    end
end
```


```ruby
# config/initializers/types.rb
# frozen_string_literal: true

# NOTE: when using custom types with Postgres arrays they must be registered (here) to work.
# When registered, they become a subtype of ActiveRecord::ConnectionAdapters::PostgreSQL::OID::Array
# which handles the array bits before invoking your custom type.
#
#     attribute :links, :link, array: true          # this works, becoming an array subtype
#     attribute :links, Type::Link.new, array: true # this acts like a non-array type
#
# Next, confirm the type with `MyModel.type_for_attribute(:links)`
Dir[Rails.root.join('lib/types/**/*.rb')].sort.each { |f| require f }

ActiveRecord::Type.register(:localized_date, LocalizedDate)
ActiveRecord::Type.register(:string, Type::String, override: true)
ActiveRecord::Type.register(:token, Type::Token)

ActiveModel::Type.register(:localized_date, LocalizedDate)
ActiveModel::Type.register(:string, Type::String, override: true)
ActiveModel::Type.register(:token, Type::Token)
```


https://stackoverflow.com/questions/3879680/how-can-i-make-rails-3-localize-my-date-formats/45743846#45743846

How Rails Transforms Attributes

Here's a quick tour of the Rails Attributes API. You can skip this section, but then you won't know how this stuff works. What fun is that?

Understanding how Rails handles user input for your attribute will let us override only one method instead of making a more complete custom type. It will also help you write better code, since rails' code is pretty good.

Since you didn't mention a model, I'll assume you have a Post with a :publish_date attribute (some would prefer the name :published_on, but I digress).
What is your type?

Find out what type :publish_date is. We don't care that it is an instance of Date, we need to know what type_for_attribute returns:

    This method is the only valid source of information for anything related to the types of a model's attributes.
```ruby
$ rails c
> post = Post.where.not(publish_date: nil).first
> post.publish_date.class
=> Date
> Post.type_for_attribute('publish_date').type
=> :date
```

Now we know the `:publish_date` attribute is a `:date` type. This is defined by `ActiveRecord::Type::Date`, which extends `ActiveModel::Type::Date` (ActiveRecord Types are here: https://api.rubyonrails.org/classes/ActiveRecord/Type.html), which extends `ActiveModel::Type::Value` (ActiveModel Types are here: https://api.rubyonrails.org/classes/ActiveModel/Type.htmlValue.html).

How is user input transformed by `ActiveRecord::Type::Date`?

So, when you set `:publish_date`, the value is passed to cast, which calls cast_value. Since form input is a String, it will try a `fast_string_to_date` then `fallback_string_to_date` which uses `Date._parse`.

If you're getting lost, don't worry. You don't need to understand rails' code to customize an attribute.
Defining a Custom Type

Now that we understand how Rails uses the attributes API, we can easily make our own. Just create a custom type to override cast_value to expect localized date strings:
```ruby
class LocalizedDate < ActiveRecord::Type::Date

  private
    # Convert localized date string to Date object. This takes I18n formatted date strings
    # from user input and casts them back to Date objects.
    def cast_value(value)
      if value.is_a?(::String)
        return if value.empty?
        format = I18n.translate("date.formats.short")
        Date.strptime(value, format) rescue nil
      elsif value.respond_to?(:to_date)
        value.to_date
      else
        value
      end
    end
end
```

See how I just copied rails' code and made a small tweak. Easy. You might want to improve on this with a call to super and move the :short format to an option or constant.

Register your type so it can be referenced by a symbol:
```ruby
# config/initializers/types.rb
ActiveRecord::Type.register(:localized_date, LocalizedDate)
```

Override the :publish_date type with your custom type:
```ruby
# app/models/post.rb
class Post < ApplicationRecord
  attribute :publish_date, :localized_date
end
```

Now you can use localized values in your form inputs:
```ruby
# app/views/posts/_form.html.erb
<%= form_for(@post) do |f| %>
  <%= f.label :publish_date %>
  <%= f.text_field :publish_date, value: (I18n.localize(value, format: :short) if value.present?) %>
<% end %>
```
