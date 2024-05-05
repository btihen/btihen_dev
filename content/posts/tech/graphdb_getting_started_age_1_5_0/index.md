---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "GraphDB - Apache AGE 1.5.0"
subtitle: "Getting Started with Postgres AGE GraphDB"
summary: ""
authors: ['btihen']
tags: ['Rails', "GraphDB", "Postgres AGE"]
categories: ["Code", "GraphDB", "Ruby Language", "Rails Framework"]
date: 2024-05-05T01:20:00+02:00
lastmod: 2024-05-05T01:20:00+02:00
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
    CREATE (f:person {first_name: "Fred", last_name: "Flintstone", given_name: "Flintstone", gender: "male"})
    return f
$$) as (person agtype);
```

This creates a vertex (node) with a `person` label and the properties (attributes) of `{ first_name: "Fred", last_name: "Flintstone", given_name: "Flintstone", gender: "male" }` to verify this lets find our record using a `MATCH` instead of `CREATE` command:

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (p:person)
    WHERE p.first_name = "Fred" AND p.last_name = "Flintstone"
    RETURN p
$$) as (person agtype);
```

Now let's enter Wilma Flintstone with:
```sql
SELECT *
FROM cypher('age_schema', $$
    CREATE (w:person {first_name: "Wilma", last_name: "Flintstone", given_name: "Slaghoople", gender: "female"})
    return w
$$) as (person agtype);
```

### Create an Relationship (Edge)

Now that we have two vertex we can connect them with an `edge` with a `married_to` label (usually a verb `likes`, or `married_to`, `works_at`, etc) we can do that with the command and the `family_name` as the edge property:
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:person), (w:person)
    WHERE f.first_name = 'Fred' AND w.first_name = 'Wilma'
    CREATE
      (f)-[e:married_to {role: "husband", family_name:f.given_name + '-' + w.given_name}]->(w)
    RETURN e
$$) as (married agtype);
```

Note: we can use an operation to use known properties to create a new property: `family_name:f.given_name + '-' + w.given_name`

Since AGE Cypher only allows uni-directional relationships, we can also have Wilma married to fred with:
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:person), (w:person)
    WHERE f.first_name = 'Fred' AND w.first_name = 'Wilma'
    CREATE
      p = (w)-[r:married_to {role: "wife", family_name:f.given_name + '-' + w.given_name}]->(f)
    RETURN r, p
$$) as (Relationship agtype, Path agtype);
```

### Create a Path (Nodes and Edges together)

Let's make a full path - with this we can create edges and nodes if they don't already exist.  For example we have Fred, but not Barney nor their workplace.  Let's add them in one cypher command:

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:person)
    WHERE f.first_name = 'Fred'
    CREATE p = (f)-[:works_at {role: "Crane Operator"}]->(:company {name: "Bedrock Quarry"})<-[:works_at]-(:person {first_name: 'Barney', last_name: "Rubble", given_name: "Rubble", gender: "male"})
    RETURN p
$$) as (path agtype);
```

First we found Fred, then we created the Company and Barney on the fly.

### Multiple Creates in one Command

Let's create Betty Rubble and her marriage with Barney.
first_name: 'Betty', last_name: 'Rubble', given_name: 'McBricker', gender: 'female'
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (ba:person)
    WHERE ba.first_name = 'Barney'
    CREATE p1 = (ba)-[:married_to {role: "husband"}]->(be:person {first_name: 'Betty', last_name: 'Rubble', given_name: 'McBricker', gender: 'female'}),
    p2 = (be)-[:married_to {role: 'wife'}]->(ba)
    RETURN p1, p2
$$) as (path1 agtype, path2 agtype);
```

we can also add the work place of Wilma and Betty:

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (b:person), (w:person)
    WHERE b.first_name = 'Betty', w.first_name = "Wilma"
    CREATE p = (b)-[:works_at {role: "Reporter"}]->(:company {name: "Bedrock News"})<-[:works_at {role: "News Anchor"}]-(w)
    RETURN p
$$) as (path agtype);
```


### Query Relations

We will start with a very generic query to find any nodes connected with a relationship:
```sql
SELECT * from cypher('age_schema', $$
        MATCH (N)-[R]-(N2)
        RETURN N,R,N2
$$) as (StartNode agtype, Relationship agtype, EndNode agtype);
```

you can see we can return 3 columns and describe these columns in `as` statement.

Now you can see we have multiple types of relationships and entities returned in one query.  What if we want to return just marriage relations:

```sql
SELECT * from cypher('age_schema', $$
        MATCH p = (c1:person)-[r:married_to]-(c2:person)
        RETURN p
$$) as (MarrageRelations agtype);
```

And if we just want to see who works where:

```sql
SELECT * from cypher('age_schema', $$
        MATCH p = (c1:person)-[r:works_at]-(c2:company)
        RETURN p
$$) as (WorkerAtCompany agtype);
```

### Deep Relationship Paths

Let's create a few more family relationships so that we can build a family tree a few levels deep.  We will build the children and grand-children from the `Flintstones` and `Rubbles`

Let's start with `Bamm-Bamm`

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:person), (m:person)
    WHERE f.first_name = 'Barney', m.first_name = "Betty"
    CREATE
      pp = (f)-[:parent_of {role: "Father"}]->(c:person {first_name: "Bamm-Bamm", last_name: "Rubble", given_name: "Rubble", gender: "male"})<-[:parent_of {role: "Mother"}]-(m),
      cp = (f)<-[:child_of {role: "Son"}]-(c)-[:child_of {role: "Son"}]-(m)
    RETURN pp, cp
$$) as (ParentalPaths agtype, ChildPaths agtype);
```
And now `Pebbles`

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:person), (m:person)
    WHERE f.first_name = 'Fred', m.first_name = "Wilma"
    CREATE
      pp = (f)-[:parent_of {role: "Father"}]->(c:person {first_name: "Pebbles", last_name: "Rubble", given_name: "Rubble", gender: "male"})<-[:parent_of {role: "Mother"}]-(m),
      cp = (f)<-[:child_of {role: "Daughter"}]-(c)-[:child_of {role: "Daughter"}]-(m)
    RETURN pp, cp
$$) as (ParentalPaths agtype, ChildPaths agtype);
```

**son of bamm-bamm & pebbles**
chip = Character.create!(species: human, first_name: 'Charleston Frederick', nick_name: 'Chip', last_name: 'Rubble', gender: 'male')
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:person), (m:person)
    WHERE f.first_name = 'Bamm-Bamm', m.first_name = "Pebbles"
    CREATE
      pp = (f)-[:parent_of {role: "Father"}]->(c:person {first_name: "Chip", last_name: "Rubble", given_name: "Rubble", gender: "male"})<-[:parent_of {role: "Mother"}]-(m),
      cp = (f)<-[:child_of {role: "Son"}]-(c)-[:child_of {role: "Son"}]-(m)
    RETURN pp, cp
$$) as (ParentalPaths agtype, ChildPaths agtype);
```

**daughter of bamm-bamm & pebbles**
roxy = Character.create!(species: human, first_name: 'Roxann Elisabeth', nick_name: 'Roxy', last_name: 'Rubble', gender: 'female')
```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH (f:person), (m:person)
    WHERE f.first_name = 'Bamm-Bamm', m.first_name = "Pebbles"
    CREATE
      pp = (f)-[:parent_of {role: "Father"}]->(c:person {first_name: "Roxann", last_name: "Rubble", given_name: "Rubble", gender: "female"})<-[:parent_of {role: "Mother"}]-(m),
      cp = (f)<-[:child_of {role: "Daughter"}]-(c)-[:child_of {role: "Daughter"}]-(m)
    RETURN pp, cp
$$) as (ParentalPaths agtype, ChildPaths agtype);
```

Now we can find all person relations `[*]` finds any paths between `persons`

```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH p = (p1:person)-[*]->(p2:person)
    RETURN p
$$) as (FamilyPath agtype);
```

Unfortunately, `[*]` in a large database can find huge amounts of data.  To control performance we can limit the path depths.

We can also explicitly list a path length: `(u)-[]->()-[]->(v)` which is the same as `(u)-[*2]->(v)`

We can allow a path match range the following allows a relationship path lenght of 1-5: `(u)-[*..5]->(v)`.

If want matches with a minimum path length of three or more we can do: `(u)-[*3..]->(v)`.

Finally, we can limit matches between 3 and 5 with: `(u)-[*3..5]->(v)`.

Finally, we can also add a match using a label for example let's list all the parents of `Chip`


```sql
SELECT *
FROM cypher('age_schema', $$
    MATCH p = (p1:person)<-[*:parent_of]-(p2:person {name: 'Chip'})
    RETURN p
$$) as (FamilyPath agtype);
```

### Multiple Path Matching

Let's say we want to match all who have the same boss:

mr_slate = Character.create!(species: human, first_name: 'George', nick_name: 'Mr.', last_name: 'Slate', gender: 'male')
let's add `Mr Slate` to our data:
```sql

```

So to find crane operators and their bosses with:

```sql
SELECT * FROM cypher('graph_name', $$
    MATCH (operator:person)-[:works_at {role: "Crane Operator"}]->(firm:company)<-[:works_at {role: "Manager"}]-(boss:person)
    RETURN firm, operator, boss
$$) as (CompanyName agtype, CraneOperator agtype, Boss agtype);
```

### Return Individual Values

We can return just matched values instead of full records, by adding property values to the return statement, ie: `RETURN company.name, w1.first_name, w2.first_name` for example:

```sql
SELECT * FROM cypher('graph_name', $$
    MATCH (operator:person)-[:works_at {role: "Crane Operator"}]->(firm:company)<-[:works_at {role: "Manager"}]-(boss:person)
    RETURN firm.name, operator.first_name, boss.last_name
$$) as (CompanyName agtype, CraneOperatorName agtype, BossName agtype);
```

### Delete and Other Features

This is hopefully, enough of an intro to inspire exploring Apache AGE. It's complete features all well documented at: https://age.apache.org/age-manual/master/

## Resources AGE

* [AGE Quickstart](https://age.apache.org/getstarted/quickstart/)
* [AGE Downloads](https://age.apache.org/download/)
* [AGE Repo](https://github.com/apache/age?tab=readme-ov-file)
* [AGE Cypher Docs](https://age.apache.org/age-manual/master/index.html)
* [Apache AGE Installation Tutorial for MacOS - compile extension](https://www.youtube.com/watch?v=0-qMwpDh0CA)


## Usage Example Resources

## Related Resources

* [GraphDB in Postgres with SQL](https://www.dylanpaulus.com/posts/postgres-is-a-graph-database/)
* [Modeling graphs and trees using recursion in PostgreSQL](https://www.linkedin.com/pulse/you-dont-need-graph-database-modeling-graphs-trees-viktor-qvarfordt-efzof/)

## Neo4J Resources

* [Graph Academy](https://graphacademy.neo4j.com/)
* [Neo4J Developer](http://neo4jrb.io/)
* [Neo4J ActiveGraph Gem](https://github.com/neo4jrb/activegraph?tab=readme-ov-file)
* [Getting Started with Neo4j and Ruby](https://neo4j.com/developer/ruby-course/)
* [Neo4J Developer / Cypher Docs](https://neo4jrb.readthedocs.io/en/stable)
