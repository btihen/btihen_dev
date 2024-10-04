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
mix phx.gen.schema Person people last_name:string first_name:string job_title:string department:string

mix phx.gen.schema Account accounts status:enum:active:inactive username:string:unique password:string:redact person_id:references:people
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
    field :department, :string

    # add a relationship to the account
    has_one :account, Fuzzy.Account

    timestamps(type: :utc_datetime)
  end

  @doc false
  def changeset(person, attrs) do
    person
    |> cast(attrs, [:last_name, :first_name, :job_title, :department])
    |> validate_required([:last_name, :first_name, :job_title, :department])
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
```

### Seeding

Now let's make some records for testing using the `seeds.exs` file.

```elixir
# priv/repo/seeds.exs
alias Fuzzy.{Repo, Person, Account}

# Step 1: People Data
people_data = [
  %{
    last_name: "Smith",
    first_name: "John",
    title: "Software Engineer",
    status: :active,
    username: "johnsmith",
    password: "password123",
    department: "Product"
  },
  %{
    last_name: "Johnson",
    first_name: "John",
    title: "Network Engineer",
    status: :active,
    username: "johnjohnson",
    password: "password123",
    department: "Operations"
  },
  %{
    last_name: "Johanson",
    first_name: "Jonathan",
    title: "QA Engineer",
    status: :active,
    username: "jonathanjohanson",
    password: "password123",
    department: "Quality"
  },
  %{
    last_name: "Smith",
    first_name: "Charles",
    title: "Tester",
    status: :active,
    username: "charlessmith",
    password: "password123",
    department: "Quality"
  },
  %{
    last_name: "Brown",
    first_name: "Charlie",
    title: "Designer",
    status: :inactive,
    username: "charliebrown",
    password: "password123",
    department: "Product"
  },
  %{
    last_name: "Johnson",
    first_name: "Emma",
    title: "Data Scientist",
    status: :active,
    username: "emmajohnson",
    password: "password123",
    department: "Research"
  },
  %{
    last_name: "Johnston",
    first_name: "Emilia",
    title: "Data Scientist",
    status: :active,
    username: "emiliajohnston",
    password: "password123",
    department: "Research"
  },
  %{
    last_name: "Williams",
    first_name: "Liam",
    title: "DevOps Engineer",
    status: :inactive,
    username: "liamwilliams",
    password: "password123",
    department: "Operations"
  },
  %{
    last_name: "Jones",
    first_name: "Olivia",
    title: "UX Researcher",
    status: :active,
    username: "oliviajones",
    password: "password123",
    department: "Research"
  },
  %{
    last_name: "Miller",
    first_name: "Noah",
    title: "Software Engineer",
    status: :inactive,
    username: "noahmiller",
    password: "password123",
    department: "Product"
  },
  %{
    last_name: "Davis",
    first_name: "Ava",
    title: "Software Engineer",
    status: :active,
    username: "avadavis",
    password: "password123",
    department: "Product"
  },
  %{
    last_name: "Garcia",
    first_name: "Sophia",
    title: "QA Engineer",
    status: :inactive,
    username: "sophiagarcia",
    password: "password123",
    department: "Quality"
  },
  %{
    last_name: "Rodriguez",
    first_name: "Isabella",
    title: "Tech Support",
    status: :active,
    username: "isabellarodriguez",
    password: "password123",
    department: "Customers"
  },
  %{
    last_name: "Martinez",
    first_name: "Mason",
    title: "Business Analyst",
    status: :active,
    username: "masonmartinez",
    password: "password123",
    department: "Research"
  },
  %{
    last_name: "Hernandez",
    first_name: "Lucas",
    title: "Systems Administrator",
    status: :inactive,
    username: "lucashernandez",
    password: "password123",
    department: "Operations"
  },
  %{
    last_name: "Lopez",
    first_name: "Amelia",
    title: "Product Manager",
    status: :active,
    username: "amelialopez",
    password: "password123",
    department: "Business"
  },
  %{
    last_name: "Gonzalez",
    first_name: "James",
    title: "Network Engineer",
    status: :inactive,
    username: "jamesgonzalez",
    password: "password123",
    department: "Operations"
  },
  %{
    last_name: "Wilson",
    first_name: "Elijah",
    title: "Cloud Architect",
    status: :active,
    username: "elijahwilson",
    password: "password123",
    department: "Operations"
  }
]

# Step 2: Insert People and Assign Accounts & Departments
Enum.each(people_data, fn person_data ->
  # Insert the Person record
  person =
    %Person{}
    |> Person.changeset(%{
      first_name: person_data.first_name,
      last_name: person_data.last_name,
      title: person_data.title,
      department: person_data.department})
    |> Repo.insert!()

  # Insert the corresponding Account record
  %Account{}
  |> Account.changeset(%{
    person_id: person.id,
    username: person_data.username,
    password: person_data.password,
    status: person_data.status
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
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 1,
      last_name: "Smith",
      first_name: "John",
      title: "Software Engineer",
      department: "Product",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    },
    score: 1.0
  },
  %{
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 2,
      last_name: "Johnson",
      first_name: "John",
      title: "Network Engineer",
      department: "Operations",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    },
    score: 1.0
  },
  %{
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 3,
      last_name: "Johanson",
      first_name: "Jonathan",
      title: "QA Engineer",
      department: "Quality",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    },
    score: 0.2222222238779068
  },
  %{
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 17,
      last_name: "Gonzalez",
      first_name: "James",
      title: "Network Engineer",
      department: "Operations",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    },
    score: 0.1666666716337204
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

You may also notice we have two records with a 100% match, but they have different lastnames, roles, etc.  In order to refine our search - let's explore multi-column fuzzy searches.

### Multi-Column Fuzzy Search

Let's try matching on first and last name using:

```elixir
iex -S mix phx.server

import Ecto.Query
alias Fuzzy.Repo
alias Fuzzy.Person

# if ewe lower the threshold we get more results
search_string = "john smith"
threshold = 0.4

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
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 1,
      last_name: "Smith",
      first_name: "John",
      title: "Software Engineer",
      department: "Product",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    },
    score: 1.0
  },
  %{
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 2,
      last_name: "Johnson",
      first_name: "John",
      title: "Network Engineer",
      department: "Operations",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    },
    score: 0.5555555820465088
  },
  %{
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 4,
      last_name: "Smith",
      first_name: "Charles",
      title: "Tester",
      department: "Quality",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    },
    score: 0.4285714328289032
  }
]
```

you will see that longer strings comparisons are more likely to match (and shorter strings, the lower the match scores).

you can / should of course include all the person fields.

### Multi-Table Fuzzy Search

To improve the search we can add multiple columns to the search (including accross tables).

Let's search first and last name, job title and username.

```elixir
iex -S mix phx.server

import Ecto.Query
alias Fuzzy.Repo
alias Fuzzy.Person

# if ewe lower the threshold we get more results
search_string = "john an engineer in product"
threshold = 0.3

from(p in Person,
  join: a in assoc(p, :account), # with associations defined in the schema
  # join: a in Account, on: a.person_id == p.id, # without associations in schema
  where:
    # needs a threshold for a binary result (include in results or not)
    fragment(
      "similarity(CONCAT_WS(' ', ?, ?, ?, ?, ?), ?) > ?",
       # concat columns with a space between them
        a.username,
      p.first_name,
      p.last_name,
      p.title,
      p.department,
      # input params
      ^search_string,
      ^threshold
    ),
  select: %{
    score:
      fragment(
        "similarity(CONCAT_WS(' ', ?, ?, ?, ?, ?), ?)",
        # concat columns with a space between them
        a.username,
        p.first_name,
        p.last_name,
        p.title,
        p.department,
        # input params
        ^search_string
      ),
    person: p
  },
  order_by:
    [
      desc: fragment(
        "similarity(CONCAT_WS(' ', ?, ?, ?, ?, ?), ?)",
        # concat columns with a space between them
        a.username,
        p.first_name,
        p.last_name,
        p.title,
        p.department,
        # input params
        ^search_string
      )
    ]
) |> Repo.all()

# returns:
[
  %{
    score: 0.5,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 1,
      last_name: "Smith",
      first_name: "John",
      title: "Software Engineer",
      department: "Product",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    }
  },
  %{
    score: 0.375,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 11,
      last_name: "Davis",
      first_name: "Ava",
      title: "Software Engineer",
      department: "Product",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    }
  },
  %{
    score: 0.3400000035762787,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 10,
      last_name: "Miller",
      first_name: "Noah",
      title: "Software Engineer",
      department: "Product",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    }
  }
]
```

Now the queries are more complicated, but also more useful.

### Fuzzy Search without Threshold

using limit.

```elixir
iex -S mix phx.server

import Ecto.Query
alias Fuzzy.Repo
alias Fuzzy.Person

# if ewe lower the threshold we get more results
search_string = "john an engineer in product"

from(p in Person,
  join: a in assoc(p, :account), # with associations defined in the schema
  # join: a in Account, on: a.person_id == p.id, # without associations in schema
  select: %{
    score:
      fragment(
        "similarity(CONCAT_WS(' ', ?, ?, ?, ?, ?), ?)",
        # concat columns with a space between them
        a.username,
        p.first_name,
        p.last_name,
        p.title,
        p.department,
        # input params
        ^search_string
      ),
    person: p
  },
  order_by:
    [
      desc: fragment(
        "similarity(CONCAT_WS(' ', ?, ?, ?, ?, ?), ?)",
        # concat columns with a space between them
        a.username,
        p.first_name,
        p.last_name,
        p.title,
        p.department,
        # input params
        ^search_string
      )
    ],
  limit: 2
) |> Repo.all()
[
  %{
    score: 0.5,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 1,
      last_name: "Smith",
      first_name: "John",
      title: "Software Engineer",
      department: "Product",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    }
  },
  %{
    score: 0.375,
    person: %Fuzzy.Person{
      __meta__: #Ecto.Schema.Metadata<:loaded, "people">,
      id: 11,
      last_name: "Davis",
      first_name: "Ava",
      title: "Software Engineer",
      department: "Product",
      account: #Ecto.Association.NotLoaded<association :account is not loaded>,
      inserted_at: ~U[2024-10-04 09:31:56Z],
      updated_at: ~U[2024-10-04 09:31:56Z]
    }
  }
]



# or just in sorting
from(p in Person,
  join: a in assoc(p, :account), # with associations defined in the schema
  select:  p,
  preload: [:account],
  order_by:
    [
      desc: fragment(
        "similarity(CONCAT_WS(' ', ?, ?, ?, ?, ?), ?)",
        # concat columns with a space between them
        a.username,
        p.first_name,
        p.last_name,
        p.title,
        p.department,
        # input params
        ^search_string
      )
    ],
  limit: 2
) |> Repo.all()
```

Or if you only every want the top response:
```elixir

from(p in Person,
  join: a in assoc(p, :account), # with associations defined in the schema
  select:  p,
  preload: [:account],
  order_by:
    [
      desc: fragment(
        "similarity(CONCAT_WS(' ', ?, ?, ?, ?, ?), ?)",
        # concat columns with a space between them
        a.username,
        p.first_name,
        p.last_name,
        p.title,
        p.department,
        # input params
        ^search_string
      )
    ],
  limit: 1
) |> Repo.one()
```

I personally prefer at least returning the score to assess the match quality even when I set the threshold.

## Conclusion

This is a straight forward way to find records from incomplete information and when used in coordination with other strategies this can be a powerful way to help users locate data.

PS - generally speaking 0.3 is considered a generous, yet healthy cutoff.

## Resources & Articles

* [Postgres Fuzzy Search](https://dev.to/moritzrieger/build-a-fuzzy-search-with-postgresql-2029)
* [Optimizing Postgres Text Search with Trigrams](https://alexklibisz.com/2022/02/18/optimizing-postgres-trigram-search)


## Docs

* [fuzzystrmatch](https://www.postgresql.org/docs/current/fuzzystrmatch.html)
* [pg_trgm](https://www.postgresql.org/docs/12/pgtrgm.html)
* [Full Text Search](https://www.postgresql.org/docs/12/textsearch.html)
