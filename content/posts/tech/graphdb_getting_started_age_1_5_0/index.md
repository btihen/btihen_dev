---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "GraphDB - Apache AGE 1.5.0"
subtitle: "Getting Started with Postgres AGE GraphDB"
summary: ""
authors: ['btihen']
tags: ['Rails', "GraphDB", "Postgres AGE"]
categories: ["Code", "GraphDB", "Ruby Language", "Rails Framework"]
date: 2024-05-05T01:20:00+02:00
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

In this article, I want to play with the new Postgres **Apache AGE** GraphDB since I primarily Postgres already and using Neo4j can be quite expensive for real-world applications.

Unfortunately, Apache AGE is not directly supported the Ruby world. This article plays with integrating AGE into rails using the PG gem.

(Note: PostgreSQL versions 11, 12, 13, 14, 15 & 16 are supported.)

## Getting Started - Install Apache Age

### Compile the Apache AGE extension

Compile and install the AGE extension into Postgres if you are already running postgres on your computer.

1. Download from: https://age.apache.org/download/ or clone using: `git clone https://github.com/apache/age.git`
2. Go into the source code directory of Apache AGE, something like `cd age` if using the Git Repo
3. Run `make install` If the path to your Postgres installation is not in the PATH variable, then add the path using: `make PG_CONFIG=/path/to/postgres/bin/pg_config install`

### Postgres Apache AGE

If you are not already running Postgres on you computer then using a **docker image** is probably the easiest way forward:

1. Get the docker image with: `docker pull apache/age`
2. Create AGE docker container with:
```
docker run \
    --name age  \
    -p 5455:5432 \
    -e POSTGRES_USER=pgUser \
    -e POSTGRES_PASSWORD=pgPassword \
    -e POSTGRES_DB=ageDB \
    -d \
    apache/age
```
3. now you can enter postgresql using:
```
docker exec -it age psql -d ageDB -U ageUser
```
although I tend to use:
```
export PGPASSWORD=agePassword

psql -d ageDB -U ageUser -h localhost -p 5455
```

### Post Install

No matter how you installed Apache Age, you now need to do the following withing postgres:
```
CREATE EXTENSION IF NOT EXISTS age;

LOAD 'age';

SET search_path = ag_catalog, "$user", public;
```

## Exploring Cypher SQL

Cypher SQL is be the basis for interacting with Apache AGE - so let's be sure the basics work within a psql session:

First we need to make an Apache AGE schema for us to use (which is cool, because that means we can within one app use both Apache AGE and normal SQL records)

### Create a Schema

To create a graph, use the create_graph function located in the ag_catalog namespace using - you can use whatever name you want here I am using `age_schema`:
```sql
SELECT create_graph('age_schema');
```

now you can view your new schema with `\dn`
```
ageDB=# \dn
        List of schemas
    Name    |       Owner
------------+-------------------
 ag_catalog | pgUser
 age_schema | postgresUser
 public     | pg_database_owner
```

### Create a Node (Vertex)

To create a single vertex (node) with label and properties, use the CREATE clause. Let's enter `Fred Flintstone`.  Given our schema `age_schema` the `CREATE` command in `psql` looks like:
```sql
SELECT *
FROM cypher('age_schema', $$
    CREATE (f:Person {first_name: "Fred", last_name: "Flintstone", given_name: "Flintstone", gender: "male"})
    return f
$$) as (person agtype);

                      person
---------------------------------------------
 {"id": 844424930131969, "label": "Person", "properties": {"gender": "male", "last_name": "Flintstone", "first_name": "Fred", "given_name": "Flintstone"}}::vertex
(1 row)
```

This creates a type `vertex` (node) with a `person` label and the properties (attributes) of `{ first_name: "Fred", last_name: "Flintstone", given_name: "Flintstone", gender: "male" }` to verify this lets find our record using a `MATCH` instead of `CREATE` command:

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (p:Person)
    WHERE p.first_name = "Fred" AND p.last_name = "Flintstone"
    RETURN p
$$) as (person agtype);

                person
-----------------------------------------------
{"id": 844424930131969, "label": "Person", "properties": {"gender": "male", "last_name": "Flintstone", "first_name": "Fred", "given_name": "Flintstone"}}::vertex
(1 row)
```

To match by ID we need a slightly different query we need to drop the prefix `p` which contains the properties so now the query looks like:
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (p:Person)
    WHERE id(p) = 844424930131969
    RETURN p
$$) as (person agtype);
```

If we want the persons first and last name we can change the return (like adding a select):
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (p:Person)
    WHERE id(p) = 844424930131969
    RETURN p.first_name, p.last_name
$$) as (FirstName agtype, LastName agtype);

 firstname |   lastname
-----------+--------------
 "Fred"    | "Flintstone"
(1 row)
```

Now let's enter Wilma Flintstone with:
```sql
SELECT *
FROM cypher('age_schema', $$
    CREATE (w:Person {first_name: "Wilma", last_name: "Flintstone", given_name: "Slaghoople", gender: "female"})
    return w
$$) as (person agtype);

                            person
--------------------------------------------------
{"id": 844424930131970, "label": "Person", "properties": {"gender": "female", "last_name": "Flintstone", "first_name": "Wilma", "given_name": "Slaghoople"}}::vertex
(1 row)
```

### Create an Relationship (Edge)

Now that we have two vertex we can connect them with an `edge` with a `married_to` label (usually a verb `likes`, or `married_to`, `works_at`, etc) we can do that with the command and the `family_name` as the edge property:
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:Person), (w:Person)
    WHERE f.first_name = 'Fred' AND w.first_name = 'Wilma'
    CREATE
      (f)-[e:MarriedTo {role: "husband", family_name:f.given_name + '-' + w.given_name}]->(w)
    RETURN e
$$) as (married_to agtype);

                    married_to
-----------------------------------------------
{"id": 1125899906842625, "label": "MarriedTo", "end_id": 844424930131970, "start_id": 844424930131969, "properties": {"role": "husband", "family_name": "Flintstone-Slaghoople"}}::edge
(1 row)
```

Note this time it returned a type `edge` and also not that the `family_name` operation worked and resulted in: `Flintstone-Slaghoople`!  There are lots of other operators documented at:

Note: we can use an operation to use known properties to create a new property: `family_name:f.given_name + '-' + w.given_name`

Since AGE Cypher only allows uni-directional relationships, we can also have Wilma married to fred with:
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:Person), (w:Person)
    WHERE f.first_name = 'Fred' AND w.first_name = 'Wilma'
    CREATE
      path = (w)-[edge:MarriedTo {role: "wife", family_name:f.given_name + '-' + w.given_name}]->(f)
    RETURN edge, path
$$) as (Relationship agtype, Path agtype);

          relationship       |            path
--------------------------------------------------------
{"id": 1125899906842626, "label": "MarriedTo", "end_id": 844424930131969, "start_id": 844424930131970, "properties": {"role": "wife", "family_name": "Flintstone-Slaghoople"}}::edge | [{"id": 844424930131970, "label": "Person", "properties": {"gender": "female", "last_name": "Flintstone", "first_name": "Wilma", "given_name": "Slaghoople"}}::vertex, {"id": 1125899906842626, "label": "MarriedTo", "end_id": 844424930131969, "start_id": 844424930131970, "properties": {"role": "wife", "family_name": "Flintstone-Slaghoople"}}::edge, {"id": 844424930131969, "label": "Person", "properties": {"gender": "male", "last_name": "Flintstone", "first_name": "Fred", "given_name": "Flintstone"}}::vertex]::path
(1 row)
```

**Note:** in this case I asked it to return both the edge and the full path we created.  (sorry, its hard to read, but wanted to demonstrate they type `path` exists too - which contains both `vertices` and `edges`)

### Create a Path (Nodes and Edges together)

Let's make a full path - with this we can create edges and nodes if they don't already exist.  For example we have Fred, but not Barney nor their workplace.

First we find Fred, then we create the Company and Barney and their relationships to each other -- all in one go. The cypher command looks like:

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:Person)
    WHERE f.first_name = 'Fred'
    CREATE p = (f)-[:WorksAt {role: "Crane Operator"}]->(:Company {name: "Bedrock Quarry"})<-[:WorksAt]-(:Person {first_name: 'Barney', last_name: "Rubble", given_name: "Rubble", gender: "male"})
    RETURN p
$$) as (path agtype);

                            path
----------------------------------------------------------
[{"id": 844424930131969, "label": "Person", "properties": {"gender": "male", "last_name": "Flintstone", "first_name": "Fred", "given_name": "Flintstone"}}::vertex, {"id": 1407374883553282, "label": "WorksAt", "end_id": 1688849860263937, "start_id": 844424930131969, "properties": {"role": "Crane Operator"}}::edge, {"id": 1688849860263937, "label": "company", "properties": {"name": "Bedrock Quarry"}}::vertex, {"id": 1407374883553281, "label": "WorksAt", "end_id": 1688849860263937, "start_id": 844424930131971, "properties": {}}::edge, {"id": 844424930131971, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Barney", "given_name": "Rubble"}}::vertex]::path
(1 row)
```

Note: we returned the full path: Fred his role at work, barney and his role at work. Pretty cool when only Fred existed before this command.  (You can also do this without a match - where all aspects are new!)

### Multiple Creates in one Command

Let's create Betty Rubble and her marriage with Barney.
first_name: 'Betty', last_name: 'Rubble', given_name: 'McBricker', gender: 'female'.

The trick to multiple creates in one command is to separate them with a comma.  So the command looks like:
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (ba:Person)
    WHERE ba.first_name = 'Barney'
    CREATE p1 = (ba)-[:MarriedTo {role: "husband"}]->(be:Person {first_name: 'Betty', last_name: 'Rubble', given_name: 'McBricker', gender: 'female'}),
    p2 = (be)-[:MarriedTo {role: 'wife'}]->(ba)
    RETURN p1, p2
$$) as (path1 agtype, path2 agtype);

            path1        |          path2
----------------------------------------------------
[{"id": 844424930131971, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Barney", "given_name": "Rubble"}}::vertex, {"id": 1125899906842627, "label": "MarriedTo", "end_id": 844424930131972, "start_id": 844424930131971, "properties": {"role": "husband"}}::edge, {"id": 844424930131972, "label": "Person", "properties": {"gender": "female", "last_name": "Rubble", "first_name": "Betty", "given_name": "McBricker"}}::vertex]::path | [{"id": 844424930131972, "label": "Person", "properties": {"gender": "female", "last_name": "Rubble", "first_name": "Betty", "given_name": "McBricker"}}::vertex, {"id": 1125899906842628, "label": "MarriedTo", "end_id": 844424930131971, "start_id": 844424930131972, "properties": {"role": "wife"}}::edge, {"id": 844424930131971, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Barney", "given_name": "Rubble"}}::vertex]::path
(1 row)
```

Alternatively, and probably more efficiently we could have created all paths together using:
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (barney:Person)
    WHERE barney.first_name = 'Barney'
    CREATE path = (barney)-[:MarriedTo {role: "husband"}]->(betty:Person {first_name: 'Betty', last_name: 'Rubble', given_name: 'McBricker', gender: 'female'})-[:MarriedTo {role: 'wife'}]->(barney)
    RETURN path
$$) as (path agtype);
```
I just wanted demonstrate two creates in one command.

Let's also add the work place of Wilma and Betty:
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (b:Person), (w:Person)
    WHERE b.first_name = 'Betty' AND w.first_name = "Wilma"
    CREATE p = (b)-[:WorksAt {role: "Reporter"}]->(:Company {name: "Bedrock News"})<-[:WorksAt {role: "News Anchor"}]-(w)
    RETURN p
$$) as (path agtype);

                            path
-----------------------------------------------------
[{"id": 844424930131972, "label": "Person", "properties": {"gender": "female", "last_name": "Rubble", "first_name": "Betty", "given_name": "McBricker"}}::vertex, {"id": 1407374883553284, "label": "WorksAt", "end_id": 1688849860263938, "start_id": 844424930131972, "properties": {"role": "Reporter"}}::edge, {"id": 1688849860263938, "label": "company", "properties": {"name": "Bedrock News"}}::vertex, {"id": 1407374883553283, "label": "WorksAt", "end_id": 1688849860263938, "start_id": 844424930131970, "properties": {"role": "News Anchor"}}::edge, {"id": 844424930131970, "label": "Person", "properties": {"gender": "female", "last_name": "Flintstone", "first_name": "Wilma", "given_name": "Slaghoople"}}::vertex]::path
(1 row)
```

### Query Relations

We will start with a very generic query to find any nodes connected with a relationship:
```sql
SELECT * from cypher('age_schema', $$
        MATCH (N)-[R]-(N2)
        RETURN N,R,N2
$$) as (StartNode agtype, Relationship agtype, EndNode agtype);

         startnode    |   relationship    |     endnode
---------------------------------------------------
 {"id": 844424930131971, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Barney", "given_name": "Rubble"}}::vertex | {"id": 1407374883553281, "label": "WorksAt", "end_id": 1688849860263937, "start_id": 844424930131971, "properties": {}}::edge  | {"id": 1688849860263937, "label": "company", "properties": {"name": "Bedrock Quarry"}}::vertex
 ...
```

you can see we can return 3 columns and describe these columns in `as` statement.

Now you can see we have a big jumble multiple types of relationships and entities returned in one query. If we want to return just marriage relations we can write:

```sql
SELECT * from cypher('age_schema', $$
        MATCH p = (c1:Person)-[r:MarriedTo]-(c2:Person)
        RETURN p
$$) as (Marriages agtype);

                          marriages
--------------------------------------------------------
[{"id": 844424930131969, "label": "Person", "properties": {"gender": "male", "last_name": "Flintstone
", "first_name": "Fred", "given_name": "Flintstone"}}::vertex, {"id": 1125899906842625, "label": "Marr
iedTo", "end_id": 844424930131970, "start_id": 844424930131969, "properties": {"role": "husband", "fam
ily_name": "Flintstone-Slaghoople"}}::edge, {"id": 844424930131970, "label": "Person", "properties": {
"gender": "female", "last_name": "Flintstone", "first_name": "Wilma", "given_name": "Slaghoople"}}::ve
rtex]::path
[{"id": 844424930131969, "label": "Person", "properties": {"gender": "male", "last_name": "Flintstone", "first_name": "Fred", "given_name": "Flintstone"}}::vertex, {"id": 1125899906842626, "label": "MarriedTo", "end_id": 844424930131969, "start_id": 844424930131970, "properties": {"role": "wife", "family_name": "Flintstone-Slaghoople"}}::edge, {"id": 844424930131970, "label": "Person", "properties": {"gender": "female", "last_name": "Flintstone", "first_name": "Wilma", "given_name": "Slaghoople"}}::vertex]::path
 ...
```

And if we just want to see who works where:

```sql
SELECT * from cypher('age_schema', $$
        MATCH p = (c1:Person)-[r:WorksAt]-(c2:Company)
        RETURN p
$$) as (Employees agtype);

                        employees
----------------------------------------------------------
[{"id": 844424930131969, "label": "Person", "properties": {"gender": "male", "last_name": "Flintstone", "first_name": "Fred", "given_name": "Flintstone"}}::vertex, {"id": 1407374883553282, "label": "WorksAt", "end_id": 1688849860263937, "start_id": 844424930131969, "properties": {"role": "Crane Operator"}}::edge, {"id": 1688849860263937, "label": "company", "properties": {"name": "Bedrock Quarry"}}::vertex]::path
[{"id": 844424930131970, "label": "Person", "properties": {"gender": "female", "last_name": "Flintstone", "first_name": "Wilma", "given_name": "Slaghoople"}}::vertex, {"id": 1407374883553283, "label": "WorksAt", "end_id": 1688849860263938, "start_id": 844424930131970, "properties": {"role": "News Anchor"}}::edge, {"id": 1688849860263938, "label": "company", "properties": {"name": "Bedrock News"}}::vertex]::path
 ...
(4 rows)
```

### Deep Relationship Paths

Let's create a few more family relationships so that we can build a family tree a few levels deep.  We will build the children and grand-children from the `Flintstones` and `Rubbles`

Let's start with `Bamm-Bamm`

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (father:Person), (mother:Person)
    WHERE father.first_name = 'Barney' and mother.first_name = "Betty"
    CREATE
      parenthood = (father)-[:ParentOf {role: "Father"}]->(child:Person {first_name: "Bamm-Bamm", last_name: "Rubble", given_name: "Rubble", gender: "male"})<-[:ParentOf {role: "Mother"}]-(mother),
      childhood = (father)<-[:ChildOf {role: "Son"}]-(child)-[:ChildOf {role: "Son"}]->(mother)
    RETURN parenthood, childhood
$$) as (Parenthood agtype, Childhood agtype);

            parenthood      |       childhood
----------------------------------------------------
[{"id": 844424930131971, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Barney", "given_name": "Rubble"}}::vertex, {"id": 1970324836974594, "label": "parent_of", "end_id": 844424930131973, "start_id": 844424930131971, "properties": {"role": "Father"}}::edge, {"id": 844424930131973, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Bamm-Bamm", "given_name": "Rubble"}}::vertex, {"id": 1970324836974593, "label": "parent_of", "end_id": 844424930131973, "start_id": 844424930131972, "properties": {"role": "Mother"}}::edge, {"id": 844424930131972, "label": "Person", "properties": {"gender": "female", "last_name": "Rubble", "first_name": "Betty", "given_name": "McBricker"}}::vertex]::path | [{"id": 844424930131971, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Barney", "given_name": "Rubble"}}::vertex, {"id": 2251799813685250, "label": "child_of", "end_id": 844424930131971, "start_id": 844424930131973, "properties": {"role": "Son"}}::edge, {"id": 844424930131973, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Bamm-Bamm", "given_name": "Rubble"}}::vertex, {"id": 2251799813685249, "label": "child_of", "end_id": 844424930131972, "start_id": 844424930131973, "properties": {"role": "Son"}}::edge, {"id": 844424930131972, "label": "Person", "properties": {"gender": "female", "last_name": "Rubble", "first_name": "Betty", "given_name": "McBricker"}}::vertex]::path
(1 row)
```
And now `Pebbles`

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:Person), (m:Person)
    WHERE f.first_name = 'Fred' and m.first_name = "Wilma"
    CREATE
      pp = (f)-[:ParentOf {role: "Father"}]->(c:Person {first_name: "Pebbles", last_name: "Rubble", given_name: "Rubble", gender: "male"})<-[:ParentOf {role: "Mother"}]-(m),
      cp = (f)<-[:ChildOf {role: "Daughter"}]-(c)-[:ChildOf {role: "Daughter"}]->(m)
    RETURN pp, cp
$$) as (ParentalPaths agtype, ChildPaths agtype);

            parenthood      |       childhood
----------------------------------------------------
[{"id": 844424930131969, "label": "Person", "properties": {"gender": "male", "last_name": "Flintstone", "first_name": "Fred", "given_name": "Flintstone"}}::vertex, {"id": 1970324836974596, "label": "parent_of", "end_id": 844424930131974, "start_id": 844424930131969, "properties": {"role": "Father"}}::edge, {"id": 844424930131974, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Pebbles", "given_name": "Rubble"}}::vertex, {"id": 1970324836974595, "label": "parent_of", "end_id": 844424930131974, "start_id": 844424930131970, "properties": {"role": "Mother"}}::edge, {"id": 844424930131970, "label": "Person", "properties": {"gender": "female", "last_name": "Flintstone", "first_name": "Wilma", "given_name": "Slaghoople"}}::vertex]::path | [{"id": 844424930131969, "label": "Person", "properties": {"gender": "male", "last_name": "Flintstone", "first_name": "Fred", "given_name": "Flintstone"}}::vertex, {"id": 2251799813685252, "label": "child_of", "end_id": 844424930131969, "start_id": 844424930131974, "properties": {"role": "Daughter"}}::edge, {"id": 844424930131974, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Pebbles", "given_name": "Rubble"}}::vertex, {"id": 2251799813685251, "label": "child_of", "end_id": 844424930131970, "start_id": 844424930131974, "properties": {"role": "Daughter"}}::edge, {"id": 844424930131970, "label": "Person", "properties": {"gender": "female", "last_name": "Flintstone", "first_name": "Wilma", "given_name": "Slaghoople"}}::vertex]::path
(1 row)
```

**son of bamm-bamm & pebbles**
chip = Character.create!(species: human, first_name: 'Charleston Frederick', nick_name: 'Chip', last_name: 'Rubble', gender: 'male')
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:Person), (m:Person)
    WHERE f.first_name = 'Bamm-Bamm' and m.first_name = "Pebbles"
    CREATE
      pp = (f)-[:ParentOf {role: "Father"}]->(c:Person {first_name: "Chip", last_name: "Rubble", given_name: "Rubble", gender: "male"})<-[:ParentOf {role: "Mother"}]-(m),
      cp = (f)<-[:ChildOf {role: "Son"}]-(c)-[:ChildOf {role: "Son"}]->(m)
    RETURN pp, cp
$$) as (ParentalPaths agtype, ChildPaths agtype);

            parenthood      |       childhood
----------------------------------------------------
[{"id": 844424930131973, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Bamm-Bamm", "given_name": "Rubble"}}::vertex, {"id": 1970324836974598, "label": "parent_of", "end_id": 844424930131975, "start_id": 844424930131973, "properties": {"role": "Father"}}::edge, {"id": 844424930131975, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Chip", "given_name": "Rubble"}}::vertex, {"id": 1970324836974597, "label": "parent_of", "end_id": 844424930131975, "start_id": 844424930131974, "properties": {"role": "Mother"}}::edge, {"id": 844424930131974, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Pebbles", "given_name": "Rubble"}}::vertex]::path | [{"id": 844424930131973, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Bamm-Bamm", "given_name": "Rubble"}}::vertex, {"id": 2251799813685254, "label": "child_of", "end_id": 844424930131973, "start_id": 844424930131975, "properties": {"role": "Son"}}::edge, {"id": 844424930131975, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Chip", "given_name": "Rubble"}}::vertex, {"id": 2251799813685253, "label": "child_of", "end_id": 844424930131974, "start_id": 844424930131975, "properties": {"role": "Son"}}::edge, {"id": 844424930131974, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Pebbles", "given_name": "Rubble"}}::vertex]::path
(1 row)
```

**daughter of bamm-bamm & pebbles**
roxy = Character.create!(species: human, first_name: 'Roxann Elisabeth', nick_name: 'Roxy', last_name: 'Rubble', gender: 'female')
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:Person), (m:Person)
    WHERE f.first_name = 'Bamm-Bamm' and m.first_name = "Pebbles"
    CREATE
      pp = (f)-[:ParentOf {role: "Father"}]->(c:Person {first_name: "Roxann", last_name: "Rubble", given_name: "Rubble", gender: "female"})<-[:ParentOf {role: "Mother"}]-(m),
      cp = (f)<-[:ChildOf {role: "Daughter"}]-(c)-[:ChildOf {role: "Daughter"}]->(m)
    RETURN pp, cp
$$) as (ParentalPaths agtype, ChildPaths agtype);

            parenthood      |       childhood
----------------------------------------------------
[{"id": 844424930131973, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Bamm-Bamm", "given_name": "Rubble"}}::vertex, {"id": 1970324836974600, "label": "parent_of", "end_id": 844424930131976, "start_id": 844424930131973, "properties": {"role": "Father"}}::edge, {"id": 844424930131976, "label": "Person", "properties": {"gender": "female", "last_name": "Rubble", "first_name": "Roxann", "given_name": "Rubble"}}::vertex, {"id": 1970324836974599, "label": "parent_of", "end_id": 844424930131976, "start_id": 844424930131974, "properties": {"role": "Mother"}}::edge, {"id": 844424930131974, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Pebbles", "given_name": "Rubble"}}::vertex]::path | [{"id": 844424930131973, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Bamm-Bamm", "given_name": "Rubble"}}::vertex, {"id": 2251799813685256, "label": "child_of", "end_id": 844424930131973, "start_id": 844424930131976, "properties": {"role": "Daughter"}}::edge, {"id": 844424930131976, "label": "Person", "properties": {"gender": "female", "last_name": "Rubble", "first_name": "Roxann", "given_name": "Rubble"}}::vertex, {"id": 2251799813685255, "label": "child_of", "end_id": 844424930131974, "start_id": 844424930131976, "properties": {"role": "Daughter"}}::edge, {"id": 844424930131974, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Pebbles", "given_name": "Rubble"}}::vertex]::path
```

Now we can find all person relations `[*]` finds any paths between `persons`

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH p = (p1:Person)-[*]->(p2:Person)
    RETURN p
$$) as (PeoplePaths agtype);
```

Unfortunately, `[*]` this is very SLOW!
To control performance we can limit the path depths.

We can also explicitly list a path length: `(u)-[]->()-[]->(v)` which is the same as `(u)-[*2]->(v)`

We can allow a path match range the following allows a relationship path lenght of 1-5: `(u)-[*..5]->(v)`.

If want matches with a minimum path length of three or more we can do: `(u)-[*3..]->(v)`.

Finally, we can limit matches between 3 and 5 with: `(u)-[*3..5]->(v)`.

Finally, we can also add a match using a label for example let's list all the parents of `Chip`


```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH path = (child:Person {first_name: 'Chip'})-[:ChildOf]->(:Person)
    RETURN path
$$) as (FamilyPath agtype);

                      familypath
----------------------------------------------------------
[{"id": 844424930131975, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Chip", "given_name": "Rubble"}}::vertex, {"id": 2251799813685254, "label": "child_of", "end_id": 844424930131973, "start_id": 844424930131975, "properties": {"role": "Son"}}::edge, {"id": 844424930131973, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Bamm-Bamm", "given_name": "Rubble"}}::vertex]::path
[{"id": 844424930131975, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Chip", "given_name": "Rubble"}}::vertex, {"id": 2251799813685253, "label": "child_of", "end_id": 844424930131974, "start_id": 844424930131975, "properties": {"role": "Son"}}::edge, {"id": 844424930131974, "label": "Person", "properties": {"gender": "male", "last_name": "Rubble", "first_name": "Pebbles", "given_name": "Rubble"}}::vertex]::path
(2 rows)
```

Cool now we have the two parents Chip - might be nicer to just list the parents and not the paths with:
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (child:Person {first_name: 'Chip'})-[:ChildOf]->(parent:Person)
    RETURN parent.first_name
$$) as (ParentName agtype);

 parentname
-------------
 "Bamm-Bamm"
 "Pebbles"
(2 rows)
```

now to query parents of parents (family tree)
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (child:Person {first_name: 'Chip'})-[:ChildOf]->(:Person)-[:ChildOf]->(parent:Person)
    RETURN parent.first_name
$$) as (Grandparents agtype);

 grandparents
------------
 "Fred"
 "Wilma"
 "Barney"
 "Betty"
(4 rows)
```

as described earlier a nicer way to rewrite this is as:
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (child:Person {first_name: 'Chip'})-[:ChildOf *2]->(parent:Person)
    RETURN parent.first_name
$$) as (Grandparents agtype);

 grandparents
--------------
 "Fred"
 "Wilma"
 "Barney"
 "Betty"
(4 rows)
```

to get both parents and grandparent we query using:
```sql
FROM cypher('age_schema', $$
    MATCH (child:Person {first_name: 'Chip'})-[:ChildOf *1..2]->(parent:Person)
    RETURN parent.first_name
$$) as (Relatives agtype);
  relatives
-------------
 "Fred"
 "Wilma"
 "Barney"
 "Betty"
 "Bamm-Bamm"
 "Pebbles"
(6 rows)
```

### Multiple Path Matching

Let's say we want to match all who have the same boss:

mr_slate = Character.create!(species: human, first_name: 'George', nick_name: 'Mr.', last_name: 'Slate', gender: 'male')
let's add `Mr Slate` to our data:
```sql
SELECT * FROM cypher('age_schema', $$
    MATCH (firm:Company {name: 'Bedrock Quarry'})
    CREATE path = (boss:Person {first_name: 'George', last_name: 'Slate', given_name: 'Slate', gender: 'male'})-[:WorksAt {role: "Manager"}]->(firm)
    RETURN path
$$) as (Boss agtype);

                            boss
-------------------------------------------------------
[{"id": 844424930131977, "label": "Person", "properties": {"gender": "male", "last_name": "Slate", "first_name": "George", "given_name": "Slate"}}::vertex, {"id": 1407374883553285, "label": "WorksAt", "end_id": 1688849860263937, "start_id": 844424930131977, "properties": {"role": "Manager"}}::edge, {"id": 1688849860263937, "label": "company", "properties": {"name": "Bedrock Quarry"}}::vertex]::path
(1 row)
```

So to find people with known bosses:

```sql
SELECT * FROM cypher('age_schema', $$
    MATCH (employee:Person)-[job:WorksAt]->(firm:Company)<-[:WorksAt {role: "Manager"}]-(boss:Person)
    RETURN employee.first_name, job.role, firm.name, boss.last_name
$$) as (employee agtype, job agtype, firm agtype, boss agtype);
 employee |       job        |       firm       |  boss
----------+------------------+------------------+---------
 "Fred"   | "Crane Operator" | "Bedrock Quarry" | "Slate"
 "Barney" |                  | "Bedrock Quarry" | "Slate"
(2 rows)
```

OOPS - we forgot to add Barney's role.  We can fix this with:
```sql
SELECT * FROM cypher('age_schema', $$
    MATCH (employee:Person {first_name: "Barney"})-[job:WorksAt]->(firm:Company)
    SET job.role = 'Crane Operator'
    RETURN employee.first_name, job.role, firm.name
$$) as (employee agtype, job agtype, firm agtype);

 employee |       job        |       firm
----------+------------------+------------------
 "Barney" | "Crane Operator" | "Bedrock Quarry"
```

PS we can remove a property with: `SET job.role = NULL`


### Many Other Features

This is hopefully, enough of an intro to inspire exploring Apache AGE. It's complete features all well documented at: https://age.apache.org/age-manual/master/

## Resources

### Usage Example Resources

### Resources AGE

* [AGE Quickstart](https://age.apache.org/getstarted/quickstart/)
* [AGE Downloads](https://age.apache.org/download/)
* [AGE Repo](https://github.com/apache/age?tab=readme-ov-file)
* [AGE Cypher Docs](https://age.apache.org/age-manual/master/index.html)
* [Apache AGE Installation Tutorial for MacOS - compile extension](https://www.youtube.com/watch?v=0-qMwpDh0CA)

* [APACHE AGE: Getting Started Part 1 (PosgreSQL Installation)](https://dev.to/shadycj/apache-age-getting-started-part-1posgresql-installation-4o5o)
* [APACHE AGE: Getting Started Part 2 (AGE Installation)](https://dev.to/shadycj/apache-age-getting-started-part-2age-installation-33ed)
* [APACHE AGE: Getting Started Part 3 (AGE Installation via Docker)](https://dev.to/shadycj/apache-age-getting-started-part-3age-installation-via-docker-2lig)

### Related Resources

* [GraphDB in Postgres with SQL](https://www.dylanpaulus.com/posts/postgres-is-a-graph-database/)
* [Modeling graphs and trees using recursion in PostgreSQL](https://www.linkedin.com/pulse/you-dont-need-graph-database-modeling-graphs-trees-viktor-qvarfordt-efzof/)

### Neo4J Resources

* [Graph Academy](https://graphacademy.neo4j.com/)
* [Neo4J Developer](http://neo4jrb.io/)
* [Neo4J ActiveGraph Gem](https://github.com/neo4jrb/activegraph?tab=readme-ov-file)
* [Getting Started with Neo4j and Ruby](https://neo4j.com/developer/ruby-course/)
* [Neo4J Developer / Cypher Docs](https://neo4jrb.readthedocs.io/en/stable)
