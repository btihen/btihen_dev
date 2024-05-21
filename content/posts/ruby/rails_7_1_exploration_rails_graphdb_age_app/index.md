---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x - GraphDB App with AGE"
subtitle: "Using AGE in a Rails App"
summary: ""
authors: ['btihen']
tags: ['Rails', "GraphDB", "Postgres AGE"]
categories: ["Code", "GraphDB", "Ruby Language", "Rails Framework"]
date: 2024-05-20T01:20:00+02:00
lastmod: 2024-05-20T01:20:00+02:00
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

This code can be found at: https://github.com/btihen-dev/rails_graphdb_age_app

## Getting Started

I will assume you have the AGE extension already installed in your Postgres instance and you understand OpenCypher at least casually.  If not please first visit: https://btihen.dev/posts/tech/graphdb_getting_started_age_1_5_0/

As we have done in a few apps, we will build a Flintstone's App.
Let's create our rails app:
```
rails _7.1.3.2_ new graphdb_age -T -d postgresql
cd graphdb_age
```

To do this config we need to update our config & create migrations

### Apache AGE setup

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

## Apache AGE Test

access using:
```bash
# simplify the login if you wish
export PGPASSWORD=postgresPW

# now access the database
psql -d myPostgresDb -U postgresUser -h localhost -p 5455
```

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

## Rails Setup

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

now let's check our setup with:
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

We can ignore the warnings:
```
unknown OID 4089: failed to recognize type of 'namespace'. It will be treated as String.
unknown OID 2205: failed to recognize type of 'relation'. It will be treated as String.
```

Now add `schema_search_path: 'ag_catalog,age_schema,public'` to the database config:

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

Bow we need to fix the `db/schema.rb` file:
```ruby
# db/schema.rb
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

this allows us to run tests.

## Overview

I feel like it makes sense to separate the node and edges (but this isn't necessary). Here is the structure I will use:
```
├── app
│   ...
│   ├── controllers
│   │   ├── application_controller.rb
│   │   ├── concerns
│   │   └── people_controller.rb
│   ├── graphs
│   │   ├── edges
│   │   │   └── works_at.rb
│   │   └── nodes
│   │       ├── company.rb
│   │       ├── person.rb
│   │       └── pet.rb
│   ...
│   ├── lib
│   │   └── apache_age
│   │       ├── class_methods.rb
│   │       ├── common_methods.rb
│   │       ├── edge.rb
│   │       ├── entity.rb
│   │       └── vertex.rb
│   ...
│   └── views
│       ├── layouts
│       │   ├── application.html.erb
│       │   ├── mailer.html.erb
│       │   └── mailer.text.erb
│       └── people
│           ├── _form.html.erb
│           ├── _person.html.erb
│           ├── edit.html.erb
│           ├── index.html.erb
│           ├── new.html.erb
│           ├── show.html.erb
```

## Graph Models

**Edges**

```ruby
# app/graphs/edges/works_at.rb
module Edges
  class WorksAt
    include ApacheAge::Edge

    attribute :employee_role, :string
    validates :employee_role, presence: true
  end
end
```

**Nodes**

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

and

```ruby
# app/graphs/nodes/person.rb
module Nodes
  class Person
    include ApacheAge::Vertex

    attribute :first_name, :string
    attribute :last_name, :string
    attribute :given_name, :string
    attribute :gender, :string

    validates :gender, :first_name, :last_name, :given_name,
              presence: true

    def initialize(**attributes)
      super
      # use unless present? since attributes when empty sets to "" by default
      self.nick_name = first_name unless nick_name.present?
      self.given_name = last_name unless given_name.present?
    end
  end
end
```

## Router

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :people

  get 'up' => 'rails/health#show', as: :rails_health_check

  root 'people#index'
end
```

## contoller

```ruby
class PeopleController < ApplicationController
  before_action :set_person, only: %i[show edit update destroy]

  # GET /people or /people.json
  def index
    @people = Nodes::Person.all
  end

  # GET /people/1 or /people/1.json
  def show; end

  # GET /people/new
  def new
    @person = Nodes::Person.new
  end

  # GET /people/1/edit
  def edit; end

  # POST /people or /people.json
  def create
    @person = Nodes::Person.new(**person_params)
    respond_to do |format|
      if @person.save
        format.html { redirect_to person_url(@person), notice: 'Person was successfully created.' }
        format.json { render :show, status: :created, location: @person }
      else
        format.html { render :new, status: :unprocessable_entity }
        format.json { render json: @person.errors, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /people/1 or /people/1.json
  def update
    respond_to do |format|
      if @person.update(**person_params)
        format.html { redirect_to person_url(@person), notice: 'Person was successfully updated.' }
        format.json { render :show, status: :ok, location: @person }
      else
        format.html { render :edit, status: :unprocessable_entity }
        format.json { render json: @person.errors, status: :unprocessable_entity }
      end
    end
  end

  # DELETE /people/1 or /people/1.json
  def destroy
    @person.destroy!

    respond_to do |format|
      format.html { redirect_to people_url, notice: 'Person was successfully destroyed.' }
      format.json { head :no_content }
    end
  end

  private

  # Use callbacks to share common setup or constraints between actions.
  def set_person
    @person = Nodes::Person.find(params[:id])
  end

  # Only allow a list of trusted parameters through.
  def person_params
    # params.fetch(:person, {})
    params.require(:nodes_person).permit(:first_name, :last_name, :nick_name, :given_name, :gender)
  end
end
```

## Views

Note you need to use:
`<%= form_with(model: person, url: form_url) do |form| %>`

```ruby
# app/views/people/_form.html.erb
<%= form_with(model: person, url: form_url) do |form| %>
  <% if person.errors.any? %>
    <div style="color: red">
      <h2><%= pluralize(person.errors.count, "error") %> prohibited this person from being saved:</h2>

      <ul>
        <% person.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= form.label :first_name, style: "display: block" %>
    <%= form.text_field :first_name %>
  </div>

  <div>
    <%= form.label :nick_name, style: "display: block" %>
    <%= form.text_field :nick_name %>
  </div>
  <div>
    <%= form.label :last_name, style: "display: block" %>
    <%= form.text_field :last_name %>
  </div>

  <div>
    <%= form.label :given_name, style: "display: block" %>
    <%= form.text_field :given_name %>
  </div>

  <div>
    <%= form.label :gender, style: "display: block" %>
    <%= form.text_field :gender %>
  </div>

  <div>
    <%= form.submit %>
  </div>
<% end %>
```

```ruby
# app/views/people/_person.html.erb
<div id="<%= dom_id person %>">
  <p>
    <strong>First Name:</strong>
    <%= person.first_name %>
  </p>
  <p>
    <strong>Nick Name:</strong>
    <%= person.nick_name %>
  </p>
  <p>
    <strong>Last Name:</strong>
    <%= person.last_name %>
  </p>
  <p>
    <strong>Given Name:</strong>
    <%= person.given_name %>
  </p>
  <p>
    <strong>Gender:</strong>
    <%= person.gender %>
  </p>
</div>
```

note you need to use a render with:
`<%= render "form", person: @person, form_url: person_path(@person) %>`
so it will look like:

```ruby
# app/views/people/edit.html.erb
<h1>Editing person</h1>

<!-- # person_path(@person)  PATCH /people/:id(.:format)  people#update -->
<%= render "form", person: @person, form_url: person_path(@person) %>

<br>

<div>
  <%= link_to "Show this person", person_path(@person) %> |
  <%= link_to "Back to people", people_path %>
</div>
```

```ruby
# app/views/people/index.html.erb
<p style="color: green"><%= notice %></p>

<h1>People</h1>

<div id="people">
  <% @people.each do |person| %>
    <%= render 'person', person: person %>
    <p>
      <%= link_to "Show this person", person_path(person) %>
    </p>
    <p>
      <%= link_to "Edit this person", edit_person_path(person) %>
    </p>
    <hr>
  <% end %>
</div>

<%= link_to "New person", new_person_path %>
```

```ruby
# app/views/people/new.html.erb
<h1>New person</h1>

<!-- # people_path(@person) PUT /people_path => people#update -->
<%= render "form", person: @person, form_url: people_path(@person) %>

<br>

<div>
  <%= link_to "Back to people", people_path %>
</div>
```

```ruby
# app/views/people/show.html.erb
<p style="color: green"><%= notice %></p>

<%= render 'person', person: @person %>

<div>
  <%= link_to "Edit this person", edit_person_path(@person) %> |
  <%= link_to "Back to people", people_path %>

  <%= button_to("Destroy this person", person_path(@person), method: :delete, data: { confirm: 'Are you sure?' }) %>
</div>

```

## Library Code -- Appendix

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

## Resources

### Graph DB Design

* https://github.com/neo4jrb/activegraph
* https://apache-age.medium.com/what-is-data-modeling-graph-db-86ccd7b5989e

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
