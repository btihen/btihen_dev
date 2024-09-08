---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x - Graph AGE Integration Exploration"
subtitle: "Explore Rails Integration with Postgres Apache AGE GraphDB"
summary: ""
authors: ['btihen']
tags: ['Rails', "GraphDB", "Postgres AGE"]
categories: ["Code", "GraphDB", "Ruby Language", "Rails Framework"]
date: 2024-05-06T01:20:00+02:00
lastmod: 2024-05-21T01:20:00+02:00
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

Rails code can be found at: https://github.com/btihen-dev/rails_graphdb_age_app

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

## Test Postgres Connections

No matter how you installed Apache Age - lets access postgresql:
```bash
# simplify the login if you wish
export PGPASSWORD=postgresPW

# now access the database
psql -d myPostgresDb -U postgresUser -h localhost -p 5455
```

you now can test with the following cypher / SQL commands:
```sql
--- ensure age extension is installed
CREATE EXTENSION IF NOT EXISTS age;

--- load the extension
LOAD 'age';

--- load ag_catalog to the search path
SET search_path = ag_catalog, "$user", public;

--- create the graph schema
SELECT create_graph('age_schema');

--- create a node
SELECT *
FROM cypher('graph_name', $$
    CREATE (a {name: 'Andres'})
    RETURN a
$$) as (a agtype);

--- select the node
SELECT * FROM cypher('graph_name', $$
MATCH (v)
RETURN v
$$) as (v agtype);
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
quarry.create
# {"id"=>1125899906842625,
#  "label"=>"Company",
#  "properties"=>{"name"=>"Bedrock Quarry", "gender"=>"", "last_name"=>"", "first_name"=>"", "given_name"=>""}}
```

We probaly need to add `find` and `find_by`, but lets do that later.  Let's explore edges next.

### Edges First Exploration
```ruby
module ApacheAge
  module Vertex

    def age_type = 'vertex'
    def age_label = self.class.name.split('::').last
    def age_graph = self.class.name.split('::').first.underscore

    def persisted? = id.present?
    def to_h = attributes.symbolize_keys.to_hash
    def age_properties = attributes.except('id').symbolize_keys

    def age_hash
      {
        id: id,
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
      return nil if id.blank?

      # we start the alias with a since we can't start with a number
      'a' + Digest::SHA256.hexdigest(id.to_s).to_i(16).to_s(36)[0..9]
    end

    def age_node(alias_name = nil)
      alias_name = alias_name || age_alias
      "(#{age_alias}:#{age_label} #{properties_to_s})"
    end

    def create
      return self if id.present?

      age_result = ActiveRecord::Base.connection.execute(create_sql)
      json_data = age_result.to_a.first[age_label.downcase].split("::#{age_type}").first
      json_data = age_result.to_a.first.values.first.split("::#{age_type}").first

      hash = JSON.parse(json_data)

      self.id = hash['id']
      # @id = hash['id']
      self
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

module ApacheAge
  module Edge
    def age_type = 'edge'
    def age_label = self.class.name.split('::').last
    def age_graph = self.class.name.split('::').first.underscore

    def persisted? = id.present?

    def to_h
      base_hash =  attributes.except('start_node', 'end_node')
      base_hash[:start_node] = start_node.to_h
      base_hash[:end_node] = end_node.to_h
      base_hash.symbolize_keys
    end

    def age_properties = attributes.except('id', 'start_node', 'end_node', 'start_id', 'end_id').symbolize_keys

    def age_hash
      {
        id: id,
        end_id: end_id,
        start_id: start_id,
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
      return nil if id.blank?

      # we start the alias with a since we can't start with a number
      'a' + Digest::SHA256.hexdigest(id.to_s).to_i(16).to_s(36)[0..9]
    end

    def age_edge(alias_name = nil)
      alias_name = alias_name || age_alias
      "[#{age_alias}:#{age_label} #{properties_to_s}]"
    end

    def create
      return self if id.present?

      age_result = ActiveRecord::Base.connection.execute(create_sql)
      # json_data = age_result.to_a.first[age_label.downcase].split("::#{age_type}").first
      json_data = age_result.to_a.first.values.first.split("::#{age_type}").first

      hash = JSON.parse(json_data)

      self.id = hash['id']
      self.end_id = hash['end_id']
      self.start_id = hash['start_id']

      self
    end

    def create_sql
      self.start_node = start_node.create unless start_node.persisted?
      self.end_node = end_node.create unless end_node.persisted?
      <<-SQL
        SELECT *
        FROM cypher('#{age_graph}', $$
            MATCH (start_vertex:#{start_node.age_label}), (end_vertex:#{end_node.age_label})
            WHERE id(start_vertex) = #{start_node.id} and id(end_vertex) = #{end_node.id}
            CREATE (start_vertex)-[edge#{to_s}]->(end_vertex)
            RETURN edge
        $$) as (edge agtype);
      SQL
    end
  end
end

module AgeSchema
  class WorksAt
    include ActiveModel::Model
    include ActiveModel::Attributes
    include ActiveModel::Validations
    include ApacheAge::Edge

    attribute :role, :string

    attribute :id, :integer
    attribute :end_id, :integer
    attribute :start_id, :integer
    attribute :end_node #, :vertex
    attribute :start_node #, :vertex

    def initialize(id: nil, role:, start_node:, end_node:)
      super # without this `@attributes` is nil and creates
      self.id = id
      self.role = role
      self.end_id = end_node.id
      self.start_id = start_node.id
      self.start_node = start_node
      self.end_node = end_node
    end
  end
end

fred = AgeSchema::Person.new(first_name: 'Fred', last_name: 'Flintstone', gender: 'male')
quarry = AgeSchema::Company.new(name: 'Bedrock Quarry')
crane_ops = AgeSchema::WorksAt.new(role: 'Crane Operator', start_node: fred, end_node: quarry)
crane_ops.create_sql
crane_ops.create
```

Next, let's work with the similarities in both Vertex and Node code.  Let's see if we can resolve that next.

### Common Code Exploration

```ruby
module ApacheAge
  module CommonEntity
    def age_label = self.class.name.split('::').last
    def age_graph = self.class.name.split('::').first.underscore
    def persisted? = id.present?
    def to_s = ":#{age_label} #{properties_to_s}"
    def base_to_h = attributes.to_hash
    def base_properties = attributes.except('id')

    def base_hash
      {
        id: id,
        label: age_label,
        properties: age_properties
      }.transform_keys(&:to_s)
    end

    def properties_to_s
      string_values =
        age_properties.each_with_object([]) do |(key,val), array|
          array << "#{key}: '#{val}'"
        end
      "{#{string_values.join(', ')}}"
    end

    def age_alias
      return nil if id.blank?

      # we start the alias with a since we can't start with a number
      'a' + Digest::SHA256.hexdigest(id.to_s).to_i(16).to_s(36)[0..9]
    end

    def execute_sql
      return self.age_hash if id.present?

      age_result = ActiveRecord::Base.connection.execute(create_sql)
      json_data = age_result.to_a.first.values.first.split("::#{age_type}").first

      JSON.parse(json_data)
    end
  end
end

module ApacheAge
  module Vertex
    include ApacheAge::CommonEntity

    def age_type = 'vertex'
    def to_h = base_to_h.symbolize_keys
    def age_properties = base_properties.symbolize_keys
    def age_hash = base_hash.with_indifferent_access

    def create
      return self if id.present?

      response_hash = execute_sql
      self.id = response_hash['id']

      self
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

module ApacheAge
  module Edge
    include ApacheAge::CommonEntity

    def age_type = 'edge'
    def age_hash = base_hash.merge(end_id:, start_id:)
    def age_properties = base_properties.except('start_node', 'end_node', 'start_id', 'end_id').symbolize_keys

    def to_h
      base_h = base_to_h.except('start_node', 'end_node')
      base_h['start_node'] = start_node.to_h
      base_h['end_node'] = end_node.to_h
      base_h.with_indifferent_access
    end

    def create
      return self if id.present?

      response_hash = execute_sql
      self.id = response_hash['id']
      self.end_id = response_hash['end_id']
      self.start_id = response_hash['start_id']

      self
    end

    def create_sql
      self.start_node = start_node.create unless start_node.persisted?
      self.end_node = end_node.create unless end_node.persisted?
      <<-SQL
        SELECT *
        FROM cypher('#{age_graph}', $$
            MATCH (start_vertex:#{start_node.age_label}), (end_vertex:#{end_node.age_label})
            WHERE id(start_vertex) = #{start_node.id} and id(end_vertex) = #{end_node.id}
            CREATE (start_vertex)-[edge#{to_s}]->(end_vertex)
            RETURN edge
        $$) as (edge agtype);
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

module AgeSchema
  class WorksAt
    include ActiveModel::Model
    include ActiveModel::Attributes
    include ActiveModel::Validations
    include ApacheAge::Edge

    attribute :role, :string

    attribute :id, :integer
    attribute :end_id, :integer
    attribute :start_id, :integer
    attribute :end_node #, :vertex
    attribute :start_node #, :vertex

    def initialize(id: nil, role:, start_node:, end_node:)
      super # without this `@attributes` is nil and creates
      self.id = id
      self.role = role
      self.end_id = end_node.id
      self.start_id = start_node.id
      self.start_node = start_node
      self.end_node = end_node
    end
  end
end

fred = AgeSchema::Person.new(first_name: 'Fred', last_name: 'Flintstone', gender: 'male')
# fred.create
# fred.id

quarry = AgeSchema::Company.new(name: 'Bedrock Quarry')
# quarry.create
# quarry.id

works_at = AgeSchema::WorksAt.new(role: 'Crane Operator', start_node: fred, end_node: quarry)
works_at.create
works_at.id

fred.to_h
quarry.to_h
works_at.to_h

fred.age_hash
quarry.age_hash
works_at.age_hash

fred.age_properties
quarry.age_properties
works_at.age_properties
```


### Unique Property Constraints

Things to consider.
* Can we separate AGE `properties` from virtual `attributes`?
* Can we support all the property data-types?, :string, :array, etc?

I think its easiest to create a rails schema and declare properties:

```ruby
attribute :id, :integer
attribute :nick_name, :string
attribute :name, :string,
attribute :social_security_number, :integer
attribute :auth_roles, :array, default: ['user']

validates :name, required: true
validates :social_security_number, unique: true
validates :auth_roles, required: true

# class name is the label_name
# id is stored, but not a property
# nick_name is a virtual attribute
properties [:name, :social_security_number, :auth_roles]
```

Perhaps (how Neo4j works) - but that feels complicated:
```ruby
property :nick_name, :string
property :name, :string, required: true
property :social_security_umber, :integer, unique: true
property :auth_roles, :array, required: true, default: ['user']
```

**Rails Code**

https://github.com/neo4jrb/activegraph/blob/8e2ba4d117f5702633b0aa7099c71923a100c40d/lib/active_graph/shared/property.rb

```ruby
module ActiveGraph::Shared
  module Property
    extend ActiveSupport::Concern

    include ActiveGraph::Shared::MassAssignment
    include ActiveGraph::Shared::TypecastedAttributes
    include ActiveModel::Dirty
    ...
    module ClassMethods
      extend Forwardable

      def_delegators :declared_properties, :serialized_properties, :serialized_properties=, :serialize, :declared_property_defaults

      VALID_PROPERTY_OPTIONS = %w(type default index constraint serializer typecaster).map(&:to_sym)
      # Defines a property on the class
      #
      # See active_attr gem for allowed options, e.g which type
      # Notice, in ActiveGraph you don't have to declare properties before using them, see the ActiveGraph::Core api.
      #
      # @example Without type
      #    class Person
      #      # declare a property which can have any value
      #      property :name
      #    end
      #
      # @example With type and a default value
      #    class Person
      #      # declare a property which can have any value
      #      property :score, type: Integer, default: 0
      #    end
      #
      # @example With an index
      #    class Person
      #      # declare a property which can have any value
      #      property :name, index: :exact
      #    end
      #
      # @example With a constraint
      #    class Person
      #      # declare a property which can have any value
      #      property :name, constraint: :unique
      #    end
      def property(name, options = {})
        invalid_option_keys = options.keys.map(&:to_sym) - VALID_PROPERTY_OPTIONS
        fail ArgumentError, "Invalid options for property `#{name}` on `#{self.name}`: #{invalid_option_keys.join(', ')}" if invalid_option_keys.any?
        build_property(name, options) do |prop|
          attribute(prop)
        end
      end
      ...
    end
  end
end

```

**DB RESEARCH**

https://stackoverflow.com/questions/75903997/how-can-i-declare-primary-key-in-apache-age


Only allow 1 SSN entry in `SoftwareEngineer`

Use this query:
```sql
CREATE OR REPLACE FUNCTION create_pk(properties agtype)
RETURNS agtype AS
$BODY$
SELECT agtype_access_operator($1, '"SocialSecurityNumber"');
$BODY$
LANGUAGE sql IMMUTABLE;

CREATE UNIQUE INDEX person_pk_idx ON staff_details."SoftwareEngineer"
(create_pk(properties));
```

**Trigger A**

n Apache AGE, you can achieve the uniqueness of the property SocialSecurityNumber using triggers. You will need to create the trigger function at first:
```sql
CREATE OR REPLACE FUNCTION check_unique_ssn()
RETURNS TRIGGER AS $$
BEGIN
    IF EXISTS (
        SELECT 1 FROM SoftwareEngineer
        WHERE SocialSecurityNumber = NEW.SocialSecurityNumber
    ) THEN
        RAISE EXCEPTION 'A SoftwareEngineer with the same SocialSecurityNumber already exists';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```
Then, you need to create the trigger and insert the values into it:

```sql
CREATE TRIGGER enforce_unique_ssn
BEFORE INSERT ON SoftwareEngineer
FOR EACH ROW
EXECUTE FUNCTION check_unique_ssn();

INSERT INTO SoftwareEngineer (name, SocialSecurityNumber, date)
VALUES ('Muneeb', '12345', NOW());
```

**Using MERGE & COALESCE**

```sql
SELECT * FROM cypher('staff_details', $$
    MERGE (e:SoftwareEngineer {
        social_security_umber: '12345'
    })
    SET e.name = 'Muneeb', e.date = COALESCE(e.date, timestamp())
    RETURN e
$$) as (e agtype);

SELECT * FROM staff_details."SoftwareEngineer";
```

[MERGE documentation](https://age.apache.org/age-manual/master/clauses/merge.html)
[COALESCE documentation](https://age.apache.org/age-manual/master/functions/scalar_functions.html#coalesce)

**Trigger B**

There is a discussion on a similar issue on Github. Kindly check it out. From what I have tried and that works for your situation is:

Create vertex label (if it is not created already) before running the above CREATE INDEX query as

`SELECT * FROM create_vlabel('staff_details', 'SoftwareEngineer');`

Create Function "get_ssn" that will return the property SocialSecurityNumber from the "properties" column

```sql
CREATE OR REPLACE FUNCTION get_ssn(properties agtype)
RETURNS agtype
AS
$BODY$
select agtype_access_operator($1, '"SocialSecurityNumber"');
$BODY$
LANGUAGE sql
IMMUTABLE;
Create a unique index on the property "SocialSecurityNumber"

CREATE UNIQUE INDEX person_ssn_idx ON staff_details."SoftwareEngineer"(get_ssn(properties)) ;
```

Now when you try to add another node with the same SocialSecurityNumber, you get the error:

ERROR:  `duplicate key value violates unique constraint "person_ssn_idx" DETAIL:  Key (get_ssn(properties))=("12345") already exists.`

* https://github.com/apache/age/issues/45

`JoshInnis` commented on Nov 29, 2021

A graph name is a schema and a label name is a table. Id and properties are columns in vertex table. Id, start_id, end_id, and properties are columns in the edge tables. Use the agtype_access_operator(properties, key) to get to get a property value.

Knowing all that you can use Postges' standard DDL language to implement constraints, indices and unique values.

```sql
ALTER TABLE graph_name.label_name
ADD CONSTRAINT constraint_name
CHECK(agtype_access_operator(properties, "name_of_property") != '"Check against here"'::agtype);
```

`pdpotter` commented on Nov 30, 2021
Since indices require an immutable function, an additional function will still need to be created for them. When I create a get_id function with

```sql
CREATE OR REPLACE FUNCTION get_id(properties agtype)
  RETURNS agtype
AS
$BODY$
    select agtype_access_operator($1, '"id"');
$BODY$
LANGUAGE sql
IMMUTABLE;
```

and use it in an index with
```sql
CREATE UNIQUE INDEX person_id_idx ON mygraph.person(get_id(properties));
```

the creation of vertices with the same id will be prevented

ERROR:  `duplicate key value violates unique constraint "person_id_idx"`
DETAIL:  `Key (get_id(properties))=(2250) already exists.`
but the index will still not be used when trying to match vertices with a specific id:
```sql
SELECT * FROM ag_catalog.cypher('mygraph', $$EXPLAIN ANALYZE MATCH (a:person {id:2250}) return a$$) as (a agtype);
```

Is there a way to use indices when matching?

`JoshInnis` commented on Nov 30, 2021
Indices cannot currently be used while matching. There will need to be some re factoring done to allow the planner to realize opportunities where the indices can be used.


**Constraints**: You can create a uniqueness constraint on the SocialSecurityNumber property of the SoftwareEngineer label. This will enforce uniqueness and prevent duplicate nodes with the same SocialSecurityNumber from being created. Here's an example of how to create a uniqueness constraint using Cypher:

```sql
CREATE CONSTRAINT ON (se:SoftwareEngineer) ASSERT se.SocialSecurityNumber IS UNIQUE;
```

**Triggers**: You can also use triggers to enforce the uniqueness constraint at the database level. A trigger can be created to check if a SoftwareEngineer node with the same SocialSecurityNumber already exists before inserting a new node. Here's an example of how to create a trigger using Cypher:

```sql
CREATE TRIGGER check_unique_social_security_number
BEFORE INSERT ON SoftwareEngineer
FOR EACH ROW
BEGIN
    IF EXISTS (
        MATCH (se:SoftwareEngineer {SocialSecurityNumber: NEW.SocialSecurityNumber})
        RETURN se
    )
    THEN
        RAISE EXCEPTION 'A SoftwareEngineer with the same SocialSecurityNumber already exists.';
    END IF;
END;
```

With these constraints and triggers in place, if you try to insert a SoftwareEngineer node with a duplicate SocialSecurityNumber, it will result in an exception being raised, preventing the creation of the duplicate node.

**Note**: The examples provided assume that you have already created the SoftwareEngineer label and relevant properties in your graph schema. Adjust the Cypher statements accordingly based on your schema design.

### Migration Exploration

* Create a Node/Edge Label ('Table')

staff_details - Graph Name
SoftwareEngineer - Label Name
SocialSecurityNumber - Unique Property Name

`SELECT * FROM create_vlabel('staff_details', 'SoftwareEngineer');`

* Create Unique Fields

`CREATE CONSTRAINT ON (se:SoftwareEngineer) ASSERT se.SocialSecurityNumber IS UNIQUE;`

* Create Required Fields

Can this be done on a DB Level (or just within Rails)?


### Relationship Explorations

Can we Automate queries for relationships between nodes?  maybe something like:
```
# OUTGOING (within Person class)
one_link :outgoing :company, via: works_at
one_link :outgoing :employeer, node: Company, via: works_for, edge: WorksAt

many_links :outgoing :companies, via: works_at
many_links :outgoing :firms, node: Company, via: works_for, edge: WorksAt

# INCOMING (within Company class)
many_links :incoming :people, via: works_at
many_links :incoming :employees, node: Person, via: works_for, edge: WorksAt
```

**Create code**

From 'Neo4j/ActiveGraph'

https://github.com/neo4jrb/activegraph/blob/8e2ba4d117f5702633b0aa7099c71923a100c40d/lib/active_graph/node/has_n.rb#L409

```ruby
module ActiveGraph::Node
  module HasN

    extend ActiveSupport::Concern
    ...
    module ClassMethod
      ...
      def has_many(direction, name, options = {})
        name = name.to_sym
        build_association(:has_many, direction, name, options)

        define_has_many_methods(name, options)
      end

      def has_one(direction, name, options = {})
        name = name.to_sym
        build_association(:has_one, direction, name, options)

        define_has_one_methods(name, options)
      end

      private
      ...
    end
  end
end
```

## Testing

In order to run tests we need a valid schema.rb file.
It turns out in out case rails builds an incorrect `db/schema.rb` file.

To fix it just copy the migration into the schema - so it should look like:
```ruby
ActiveRecord::Schema[7.1].define(version: 20_240_505_183_043) do
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

Now commit this - as rails will keep changing it back and if its committed you can now restore you fix after each migration.

## Code

The up-tp-date code is at: https://github.com/btihen-dev/rails_graphdb_age_app

A quick summary of the working code within rails:

### Library Code

```ruby
# app/lib/apache_age/vertex.rb
module ApacheAge
  module Vertex
    extend ActiveSupport::Concern
    # include ApacheAge::Entity

    included do
      include ActiveModel::Model
      include ActiveModel::Dirty
      include ActiveModel::Attributes

      attribute :id, :integer

      extend ApacheAge::ClassMethods
      include ApacheAge::CommonMethods
    end

    def age_type = 'vertex'

    # AgeSchema::Nodes::Company.create(company_name: 'Bedrock Quarry')
    # SELECT *
    # FROM cypher('age_schema', $$
    #     CREATE (company:Company {company_name: 'Bedrock Quarry'})
    # RETURN company
    # $$) as (Company agtype);
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

    # So far just properties of string type with '' around them
    def update_sql
      alias_name = age_alias || age_label.downcase
      set_caluse =
        age_properties.map { |k, v| v ? "#{alias_name}.#{k} = '#{v}'" : "#{alias_name}.#{k} = NULL" }.join(', ')
      <<-SQL
        SELECT *
        FROM cypher('#{age_graph}', $$
            MATCH (#{alias_name}:#{age_label})
            WHERE id(#{alias_name}) = #{id}
            SET #{set_caluse}
            RETURN #{alias_name}
        $$) as (#{age_label} agtype);
      SQL
    end
  end
end
```

```ruby
# app/lib/apache_age/edge.rb
module ApacheAge
  module Edge
    extend ActiveSupport::Concern

    included do
      include ActiveModel::Model
      include ActiveModel::Dirty
      include ActiveModel::Attributes

      attribute :id, :integer
      attribute :end_id, :integer
      attribute :start_id, :integer
      attribute :end_node # :vertex
      attribute :start_node # :vertex

      validates :end_node, :start_node, presence: true

      extend ApacheAge::ClassMethods
      include ApacheAge::CommonMethods
    end

    def age_type = 'edge'

    # AgeSchema::Edges::WorksAt.create(
    #   start_node: fred, end_node: quarry, employee_role: 'Crane Operator'
    # )
    # SELECT *
    # FROM cypher('age_schema', $$
    #     MATCH (start_vertex:Person), (end_vertex:Company)
    #     WHERE id(start_vertex) = 1125899906842634 and id(end_vertex) = 844424930131976
    #     CREATE (start_vertex)-[edge:WorksAt {employee_role: 'Crane Operator'}]->(end_vertex)
    #     RETURN edge
    # $$) as (edge agtype);
    def create_sql
      self.start_node = start_node.save unless start_node.persisted?
      self.end_node = end_node.save unless end_node.persisted?
      <<-SQL
        SELECT *
        FROM cypher('#{age_graph}', $$
            MATCH (from_node:#{start_node.age_label}), (to_node:#{end_node.age_label})
            WHERE id(from_node) = #{start_node.id} and id(to_node) = #{end_node.id}
            CREATE (from_node)-[edge#{self}]->(to_node)
            RETURN edge
        $$) as (edge agtype);
      SQL
    end

    # So far just properties of string type with '' around them
    def update_sql
      alias_name = age_alias || age_label.downcase
      set_caluse =
        age_properties.map { |k, v| v ? "#{alias_name}.#{k} = '#{v}'" : "#{alias_name}.#{k} = NULL" }.join(', ')
      <<-SQL
        SELECT *
        FROM cypher('#{age_graph}', $$
            MATCH ()-[#{alias_name}:#{age_label}]->()
            WHERE id(#{alias_name}) = #{id}
            SET #{set_caluse}
            RETURN #{alias_name}
        $$) as (#{age_label} agtype);
      SQL
    end
  end
end
```

```ruby
# app/lib/apache_age/entity.rb
module ApacheAge
  class Entity
    class << self
      def find_by(attributes)
        where_clause = attributes.map { |k, v| "find.#{k} = '#{v}'" }.join(' AND ')
        handle_find(where_clause)
      end

      def find(id)
        where_clause = "id(find) = #{id}"
        handle_find(where_clause)
      end

      private

      def age_graph = 'age_schema'

      def handle_find(where_clause)
        # try to find a vertex
        match_node = '(find)'
        cypher_sql = find_sql(match_node, where_clause)
        age_response = execute_find(cypher_sql)

        if age_response.nil?
          # if not a vertex try to find an edge
          match_edge = '()-[find]->()'
          cypher_sql = find_sql(match_edge, where_clause)
          age_response = execute_find(cypher_sql)
          return nil if age_response.nil?
        end

        instantiate_result(age_response)
      end

      def execute_find(cypher_sql)
        age_result = ActiveRecord::Base.connection.execute(cypher_sql)
        return nil if age_result.values.first.nil?

        age_result
      end

      def instantiate_result(age_response)
        age_type = age_response.values.first.first.split('::').last
        json_string = age_response.values.first.first.split('::').first
        json_data = JSON.parse(json_string)

        age_label = json_data['label']
        attribs = json_data.except('label', 'properties')
                           .merge(json_data['properties'])
                           .symbolize_keys

        "#{json_data['label'].gsub('__', '::')}".constantize.new(**attribs)
      end

      def find_sql(match_clause, where_clause)
        <<-SQL
          SELECT *
          FROM cypher('#{age_graph}', $$
              MATCH #{match_clause}
              WHERE #{where_clause}
              RETURN find
          $$) as (found agtype);
        SQL
      end
    end
  end
end
```

```ruby
# app/lib/apache_age/class_methods.rb
module ApacheAge
  module ClassMethods
    # for now we only allow one predestined graph
    def create(attributes) = new(**attributes).save

    def find_by(attributes)
      where_clause = attributes.map { |k, v| "find.#{k} = '#{v}'" }.join(' AND ')
      cypher_sql = find_sql(where_clause)
      execute_find(cypher_sql)
    end

    def find(id)
      where_clause = "id(find) = #{id}"
      cypher_sql = find_sql(where_clause)
      execute_find(cypher_sql)
    end

    def all
      age_results = ActiveRecord::Base.connection.execute(all_sql)
      return [] if age_results.values.count.zero?

      age_results.values.map do |result|
        json_string = result.first.split('::').first
        hash = JSON.parse(json_string)
        attribs = hash.except('label', 'properties').merge(hash['properties']).symbolize_keys

        new(**attribs)
      end
    end

    # Private stuff

    def age_graph = 'age_schema'
    def age_label = name.gsub('::', '__')
    def age_type = name.constantize.new.age_type

    def match_clause
      age_type == 'vertex' ? "(find:#{age_label})" : "()-[find:#{age_label}]->()"
    end

    def execute_find(cypher_sql)
      age_result = ActiveRecord::Base.connection.execute(cypher_sql)
      return nil if age_result.values.count.zero?

      age_type = age_result.values.first.first.split('::').last
      json_data = age_result.values.first.first.split('::').first

      hash = JSON.parse(json_data)
      attribs = hash.except('label', 'properties').merge(hash['properties']).symbolize_keys

      new(**attribs)
    end

    def all_sql
      <<-SQL
      SELECT *
      FROM cypher('#{age_graph}', $$
          MATCH #{match_clause}
          RETURN find
      $$) as (#{age_label} agtype);
      SQL
    end

    def find_sql(where_clause)
      <<-SQL
        SELECT *
        FROM cypher('#{age_graph}', $$
            MATCH #{match_clause}
            WHERE #{where_clause}
            RETURN find
        $$) as (#{age_label} agtype);
      SQL
    end
  end
end
```

```ruby
# app/lib/apache_age/common_methods.rb
module ApacheAge
  module CommonMethods
    def initialize(**attributes)
      super
      return self unless age_type == 'edge'

      self.end_id ||= end_node.id if end_node
      self.start_id ||= start_node.id if start_node
      self.end_node ||= Entity.find(end_id) if end_id
      self.start_node ||= Entity.find(start_id) if start_id
    end

    # for now we just can just use one schema
    def age_graph = 'age_schema'
    def age_label = self.class.name.gsub('::', '__')
    def persisted? = id.present?
    def to_s = ":#{age_label} #{properties_to_s}"

    def to_h
      base_h = attributes.to_hash
      if age_type == 'edge'
        # remove the nodes (in attribute form and re-add in hash form)
        base_h = base_h.except('start_node', 'end_node')
        base_h[:end_node] = end_node.to_h if end_node
        base_h[:start_node] = start_node.to_h if start_node
      end
      base_h.symbolize_keys
    end

    def update_attributes(attribs)
      attribs.except(id:).each do |key, value|
        send("#{key}=", value) if respond_to?("#{key}=")
      end
    end

    def update(attribs)
      update_attributes(attribs)
      save
    end

    def save
      return false unless valid?

      cypher_sql = (persisted? ? update_sql : create_sql)
      response_hash = execute_sql(cypher_sql)

      self.id = response_hash['id']

      if age_type == 'edge'
        self.end_id = response_hash['end_id']
        self.start_id = response_hash['start_id']
        # reload the nodes? (can we change the nodes?)
        # self.end_node = ApacheAge::Entity.find(end_id)
        # self.start_node = ApacheAge::Entity.find(start_id)
      end

      self
    end

    def destroy
      match_clause = (age_type == 'vertex' ? "(done:#{age_label})" : "()-[done:#{age_label}]->()")
      delete_clause = (age_type == 'vertex' ? 'DETACH DELETE done' : 'DELETE done')
      cypher_sql =
        <<-SQL
        SELECT *
        FROM cypher('#{age_graph}', $$
            MATCH #{match_clause}
            WHERE id(done) = #{id}
 	          #{delete_clause}
            return done
        $$) as (deleted agtype);
        SQL

      hash = execute_sql(cypher_sql)
      return nil if hash.blank?

      self.id = nil
      self
    end
    alias destroy! destroy
    alias delete destroy

    # private

    def age_properties
      attrs = attributes.except('id')
      attrs = attrs.except('end_node', 'start_node', 'end_id', 'start_id') if age_type == 'edge'
      attrs.symbolize_keys
    end

    def age_hash
      hash =
        {
          id:,
          label: age_label,
          properties: age_properties
        }
      hash.merge!(end_id:, start_id:) if age_type == 'edge'
      hash.transform_keys(&:to_s)
    end

    def properties_to_s
      string_values =
        age_properties.each_with_object([]) do |(key, val), array|
          array << "#{key}: '#{val}'"
        end
      "{#{string_values.join(', ')}}"
    end

    def age_alias
      return nil if id.blank?

      # we start the alias with a since we can't start with a number
      'a' + Digest::SHA256.hexdigest(id.to_s).to_i(16).to_s(36)[0..9]
    end

    def execute_sql(cypher_sql)
      age_result = ActiveRecord::Base.connection.execute(cypher_sql)
      age_type = age_result.values.first.first.split('::').last
      json_data = age_result.values.first.first.split('::').first
      # json_data = age_result.to_a.first.values.first.split("::#{age_type}").first

      JSON.parse(json_data)
    end
  end
end
```

### Implementation Classes

```ruby
# app/graphs/nodes/company.rb
module Nodes
  class Company
    include ApacheAge::Vertex

    attribute :company_name, :string
    validates :company_name, presence: true
  end
end
```

### Usage

This will be described in detail in the article:

but here is a simple usage example:
```ruby
bedrock_quarry = Nodes::Company.create(company_name: 'Bedrock Quarry Company')
```

## Resources

### Graph App Example Resources

* [Building a Real-Time Product Recommender System with Graph Databases: Leveraging Neo4j and BigQuery for E-commerce Data Analysis](https://medium.com/badal-io/building-a-real-time-product-recommender-system-with-graph-databases-leveraging-neo4j-and-bigquery-65b5b361d276)
* [Recommendation System Using Graph Database](https://47billion.com/blog/recommendation-system-using-graph-database/)
* [How to Build a Real-Time Recommendation Engine Using Graph Databases](https://www.kdnuggets.com/2023/08/build-realtime-recommendation-engine-graph-databases.html)
* [Advanced Usage within multiple Graphs, Use Cases & Demo](https://apache-age.medium.com/webinar-demonstration-of-apache-age-a-graph-extension-for-postgresql-db82b0cfc5d8)
* [Neo4j movies example](https://www.youtube.com/watch?v=YDWkPFijKQ4)
* [Neo4j Recommendation Engine Example](https://www.youtube.com/watch?v=QIrxsNQ9JEs)
* [Oracle Graph Recommendations](https://www.youtube.com/watch?v=eqQ5nEY5FEg)
* [ArangoDB Better Recommendations Example](https://www.youtube.com/watch?v=G4KVAnZoyRg)
* https://apache-age.medium.com/webinar-demonstration-of-apache-age-a-graph-extension-for-postgresql-db82b0cfc5d8
* https://www.fabiomarini.net/postgresql-with-apache-age-playing-more-seriously-with-graph-databases/

### Graph DB Design

* https://github.com/neo4jrb/activegraph
* https://apache-age.medium.com/what-is-data-modeling-graph-db-86ccd7b5989e

### Apache AGE SQL / Migration / Indexes

* https://github.com/apache/age/issues/45
* https://stackoverflow.com/questions/75903997/how-can-i-declare-primary-key-in-apache-age


### Rails with Apache AGE

* [Working with multi-schema database in Rails](https://learnitnow.medium.com/working-with-multi-schema-database-in-rails-7e60aef8ff86)
* [Using multiple PostgreSQL schemas with Rails models](https://stackoverflow.com/questions/8806284/using-multiple-postgresql-schemas-with-rails-models)
* [Rails Guides: Multiple Databases with Active Record](https://guides.rubyonrails.org/active_record_multiple_databases.html#connecting-to-databases-without-managing-schema-and-migrations)

### AGE Management

* https://matheusfarias03.github.io/AGE-quick-guide/

### AGE SQL

* https://dev.to/danielwambo/exploring-apache-age-a-graph-database-built-on-postgresqlapache-age-cypher-postgres-346a
* https://dev.to/danielwambo/apache-age-a-technical-deep-dive-48n

### INTRESTSING UNTESTED RESOURCES

* https://blog.howtoclicks.com/blog/apache-age-example/
* https://dev.to/abhikmr2046/creating-new-function-in-apache-age-ioj
* https://dev.to/k1hara/creating-user-defined-functions-in-apache-age-1hha


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


### BiDirectional Edges Exploration

```ruby
module Edge
  class MarriedTo
    attr_reader :husband, :wife, :husbands_marriage, :wifes_marriage

    AGE_TYPE = 'edge'.freeze
    CLASS_NAME = 'Person'.freeze
    GRAPH_NAME = 'age_schema'.freeze

    def initialize(husband:, wife:)
      @husband = husband
      @wife = wife
    end

    def create
      @husband = husband.create unless husband.persisted?
      @wife = wife.create unless wife.persisted?

      age_results = create_sql

      json_data = age_result.to_a.first[AGE_TYPE].split('::vertex')
    end

    def create_sql
      <<-SQL
        SELECT *
        FROM cypher('age_schema', $$
            MATCH (husband:Person), (wife:Person)
            WHERE id(husband) = #{husband.id} and id(wife) = #{wife.id}
            CREATE path = (husband)-[husbands_marriage:MarriedTo {role: "husband"}]->(wife:Person)-[wifes_marriage:MarriedTo {role: 'wife'}]->(husband)
            RETURN path
        $$) as (path agtype);
      SQL
    end
  end
end


fred = Person.new(first_name: 'Fred', last_name: 'Flintstone', gender: 'male')
result = create_person(fred)

wilma = Person.new(first_name: 'Wilma', last_name: 'Flintstone', given_name: 'Slaghoople', gender: 'female')
result = create_person(wilma)
```

## Make some Models

lets put our nodes in a different folder than `models` which we can use for full pledged `ActiveRecords`

```
mkdir app/age_nodes
mkdir app/age_nodes/base_node.rb
touch app/age_nodes/person.rb
```

consider:
https://stackoverflow.com/questions/8806284/using-multiple-postgresql-schemas-with-rails-models
```
PostgreSQL adapter schema_search_path in database.yml does solve your problem?

development:
  adapter: postgresql
  encoding: utf-8
  database: solidus
  host: 127.0.0.1
  port: 5432
  username: postgres
  password: postgres
  schema_search_path: "discogs,public"
Or, you can to specify different connections for each schema:

public_schema:
  adapter: postgresql
  encoding: utf-8
  database: solidus
  host: 127.0.0.1
  port: 5432
  username: postgres
  password: postgres
  schema_search_path: "public"

discogs_schema:
  adapter: postgresql
  encoding: utf-8
  database: solidus
  host: 127.0.0.1
  port: 5432
  username: postgres
  password: postgres
  schema_search_path: "discogs"
After each connection defined, create two models:

class PublicSchema < ActiveRecord::Base
  self.abstract_class = true
  establish_connection :public_schema
end

class DiscoGsSchema < ActiveRecord::Base
  self.abstract_class = true
  establish_connection :discogs_schema
end
And, all your models inherit from the respective schema:

class MyModelFromPublic < PublicSchema
  set_table_name :my_table_name
end

class MyOtherModelFromDiscoGs < DiscoGsSchema
  set_table_name :disco
end
```


### APPENDIX (Advanced Usage)

https://matheusfarias03.github.io/AGE-quick-guide/

```
AGE-quick-guide
Apache AGE Quick Guide
Introduction
This is a quick guide I made for the Apache AGE. It sums up some things I have read from different sources to have a concise understanding on how to use AGE, how it works internally, and how to create some data with it. This guide shows how to set AGE with PostgreSQL, the internals of this extension and some Cypher queries to use with it. Documentations and further explanation of the topics covered here can be viewed in the links provided at the “References” section.

Overview
Apache AGE is a PostgreSQL extension that provides graph database functionality. AGE is an acronym for “A Graph Extension”, and is inspired by Bitnine’s fork of PostgreSQL 10, AgensGraph, which is a multi-model database. The goal of AGE is to create a single storage that can handle both relational and graph model data so that users can use standard ANSI SQL along with openCypher, the Graph query language.

Setup
First of all, you will need to install the essential libraries according to each OS. Building AGE from source depends on the following Linux libraries:

Ubuntu : sudo apt-get install build-essential libreadline-dev zlib1g-dev flex bison ;
Fedora : dnf install gcc glibc bison flex readline readline-devel zlib zlib-devel ;
CentOS : yum install gcc glibc glib-common readline readline-devel zlib zlib-devel flex bison .
You will also need to install a AGE compatible version of Postgres. For now, AGE only supports Postgres 11 and the ALPHA of 12. To install Postgres from a package manager:

Postgres 11 : sudo apt install postgresql-server-dev-11
Postgres 12 : sudo apt install postgresql-12
Then, clone the github repository at : https://github.com/apache/agex

Now, inside the AGE directory, build and install the extension with the following command :

make PG_CONFIG=/path/to/postgres/bin/pg_config install
Post Installation AGE Setup
After the installation, open a connection to a running instance of your database and run the CREATE EXTENSION command to have AGE be installed on the server.

CREATE EXTENSION age;
Per Session Instructions
For every connection of AGE you start you will need to load AGE extension.

LOAD 'age';
It is recommended adding ag_catalog to the sarch_path to simplify the queries. If not done so, remember to add ‘ag_catalog’ to your cypher query function calls.

SET search_path = ag_catalog, "$user", public;
Graphs
A graph consist of a set of vertices and edges, where each individual node and edge possesses a map properties. A vertex is the basic object of a graph, that can exist independently of everything else in the graph. An edge creates a directed connection between two vertices.

Vertex
An agtype vertex is its graphid, label, and it’s properties:

{
    "id" : 1125899906842777,
    "label": "city",
    "properties": {"key": "value"}
}::vertex
Edge
An edge is its graphid, label, the graphid of its start and end vertex and its properties:

{
    "id": 1125899906842777,
    "label": "has_city",
    "start_id": 1125899906842777,
    "end_id": 1125899906842777,
    "properties": {"key": "value"}
}::edge
Path
A path is a list of alternating vertices and edges:

[vertex, edge, vertex... etc]::path
Creating a Graph
To create a graph, use the create_graph function, located in the ag_catalog namespace. This function will not return any results. The graph is created if there is no error message. Tables needed to setup the graph are autmoatically created.

Example:

SELECT * FROM ag_catalog.create_graph('graph_name');
Deleting a Graph
Use drop_graph function to delete the graph. It is located in the ag_catalog namespace. This function will not return any results. If there is no error message the graph has been deleted. It is recommended to set the cascade option to true, otherwise everything in the graph must be manually dropped with SQL DDL commands.

Example:

SELECT * FROM ag_catalog.drop_graph('graph_name', true);
How Data is Stored in Apache AGE
Graphs
AGE uses a Postgres namespace for every individual graph. Data for a graph is stored in the table ag_catalog.ag_graph. Running the query:

SELECT oid, * FROM ag_catalog.ag_graph;
will return the following result:

   oid   |    name   |     namespace     |
---------+-----------+-------------------+
  16937  |  database |     database      |
(1 row)
- oid is the Postgres Object Identifier, which are 4-byte integers. All the database objects in PostgreSQL are internally managed by respective OIDs.

Labels
A label is a table that holds all vertices/edges associated with the label. The data for a label is stored in the ag_label. By typing the following query, you can view the available labels:

SELECT oid, * FROM ag_catalog.ag_label;
    oid   |        name       | graph | id | kind |        relation          |
----------+-------------------+-------+----+------+--------------------------+
   16950  | _age_label_vertex | 16937 |  1 | v    | database.ag_label_vertex |
   16963  | _age_label_edge   | 16937 |  2 | e    | database.ag_label_edge   |
   16975  | Country           | 16937 |  3 | v    | database."Country"       |
   16987  | City              | 16937 |  4 | v    | database."City"          |
   16999  | has_city          | 16937 |  5 | e    | database.has_city        |
With each column of the table above, we can identify some field names:

oid : An unique Postgres identifier per label ;
name: The name of the label ;
graph: Oid from ag_graph ;
id : The id for the label, it is unique for each graph ;
kind : Shows if it is a vertex “v” or an edge “e” ;
relation : Schema qualified name of table name for label.
Vertices
Vertices have unique identifiers and a set of properties associated with them. You can see all the available vertices of a selected label with the following query (The “City” is just an example of a label and “database” the name of the database you are working with):

SELECT * FROM database."City";
With this query, it will show the id and properties as columns of each vertex. The id will be an integer number and the properties will be stored as a JSON object with key : value pairs.

Edges
Each edge has an unique identifier, the identifier of the vertices it is associated with and it’s properties. Below is an example on how to show the edges available on the graph:

SELECT * FROM cypher_vle.edge;
This query will show the id of the edge, start_id, end_id, and properties. The start_id and end_id will show the id of the vertex it starts with and ends with, respectively.

Cypher
Cypher is a declarative graph query language that allows data querying in a graph. Cypher queries are assembled with patterns of nodes and relationships with any specific filter on label or properties to use the CRUD methods.

Like SQL, Cypher has a variety of keywords for specifying, filtering patterns, and returning results. The most common are: MATCH, WHERE and RETURN.

MATCH is used for finding nodes, relationships, or combinations of nodes and relationships together ;
WHERE in cypher is used to add additional constraints to patterns and filter unwanted patterns ;
RETURN in cypher formats and organizes the way that the result will be outputted.
An example of a cypher query would be the following:

MATCH (jorge:Actor {name: 'Jorge Mário da Silva'})-[:ACTED_IN]->(movie:Movie)
WHERE movie.year < $yearParameter
RETURN movie
This example will search for the pattern of the node with the Actor label and with the name “Jorge Mário da Silva” connected by a relationship (ACTED_IN) to another node, in this case, a node with the “Movie” label. The WHERE clause filters to only keep patterns where the Movie node in the MATCH clause has a year property that is less than the value passed in $yearParameter.

- A more indepth information about cypher syntax and a guide about it can be accessed on: https://neo4j.com/developer/cypher/intro-cypher/

Using Cypher with AGE
Cypher queries are constructed using a function called cypher in ag_catalog, which returns a Postgres SETOF records. A RECORD in Postgres is a placeholder data type that allows the user to define the output columns at runtime. And SETOF means that multiple rows might be returned.

The syntax for using cypher with AGE is: cypher(graph_name, query_string, parameters).

graph_name is the graph for the Cypher query ;
query_string is the Cypher query to be executed ;
parameters is an optional map of parameters used for Prepared Statements. Default is NULL.
The query utilising this syntax would be something like this:

SELECT * FROM cypher(
  'graph_name',
  $$ /*Cypher query here*/ $$
) AS (result1 agtype, result2 agtype);
Note that the “AS (result1 agtype, result2 agtype)” defines the RECORD of the query.

CRUD
CREATE, READ, UPDATE, DELETE (CRUD) are the four basic operations of persistent storage. In this section we can see some examples of the CRUD operations with Cypher on AGE. More detailed explanations are available on the links at the end of each operation.

Create
The CREATE clause is used to create graph vertices and edges. A create clause that is not followed by another clause is called a terminal clause. When a cypher query ends with a terminal clause, no results will be returned from the cypher function call. However, the cypher function call still requires a column list definition. When cypher ends with a terminal node, define a single value in the column list definition: no data will be returned in this variable.

A query for creating a new vertex with labels can be done as:

SELECT *
FROM cypher(
  'graph_name',
  $$ CREATE (:Person {name: 'Dennis Ritchie', title: 'Programmer'})$$
) as (n agtype);
To create an edge between two vertices, we first get the two vertices. Once the nodes are loaded, we simply create an edge between them.

SELECT * FROM cypher(
  'graph_name',
  $$ MATCH (a:Person), (b:Person)
  WHERE a.name = 'Node A' AND b.name = 'Node B'
  CREATE (a)-[e:RELTYPE]->(b)
  RETURN e $$
) as (e agtype);
- More about the CREATE clause with AGE on: https://age.apache.org/age-manual/master/clauses/create.html

Read
The MATCH clause allows you to specify the patterns Cypher will search for in the database. This is the primary way of getting data into the current set of bindings.

By just specifying a pattern with a single vertex and no labels, all vertices in the graph will be returned.

SELECT * FROM cypher(
  'graph_name',
  $$ MATCH (v)
  RETURN v $$
) as (v agtype);
To return related vertices within a table, the -[]- symbol is used. It means “related to”, without regarding to the type or relation to the edge.

SELECT * FROM cypher(
  'graph_name',
  $$ MATCH (director {name: 'John Carpenter'})-[]-(movie)
  RETURN movie.title $$
) as (title agtype);
- More about the MATCH clause with AGE on: https://age.apache.org/age-manual/master/clauses/match.html

Update
The SET clause is used to update labels on nodes and properties on vertices and edges. To remove a property, normally it is used REMOVE, but sometimes the SET command also works.

An example of the SET clause with cypher on AGE:

SELECT *
FROM cypher(
  'graph_name',
  $$ MATCH (v {name: 'Andres'})
  SET v.surname = 'Taylor'
  RETURN v $$
) as (v agtype);
An example of the REMOVE clause with cypher on AGE:

SELECT *
FROM cypher('graph_name', $$
    MATCH (andres {name: 'Andres'})
    REMOVE andres.age
    RETURN andres
$$) as (andres agtype);
- More about the SET clause at: https://age.apache.org/age-manual/master/clauses/set.html

- More about the REMOVE clause at: https://age.apache.org/age-manual/master/clauses/remove.html

Delete
The DELETE clause is used to delete graph elements—nodes, relationships or paths. You cannot delete a node without also deleting edges that start or end on said vertex. Either explicitly delete the vertices,or use DETACH DELETE.

Deleting single vertex :

SELECT *
FROM cypher('graph_name', $$
	MATCH (v:Useless)
	DELETE v
$$) as (v agtype);
Deleting all vertices (use the DETACH option to first delete a vertice’s edges then delete the vertex itself) :

SELECT *
FROM cypher('graph_name', $$
	MATCH (v:Useless)
	DETACH DELETE v
$$) as (v agtype);
-More about the DELETE clause at: https://age.apache.org/age-manual/master/clauses/delete.html

References
Internals of PostgreSQL - https://www.interdb.jp/pg/index.html
Apache AGE Documentation - https://age.apache.org/age-manual/master
Neo4j Intro to Cypher - https://neo4j.com/developer/cypher/intro-cypher/
Cypher on Wikipedia - https://en.wikipedia.org/wiki/Cypher_(query_language)
```
