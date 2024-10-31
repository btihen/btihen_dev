---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Phoenix / Ecto - Fuzzy Postgres Search using SIMILARITY"
subtitle: "Simple Effective Scored Fuzzy Searches"
# Summary for listings and search engines
summary: "Elixir, Ecto and Phoenix have a simple and powerful way to do scored fuzzy searches"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "Ecto", "Fuzzy", "Postgres"]
categories: ["Code", "Elixir Language", "Phoenix Framework", "PostgreSQL"]
date: 2024-10-04T01:01:53+02:00
lastmod: 2024-10-04T01:01:53+02:00
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

My co-worker [Gernot Kogler](https://www.linkedin.com/in/gernot-kogler-075513162/), introduced me to the trigram scoring searches using `similarity` and `word_similarity` - this is a simple and very effective way to do fuzzy searches.

If interested, a good article [Optimizing Postgres Text Search with Trigrams](https://alexklibisz.com/2022/02/18/optimizing-postgres-trigram-search) that explains the details of the scoring and optimizing the search speed.


In fact, there are several ways to do fuzzy searches in Postgres, this is probably an incomplete list, but a good starting place for those who want to explore further.

* `LIKE` & `ILIKE` - single column (exact partial matches)
* `grep` - searches matches
* `similarity` & `word_similarity` - multiple columns (scored partial)
* `pg_trgrm` (trigram) extension - efficient scored partial matches
* `fuzzystrmatch` - fuzzy searches
* `levenshtein`
* Full Text (Document) Search

In this article, we will only explore `similarity` and `word_similarity`.

## Getting Started

Let's create a test project that finds the best matching person in our database, from a 'description'.

### Requirements

Let's get the newest [erlang](https://www.erlang.org/downloads), [elixir](https://github.com/elixir-lang/elixir/releases) and [phoenix](https://hexdocs.pm/phoenix/releases.html).

```bash
# macos 15 seems to need this
ulimit -n 10240

# install the newest erlang found at: https://www.erlang.org/downloads
asdf install erlang 27.1
asdf global erlang 27.1

# install the newest elixir (with a matching erlang release) https://github.com/elixir-lang/elixir/releases
asdf install elixir 1.17.3-otp-27
asdf global elixir 1.17.3-otp-27

# install the newest phoenix
elixir -v
mix local.hex
mix archive.install hex phx_new # 1.7.14
```

### Phoenix Project

Now that we have the requirements installed, let's create a new phoenix project (with ecto!).

```bash
# create a new phoenix project
mix phx.new fuzzy

cd fuzzy
mix ecto.create

# test phoenix
iex -S mix phx.server

# assuming all went well
git init
git add .
git commit -m "initial commit"
```

### Add DB extensions

I always add `citext` and for this article we also need the `pg_trgm` extension.

```bash
mix ecto.gen.migration add_pg_extensions
```

```elixir
# priv/repo/migrations/20241003142425_add_pg_extensions.exs
defmodule Fuzzy.Repo.Migrations.AddPgExtensions do
  use Ecto.Migration

  def change do
    execute("CREATE EXTENSION IF NOT EXISTS citext", "DROP EXTENSION IF EXISTS citext")
    execute("CREATE EXTENSION IF NOT EXISTS pg_trgm", "DROP EXTENSION IF EXISTS pg_trgm")
  end
end
```
now migrate
```bash
mix ecto.migrate
```

### Create Person

Let's create a simple person schema and learn to do a fuzzy search.

```bash
mix phx.gen.schema Person people last_name:string first_name:string

mix phx.gen.schema Account accounts status:enum:active:inactive username:string:unique password:string:redact person_id:references:people

mix phx.gen.schema Department departments dept_name:string

mix phx.gen.schema Job jobs job_title:string department_id:references:departments

mix phx.gen.schema PersonJobs person_jobs seniority:enum:intern:junior:professional:senior:lead:executive person_id:references:people job_id:references:jobs
```

**Migration Update**
now let's ensure that username is case insensitive (using the `citext` extension). We change `:username` column's type from `:string` to `:citext`.

```elixir
# priv/repo/migrations/20241003150857_create_accounts.exs
defmodule Fuzzy.Repo.Migrations.CreateAccounts do
  use Ecto.Migration

  def change do
    create table(:accounts) do
      add :status, :string
      # add :username, :string
      add :username, :citext # now for Ecto this is a case insensitive string
      #              ^^^^^^^
      add :password, :string
      add :person_id, references(:people, on_delete: :nothing)

      timestamps(type: :utc_datetime)
    end

    create unique_index(:accounts, [:username])
    create index(:accounts, [:person_id])
  end
end
```

now we can migrate:

```bash
mix ecto.migrate
```

**Add Relationships**

By adding relationships we can simplify our queries.

```elixir
# lib/fuzzy/person.ex
defmodule Fuzzy.Person do
  use Ecto.Schema
  import Ecto.Changeset

  schema "people" do
    field :last_name, :string
    field :first_name, :string
    field :job_title, :string

    # add a relationship to the account
    has_one :account, Fuzzy.Account

    timestamps(type: :utc_datetime)
  end

  @doc false
  def changeset(person, attrs) do
    person
    |> cast(attrs, [:last_name, :first_name, :job_title])
    |> validate_required([:last_name, :first_name, :job_title])
  end
end

# lib/fuzzy/account.ex
defmodule Fuzzy.Account do
  use Ecto.Schema
  import Ecto.Changeset

  schema "accounts" do
    field :status, Ecto.Enum, values: [:active, :inactive]
    field :username, :string
    field :password, :string, redact: true
    # field :person_id, :id
    belongs_to :person, Fuzzy.Person

    timestamps(type: :utc_datetime)
  end

  @doc false
  def changeset(account, attrs) do
    account
    |> cast(attrs, [:status, :username, :password, :person_id])
    |> validate_required([:status, :username, :password, :person_id])
    |> unique_constraint(:username)
  end
end

# lib/fuzzy/department.ex
defmodule Fuzzy.Department do
  use Ecto.Schema
  import Ecto.Changeset

  schema "departments" do
    field :dept_name, :string

    # add a relationship to the person_department
    belongs_to :person_department, Fuzzy.PersonDepartment

    timestamps(type: :utc_datetime)
  end

  @doc false
  def changeset(department, attrs) do
    department
    |> cast(attrs, [:dept_name, :person_department_id])
    |> validate_required([:dept_name, :person_department_id])
  end
end

# lib/fuzzy/person_department.ex
defmodule Fuzzy.PersonDepartment do
  use Ecto.Schema
  import Ecto.Changeset

  schema "person_departments" do
    # field :person_id, :id
    # field :department_id, :id

    belongs_to :person, Fuzzy.Person
    belongs_to :department, Fuzzy.Department

    timestamps(type: :utc_datetime)
  end

  @doc false
  def changeset(person_departments, attrs) do
    person_departments
    |> cast(attrs, [:person_id, :department_id])
    |> validate_required([:person_id, :department_id])
  end
end
```

### Seeding

Now let's make some records for testing using the `seeds.exs` file.

```elixir
# priv/repo/seeds.exs
alias Fuzzy.{Repo, Person, Account, Department, PersonDepartment}

# priv/repo/seeds.exs

alias Fuzzy.{Repo, Person, Account, Department, Job, PersonJobs}

# Step 1: Insert Departments
departments = [
  %{dept_name: "Development"},
  %{dept_name: "Operations"},
  %{dept_name: "Research"},
  %{dept_name: "Customers"},
  %{dept_name: "Business"}
]

departments_map =
  Enum.map(departments, fn dept_data ->
    %Department{}
    |> Department.changeset(dept_data)
    |> Repo.insert!()
  end)
  |> Enum.reduce(%{}, fn dept, acc -> Map.put(acc, dept.dept_name, dept.id) end)

# Step 2: Insert Jobs
jobs_data = [
  %{job_title: "Software Engineer", department: "Development"},
  %{job_title: "Network Engineer", department: "Development"},
  %{job_title: "Quality Assurance Engineer", department: "Research"},
  %{job_title: "Designer", department: "Research"},
  %{job_title: "Data Scientist", department: "Research"},
  %{job_title: "DevOps Engineer", department: "Development"},
  %{job_title: "UX Researcher", department: "Customers"},
  %{job_title: "Backend Engineer", department: "Development"},
  %{job_title: "Frontend Developer", department: "Development"},
  %{job_title: "Tech Support Specialist", department: "Customers"},
  %{job_title: "Business Analyst", department: "Business"},
  %{job_title: "Systems Administrator", department: "Development"},
  %{job_title: "Project Manager", department: "Business"},
  %{job_title: "Cloud Architect", department: "Development"}
]

jobs_map =
  Enum.map(jobs_data, fn job_data ->
    department_id = departments_map[job_data.department]

    %Job{}
    |> Job.changeset(%{job_title: job_data.job_title, department_id: department_id})
    |> Repo.insert!()
  end)
  |> Enum.reduce(%{}, fn job, acc -> Map.put(acc, job.job_title, job.id) end)

# Step 3: Insert People and Accounts
people_data = [
  %{last_name: "Smith", first_name: "John", job_title: "Software Engineer", status: :active, username: "johnsmith", password: "password123", role_level: :senior},
  %{last_name: "Johnson", first_name: "John", job_title: "Network Engineer", status: :active, username: "johnjohnson", password: "password123", role_level: :professional},
  %{last_name: "Johanson", first_name: "Jonathan", job_title: "Quality Assurance Engineer", status: :active, username: "jonathanjohanson", password: "password123", role_level: :junior},
  %{last_name: "Smith", first_name: "Charles", job_title: "Quality Assurance Engineer", status: :active, username: "charlessmith", password: "password123", role_level: :professional},
  %{last_name: "Brown", first_name: "Charlie", job_title: "Designer", status: :inactive, username: "charliebrown", password: "password123", role_level: :junior},
  %{last_name: "Johnson", first_name: "Emma", job_title: "Data Scientist", status: :active, username: "emmajohnson", password: "password123", role_level: :senior},
  %{last_name: "Johnston", first_name: "Emilia", job_title: "Data Scientist", status: :active, username: "emiliajohnston", password: "password123", role_level: :professional},
  %{last_name: "Williams", first_name: "Liam", job_title: "DevOps Engineer", status: :inactive, username: "liamwilliams", password: "password123", role_level: :professional},
  %{last_name: "Jones", first_name: "Olivia", job_title: "UX Researcher", status: :active, username: "oliviajones", password: "password123", role_level: :lead},
  %{last_name: "Miller", first_name: "Noah", job_title: "Backend Engineer", status: :inactive, username: "noahmiller", password: "password123", role_level: :junior},
  %{last_name: "Davis", first_name: "Ava", job_title: "Frontend Developer", status: :active, username: "avadavis", password: "password123", role_level: :senior},
  %{last_name: "Garcia", first_name: "Sophia", job_title: "Quality Assurance Engineer", status: :inactive, username: "sophiagarcia", password: "password123", role_level: :junior},
  %{last_name: "Rodriguez", first_name: "Isabella", job_title: "Tech Support Specialist", status: :active, username: "isabellarodriguez", password: "password123", role_level: :professional},
  %{last_name: "Martinez", first_name: "Mason", job_title: "Business Analyst", status: :active, username: "masonmartinez", password: "password123", role_level: :lead},
  %{last_name: "Hernandez", first_name: "Lucas", job_title: "Systems Administrator", status: :inactive, username: "lucashernandez", password: "password123", role_level: :professional},
  %{last_name: "Lopez", first_name: "Amelia", job_title: "Project Manager", status: :active, username: "amelialopez", password: "password123", role_level: :lead},
  %{last_name: "Gonzalez", first_name: "James", job_title: "Network Engineer", status: :inactive, username: "jamesgonzalez", password: "password123", role_level: :junior},
  %{last_name: "Wilson", first_name: "Elijah", job_title: "Cloud Architect", status: :active, username: "elijahwilson", password: "password123", role_level: :executive}
]

Enum.each(people_data, fn person_data ->
  # Insert Person
  person =
    %Person{}
    |> Person.changeset(%{first_name: person_data.first_name, last_name: person_data.last_name})
    |> Repo.insert!()

  # Insert Account
  %Account{}
  |> Account.changeset(%{
    person_id: person.id,
    username: person_data.username,
    password: person_data.password,
    status: person_data.status
  })
  |> Repo.insert!()

  # Get job_id and insert into PersonJobs
  job_id = jobs_map[person_data.job_title]

  %PersonJobs{}
  |> PersonJobs.changeset(%{
    person_id: person.id,
    job_id: job_id,
    role_level: person_data.role_level
  })
  |> Repo.insert!()
end)
```

now let's run the seed file:
```bash
mix run priv/repo/seeds.exs
```

### Test Setup

let's see if we can access our records using `iex`:
```elixir
# enter iex (within the context of phoenix)
iex -S mix phx.server

# load the Query module and simplify with aliases
import Ecto.Query
alias Fuzzy.Account
alias Fuzzy.Person
alias Fuzzy.Repo

# Using pipeline syntax
last_pipeline_person =
  (
    Person
    |> order_by(desc: :inserted_at)
    |> Repo.all()
    |> Repo.preload(:account)
  )

# using Ecto from details:
last_from_person =
  Repo.all(from p in Person, order_by: [desc: p.inserted_at], preload: [:account])
```

If all this works we are ready to go:
```bash
git add .
git commit -m "database setup and added people"
```

## Simple Fuzzy Search

Now that we have some records, let's create a simple fuzzy search module.

### Simple Fuzzy `first_name` Search

```elixir
# enter iex (within the context of phoenix)
iex -S mix phx.server

# load the Query module and simplify with aliases
import Ecto.Query
alias Fuzzy.Repo
alias Fuzzy.Person

search_string = "John"
threshold = 0.05

from(p in Person,
  where:
    fragment(
      "word_similarity(?, ?) > ?",
      p.first_name,
      ^search_string,
      ^threshold
    ),
  select: %{
    score:
      fragment(
        "word_similarity(?, ?)",
        p.first_name,
        ^search_string
      ),
    person: p
  },
  order_by: [
    desc: fragment(
      "word_similarity(?, ?)",
      p.first_name,
      ^search_string
    )
  ]
)
|> Repo.all()

[
  %{
    score: 1.0,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 1,
      last_name: "Smith",
      first_name: "John",
      job_title: "Software Engineer",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      person_departments: #Ecto.Association.NotLoaded<association :person_departments is not loaded>,
      departments: #Ecto.Association.NotLoaded<association :departments is not loaded>,
      inserted_at: ~U[2024-10-03 19:32:30Z],
      updated_at: ~U[2024-10-03 19:32:30Z]
    }
  },
  %{
    score: 1.0,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 2,
      last_name: "Johnson",
      first_name: "John",
      job_title: "Network Engineer",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      person_departments: #Ecto.Association.NotLoaded<association :person_departments is not loaded>,
      departments: #Ecto.Association.NotLoaded<association :departments is not loaded>,
      inserted_at: ~U[2024-10-03 19:32:30Z],
      updated_at: ~U[2024-10-03 19:32:30Z]
    }
  },
  %{
    score: 0.2222222238779068,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 3,
      last_name: "Johanson",
      first_name: "Jonathan",
      job_title: "Quality Assurance Engineer",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      person_departments: #Ecto.Association.NotLoaded<association :person_departments is not loaded>,
      departments: #Ecto.Association.NotLoaded<association :departments is not loaded>,
      inserted_at: ~U[2024-10-03 19:32:30Z],
      updated_at: ~U[2024-10-03 19:32:30Z]
    }
  },
  %{
    score: 0.1666666716337204,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 17,
      last_name: "Gonzalez",
      first_name: "James",
      job_title: "Network Engineer",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      person_departments: #Ecto.Association.NotLoaded<association :person_departments is not loaded>,
      departments: #Ecto.Association.NotLoaded<association :departments is not loaded>,
      inserted_at: ~U[2024-10-03 19:32:30Z],
      updated_at: ~U[2024-10-03 19:32:30Z]
    }
  }
]
```

If you get the following error, Then you need to fix (or install) the `pg_trgm` extension.
```bash
** (Postgrex.Error) ERROR 42883 (undefined_function) function word_similarity(character varying, unknown) does not exist
```

Play with the search string a bit (and the threshold) to see how it works.
You will probably find that search strings LONGER than 3 characters work best.

A score of 1 means a perfect match (100% quality match).
The closer to 0, the worse, the match.
The closer to 1, the better, the match.

For this reason I have sorted the results by score.

### Multi-Column Fuzzy Search

```elixir
iex -S mix phx.server

import Ecto.Query
alias Fuzzy.Repo
alias Fuzzy.Person

# if ewe lower the threshold we get more results
username = "john"
threshold = 0.05


```elixir
# enter iex (within the context of phoenix)
iex -S mix phx.server

# load the Query module and simplify with aliases
import Ecto.Query
alias Fuzzy.Repo
alias Fuzzy.Person

search_string = "John"
threshold = 0.05

from(p in Person,
  where:
    fragment(
      "word_similarity(CONCAT_WS(' ', ?, ?), ?) > ?",
      p.first_name,
      p.last_name,
      ^search_string,
      ^threshold
    ),
  select: %{
    score:
      fragment(
        "word_similarity(CONCAT_WS(' ', ?, ?), ?)",
        p.first_name,
        p.last_name,
        ^search_string
      ),
    person: p
  },
  order_by: [
    desc: fragment(
      "word_similarity(CONCAT_WS(' ', ?, ?), ?)",
      p.first_name,
      p.last_name,
      ^search_string
    )
  ]
)
|> Repo.all()


[
  %{
    score: 1.0,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 1,
      last_name: "Smith",
      first_name: "John",
      job_title: "Software Engineer",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-03 19:32:30Z],
      updated_at: ~U[2024-10-03 19:32:30Z]
    }
  },
  %{
    score: 1.0,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 2,
      last_name: "Johnson",
      first_name: "John",
      job_title: "Network Engineer",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-03 19:32:30Z],
      updated_at: ~U[2024-10-03 19:32:30Z]
    }
  },
  %{
    score: 0.2222222238779068,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 3,
      last_name: "Johanson",
      first_name: "Jonathan",
      job_title: "Quality Assurance Engineer",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-03 19:32:30Z],
      updated_at: ~U[2024-10-03 19:32:30Z]
    }
  },
  %{
    score: 0.1666666716337204,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 17,
      last_name: "Gonzalez",
      first_name: "James",
      job_title: "Network Engineer",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-03 19:32:30Z],
      updated_at: ~U[2024-10-03 19:32:30Z]
    }
  }
]
```

you will see that longer strings comparisons are more likely to match (and shorter strings, the lower the match scores).  Oddly, matches at the start of a string seem to score higher too.

### Multi-Column Fuzzy Search

To improve the search we can add multiple columns to the search (including accross tables).

Let's search first and last name, job title and username.

```elixir
iex -S mix phx.server

import Ecto.Query
alias Fuzzy.Repo
alias Fuzzy.Person

# if ewe lower the threshold we get more results
search_string = "Engineer John"
threshold = 0.05

from(p in Person,
  where: p.status == :active,
  where:
    # needs a threshold for a binary result (include in results or not)
    fragment(
      "similarity(CONCAT_WS(' ', ?, ?, ?, ?), ?) > ?",
       # concat columns with a space between them
      p.username,
      p.first_name,
      p.last_name,
      p.job_title,
      # input params
      ^search_string,
      ^threshold
    ),
  select: %{
    score:
      fragment(
        "similarity(CONCAT_WS(' ', ?, ?, ?, ?), ?)",
        # concat columns with a space between them
        p.username,
        p.first_name,
        p.last_name,
        p.job_title,
        # input params
        ^search_string
      ),
    person: p
  },
  order_by:
    [
      desc: fragment(
        "similarity(CONCAT_WS(' ', ?, ?, ?, ?), ?)",
        # concat columns with a space between them
        p.username,
        p.first_name,
        p.last_name,
        p.job_title,
        # input params
        ^search_string
      )
    ]
) |> Repo.all()

# returns:
[
  %{
    person: #Fuzzy.Person<
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 2,
      status: :active,
      username: "johnjohnson",
      last_name: "Johnson",
      first_name: "John",
      job_title: "Network Engineer",
      inserted_at: ~U[2024-10-03 18:06:41Z],
      updated_at: ~U[2024-10-03 18:06:41Z],
      ...
    >,
    score: 0.5
  },
  %{
    person: #Fuzzy.Person<
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 1,
      status: :active,
      username: "johnsmith",
      last_name: "Smith",
      first_name: "John",
      job_title: "Software Engineer",
      inserted_at: ~U[2024-10-03 18:06:41Z],
      updated_at: ~U[2024-10-03 18:06:41Z],
      ...
    >,
    score: 0.46666666865348816
  },
  %{
    person: #Fuzzy.Person<
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 3,
      status: :active,
      username: "jonathanjohanson",
      last_name: "Johanson",
      first_name: "Jonathan",
      job_title: "Quality Assurance Engineer",
      inserted_at: ~U[2024-10-03 18:06:41Z],
      updated_at: ~U[2024-10-03 18:06:41Z],
      ...
    >,
    score: 0.260869562625885
  },
  ...
]
```

Now the queries are more complicated, but also more useful.
We are now getting some useful results with a threshold well over 0.3!

### HasMany Contacts

```bash
mix phx.gen.schema Department departments dept_name:string

mix phx.gen.schema PersonDepartments person_departments person_id:references:people department_id:references:departments
```



### Shared Addresses

```bash
mix phx.gen.schema Address street:string city:string state:string zip:string

mix phx.gen.schema PersonAddresses person_id:references:people address_id:references:addresses
```

## Resources & Articles

* [Postgres Fuzzy Search](https://dev.to/moritzrieger/build-a-fuzzy-search-with-postgresql-2029)
* [Optimizing Postgres Text Search with Trigrams](https://alexklibisz.com/2022/02/18/optimizing-postgres-trigram-search)
* []()


## Docs

* [fuzzystrmatch](https://www.postgresql.org/docs/current/fuzzystrmatch.html)
* [pg_trgm](https://www.postgresql.org/docs/12/pgtrgm.html)
* [Full Text Search](https://www.postgresql.org/docs/12/textsearch.html)
* []()




```elixir
defmodule GaraioRemGraphqlWeb.Resolvers.Kreditorenauftrag do
  alias GaraioRemGraphql.Repo
  alias GaraioRemGraphql.Rechnungswesen
  alias GaraioRemGraphql.Rechnungswesen.Kreditorenauftrag
  alias GaraioRemGraphqlWeb.Resolvers.Internal.Errors

  import Ecto.Query, warn: false
  import GaraioRemGraphql.Util.Pagination, only: [validate_cursor: 1]

  # Deprecated
  def all(_args, %{context: %{client_info: client_info}}) do
    {:ok, Rechnungswesen.kreditorenauftraege_fuer(client_info) |> Repo.all()}
  end

  # Deprecated
  def all(_args, _info) do
    {:error,
     %{
       message: "not authenticated",
       extensions: %{code: "not_authenticated"}
     }}
  end

  def all_paginated(args, %{context: %{client_info: client_info}}) do
    with {:ok, _decoded_cursor} <- validate_cursor(args[:cursor]),
         %{entries: entries, metadata: metadata} <-
           Repo.paginate(
             Rechnungswesen.kreditorenauftraege_fuer(client_info),
             cursor_fields: [:referenz],
             after: args[:cursor],
             limit: Application.fetch_env!(:garaio_rem_graphql, :page_size)
           ) do
      {:ok, %{page: entries, cursor: metadata.after}}
    end
  end

  def all_paginated(_args, _info) do
    {:error,
     %{
       message: "not authenticated",
       extensions: %{code: "not_authenticated"}
     }}
  end

  def find_supplier_order(
        %{supplier_reference: supplier_reference} = search_args,
        %{context: %{client_info: _client_info}}
      ) do
    with {:ok, order} <- resolve_unique_supplier_order(supplier_reference, search_args) do
      {:ok, preload_order_associations(order)}
    else
      _ ->
        Errors.no_unique_match(search_args)
    end
  end

  defp resolve_unique_supplier_order(supplier_reference, search_args) do
    base_query = core_query(supplier_reference)

    with {:error, :no_unique_match} <- external_reference_search(base_query, search_args),
         {:error, :no_unique_match} <- internal_order_reference_search(base_query, search_args),
         {:error, :no_unique_match} <- masterdata_reference_search(base_query, search_args),
         {:error, :no_unique_match} <- order_detail_description_search(base_query, search_args) do
      {:error, :no_unique_match}
    else
      {:ok, order} -> {:ok, order}
      _ -> {:error, :no_unique_match}
    end
  end

  defp external_reference_search(base_query, search_args) do
    with %{external_order_reference: external_order_reference}
         when is_binary(external_order_reference) <- search_args,
         [order] <- external_reference_query(base_query, external_order_reference) |> Repo.all() do
      {:ok, order}
    else
      _ -> {:error, :no_unique_match}
    end
  end

  defp internal_order_reference_search(base_query, search_args) do
    with %{internal_order_reference: internal_order_reference}
         when is_binary(internal_order_reference) <- search_args,
         [order] <- internal_reference_query(base_query, internal_order_reference) |> Repo.all() do
      {:ok, order}
    else
      _ -> {:error, :no_unique_match}
    end
  end

  defp masterdata_reference_search(base_query, search_args) do
    with %{masterdata_reference: masterdata_reference}
         when is_binary(masterdata_reference) <- search_args,
         [order] <- masterdata_reference_query(base_query, masterdata_reference) |> Repo.all() do
      {:ok, order}
    else
      _ -> {:error, :no_unique_match}
    end
  end

  defp order_detail_description_search(base_query, search_args) do
    with %{order_detail_description: order_detail_description}
         when is_binary(order_detail_description) <- search_args,
         [order] <- fuzzy_description_logic(base_query, order_detail_description) do
      {:ok, order}
    else
      _ -> {:error, :no_unique_match}
    end
  end

  defp fuzzy_description_logic(base_query, order_detail_description) do
    fuzzy_results =
      fuzzy_description_query(base_query, order_detail_description)
      |> Repo.all()

    case fuzzy_results do
      [one_order] ->
        # if there is only one - return only the oder (without the associated score)
        [one_order.order]

      fuzzy_results when length(fuzzy_results) > 1 ->
        [first | _rest] = fuzzy_results
        first_score = first.score

        # find orders with the top score - return only the oder (without the associated score)
        fuzzy_results
        |> Enum.filter(fn %{score: score} -> score == first_score end)
        |> Enum.map(fn %{order: order} -> order end)

      _ ->
        # no results or unexpected results
        fuzzy_results
    end
  end

  defp preload_order_associations(order) do
    order
    |> Repo.preload([
      :kreditor,
      :buchhaltung,
      :kreditorenauftrag_positionen,
      detail: [:verwaltungseinheit, :haus, :objekt]
    ])
  end

  defp core_query(supplier_reference) do
    from(order in Kreditorenauftrag,
      join: supplier in assoc(order, :kreditor),
      join: account in assoc(order, :buchhaltung),
      left_join: detail in assoc(order, :detail),
      left_join: verwaltungseinheit in assoc(detail, :verwaltungseinheit),
      left_join: haus in assoc(detail, :haus),
      left_join: objekt in assoc(detail, :objekt),
      left_join: positions in assoc(order, :kreditorenauftrag_positionen),
      limit: 2,
      where:
        order.aasm_state == "erledigt" and
          order.type == "Kreditorenauftrag" and
          is_nil(detail.kreditor_rechnung_id) and
          supplier.referenz == ^supplier_reference
    )
  end

  defp external_reference_query(base_query, external_order_reference) do
    from(order in base_query,
      where: order.externe_rechnungs_nr == ^external_order_reference
    )
  end

  defp internal_reference_query(base_query, internal_order_reference) do
    from(order in base_query,
      where: order.referenz == ^internal_order_reference
    )
  end

  defp masterdata_reference_query(base_query, masterdata_reference) do
    from(order in base_query,
      left_join: detail in assoc(order, :detail),
      left_join: verwaltungseinheit in assoc(detail, :verwaltungseinheit),
      left_join: haus in assoc(detail, :haus),
      left_join: objekt in assoc(detail, :objekt),
      where:
        haus.referenz == ^masterdata_reference or
          objekt.referenz == ^masterdata_reference or
          verwaltungseinheit.referenz == ^masterdata_reference
    )
  end

  defp fuzzy_description_query(base_query, order_detail_description, similarity_threshold \\ 0.3) do
    subquery =
      from(order in base_query,
        left_join: detail in assoc(order, :detail),
        where:
          fragment(
            "word_similarity(?, ?) > ?",
            detail.betreff,
            ^order_detail_description,
            ^similarity_threshold
          ),
        select: %{
          id: order.id,
          score:
            fragment(
              "word_similarity(?, ?)",
              detail.betreff,
              ^order_detail_description
            )
        },
        distinct: order.id
      )

    # IMPORTANT: the mainquery with subquery ensures that the order is done AFTER the distinct by id in subquery
    from(s in subquery(subquery),
      join: k in Kreditorenauftrag,
      on: s.id == k.id,
      order_by: [desc: s.score],
      select: %{
        order: k,
        score: s.score
      },
      limit: 2
    )
  end
end
```
