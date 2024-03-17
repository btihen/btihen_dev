---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.x API Exploration"
subtitle: "Exploring a JSON API in Rails 7.1.x"
summary: ""
authors: ['btihen']
tags: ['Rails API']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2024-03-17T01:20:00+02:00
lastmod: 2024-03-17T01:20:00+02:00
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

I was recently asked about Rails API applications and considerations.  I thought it would be interesting to explore some ideas.  Here is what I explored.

This code can be found at: (coming soon)

## Getting Started

```bash
bin/rails new flintstones_api --api
cd flintstones_api
bin/rails g scaffold Person first_name last_name given_last_name gender
bin/rails g model PersonRelationships role_one role_two \
                  person_one:references person_two:references
```

Let's update the migration to make the first & last name required as well as gender

```ruby
class CreatePeople < ActiveRecord::Migration[7.1]
  def change
    create_table :people do |t|
      t.string :first_name, null: false
      t.string :last_name, null: false
      t.string :given_last_name
      t.string :gender, null: false

      t.timestamps
    end
  end
end
```

Let's create a quick test in our seed file.

```ruby
zed = Person.create!(first_name: "Zed", last_name: "Flintstone", gender: 'male')
jed = Person.create!(first_name: "Jed", last_name: "Flintstone", gender: 'male')
rock = Person.create!(first_name: "Rockbottom", last_name: "Flintstone", gender: 'male')
```

now migrate and make seeds

```bash
bin/rails db:migrate
bin/rails db:seed
```

## NAMESPACE REST API

Now that we have a basic setup that works lets setup the API code.

I like to namespace to allow for easy versioning.

```bash
mkdir app/controllers/api
mkdir app/controllers/api/v1
mv app/controllers/people_controller.rb app/controllers/api/v1/.
```

now that we have the new location and moved our `people_controller` there.  Let's fix it's namespace using:

```ruby
module Api
  module V1
	  class PeopleController < ApplicationController
      # ...
	  end
  end
end
```

now it looks like:
```ruby
module Api
  module V1
    class PeopleController < ApplicationController
      before_action :set_person, only: %i[ show update destroy ]

      # GET /people
      def index
        @people = Person.all

        render json: @people
      end

      # GET /people/1
      def show
        render json: @person
      end

      # POST /people
      def create
        @person = Person.new(person_params)

        if @person.save
          render json: @person, status: :created, location: @person
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # PATCH/PUT /people/1
      def update
        if @person.update(person_params)
          render json: @person
        else
          render json: @person.errors, status: :unprocessable_entity
        end
      end

      # DELETE /people/1
      def destroy
        @person.destroy!
      end

      private

      # Use callbacks to share common setup or constraints between actions.
      def set_person
        @person = Person.find(params[:id])
      end

      # Only allow a list of trusted parameters through.
      def person_params
        params.require(:person).permit(:first_name, :last_name)
      end
    end
  end
end
```

now let's update our routes to reflect our API namespace in the url path too using:

```ruby
Rails.application.routes.draw do
	namespace :api do
	  namespace :v1 do
		  resources :people
	  end
	end

  # Reveal health status on /up that returns 200 if the app boots with no exceptions, otherwise 500.
  # Can be used by load balancers and uptime monitors to verify that the app is live.
  get "up" => "rails/health#show", as: :rails_health_check

  # Defines the root path route ("/")
  # root "posts#index"
end
```

Let's test:

```bash
bin/rails s -p 3030

curl http://localhost:3030/api/v1/people.json

#this returns way to much info:
[{"id":1,"first_name":"Zed","last_name":"Flintstone","given_last_name":null,"gender":"male","created_at":"2024-03-16T15:29:19.306Z","updated_at":"2024-03-16T15:29:19.306Z"},
{"id":2,"first_name":"Jed","last_name":"Flintstone","given_last_name":null,"gender":"male","created_at":"2024-03-16T15:29:19.308Z","updated_at":"2024-03-16T15:29:19.308Z"},
{"id":3,"first_name":"Rockbottom","last_name":"Flintstone","given_last_name":null,"gender":"male","created_at":"2024-03-16T15:29:19.317Z","updated_at":"2024-03-16T15:29:19.317Z"},
...
```


Let's change the controller to return only: id, first_name, last_name, and gender by changing our select from `Person.all` to `Person.select(:id, :first_name, :last_name, :gender, :given_last_name).all` or `render json: @people.as_json(only: [:id, :first_name, :last_name, :gender, :given_last_name])`.

```ruby
# app/controllers/api/v1/people_controller.rb
      def index
        @people = Person.all

        render json: @people.as_json(only: [:id, :first_name, :last_name, :gender, :given_last_name])
      end

      def show
        render json: @person.as_json(only: [:id, :first_name, :last_name, :gender, :given_last_name])
      end

      # ...
```

For completeness we need to ensure all our critical attributes are listed for the create/update to work (we need `:gender` and `given_last_name`).

```ruby
# app/controllers/api/v1/people_controller.rb
    private

    # Only allow a list of trusted parameters through.
    def person_params
      params.require(:person).permit(:first_name, :last_name, :gender, :given_last_name)
    end
```

Now let's try the API **index** again - now we get a nice tidy response:

```bash
# browser style
curl http://localhost:3030/api/v1/people.json
# or API Style
curl http://localhost:3030/api/v1/people -H "Accept: application/json"

[{"id":1,"first_name":"Zed","last_name":"Flintstone","gender":"male"},
{"id":2,"first_name":"Jed","last_name":"Flintstone","gender":"male"},
{"id":3,"first_name":"Rockbottom","last_name":"Flintstone","gender":"male"}
# ...
```


Now let's try the **show** API  for person 2 using:

```bash
curl -X GET http://localhost:3030/api/v1/people/2.json
{"id":2,"first_name":"Jed","last_name":"Flintstone","gender":"male"}

# or

curl -X GET http://localhost:3030/api/v1/people/2 -H "Accept: application/json"
{"id":2,"first_name":"Jed","last_name":"Flintstone","gender":"male"}
```

NOTE: Once you allow **delete** you should also adjust AND TEST the `dependent: :destroy` to ensure no hanging relationships in PeopleRelationships! (not yet included)
```
curl -X DELETE http://localhost:3030/api/v1/people/2 \
     -H "Accept: application/json"
```

Update (patch / **put**):
```
curl -X PATCH http://localhost:3030/api/v1/people/2 \
     -H "Content-Type: application/json" \
     -d '{"first_name": "NewFirstName", "last_name": "NewLastName"}'
```

**Create** a new person:
```
curl -X POST http://localhost:3030/api/v1/people \
     -H "Content-Type: application/json" \
     -d '{"first_name": "John", "last_name": "Doe"}'
```

### RELATIONSHIPS

It's nice to list users, but we are also interested in finding how people are related.  Let's set relationships.

```bash
bin/rails g model PersonRelationships role_one role_two \
                  person_one:references person_two:references
```

since we have two keys referring to the same table we need to include the table name in the migration.

```ruby
class CreatePersonRelationships < ActiveRecord::Migration[7.1]
  def change
    create_table :person_relationships do |t|
      t.string :role_one
      t.string :role_two
      t.references :person_one, null: false, foreign_key: { to_table: :people }, index: true
      t.references :person_two, null: false, foreign_key: { to_table: :people }, index: true

      t.timestamps
    end
  end
end
```

let's update our person model.  Given we don't know if a person is referenced using the first or second entry in the table we need to do a complicated `has_many` and we will add a convenience method `related_people`.

```ruby
class Person < ApplicationRecord
  has_many :person_relationships_one, class_name: 'PersonRelationship', foreign_key: :person_one_id
  has_many :person_relationships_two, class_name: 'PersonRelationship', foreign_key: :person_two_id

  # relationships when using `_1.map(&attributes)`
  has_many :related_people_one,
           -> { select('people.*, person_relationships.role_two AS relationship')
                .joins(:person_relationships_one) },
           through: :person_relationships_one, source: :person_two
  has_many :related_people_two,
           -> { select('people.*, person_relationships.role_one AS relationship')
                .joins(:person_relationships_two) },
           through: :person_relationships_two, source: :person_one

  # use `.to_json` or `_1.map(&attributes)` to include the relationship!
  def related_people = (related_people_one + related_people_two).uniq
end
```

given that the names we are using don't match the class name we need to include it here.

```ruby
class PersonRelationship < ApplicationRecord
  belongs_to :person_one, class_name: 'Person'
  belongs_to :person_two, class_name: 'Person'
end
```

Let's update the seed file with a few relationships.

```ruby
zed = Person.create!(first_name: "Zed", last_name: "Flintstone", gender: 'male')
jed = Person.create!(first_name: "Jed", last_name: "Flintstone", gender: 'male')
PersonRelationship.create!(person_one: zed, role_one: "sibling", person_two: jed, role_two: "sibling")

rock = Person.create!(first_name: "Rockbottom", last_name: "Flintstone", gender: 'male')
PersonRelationship.create!(person_one: jed, role_one: "parent", person_two: rock, role_two: "child")

ed = Person.create!(first_name: "Ed", last_name: "Flintstone", gender: 'male')
PersonRelationship.create!(person_one: rock, role_one: "parent", person_two: ed, role_two: "child")

giggles = Person.create!(first_name: "Giggles", last_name: "Flintstone", gender: 'male')
PersonRelationship.create!(person_one: rock, role_one: "parent", person_two: giggles, role_two: "child")
PersonRelationship.create!(person_one: ed, role_one: "sibling", person_two: giggles, role_two: "sibling")

tex = Person.create!(first_name: "Tex", last_name: "Hardrock", gender: 'male')
edna = Person.create!(first_name: "Edna", last_name: "Flintstone", gender: 'female', given_last_name: "Hardrock")
PersonRelationship.create!(person_one: tex, role_one: "sibling", person_two: edna, role_two: "sibling")
PersonRelationship.create!(person_one: ed, role_one: "spouse", person_two: edna, role_two: "spouse")

fred = Person.create!(first_name: "Fred", last_name: "Flintstone", gender: 'male')
PersonRelationship.create!(person_one: ed, role_one: "parent", person_two: fred, role_two: "child")
PersonRelationship.create!(person_one: edna, role_one: "parent", person_two: fred, role_two: "child")

joe = Person.create!(first_name: "Joe", last_name: "Rockhead", gender: 'male')
PersonRelationship.create!(person_one: joe, role_one: "friend", person_two: fred, role_two: "friend")

stoney = Person.create!(first_name: "Stoney", last_name: "Curtis", gender: 'male')
perry = Person.create!(first_name: "Perry", last_name: "Masonry", gender: 'male' )
arnlod = Person.create!(first_name: "Arnold", last_name: "Granite", gender: 'male')

ricky = Person.create!(first_name: "Ricky", last_name: "Slaghoople", gender: 'male')
pearl = Person.create!(first_name: "Pearl", last_name: "Slaghoople", gender: 'female')
PersonRelationship.create!(person_one: ricky, role_one: "spouse", person_two: pearl, role_two: "spouse")

wilma = Person.create!(first_name: "Wilma", last_name: "Flintstone", gender: 'female')
PersonRelationship.create!(person_one: ricky, role_one: "parent", person_two: wilma, role_two: "child")
PersonRelationship.create!(person_one: pearl, role_one: "parent", person_two: wilma, role_two: "child")
PersonRelationship.create!(person_one: wilma, role_one: "spouse", person_two: fred, role_two: "spouse")

pebbles = Person.create!(first_name: "Pebbles", last_name: "Flintstone", gender: "female")
PersonRelationship.create!(person_one: wilma, role_one: "parent", person_two: pebbles, role_two: "child")
PersonRelationship.create!(person_one: fred, role_one: "parent", person_two: pebbles, role_two: "child")

bob = Person.create!(first_name: "Bob", last_name: "Rubble", gender: 'male')
flo = Person.create!(first_name: "Flo Slate", last_name: "Rubble", gender: 'female')
PersonRelationship.create!(person_one: bob, role_one: "spouse", person_two: flo, role_two: "spouse")

barney = Person.create!(first_name: "Barney", last_name: "Rubble", gender: 'male')
PersonRelationship.create!(person_one: bob, role_one: "parent", person_two: barney, role_two: "child")
PersonRelationship.create!(person_one: flo, role_one: "parent", person_two: barney, role_two: "child")
PersonRelationship.create!(person_one: barney, role_one: "friend", person_two: fred, role_two: "friend")


dusty = Person.create!(first_name: "Dusty", last_name: "Rubble", gender: 'male')
PersonRelationship.create!(person_one: bob, role_one: "parent", person_two: dusty, role_two: "child")
PersonRelationship.create!(person_one: flo, role_one: "parent", person_two: dusty, role_two: "child")

brick = Person.create!(first_name: "Brick", last_name: "McBricker", gender: "male")
jean = Person.create!(first_name: "Jean", last_name: "McBricker", gender: "female")
PersonRelationship.create!(person_one: brick, role_one: "spouse", person_two: jean, role_two: "spouse")

betty = Person.create!(first_name: "Betty", last_name: "Rubble", given_last_name: "McBricker", gender: "female")
PersonRelationship.create!(person_one: brick, role_one: "parent", person_two: betty, role_two: "child")
PersonRelationship.create!(person_one: jean, role_one: "parent", person_two: betty, role_two: "child")
PersonRelationship.create!(person_one: betty, role_one: "friend", person_two: wilma, role_two: "spouse")

brad = Person.create!(first_name: "Brad", last_name: "McBricker", gender: "male")
PersonRelationship.create!(person_one: brick, role_one: "parent", person_two: brad, role_two: "child")
PersonRelationship.create!(person_one: jean, role_one: "parent", person_two: brad, role_two: "child")

bamm = Person.create!(first_name: "Bamm-Bamm", last_name: "Rubble", gender: 'male')
PersonRelationship.create!(person_one: betty, role_one: "parent", person_two: bamm, role_two: "child")
PersonRelationship.create!(person_one: barney, role_one: "parent", person_two: bamm, role_two: "child")

PersonRelationship.create!(person_one: pebbles, role_one: "spouse", person_two: bamm, role_two: "spouse")

roxy = Person.create!(first_name: "Roxy", last_name: "Rubble", gender: "female")
PersonRelationship.create!(person_one: betty, role_one: "parent", person_two: roxy, role_two: "child")
PersonRelationship.create!(person_one: barney, role_one: "parent", person_two: roxy, role_two: "child")

chip = Person.create!(first_name: "Chip", last_name: "Rubble", gender: "male")
PersonRelationship.create!(person_one: betty, role_one: "parent", person_two: chip, role_two: "child")
PersonRelationship.create!(person_one: barney, role_one: "parent", person_two: chip, role_two: "child")
```

given the large change in the seeds file lets just reset the setup
```bash
bin/rails db:drop
bin/rails db:setup
```

Console Test our code:

```ruby
# jed is the center of our relationships and is associated with both person_one and person_two
jed = Person.find_by(first_name: 'Jed')
jed.to_json
jed.attributes

# returns the relationships (associations in the first column of the join table)
jed.related_people
# we expect to find zed and rockbottom - when we debug using:
jed.related_people_one
jed.related_people_two
# we see only one works
```
unfortunately, this doesn't work for jed (although it works for more complicated cases)

```ruby
fred = Person.find_by(first_name: 'Fred')
fred.related_people # returns the relationships as expected
```

Let's try again - this isn't reliable.

By analyzing Fred's queries (that worked) - I build a reliable query - so lets rewrite our model using:
```ruby
class Person < ApplicationRecord
  # original with ONE USER
  RELATED_IS_SQL = <<-SQL.freeze
    SELECT people.*, person_relationships.role_two AS relationship
    FROM people
    INNER JOIN person_relationships ON people.id = person_relationships.person_two_id
    WHERE person_relationships.person_one_id = ?
    UNION
    SELECT people.*, person_relationships.role_one AS relationship
    FROM people
    INNER JOIN person_relationships ON people.id = person_relationships.person_one_id
    WHERE person_relationships.person_two_id = ?
  SQL

  # use `.to_json` or `_1.map(&attributes)` to include the relationship!
  def related_people = Person.find_by_sql([RELATED_ONE, [id], [id]])
end
```

Now jed and all people should find all relations using:
```ruby
jed = Person.find_by(first_name: 'Jed')
jed.related_people

fred = Person.find_by(first_name: 'Fred')
fred.related_people
```

ok now we can build our relations API!

## RELATIONS API

It is generally a good idea to stick with the normal controller calls and create a new resource.

So if we want people and their immediate relations we can make a new controller:
```
touch app/controllers/api/v1/people_with_relations_controller.rb
```

let's decide what we want our API to return - the desired person with relations listed in an array - these 'related persons should return the relation `role`':
```ruby
{ "id"=>8,
  "first_name"=>"Fred",
  "last_name"=>"Flintstone",
	"given_last_name" => nil,
	"gender" => 'male',
  "relations"=> [
	  {"id"=>16,
		  "first_name"=>"Pebbles",
		  "last_name"=>"Flintstone",
	    "given_last_name"=>nil,
		  "gender"=>"female",
		  "relationship"=>"child"},
    {"id"=>4,
		 "first_name"=>"Ed",
		 "last_name"=>"Flintstone",
		 "given_last_name"=>nil,
		 "gender"=>"male",
		"relationship"=>"parent"},
    {"id"=>7,
		 "first_name"=>"Edna",
		 "last_name"=>"Flintstone",
		 "given_last_name"=>"Hardrock",
		 "gender"=>"female",
		 "relationship"=>"parent"},
    {"id"=>15,
		 "first_name"=>"Wilma",
		 "last_name"=>"Flintstone",
		 "given_last_name"=>nil,
		 "gender"=>"female",
		 "relationship"=>"spouse"},
    {"id"=>19,
		 "first_name"=>"Barney",
		 "last_name"=>"Rubble",
		 "given_last_name"=>nil,
		 "gender"=>"male",
		 "relationship"=>"friend"}
  ]
}
```

lets try some code:
```ruby
      params = {}
      params[:id] = 8
			person = Person.where(id: params[:id]).select(:id, :first_name, :last_name, :gender).first
			relations = person.related_people.map { _1.attributes.except('created_at', 'updated_at') }
			@person = person.attributes.except('created_at', 'updated_at').merge(relations: relations)
```

cool this looks good - lets try in the controller:

```ruby
# app/controllers/api/v1/people_with_relations_controller.rb
module Api
  module V1
    class PeopleWithRelationsController < ApplicationController
      # GET /people.json
      def index
        people = Person.select(:id, :first_name, :last_name, :gender).all
        @people = people.map do |person|
          relations = person.related_people.map { _1.attributes.except('created_at', 'updated_at') }
          person.attributes.except('created_at', 'updated_at').merge(related_people: relations)
        end

		    render json: @people
      end

      # GET /people/1.json
      def show
        person = Person.where(id: params[:id]).select(:id, :first_name, :last_name, :gender).first
        relations = person.related_people.map { _1.attributes.except('created_at', 'updated_at') }
        @person = person.attributes.except('created_at', 'updated_at').merge(related_people: relations)

		    render json: @person
      end
    end
  end
end
```

```ruby
bin/rails s -p 3030
curl -X GET http://localhost:3030/api/v1/people_with_relations/8.json

{ "id":8,
	"first_name":"Fred",
	"last_name":"Flintstone",
	"gender":"male",
	"related_people": [
	  { "id":16,
		  "first_name":"Pebbles",
		  "last_name":"Flintstone",
		  "given_last_name":null,
		  "gender":"female",
		  "relationship":"child" },
	  { "id":4,
		  "first_name":"Ed",
		  "last_name":"Flintstone",
		  "given_last_name":null,
		  "gender":"male",
		  "relationship":"parent" },
	  { "id":7,
			"first_name":"Edna",
		  "last_name":"Flintstone",
		  "given_last_name":"Hardrock",
		  "gender":"female",
		  "relationship":"parent" },
	  { "id":15,
		  "first_name":"Wilma",
		  "last_name":"Flintstone",
		  "given_last_name":null,
		  "gender":"female",
		  "relationship":"spouse" },
	  { "id":19,
		  "first_name":"Barney",
		  "last_name":"Rubble",
		  "given_last_name":null,
		  "gender":"male",
		  "relationship":"friend" }
  ]
}%
```

OK - looks good.  Let's see what the rails log looks like:
```ruby
Started GET "/api/v1/people_with_relations/8.json" for ::1 at 2024-03-17 13:50:53 +0100
Processing by Api::V1::PeopleWithRelationsController#show as JSON
  Parameters: {"id"=>"8"}
  Person Load (0.1ms)  SELECT "people"."id", "people"."first_name", "people"."last_name", "people"."gender" FROM "people" WHERE "people"."id" = ? ORDER BY "people"."id" ASC LIMIT ?  [["id", 8], ["LIMIT", 1]]
  ↳ app/controllers/api/v1/people_with_relations_controller.rb:25:in 'show'
  Person Load (0.1ms)  SELECT people.*, person_relationships.role_two AS relationship FROM "people" INNER JOIN "person_relationships" ON "people"."id" = "person_relationships"."person_two_id" INNER JOIN "person_relationships" "person_relationships_ones_people" ON "person_relationships_ones_people"."person_one_id" = "people"."id" WHERE "person_relationships"."person_one_id" = ?  [["person_one_id", 8]]
  ↳ app/models/person.rb:15:in 'related_people'
  Person Load (0.1ms)  SELECT people.*, person_relationships.role_one AS relationship FROM "people" INNER JOIN "person_relationships" ON "people"."id" = "person_relationships"."person_one_id" INNER JOIN "person_relationships" "person_relationships_twos_people" ON "person_relationships_twos_people"."person_two_id" = "people"."id" WHERE "person_relationships"."person_two_id" = ?  [["person_two_id", 8]]
  ↳ app/models/person.rb:15:in 'related_people'
Completed 200 OK in 4ms (Views: 0.1ms | ActiveRecord: 0.3ms | Allocations: 3019)
```

Three queires - makes sense.
One for the pirmary person, one in the case when our relation is in role_one and another when our relation is in role_two.

OK let's try index now:

```ruby
people = Person.select(:id, :first_name, :last_name, :gender).all
@people = people.map do |person|
            relations = person.related_people.map { _1.attributes.except('created_at', 'updated_at') }
            person.attributes.except('created_at', 'updated_at').merge(related_people: relations)
          end
[{"id"=>1, "first_name"=>"Zed", "last_name"=>"Flintstone", "gender"=>"male", :related_people=>[]},
 {"id"=>2, "first_name"=>"Jed", "last_name"=>"Flintstone", "gender"=>"male", :related_people=>[]},
 {"id"=>3,
  "first_name"=>"Rockbottom",
  "last_name"=>"Flintstone",
  "gender"=>"male",
  :related_people=>[{"id"=>4, "first_name"=>"Ed", "last_name"=>"Flintstone", "given_last_name"=>nil, "gender"=>"male", "relationship"=>"child"}]},
 {"id"=>4,
  "first_name"=>"Ed",
  "last_name"=>"Flintstone",
  "gender"=>"male",
  :related_people=>
   [{"id"=>7, "first_name"=>"Edna", "last_name"=>"Flintstone", "given_last_name"=>"Hardrock", "gender"=>"female", "relationship"=>"spouse"},
    {"id"=>8, "first_name"=>"Fred", "last_name"=>"Flintstone", "given_last_name"=>nil, "gender"=>"male", "relationship"=>"child"}]},
	...
```

now lets try via the controller

```ruby
curl -X GET http://localhost:3030/api/v1/people_with_relations.json

[ {"id":1,"first_name":"Zed","last_name":"Flintstone","gender":"male","related_people":[]},
	{"id":2,"first_name":"Jed","last_name":"Flintstone","gender":"male","related_people":[]},
	{"id":3,"first_name":"Rockbottom","last_name":"Flintstone","gender":"male",
		"related_people": [
		  { "id":4,
				"first_name":"Ed",
			  "last_name":"Flintstone",
			  "given_last_name":null,
			  "gender":"male","relationship":"child" }
		]},
  { "id":4,
	  "first_name":"Ed",
	  "last_name":"Flintstone",
	  "gender":"male",
	  "related_people": [
		  { "id":7,
			  "first_name":"Edna",
			  "last_name":"Flintstone",
			  "given_last_name":"Hardrock",
			  "gender":"female",
			  "relationship":"spouse"}
		 ]},
	...
```

ok - this looks good lets check the logs:
```ruby
Started GET "/api/v1/people_with_relations.json" for ::1 at 2024-03-17 14:19:57 +0100
Processing by Api::V1::PeopleWithRelationsController#index as JSON
  Person Load (0.6ms)  SELECT "people"."id", "people"."first_name", "people"."last_name", "people"."gender" FROM "people"
  ↳ app/controllers/api/v1/people_with_relations_controller.rb:8:in `map'
  Person Load (0.1ms)  SELECT people.*, person_relationships.role_two AS relationship FROM "people" INNER JOIN "person_relationships" ON "people"."id" = "person_relationships"."person_two_id" INNER JOIN "person_relationships" "person_relationships_ones_people" ON "person_relationships_ones_people"."person_one_id" = "people"."id" WHERE "person_relationships"."person_one_id" = ?  [["person_one_id", 1]]
  ↳ app/models/person.rb:15:in `related_people'
  Person Load (0.0ms)  SELECT people.*, person_relationships.role_one AS relationship FROM "people" INNER JOIN "person_relationships" ON "people"."id" = "person_relationships"."person_one_id" INNER JOIN "person_relationships" "person_relationships_twos_people" ON "person_relationships_twos_people"."person_two_id" = "people"."id" WHERE "person_relationships"."person_two_id" = ?  [["person_two_id", 1]]
  ↳ app/models/person.rb:15:in `related_people'
  Person Load (0.0ms)  SELECT people.*, person_relationships.role_two AS relationship FROM "people" INNER JOIN "person_relationships" ON "people"."id" = "person_relationships"."person_two_id" INNER JOIN "person_relationships" "person_relationships_ones_people" ON "person_relationships_ones_people"."person_one_id" = "people"."id" WHERE "person_relationships"."person_one_id" = ?  [["person_one_id", 2]]
  ↳ app/models/person.rb:15:in `related_people'
  Person Load (0.0ms)  SELECT people.*, person_relationships.role_one AS relationship FROM "people" INNER JOIN "person_relationships" ON "people"."id" = "person_relationships"."person_one_id" INNER JOIN "person_relationships" "person_relationships_twos_people" ON "person_relationships_twos_people"."person_two_id" = "people"."id" WHERE "person_relationships"."person_two_id" = ?  [["person_two_id", 2]]
  ↳ app/models/person.rb:15:in `related_people'
  Person Load (0.1ms)  SELECT people.*, person_relationships.role_two AS relationship FROM "people" INNER JOIN "person_relationships" ON "people"."id" = "person_relationships"."person_two_id" INNER JOIN "person_relationships" "person_relationships_ones_people" ON "person_relationships_ones_people"."person_one_id" = "people"."id" WHERE "person_relationships"."person_one_id" = ?  [["person_one_id", 3]]
  ↳ app/models/person.rb:15:in `related_people'
  Person Load (0.0ms)  SELECT people.*, person_relationships.role_one AS relationship FROM "people" INNER JOIN "person_relationships" ON "people"."id" = "person_relationships"."person_one_id" INNER JOIN "person_relationships" "person_relationships_twos_people" ON "person_relationships_twos_people"."person_two_id" = "people"."id" WHERE "person_relationships"."person_two_id" = ?  [["person_two_id", 3]]
  ↳ app/models/person.rb:15:in `related_people'
```

Hmm now we have an n+1 - one query to get all the people and two for each person to get relations.  (Can we improve his?) This well be very poor performance with lots of records

### Fixing N+1

to fix this lets see if we can adjust our SQL to handle many users and sort them by the calling `person_id` and we will change our where statements from `WHERE person_relationships.person_one_id = ?` to: `WHERE person_relationships.person_one_id IN (?)` so lets try:
```ruby
# app/models/person.rb
  def self.related_people_for(person_ids)
	  person_ids = Array(person_ids)
    # our select returns only the values we want from related persons
    # (as well as the relationship and the root person)
    sql = <<-SQL
      SELECT people.id, people.first_name, people.last_name, people.gender, person_relationships.role_two AS relationship, person_relationships.person_one_id AS person_id
      FROM people
      INNER JOIN person_relationships ON people.id = person_relationships.person_two_id
      WHERE person_relationships.person_one_id IN (?)
      UNION
      SELECT people.id, people.first_name, people.last_name, people.gender, person_relationships.role_one AS relationship, person_relationships.person_two_id AS person_id
      FROM people
      INNER JOIN person_relationships ON people.id = person_relationships.person_one_id
      WHERE person_relationships.person_two_id IN (?)
    SQL

    # find all our related people
    related_people = Person.find_by_sql([sql, id, id])

    # group related users by their root_person_id
    related_people_hash = related_people.group_by { |rp| rp.person_id }

    # convert our related people into a hash
    related_people_hash.transform_values! do |related_people_arr|
      related_people_arr.map do |related_person|
        related_person.attributes.except('created_at', 'updated_at', 'person_id')
      end
    end
  end
```

let's build / test some index code:
```ruby
people = Person.select(:id, :first_name, :last_name, :gender).all
related_people_grouped_hash = Person.related_people_for(people.pluck(:id))

@people = people.map do |person|
  person_attribs = person.attributes.except('created_at', 'updated_at')
  person_attribs.merge(related_people: related_people_grouped_hash[person.id] || [])
end

JSON.parse @people.to_json
```

nice gets the correct answer and also only 3 quiries!

lets update our controller
```ruby
# app/controllers/api/v1/people_with_relations_controller.rb
module Api
  module V1
    class PeopleWithRelationsController < ApplicationController
      # GET /people.json
      def index
        people = Person.select(:id, :first_name, :last_name, :gender).all
        related_people_grouped_hash = Person.related_people_for(people.pluck(:id))
        @people = people.map do |person|
          person_attribs = person.attributes.except('created_at', 'updated_at')
          person_attribs.merge(related_people: related_people_grouped_hash[person.id] || [])
        end
        render json: @people
      end

      # GET /people/1.json
      def show
        # not more efficient than the above, but uses the same logic as index (less to test) :)
        person = Person.find(params[:id])
        person_attribs = person.attributes.except('created_at', 'updated_at')
        related_people_grouped_hash = Person.related_people_for(params[:id])
        @person = person_attribs.merge(related_people: related_people_grouped_hash[params[:id].to_i] || [])
			  render json: @person
      end
    end
  end
end
```

here is the current model:
```ruby
# app/models/person.rb
class Person < ApplicationRecord
  RELATED_ONE = <<-SQL.freeze
    SELECT people.*, person_relationships.role_two AS relationship
    FROM people
    INNER JOIN person_relationships ON people.id = person_relationships.person_two_id
    WHERE person_relationships.person_one_id = ?
    UNION
    SELECT people.*, person_relationships.role_one AS relationship
    FROM people
    INNER JOIN person_relationships ON people.id = person_relationships.person_one_id
    WHERE person_relationships.person_two_id = ?
  SQL

  # use `.to_json` or `_1.map(&attributes)` to include the relationship!
  def related_people = Person.find_by_sql([RELATED_ONE, [id], [id]])

  RELATED_SQL = <<-SQL.freeze
    SELECT people.id, people.first_name, people.last_name, people.gender, person_relationships.role_two AS relationship, person_relationships.person_one_id AS person_id
    FROM people
    INNER JOIN person_relationships ON people.id = person_relationships.person_two_id
    WHERE person_relationships.person_one_id IN (?)
    UNION
    SELECT people.id, people.first_name, people.last_name, people.gender, person_relationships.role_one AS relationship, person_relationships.person_two_id AS person_id
    FROM people
    INNER JOIN person_relationships ON people.id = person_relationships.person_one_id
    WHERE person_relationships.person_two_id IN (?)
  SQL

  # not useful anymore
  # has_many :person_relationships_one, class_name: 'PersonRelationship', foreign_key: :person_one_id
  # has_many :person_relationships_two, class_name: 'PersonRelationship', foreign_key: :person_two_id

  # # relationships when using `_1.map(&attributes)`
  # has_many :related_people_one,
  #          -> { select('people.*, person_relationships.role_two AS relationship').joins(:person_relationships_one) },
  #          through: :person_relationships_one, source: :person_two, class_name: 'Person'
  # has_many :related_people_two,
  #          -> { select('people.*, person_relationships.role_one AS relationship').joins(:person_relationships_two) },
  #          through: :person_relationships_two, source: :person_one, class_name: 'Person'

  # # use `.to_json` or `_1.map(&attributes)` to include the relationship!
  # def related_people = (related_people_one + related_people_two).uniq

  # handles many ids
  def self.related_people_for(person_ids)
    # ensure we always have an array
    person_ids = Array(person_ids)

    # query for related people
    related_people = Person.find_by_sql([RELATED_SQL, person_ids, person_ids])

    # grouping allows us to return a hash with the person_id as the key (neccesary for index controller)
    related_people_hash = related_people.group_by(&:person_id)

    # convert values to a hash (only including desired values)
    related_people_hash.transform_values! do |related_people_arr|
      related_people_arr.map do |related_person|
        related_person.attributes.except('created_at', 'updated_at', 'person_id')
      end
    end
  end
end
```

**testing the api**

lets try show:
```bash
curl -X GET http://localhost:3030/api/v1/people_with_relations/8.json

{"id":8,"first_name":"Fred","last_name":"Flintstone","gender":"male",
 "related_people":
 {"8":[{"id":4,"first_name":"Ed","last_name":"Flintstone","gender":"male","relationship":"parent"},{"id":7,"first_name":"Edna","last_name":"Flintstone","gender":"female","relationship":"parent"},{"id":9,"first_name":"Joe","last_name":"Rockhead","gender":"male","relationship":"friend"},{"id":15,"first_name":"Wilma","last_name":"Flintstone","gender":"female","relationship":"spouse"},{"id":16,"first_name":"Pebbles","last_name":"Flintstone","gender":"female","relationship":"child"},{"id":19,"first_name":"Barney","last_name":"Rubble","gender":"male","relationship":"friend"}]}}%
```

show logs:
```
Started GET "/api/v1/people_with_relations/8.json" for ::1 at 2024-03-17 15:31:28 +0100
Processing by Api::V1::PeopleWithRelationsController#show as JSON
  Parameters: {"id"=>"8"}
  Person Pluck (0.0ms)  SELECT "people"."id" FROM "people" WHERE "people"."id" = ?  [["id", 8]]
  ↳ app/controllers/api/v1/people_with_relations_controller.rb:35:in `show'
  Person Load (0.1ms)        SELECT people.id, people.first_name, people.last_name, people.gender, person_relationships.role_two AS relationship, person_relationships.person_one_id AS person_id
      FROM people
      INNER JOIN person_relationships ON people.id = person_relationships.person_two_id
      WHERE person_relationships.person_one_id IN (8)
      UNION
      SELECT people.id, people.first_name, people.last_name, people.gender, person_relationships.role_one AS relationship, person_relationships.person_two_id AS person_id
      FROM people
      INNER JOIN person_relationships ON people.id = person_relationships.person_one_id
      WHERE person_relationships.person_two_id IN (8)

  ↳ app/models/person.rb:68:in `related_people_for'
  Person Load (0.1ms)  SELECT "people"."id", "people"."first_name", "people"."last_name", "people"."gender" FROM "people" WHERE "people"."id" = ?  [["id", 8]]
  ↳ app/controllers/api/v1/people_with_relations_controller.rb:37:in `map'
Completed 200 OK in 12ms (Views: 0.1ms | ActiveRecord: 1.1ms | Allocations: 7119)
```

ok still 3 queiries, but should scale!

**index**

Index logs:
```
Started GET "/api/v1/people_with_relations.json" for ::1 at 2024-03-17 15:25:50 +0100
Processing by Api::V1::PeopleWithRelationsController#index as JSON
  Person Pluck (0.1ms)  SELECT "people"."id" FROM "people"
  ↳ app/controllers/api/v1/people_with_relations_controller.rb:8:in `index'
  Person Load (0.3ms)        SELECT people.id, people.first_name, people.last_name, people.gender, person_relationships.role_two AS relationship, person_relationships.person_one_id AS person_id
      FROM people
      INNER JOIN person_relationships ON people.id = person_relationships.person_two_id
      WHERE person_relationships.person_one_id IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27)
      UNION
      SELECT people.id, people.first_name, people.last_name, people.gender, person_relationships.role_one AS relationship, person_relationships.person_two_id AS person_id
      FROM people
      INNER JOIN person_relationships ON people.id = person_relationships.person_one_id
      WHERE person_relationships.person_two_id IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27)

  ↳ app/models/person.rb:68:in `related_people_for'
  Person Load (0.1ms)  SELECT "people"."id", "people"."first_name", "people"."last_name", "people"."gender" FROM "people"
  ↳ app/controllers/api/v1/people_with_relations_controller.rb:10:in `map'
Completed 200 OK in 4ms (Views: 0.3ms | ActiveRecord: 0.4ms | Allocations: 5125)
```

cool - now we can get all with relations in 3 queires!  Now ts performant!

of course with a lot of records index should also limit returns and implement pagination 'we can do this in v2' as we grow!

```bash
git add .
git commit -m "implement efficient person with relations"
```

### pets

```bash
bin/rails g scaffold Pet name species # owner:references:person
bin/rails g model PetPeople pet person
```

## Refactoring

using raw SQL is generally not so desired so we can update our code with:
```ruby
class Person < ApplicationRecord
  def self.with_relations(person_ids)
    person_ids = Array(person_ids).map(&:to_i)

    query1 = Person.select("people.*, person_relationships.role_one AS relationship, person_relationships.person_two_id AS person_id")
                        .joins("INNER JOIN person_relationships ON people.id = person_relationships.person_one_id")
                        .where(person_relationships: { person_two_id: person_ids })

    query2 = Person.select("people.*, person_relationships.role_two AS relationship, person_relationships.person_one_id AS person_id")
                    .joins("INNER JOIN person_relationships ON people.id = person_relationships.person_two_id")
                    .where(person_relationships: { person_one_id: person_ids })

    Person.find_by_sql(query1.to_sql + " UNION " + query2.to_sql)
  end

  # this is an alternative way to scope instead of using the lambda and a scope statement (returns a hash)
  def self.related_people_for(person_ids)
    # query for related people
    related_people = with_relations(person_ids)

    # grouping allows us to return a hash with the person_id as the key (necessary for index controller)
    related_people_grouped = related_people.group_by(&:person_id)

    # removes unwanted attributes (possibly to later or confogure json to do this)
    related_people_grouped.transform_values! do |related_people_arr|
      related_people_arr.map do |related_person|
        related_person.attributes.except('created_at', 'updated_at', 'person_id')
      end
    end
  end

  # this is equivalent to: with_relations(person_ids) -
  # for many using scope is clearer and more standard
  # (they are both used the same way)
  # People.with_related_people(ids)
  # People.People.with_related_people(ids)
  scope :with_related_people, lambda { |person_ids| c(person_ids) }

  # this only works for an instantiated person - this works just like a has_many would
  # jed = Person.find_by(first_name: 'Jed')
  # jed.related_people
  def related_people = with_relations(id)
end
```
```
