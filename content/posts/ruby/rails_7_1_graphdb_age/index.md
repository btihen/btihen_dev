---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x - Graph App"
subtitle: "Using a Postgres AGE GraphDB"
summary: ""
authors: ['btihen']
tags: ['Rails', "GraphDB", "Postgres AGE"]
categories: ["Code", "GraphDB", "Ruby Language", "Rails Framework"]
date: 2024-05-06T01:20:00+02:00
lastmod: 2024-05-06T01:20:00+02:00
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

Graph Databases are very interesting technologies for managing social networks, recommendation engines, etc.


In this article, I want to explore using Apache AGE within a Rails Application.  This article assumes a working knowledge of OpenCypher and Apache AGE.  If you want to get started you can find an Intro and further resources at: https://btihen.dev/posts/tech/graphdb_getting_started_age_1_5_0/

This code can be found at: https://github.com/btihen-dev/rails_graphdb_age

NO CODE YET AVAILABLE - this is a Document in PROGRESS!

## Getting Started

I will assume you have the AGE extension already installed in your Postgres instance and you understand OpenCypher at least casually.  If not please first visit: https://btihen.dev/posts/tech/graphdb_getting_started_age_1_5_0/

As we have done in a few apps, we will build a Flintstone's App.
Let's create our rails app:
```
rails _7.1.3.2_ new graphdb_age -T -d postgresql \
      --css=bootstrap
cd graphdb_age
```

## Config Postgress Connections

No matter how you installed Apache Age, you now need to do the following withing postgres:
```
CREATE EXTENSION IF NOT EXISTS age;

LOAD 'age';

SET search_path = ag_catalog, "$user", public;
```

To do this config we need to update our config & create migrations

### Config

I assume you have installed Apache AGE and understand Cypher - if not please read [GraphDB - Apache AGE 1.5.0](https://btihen.dev/posts/tech/graphdb_getting_started_age_1_5_0/)

Quick install summary using Docker:
```bash
docker pull apache/age
```

Start Docker with (please change the password & username)
```bash
docker run \
    --name myPostgresDb  \
    -p 5455:5432 \
    -e POSTGRES_USER=postgresUser \
    -e POSTGRES_PASSWORD=postgresPW \
    -e POSTGRES_DB=postgresDB \
    -d \
    apache/age
```

Now configuring the DB `config/database.yml` to look like:
```yml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  username: postgresUser
  password: postgresPW
  host: localhost
  port: 5455
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
...
```

now lets check our setup with:
```
bin/rails db:setup
```

### Migration for the Schema

we can create the migration with:
```bash
bin/rails g migration AddAgeSetup
```

Now write the migration:
```ruby
class AddAgeSetup < ActiveRecord::Migration[7.1]
  def change
    # allow age extension
    execute("CREATE EXTENSION IF NOT EXISTS age;")
    # load the age code
    execute("LOAD 'age';")
    # load the ag_catalog into the search path
    execute('SET search_path = ag_catalog, "$user", public;')
    # creates our AGE schema
    # USE: `execute("SELECT create_graph('age_schema');")`, as we need to use: `create_graph`
    # NOT: `ActiveRecord::Base.connection.create_schema('age_schema')`
    execute("SELECT create_graph('age_schema');")
  end
end
```

Now migrate:
```bash
bin/rails db:migrate
```

Now we see the warnings:
```
unknown OID 4089: failed to recognize type of 'namespace'. It will be treated as String.
unknown OID 2205: failed to recognize type of 'relation'. It will be treated as String.
```

Let's log into our DB and see why these warning are happening:
```bash
# do this once
export PGPASSWORD=postgresPW

# now access the database
psql -d graphdb_age_development -U postgresUser -h localhost -p 5455

# let's check for our schema using `\dn`:
graphdb_age_development=# \dn

# and we see
        List of schemas
    Name    |       Owner
------------+-------------------
 ag_catalog | postgresUser
 age_schema | postgresUser
 public     | pg_database_owner
(3 rows)

# exit with `\q`
graphdb_age_development=# \q
```

So we thought we created one, but we really have 2 new schemas `ag_catalog` has our age configs the `OID`s that couldn't be found.

To fix this we add our graph schemas to `config/database.yml` - using (put `ag_catalog` BEFORE `age_schema` as we need to access the AGE infos before we use `age_schema`) thus we add:
`schema_search_path: 'ag_catalog,age_schema,public'`
This needs to be added to the DB Application config - so the PG config file will now look like:

```yml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  username: postgresUser
  password: postgresPW
  host: localhost
  port: 5455
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  schema_search_path: 'ag_catalog,age_schema,public'
...
```

Now our config should be complete and we can use:
`bin/rails db`
instead of using:
`psql -d graphdb_age_development -U postgresUser -h localhost -p 5455`

NOTE: if you load `ag_catalog,age_schema` at our top level of yaml file (as the documentation suggests) you will get the error:
`'{ schema_search_path => ag_catalog,age_schema,public }' is not a valid configuration. Expected 'ag_catalog,age_schema,public' to be a URL string or a Hash. (ActiveRecord::DatabaseConfigurations::InvalidConfigurationError)`

## Let Explore

### First Exploration

Before we create a Rails strategy, let's explore a bit using the Rails console by using:
`bin/rails c`

Now let's try to create some nodes using raw Cypher SQL.  We can start with a very simple Data class to help use create People.

```ruby
Person = Data.define(:first_name, :last_name, :given_name, :gender) do
  def initialize(first_name:, last_name:, given_name: nil, gender:)
    given_name = given_name || last_name
    super(first_name:, last_name:, given_name:, gender:)
  end

  def to_s
    %{:Person {first_name: '#{first_name}', last_name: '#{last_name}', given_name: '#{given_name}', gender: '#{gender}'}}
  end
end

zed = Person.new(first_name: 'Zed', last_name: 'Flintstone', gender: 'male')
#<data Person first_name="Zed", last_name="Flintstone", given_name="Flintstone", gender="male">

zed.to_s
# "p:Person {first_name: 'Zed', last_name: 'Flintstone', given_name: 'Flintstone', gender: 'male'}"

def create_person(person)
  schema_name = 'age_schema'
  node = 'node'

  sql = <<-SQL
    SELECT *
    FROM cypher('#{schema_name}', $$
        CREATE (#{node}#{person})
    RETURN #{node}
    $$) as (#{node} agtype);
  SQL

  result = ActiveRecord::Base.connection.execute(sql)
  json_data = result.to_a.first[node].split('::vertex').first

  JSON.parse(json_data)
end

result = create_person(zed)

{"id"=>844424930131972,
 "label"=>"Person",
 "properties"=>
 {"gender"=>"male", "last_name"=>"Flintstone", "first_name"=>"Zed", "given_name"=>"Flintstone"}}

exit
```

### Second Exploration

Let's drop our database and see if we can make a more useful Ruby class that might work better as an Entity (an Object with a unique ID):

```bash
bin/rails db:drop
bin/rails db:create
bin/rails db:migrate
bin/rails c
```

Ok this looks interesting - maybe we can make a regular class using:

```ruby
# not necessary
ActiveRecord::Base.connection.execute('CREATE TYPE Person')

# without a namespace I get active record errors
class Person
end
zed = Person.new
`exec': ERROR:  relation "base_nodes" does not exist (PG::UndefinedTable) LINE 10:  WHERE a.attrelid = '"base_nodes"'::regclass`

# with a namespace things work again
module Node
  class Person
    attr_reader :age_hash, :id, :first_name, :last_name, :given_name, :gender

    AGE_TYPE = 'node'.freeze
    CLASS_NAME = 'Person'.freeze
    GRAPH_NAME = 'age_schema'.freeze

    def initialize(id: nil, first_name:, last_name:, given_name: nil, gender:)
      @id = id
      @given_name = given_name || last_name
      @first_name = first_name
      @last_name = last_name
      @gender = gender
    end

    def persisted? = id.present?

    def to_s
      %{:#{CLASS_NAME} {first_name: '#{first_name}', last_name: '#{last_name}', given_name: '#{given_name}', gender: '#{gender}'}}
    end

    def to_h = { id:, gender:, last_name:, first_name:, given_name: }

    def to_age
      {
        id:,
        label: CLASS_NAME,
        properties: {
          gender:, last_name:, first_name:, given_name:
        }
      }
    end

    def create
      age_result = ActiveRecord::Base.connection.execute(create_sql)
      json_data = age_result.to_a.first[AGE_TYPE].split('::vertex').first

      hash = JSON.parse(json_data)

      @id = hash['id']
      hash
    end

    private

    def create_sql
      <<-SQL
        SELECT *
        FROM cypher('#{GRAPH_NAME}', $$
            CREATE (#{AGE_TYPE}#{self})
        RETURN #{AGE_TYPE}
        $$) as (#{AGE_TYPE} agtype);
      SQL
    end
  end
end
zed = Node::Person.new(first_name: 'Zed', last_name: 'Flintstone', gender: 'male')

zed.to_s
zed.to_h
zed.persisted?
zed.create
zed.persisted?
zed.to_h
zed.to_age
```

### Third Exploration

Can we create some Abstractions to make it easier to write two different Nodes - for example `:Person` and `:Company` ?

```bash
bin/rails db:drop
bin/rails db:create
bin/rails db:migrate
bin/rails c
```

Let's make an AGE Module and a Node class and maybe we can use some of ActiveModel so we can easily access our attributes.

```ruby
## Let's see if we can make some of the constants generic
module ApacheAge
  module Vertex
    def age_type = 'vertex'
    def age_label = self.class.name.split('::').last
    def age_graph = self.class.name.split('::').first.underscore
  end
end

module StoneAge
  class Person
    include ApacheAge::Vertex
  end
end

zed = StoneAge::Person.new
zed.age_label
# 'Person'
zed.age_graph
# 'stone_age'
zed.age_type
# 'vertex'
```

# with this start we could expand Vertex and Person with ActiveModel:
```ruby
module ApacheAge
  module Vertex
    include ActiveModel::Model
    include ActiveModel::Attributes
    include ActiveModel::Validations

    def age_type = 'vertex'
    def age_label = self.class.name.split('::').last
    def age_graph = self.class.name.split('::').first.underscore

    def persisted? = @id.present?
  end
end

module StoneAge
  class Person
    include ApacheAge::Vertex

    attribute :id, :integer
    attribute :first_name, :string
    attribute :last_name, :string
    attribute :given_name, :string
    attribute :gender, :string

    def initialize(id: nil, first_name:, last_name:, given_name: nil, gender:)
      super # without this `@attributes` is nil and creates
      self.id = id
      self.given_name = given_name || last_name
      self.first_name = first_name
      self.last_name = last_name
      self.gender = gender
    end
  end
end

zed = StoneAge::Person.new(first_name: 'Zed', last_name: 'Flintstone', gender: 'male')
zed.attributes
# {"id"=>nil, "first_name"=>"Zed", "last_name"=>"Flintstone", "given_name"=>"Flintstone", "gender"=>"male"}
zed.attributes.symbolize_keys
# {:id=>nil, :first_name=>"Zed", :last_name=>"Flintstone", :given_name=>"Flintstone", :gender=>"male"}
zed.persisted?
# false
zed.age_label
# 'Person'
zed.age_graph
# 'stone_age'
zed.age_type
# 'vertex'
```

With this basis let's see if we can add to_s, to_h and to_age generically
```ruby
module ApacheAge
  module Vertex

    def age_type = 'vertex'
    def age_label = self.class.name.split('::').last
    def age_graph = self.class.name.split('::').first.underscore

    def persisted? = @id.present?
    def to_h = attributes.symbolize_keys
    def age_properties = attributes.except('id').symbolize_keys

    def age_hash
      {
        id:,
        label: age_label,
        properties: age_properties
      }
    end

    def properties_to_s
      string_values =
        age_properties.each_with_object([]) do |(key,val), array|
          array << "#{key}: '#{val}'"
        end
      "{#{string_values.join(', ')}}"
    end

    def to_s
      ":#{age_label} #{properties_to_s}"
    end

    def age_node(age_alias = nil)
      age_alias ||=
        if @id.present?
          Digest::SHA256.hexdigest(@id.to_s).to_i(16).to_s(36).gsub(/[0-9]/,'')[0..9]
        end
      "(#{age_alias}:#{age_label} #{properties_to_s})"
    end
  end
end

module StoneAge
  class Person
    include ActiveModel::Model
    include ActiveModel::Attributes
    include ActiveModel::Validations
    include ApacheAge::Vertex

    attribute :id, :integer
    attribute :first_name, :string
    attribute :last_name, :string
    attribute :given_name, :string
    attribute :gender, :string

    def initialize(id: nil, first_name:, last_name:, given_name: nil, gender:)
      super # without this `@attributes` is nil and creates
      self.id = id
      self.given_name = given_name || last_name
      self.first_name = first_name
      self.last_name = last_name
      self.gender = gender
    end
  end
end

zed = StoneAge::Person.new(first_name: 'Zed', last_name: 'Flintstone', gender: 'male')
zed.age_hash
# {:id=>nil,
#  :label=>"Person",
#  :properties=>{:first_name=>"Zed", :last_name=>"Flintstone", :given_name=>"Flintstone", :gender=>"male"}}
zed.age_label
# 'Person'
zed.age_properties
# {:first_name=>"Zed", :last_name=>"Flintstone", :given_name=>"Flintstone", :gender=>"male"}
zed.age_node
#"(:Person {first_name: 'Zed', last_name: 'Flintstone', given_name: 'Flintstone', gender: 'male'})"
zed.age_node('zed')
#"(zed:Person {first_name: 'Zed', last_name: 'Flintstone', given_name: 'Flintstone', gender: 'male'})"
```

Finally let's add `create` back in:
```ruby

module ApacheAge
  module Vertex

    def age_type = 'vertex'
    def age_label = self.class.name.split('::').last
    def age_graph = self.class.name.split('::').first.underscore

    def persisted? = @id.present?
    def to_h = attributes.symbolize_keys
    def age_properties = attributes.except('id').symbolize_keys

    def age_hash
      {
        id: @id,
        label: age_label,
        properties: age_properties
      }
    end

    def properties_to_s
      string_values =
        age_properties.each_with_object([]) do |(key,val), array|
          array << "#{key}: '#{val}'"
        end
      "{#{string_values.join(', ')}}"
    end

    def to_s
      ":#{age_label} #{properties_to_s}"
    end

    def age_alias
      return nil if @id.blank?

      # we start the alias with a since we can't start with a number
      'a' + Digest::SHA256.hexdigest(@id.to_s).to_i(16).to_s(36)[0..9]
    end

    def age_node(alias_name = nil)
      alias_name = alias_name || age_alias
      "(#{age_alias}:#{age_label} #{properties_to_s})"
    end

    def create
      age_result = ActiveRecord::Base.connection.execute(create_sql)
      json_data = age_result.to_a.first[age_label.downcase].split("::#{age_type}").first
      json_data = age_result.to_a.first.values.first.split("::#{age_type}").first

      hash = JSON.parse(json_data)

      @id = hash['id']
      hash
    end

    def create_sql
      alias_name = age_alias || age_label.downcase
      <<-SQL
        SELECT *
        FROM cypher('#{age_graph}', $$
            CREATE (#{alias_name}#{self})
        RETURN #{alias_name}
        $$) as (#{age_label} agtype);
      SQL
    end
  end
end

module AgeSchema
  class Person
    include ActiveModel::Model
    include ActiveModel::Attributes
    include ActiveModel::Validations
    include ApacheAge::Vertex

    attribute :id, :integer
    attribute :first_name, :string
    attribute :last_name, :string
    attribute :given_name, :string
    attribute :gender, :string

    def initialize(id: nil, first_name:, last_name:, given_name: nil, gender:)
      super # without this `@attributes` is nil and creates
      self.id = id
      self.given_name = given_name || last_name
      self.first_name = first_name
      self.last_name = last_name
      self.gender = gender
    end
  end
end
zed = AgeSchema::Person.new(first_name: 'Zed', last_name: 'Flintstone', gender: 'male')
zed.persisted?
# false
zed.create
# {"id"=>844424930131980,
#  "label"=>"Person",
#  "properties"=>{"gender"=>"male", "last_name"=>"Flintstone", "first_name"=>"Zed", "given_name"=>"Flintstone"}}
zed.persisted?
# true
zed.age_hash
# {:id=>844424930131981,
#  :label=>"Person",
#  :properties=>{:first_name=>"Zed", :last_name=>"Flintstone", :given_name=>"Flintstone", :gender=>"male"}}
```

Now we can make a simple second Node `Company`
```ruby
module AgeSchema
  class Company
    include ActiveModel::Model
    include ActiveModel::Attributes
    include ActiveModel::Validations
    include ApacheAge::Vertex

    attribute :id, :integer
    attribute :name, :string

    def initialize(id: nil, name:)
      super # without this `@attributes` is nil and creates
      self.id = id
      self.name = name
    end
  end
end
quarry = AgeSchema::Company.new(name: 'Bedrock Quarry')
quarry.save
# {"id"=>1125899906842625,
#  "label"=>"Company",
#  "properties"=>{"name"=>"Bedrock Quarry", "gender"=>"", "last_name"=>"", "first_name"=>"", "given_name"=>""}}
```

We probaly need to add `find` and `find_by`, but lets do that later.  Let's explore edges next.

## Seed File


## User Repository Pattern


## Resources

### Graph App Example Resources

### Rails with Apache AGE

* [Working with multi-schema database in Rails](https://learnitnow.medium.com/working-with-multi-schema-database-in-rails-7e60aef8ff86)
* [Using multiple PostgreSQL schemas with Rails models](https://stackoverflow.com/questions/8806284/using-multiple-postgresql-schemas-with-rails-models)
* [Rails Guides: Multiple Databases with Active Record](https://guides.rubyonrails.org/active_record_multiple_databases.html#connecting-to-databases-without-managing-schema-and-migrations)

### AGE Resources

* [AGE Quick-start](https://age.apache.org/getstarted/quickstart/)
* [AGE Downloads](https://age.apache.org/download/)
* [AGE Repo](https://github.com/apache/age?tab=readme-ov-file)
* [AGE Cypher Docs](https://age.apache.org/age-manual/master/index.html)
* [Apache AGE Installation Tutorial for MacOS - compile extension](https://www.youtube.com/watch?v=0-qMwpDh0CA)


### Other Graph Resources

* [GraphDB in Postgres with SQL](https://www.dylanpaulus.com/posts/postgres-is-a-graph-database/)
* [Modeling graphs and trees using recursion in PostgreSQL](https://www.linkedin.com/pulse/you-dont-need-graph-database-modeling-graphs-trees-viktor-qvarfordt-efzof/)
* [Graph Academy](https://graphacademy.neo4j.com/)
* [Neo4J Developer](http://neo4jrb.io/)
* [Neo4J ActiveGraph Gem](https://github.com/neo4jrb/activegraph?tab=readme-ov-file)
* [Getting Started with Neo4j and Ruby](https://neo4j.com/developer/ruby-course/)
* [Neo4J Developer / Cypher Docs](https://neo4jrb.readthedocs.io/en/stable)
