---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Rails with Postgres - Fuzzy Searches"
subjob_title: "Simple Effective Scored Fuzzy Searches"
# Summary for listings and search engines
summary: "With PostgreSQL we have a simple and powerful way to do scored fuzzy searches"
authors: ["btihen"]
tags: ["Rails", "Ruby", "ActiveRecord", "Fuzzy", "Similarity", "PostgreSQL", "Postgres", "PG"]
categories: ["Code", "Ruby Language", "Ruby on Rails", "PostgreSQL"]
date: 2024-10-31T01:01:53+02:00
lastmod: 2024-10-31T01:01:53+02:00
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

## Overview

Recently I was working on a project at work that required finding the appropriate record with incomplete information (that might be either mispelled or within multiple columns - thus `LIKE` and `ILIKE` are insufficient).

My co-worker [Gernot Kogler](https://www.linkedin.com/in/gernot-kogler-075513162/), introduced me to the trigram scoring searches using `similarity` and `word_similarity` - this is a simple and very effective way to do **fuzzy** searches.

If interested, a good article [Optimizing Postgres Text Search with Trigrams](https://alexklibisz.com/2022/02/18/optimizing-postgres-trigram-search) that explains the details of the scoring and optimizing the search speed.


In fact, there are several ways to do fuzzy searches in Postgres, this this article covers `pg_trgrm` (trigram) extension - efficient scored partial matches. In the coclusion there is a list of alternatives to pg_trgrm.

## Getting Started

Let's create a test project that finds the best matching person in our database, from a 'description'.

### Simple Rails Project

```bash
# macos 15 seems to need this
ulimit -n 10240

rbenv istall 3.3.5
rbenv local 3.3.5
# create a new rails project (with PG & without minitest)
rails new fuzzy -d postgresql -T

# let's check the gem file to be sure we are using PG,
# we expect to find:
gem "pg", "~> 1.1"

# lets be sure we are using uptodate gems:
cd fuzzy
bundle update

# currently if we check Gemfile.lock we will see:
rails (7.2.2)

# setup the database with
bin/rails db:create

# assuming all went well
git init
git add .
git commit -m "initial commit"
```

### Add DB extensions

To do similarity searches we need the extension: `pg_trgm`

```bash
bin/rails generate migration AddPgTrgmExtensionToDb
```

```ruby
# db/migrate/20241031122103_add_pg_trgm_extension_to_db.rb
class AddPgTrgmExtensionToDb < ActiveRecord::Migration[7.2]
  def change
    enable_extension "pg_trgm" # for similarity searches
  end
  # or if you prefer raw sql
  # def up
  #  execute("CREATE EXTENSION IF NOT EXISTS pg_trgm;")
  # end
  # def down
  #  execute("DROP EXTENSION IF EXISTS pg_trgm;")
  # end
end

```
now migrate
```bash
bin/rails db:migrate
```

### Create Models

Let's create a simple person schema and learn to do a fuzzy search.

```bash
bin/rails generate model Person last_name:string:index first_name:string:index birthdate:date

bin/rails generate model Role job_title:string department:string person:references
```

**Update Migration**

```ruby
# db/migrate/20241031124235_create_people.rb
class CreatePeople < ActiveRecord::Migration[7.2]
  def change
    create_table :people do |t|
      t.string :last_name, null: false
      t.string :first_name, null: false
      t.date :birthdate, null: false

      t.timestamps
    end

    add_index :people, [:last_name, :first_name, :birthdate], unique: true
  end
end

# and
# db/migrate/20241031124236_create_roles.rb
class CreateRoles < ActiveRecord::Migration[7.2]
  def change
    create_table :roles do |t|
      t.string :job_title, null: false
      t.string :department, null: false
      t.references :person, null: false, foreign_key: true

      t.timestamps
    end

    add_index :roles, [:person_id, :job_title, :department], unique: true
  end
end
```

now we can migrate:

```bash
mix ecto.migrate
```

**Add Relationships**

By adding relationships we can simplify our queries.

```ruby
# app/models/person.rb
class Person < ApplicationRecord
  has_many :roles

  validates :first_name, presence: true
  validates :last_name, presence: true
  validates :birthdate, presence: true

  validates :birthdate, uniqueness: { scope: [:first_name, :last_name] }
end

# and

# app/models/role.rb
class Role < ApplicationRecord
  belongs_to :person

  validates :job_title, presence: true
  validates :department, presence: true

  validates :person_id, uniqueness: { scope: [:job_title, :department] }
end
```

### Seeding

Now let's make some records for testing using the `seeds.rb` file.

```ruby
# db/seeds.rb

# Step 1: People Data
people_data = [
  {
    last_name: "Smith",
    first_name: "John",
    birthdate: Date.new(1980, 1, 1),
    job_title: "Software Engineer",
    department: "Product"
  },
  {
    last_name: "Johnson",
    first_name: "John",
    birthdate: Date.new(1970, 1, 1),
    job_title: "Network Engineer",
    department: "Operations"
  },
  {
    last_name: "Johanson",
    first_name: "Jonathan",
    birthdate: Date.new(1972, 1, 1),
    job_title: "QA Engineer",
    department: "Quality"
  },
  {
    last_name: "Smithers",
    first_name: "Charles",
    birthdate: Date.new(1974, 1, 1),
    job_title: "Tester",
    department: "Quality"
  },
  {
    last_name: "Brown",
    first_name: "Charlie",
    birthdate: Date.new(1960, 1, 1),
    job_title: "Designer",
    department: "Product"
  },
  {
    last_name: "Johnson",
    first_name: "Emma",
    birthdate: Date.new(1962, 1, 1),
    job_title: "Data Scientist",
    department: "Research"
  },
  {
    last_name: "Johnston",
    first_name: "Emilia",
    birthdate: Date.new(1966, 1, 1),
    job_title: "Data Scientist",
    department: "Research"
  },
  {
    last_name: "Williams",
    first_name: "Liam",
    birthdate: Date.new(1968, 1, 1),
    job_title: "DevOps Engineer",
    department: "Operations"
  },
  {
    last_name: "Jones",
    first_name: "Olivia",
    birthdate: Date.new(1976, 1, 1),
    job_title: "UX Researcher",
    department: "Research"
  },
  {
    last_name: "Miller",
    first_name: "Noah",
    birthdate: Date.new(1978, 1, 1),
    job_title: "Software Engineer",
    department: "Product"
  },
  {
    last_name: "Davis",
    first_name: "Ava",
    birthdate: Date.new(1982, 1, 1),
    job_title: "Software Engineer",
    department: "Product"
  },
  {
    last_name: "Garcia",
    first_name: "Sophia",
    birthdate: Date.new(1984, 1, 1),
    job_title: "QA Engineer",
    department: "Quality"
  },
  {
    last_name: "Rodriguez",
    first_name: "Isabella",
    birthdate: Date.new(1986, 1, 1),
    job_title: "Tech Support",
    department: "Customers"
  },
  {
    last_name: "Martinez",
    first_name: "Mason",
    birthdate: Date.new(1988, 1, 1),
    job_title: "Business Analyst",
    department: "Research"
  },
  {
    last_name: "Hernandez",
    first_name: "Lucas",
    birthdate: Date.new(1958, 1, 1),
    job_title: "Systems Administrator",
    department: "Operations"
  },
  {
    last_name: "Lopez",
    first_name: "Amelia",
    birthdate: Date.new(1956, 1, 1),
    job_title: "Product Manager",
    department: "Business"
  },
  {
    last_name: "Gonzalez",
    first_name: "James",
    birthdate: Date.new(1990, 1, 1),
    job_title: "Network Engineer",
    department: "Operations"
  },
  {
    last_name: "Wilson",
    first_name: "Elijah",
    birthdate: Date.new(1992, 1, 1),
    job_title: "Cloud Architect",
    department: "Operations"
  }
]

# Step 2: Insert People and Assign Accounts & Departments
people_data.each do |person_data|
  # Insert the Person record
  person =
    Person.create(
      first_name: person_data[:first_name],
      last_name: person_data[:last_name],
      birthdate: person_data[:birthdate],
      job_title: person_data[:job_title],
      department: person_data[:department]
    )

  # Insert the corresponding Account record
  Role.create!(
    person_id: person.id,
    job_title: person_data[:job_title],
    department: person_data[:department]
  )
end
```

now let's run the seed file:
```bash
bin/rails db:seed
```

### Test Setup

let's see if we can access our records using `iex`:
```ruby
bin/rails console

# tes basic models and seed
Person.first
Role.last

# test relationships
Person.first.role
Role.last.person
```
If all this works we are ready to go:
```bash
git add .
git commit -m "database setup and added people"
```

## Simple Fuzzy Search SQL

The following list consists of the features that the extension comes with, which helps in doing trigram searches:

from the [docs](https://www.postgresql.org/docs/12/pgtrgm.html):

Function	| Returns	| Description
`similarity(text, text)` | real |	Returns a number that indicates how similar the two arguments are. The range of the result is zero (indicating that the two strings are completely dissimilar) to one (indicating that the two strings are identical).
`show_trgm(text)` |	text[] | Returns an array of all the trigrams in the given string. (In practice this is seldom useful except for debugging.)
`word_similarity(text, text)`	| real | Returns a number that indicates the greatest similarity between the set of trigrams in the first string and any continuous extent of an ordered set of trigrams in the second string. For details, see the explanation below.
`strict_word_similarity(text, text)` | real | Same as word_similarity(text, text), but forces extent boundaries to match word boundaries. Since we don't have cross-word trigrams, this function actually returns greatest similarity between first string and any continuous extent of words of the second string.
`show_limit()` | real | Returns the current similarity threshold used by the % operator. This sets the minimum similarity between two words for them to be considered similar enough to be misspellings of each other, for example (deprecated).
`set_limit(real)` | real | Sets the current similarity threshold that is used by the % operator. The threshold must be between 0 and 1 (default is 0.3). Returns the same value passed in (deprecated).

### SQL Examples

Let's focus on `similarity(text1, text2)` - this calculates a trigram similarity index from 0 (no tri-gram matches) to 1 (all trigram matches), with 0 being the least similar.  Here is a basic example (building block toward more useful searches):

```sql
bin/rails db

SELECT word_similarity('word', 'two words');

 word_similarity
-------------
     0.8
```

Trigrams are substrings of three consecutive characters. For example, the word "Johns" has the following trigrams: "  j"," jo", "joh", "ohn", "hns", "ns ".  We can use `show_trgm('Johns')` to see the trigrams used by pg_trgm.
```sql
SELECT show_trgm('Johns') AS trigrams;

            trigrams
------------------------------
{"  j"," jo","hns","joh","ns ","ohn"}
(1 row)
```

Let's try something more useful and compare a db field to a string - using:
`similarity(last_name, 'Johns')`
which returns a ('real') a similarity score between 0 and 1.

NOTE: where clauses must return a binary value (true or false), thus to use a where clause, it looks like:
`WHERE similarity(last_name, 'Johns') > 0.1`

thus a basic query would look like:

```sql
bin/rails db

SELECT id, last_name, first_name,
       similarity(last_name, 'Johns') AS score
  FROM people
  WHERE similarity(last_name, 'Johns') > 0.1
  ORDER BY score DESC;

 id | last_name | first_name |  score
---+---------+---------+---------
 2  | Johnson   | John      | 0.5555556
 6  | Johnson   | Emma      | 0.5555556
 7  | Johnston  | Emilia    |       0.5
 3  | Johanson  | Jonathan  |      0.25
 9  | Jones     | Olivia    |       0.2
(5 rows)
```

To simplify (and use the default threshold of 0.3) we can rewrite the query using a `%` boolean operator:
```sql
SELECT id, last_name, first_name,
       similarity(last_name, 'Johns') AS score
  FROM people
  WHERE last_name % 'Johns'
  ORDER BY score DESC;

 id | last_name | first_name |   score
---+---------+----------+---------
  2 | Johnson   | John       | 0.5555556
  6 | Johnson   | Emma       | 0.5555556
  7 | Johnston  | Emilia     |       0.5
```

We don't actually need the `where` clause, lets see what happens if we remove it:
```sql
SELECT id, last_name, first_name,
       similarity(last_name, 'Johns') AS score
  FROM people
  ORDER BY score DESC;

 id | last_name | first_name |   score
---+---------+----------+-----------
  6 | Johnson   | Emma       | 0.5555556
  2 | Johnson   | John       | 0.5555556
  7 | Johnston  | Emilia     |       0.5
  3 | Johanson  | Jonathan   |      0.25
  9 | Jones     | Olivia     |       0.2
 11 | Davis     | Ava        |         0
 12 | Garcia    | Sophia     |         0
 13 | Rodriguez | Isabella   |         0
 14 | Martinez  | Mason      |         0
 15 | Hernandez | Lucas      |         0
 16 | Lopez     | Amelia     |         0
 17 | Gonzalez  | James      |         0
  1 | Smith     | John       |         0
 18 | Wilson    | Elijah     |         0
  4 | Smithers  | Charles    |         0
  5 | Brown     | Charlie    |         0
  8 | Williams  | Liam       |         0
 10 | Miller    | Noah       |         0
(18 rows)
```

We see our results return every record - no matter the score.  Usually we are only interested in the top results, so we can use `LIMIT` (along with `ORDER BY`) to limit the top results (this is probably faster than using the `where` clause):
```sql
SELECT id, last_name, first_name,
       similarity(last_name, 'Johns') AS score
  FROM people
  ORDER BY score DESC
  LIMIT 4;

 id | last_name | first_name |   score
----+-----------+------------+-----------
  6 | Johnson   | Emma       | 0.5555556
  2 | Johnson   | John       | 0.5555556
  7 | Johnston  | Emilia     |       0.5
  3 | Johanson  | Jonathan   |      0.25
(4 rows)
```

OK - this is looking good - let's do this for the full name (first and last name). To do this we need to concatenate the first and last name together (with a space as a separator).  We can use `CONCAT_WS` to do this:

```sql
SELECT id, first_name, last_name,
       similarity(CONCAT_WS(' ', first_name, last_name), 'Emily Johns') AS score
FROM people
ORDER BY score DESC
LIMIT 5;

 id | first_name | last_name |   score
----+------------+-----------+------------
  7 | Emilia     | Johnston  | 0.47368422
  6 | Emma       | Johnson   |  0.3888889
  2 | John       | Johnson   |     0.3125
  1 | John       | Smith     | 0.21052632
  3 | Jonathan   | Johanson  |      0.125
```

Interestingly, if we switch the order of the field names compared to the given string (at least for two fields), we get the same results:

```sql
SELECT id, last_name, first_name,
       similarity(CONCAT_WS(' ', last_name, first_name), 'Emily Johns') AS score
FROM people
ORDER BY score DESC
LIMIT 5;

 id | last_name | first_name |   score
---+---------+----------+----------
  7 | Johnston  | Emilia     | 0.47368422
  6 | Johnson   | Emma       |  0.3888889
  2 | Johnson   | John       |     0.3125
  1 | Smith     | John       | 0.21052632
  3 | Johanson  | Jonathan   |      0.125
(5 rows)
```

## Simple Fuzzy Search Rails

### Single Field Example

```ruby
threshold = 0.3
similarity_string = 'Emily Johns'
similarity_quoted = ActiveRecord::Base.connection.quote(similarity_string)

Person.where("similarity(last_name, ?) > ?", similarity_string, threshold)
      .order("similarity(last_name, #{similarity_quoted}) DESC")
      .limit(5)
#  Dangerous query method (method whose arguments are used as raw SQL) called with non-attribute argument(s): "similarity(last_name, 'Emily Johns') DESC".This method should not be called with user-provided values, such as request parameters or model attributes. Known-safe values can be passed by wrapping them in Arel.sql(). (ActiveRecord::UnknownAttributeReference)

Person.select(Arel.sql("*, similarity(last_name, #{similarity_quoted}) AS score"))
      .where("similarity(last_name, ?) > ?", similarity_string, threshold)
      .order(Arel.sql("similarity(last_name, #{similarity_quoted}) DESC"))
      .limit(5)

[#<Person:0x000000011dfce108
  id: 2,
  last_name: "Johnson",
  first_name: "John",
  birthdate: "1970-01-01",
  created_at: "2024-10-31 13:42:15.380507000 +0000",
  updated_at: "2024-10-31 13:42:15.380507000 +0000",
  score: 0.33333334>,
 #<Person:0x000000011dfcdfc8
  id: 6,
  last_name: "Johnson",
  first_name: "Emma",
  birthdate: "1962-01-01",
  created_at: "2024-10-31 13:42:15.456393000 +0000",
  updated_at: "2024-10-31 13:42:15.456393000 +0000",
  score: 0.33333334>,
 #<Person:0x000000011dfcde88
  id: 7,
  last_name: "Johnston",
  first_name: "Emilia",
  birthdate: "1966-01-01",
  created_at: "2024-10-31 13:42:15.473551000 +0000",
  updated_at: "2024-10-31 13:42:15.473551000 +0000",
  score: 0.3125>]
```


Which can be simplified to:
```ruby
compare_string = 'Emily Johns'
compare_quoted = ActiveRecord::Base.connection.quote(compare_string)
similarity_string = "similarity(last_name, #{compare_quoted})"

Person.select("*, #{similarity_string} AS score")
      .order("score DESC")
      .limit(5)
```
That's much more readable!

Note: only searching the last name doesn't fully help us narrow our choices.  let's try searching the first and last name together:

### Two Fields Example

This is where it gets a little tricky, but at the same time much more useful.  We can use `CONCAT_WS` to consolidate multiple fields into a single value (and compare that to the given string):

```ruby
Person
  .select(
    "*, similarity(CONCAT_WS(' ', last_name, first_name), 'Emily Johns') AS score"
  ).order("score DESC")
  .limit(3)

[#<Person:0x000000011ddf1060
  id: 7,
  last_name: "Johnston",
  first_name: "Emilia",
  birthdate: "1966-01-01",
  created_at: "2024-10-31 18:30:08.773495000 +0000",
  updated_at: "2024-10-31 18:30:08.773495000 +0000",
  score: 0.47368422>,
 #<Person:0x000000011ddf0f20
  id: 6,
  last_name: "Johnson",
  first_name: "Emma",
  birthdate: "1962-01-01",
  created_at: "2024-10-31 18:30:08.757941000 +0000",
  updated_at: "2024-10-31 18:30:08.757941000 +0000",
  score: 0.3888889>,
 #<Person:0x000000011ddf0de0
  id: 2,
  last_name: "Johnson",
  first_name: "John",
  birthdate: "1970-01-01",
  created_at: "2024-10-31 18:30:08.640962000 +0000",
  updated_at: "2024-10-31 18:30:08.640962000 +0000",
  score: 0.3125>]
```

Again, this is hard to read - let's try making it a bit more readable:

```ruby
compare_string = 'Emily Johns'
compare_quoted = ActiveRecord::Base.connection.quote(compare_string)
similarity_string =
  "similarity(CONCAT_WS(' ', last_name, first_name), #{compare_quoted})"

Person.select("*, #{similarity_string} AS score")
      .order("score DESC")
      .limit(3)

[#<Person:0x000000011dad49e0
  id: 7,
  last_name: "Johnston",
  first_name: "Emilia",
  birthdate: "1966-01-01",
  created_at: "2024-10-31 18:30:08.773495000 +0000",
  updated_at: "2024-10-31 18:30:08.773495000 +0000",
  score: 0.47368422>,
 #<Person:0x000000011dad48a0
  id: 6,
  last_name: "Johnson",
  first_name: "Emma",
  birthdate: "1962-01-01",
  created_at: "2024-10-31 18:30:08.757941000 +0000",
  updated_at: "2024-10-31 18:30:08.757941000 +0000",
  score: 0.3888889>,
 #<Person:0x000000011dad4760
  id: 2,
  last_name: "Johnson",
  first_name: "John",
  birthdate: "1970-01-01",
  created_at: "2024-10-31 18:30:08.640962000 +0000",
  updated_at: "2024-10-31 18:30:08.640962000 +0000",
  score: 0.3125>]
```

### Multi-Table Fuzzy Search

Lets say we want to search for a person by last_name, first_name, title and department (we aren't sure what data we have available).

```ruby
compare_string = 'Emily, a research scientist'
compare_quoted = ActiveRecord::Base.connection.quote(compare_string)
similarity_string =
  "similarity(" \
    "CONCAT_WS(' ', first_name, first_name, job_title, department), " \
    "#{compare_quoted})"

Person.joins(:roles)
      .select("*, #{similarity_string} AS score")
      .order("score DESC")
      .limit(3)

[#<Person:0x000000012277de90
  id: 7,
  last_name: "Johnston",
  first_name: "Emilia",
  birthdate: "1966-01-01",
  created_at: "2024-10-31 18:30:08.788015000 +0000",
  updated_at: "2024-10-31 18:30:08.788015000 +0000",
  job_title: "Data Scientist",
  department: "Research",
  person_id: 7,
  score: 0.6571429>,
 #<Person:0x000000012277dc10
  id: 6,
  last_name: "Johnson",
  first_name: "Emma",
  birthdate: "1962-01-01",
  created_at: "2024-10-31 18:30:08.765276000 +0000",
  updated_at: "2024-10-31 18:30:08.765276000 +0000",
  job_title: "Data Scientist",
  department: "Research",
  person_id: 6,
  score: 0.6>,
 #<Person:0x000000012277dad0
  id: 14,
  last_name: "Martinez",
  first_name: "Mason",
  birthdate: "1988-01-01",
  created_at: "2024-10-31 18:30:08.910685000 +0000",
  updated_at: "2024-10-31 18:30:08.910685000 +0000",
  job_title: "Business Analyst",
  department: "Research",
  person_id: 14,
  score: 0.22916667>]
```

Sweet, this works amayingly well!

## Efficiency & Performance with indexing

* **Trigram Indexes** using (`gist_trgm_ops`) - approximates a set of trigrams as a bitmap signature.
  - `CREATE INDEX trgm_idx ON people USING GIST (last_name gist_trgm_ops);`
  - `CREATE INDEX trgm_idx ON people USING GIN (last_name gin_trgm_ops);`
  - `CREATE INDEX trgm_idx ON people USING GIST (last_name gist_trgm_ops(siglen=32));` Example of creating such an index with a signature length of 32 bytes:

```sql
SELECT first_name, similarity(first_name, 'Emily') AS sml
  FROM people
  WHERE first_name % 'Emily'
  ORDER BY sml DESC, first_name;
```

NOTES:
* **Beginning in PostgreSQL 9.1**, these index types also support index searches for LIKE and ILIKE, for example:
`SELECT * FROM people WHERE first_name ILIKE '%emil%';`
* **Beginning in PostgreSQL 9.3**, these index types also support index searches for regular-expression matches (~ and ~* operators), for example:
`SELECT * FROM people WHERE first_name ~ '(Emily|John)';`

## Distance Searching (instead of similarity)

Use `distance` instead of `similarity`, we can use the `<->` operator instead of `(1.0 - similarity(last_name, 'Johns'))` to compare a db field to a string - using:

```sql
SELECT id, last_name, first_name,
       (1.0 - similarity(last_name, 'Johns')) AS score
  FROM people
  WHERE last_name % 'Johns'
  ORDER BY score DESC;

 id | last_name | first_name |       score
---+---------+----------+----------------
  7 | Johnston  | Emilia     |                0.5
  2 | Johnson   | John       | 0.4444444179534912
  6 | Johnson   | Emma       | 0.4444444179534912
(3 rows)


SELECT id, last_name, first_name,
       last_name <-> 'Johns' AS score
  FROM people
  WHERE last_name % 'Johns'
  ORDER BY score DESC;

 id | last_name | first_name |   score
---+---------+----------+----------
  7 | Johnston  | Emilia     |        0.5
  2 | Johnson   | John       | 0.44444442
  6 | Johnson   | Emma       | 0.44444442
```

Here is a summary of the short-cut **distance** operators:
* `text <-> text` → real - Returns the “distance” between the arguments, that is one minus the similarity() value.
* `text <<-> text` → real - Returns the “distance” between the arguments, that is one minus the word_similarity() value.
* `text <->> text` → real - Commutator of the <<-> operator.
* `text <<<-> text` → real - Returns the “distance” between the arguments, that is one minus the strict_word_similarity() value.
* `text <->>> text` → real - Commutator of the <<<-> operator.

## Boolean Search Operators (shortens where clauses)

Here is a summary of short-cut **boolean** operators (which can be used in the where clauses):
* `text % text` → Returns true if its arguments have a similarity that is greater than the current similarity threshold set by pg_trgm.similarity_threshold.
* `text <% text` → Returns true if the similarity between the trigram set in the first argument and a continuous extent of an ordered trigram set in the second argument is greater than the current word similarity threshold set by pg_trgm.word_similarity_threshold parameter.
* `text %> text` → Commutator of the <% operator.
* `text <<% text` → Returns true if its second argument has a continuous extent of an ordered trigram set that matches word boundaries, and its similarity to the trigram set of the first argument is greater than the current strict word similarity threshold set by the pg_trgm.strict_word_similarity_threshold parameter.
* `text %>> text` → Commutator of the <<% operator.

## Conclusion

`similarity`, `word_similarity` & `strict_word_similarity` - are straight forward ways to find records from partial information.

Depending on your use case, you may want to consider the following other methods:

* [Pattern Matching](https://www.postgresql.org/docs/current/functions-matching.html)
  - `LIKE` & `ILIKE` - sub-string string matching
  - [SIMILAR TO](https://www.postgresql.org/docs/current/functions-matching.html#FUNCTIONS-SIMILARTO-REGEXP)
    * `string SIMILAR TO pattern [ESCAPE escape-character]`
    * `string NOT SIMILAR TO pattern [ESCAPE escape-character]`
  - [POSIX Regular Expressions](https://www.postgresql.org/docs/current/functions-matching.html#FUNCTIONS-POSIX-REGEXP)
    * `text ~ text → boolean` ie. (`'thomas' ~ 't.*ma'` = true) - String matches regular expression, case sensitively
    * `text ~* text → boolean` i.e. (`'thomas' ~* 'T.*ma'` = true) - string matches regular expression, case-insensitively
    * `text !~ text → boolean` i.e. (`'thomas' !~ 't.*max'` = true) - String does not match regular expression, case sensitively
    * `text !~* text → boolean` i.e. (`'thomas' !~* 'T.*ma'` = false) -  - String does not match regular expression, case-insensitively

* [fuzzystrmatch](https://www.postgresql.org/docs/current/fuzzystrmatch.html) - determine string similarities and distance
  - `Soundex` (do not use with UTF-8)
  - `Metaphone` (do not use with UTF-8)
  - `Double Metaphone` (do not use with UTF-8)
  - `Levenshtein` (work with UTF-8)
  - `Daitch-Mokotoff Soundex` (work with UTF-8)

* [Full Text (Document) Search](https://www.postgresql.org/docs/17/textsearch.html) - **big topic** - can in some cases replace something like elastic search.

* [pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html) - efficient scored trigram matching
  - `similarity` - Returns a number that indicates how similar the two arguments are. The range of the result is zero (indicating that the two strings are completely dissimilar) to one (indicating that the two strings are identical).
  - `word_similarity` - Returns a number that indicates the greatest similarity between the set of trigrams in the first string and any continuous extent of an ordered set of trigrams in the second string. For details, see the explanation below.
  - `strict_word_similarity` - ame as word_similarity, but forces extent boundaries to match word boundaries. Since we don't have cross-word trigrams, this function actually returns greatest similarity between first string and any continuous extent of words of the second string.

**NOTE:** There is a gem called [pg_search](https://github.com/Casecommons/pg_search) that provides the trigram search functionality, but given how straight-forward this is, I'd recommend just doing it yourself.

## Resources & Articles

* [Postgres Fuzzy Search](https://dev.to/moritzrieger/build-a-fuzzy-search-with-postgresql-2029)
* [Optimizing Postgres Text Search with Trigrams](https://alexklibisz.com/2022/02/18/optimizing-postgres-trigram-search)
* [Awesome Autocomplete: Trigram Search in Rails and PostgreSQL](https://www.sitepoint.com/awesome-autocomplete-trigram-search-in-rails-and-postgresql/)


## Docs

* [pg_trgm](https://www.postgresql.org/docs/12/pgtrgm.html)
* [Full Text Search](https://www.postgresql.org/docs/12/textsearch.html)
* [fuzzy string match](https://www.postgresql.org/docs/current/fuzzystrmatch.html)
