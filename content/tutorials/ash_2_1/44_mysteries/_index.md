---
# Title, summary, and page position.
linktitle: '44-Mysteries'
summary: Notes for Aspects of Ash not yet understood (with working code)
weight: 45
icon: book-reader
icon_pack: fas

# Page metadata.
title: '44-Mysteries'
date: '2022-11-04T00:00:00Z'
type: book # Do not modify.
---

## Calculations

### Resource Calculation with complex expressions

If the calculation is difficult to write as a Resource Calculation, ie:

```elixir
# lib/support/resources/user.ex
  # ...
  calculations do
    calculate :full_name, :string, expr([first_name, last_name] |> Enum.join(" "))
    calculate :formal_name, :string, expr(
      last_name  <> ", " <> (
                              [first_name, middle_name]
                              |> Enum.map(fn string -> is_binary(string) end)
                              |> Enum.join(" ")
                            )
    )
  end
  # ...
```

when try to call these we get errors such as:

```elixir
Support.User
|> Ash.Query.new()
|> Ash.Query.load([:full_name, :formal_name])
|> Support.AshApi.read!()

#   warning: variable "first_name" does not exist and is being expanded to "first_name()", please use parentheses to remove the ambiguity or change the variable name
#   lib/support/resources/user.ex:86: Support.User
#
# == Compilation error in file lib/support/resources/user.ex ==
# ** (CompileError) lib/support/resources/user.ex:86: undefined function first_name/0 (there is no such import)
```

1) Medium Complex Resource calculations don't seem to recognize variables - I don't think I understand the DSL well:

```elixir
# lib/support/resources/user.ex
calculations do
  # this works:
  calculate :full_name, :string, expr(first_name <> " " <> last_name)
  # this doesn't work
  calculate :joined_names, :string, expr([first_name, middle_name] |> Enum.join(" "))
  # variable "first_name" does not exist and is being expanded to "first_name()",
end
```

**For #1**,  the biggest thing here: expr accepts a very specific subset of Elixir. Its not just an elixir expression.
<https://ash-hq.org/docs/guides/ash/latest/topics/expressions>
If you're working with the postgres data layer, for example, you have fragment() available, just like in ecto, to do whatever expression you need. However, the goal is to continue building the available expressions in Ash expressions.



### Query Calculation (with in-line expressions)

```elixir
Support.User
|> Ash.Query.new()
|> Ash.Query.calculate(:both_names, :string, expr(first_name <> " " <> last_name))
|> Ash.Query.calculate(:names_both, :string, concat([:first_name, :last_name], " "))
|> Support.AshApi.read!()
```

**For #3**, do those not work? Just remember that on the fly calculations are placed in record.calculations

REMEMBER: expr accepts a very specific subset of Elixir. Its not just an elixir expression.
<https://ash-hq.org/docs/guides/ash/latest/topics/expressions>
If you're working with the postgres data layer, for example, you have fragment() available, just like in ecto, to do whatever expression you need. However, the goal is to continue building the available expressions in Ash expressions.

### Calculations with Conditionals

If you build a calculation with a potential problem like:

```elixir
# lib/support/resources/user.ex
  calculations do
    calculate :percent_open_assignments, :float,
              expr(active_assigned_tickets / all_assigned_tickets)
  end
```

**Test** - should break with:

```elixir
recompile()

Support.User
|> Ash.Query.load(:percent_open)
|> Support.AshApi.read!()
```

**Solution** to divide by Zero and return a default value: `0` (solution from Zach)

```elixir
# lib/support/resources/user.ex
  calculations do
    calculate :percent_open_assignments, :float,
              expr(if all_assigned_tickets == 0, do:
                    0,
                    else: active_assigned_tickets / all_assigned_tickets)
  end
```

**Test** - protected now!

```elixir
recompile()

Support.User
|> Ash.Query.load(:all_reported_tickets)
|> Ash.Query.load(:percent_open_assignments)
|> Support.AshApi.read!()
```



3) How does one create on-the-fly Query Calculations?  I've tried many variations on:


```elixir
# lib/support/resources/user.ex
  calculations do
    calculate :percent_open_assignments, :float,
              expr(active_assigned_tickets / all_assigned_tickets)
  end
```
I found I need to query with:
```elixir
require Ash.Query

Support.User
|> Ash.Query.load(:assigned_open_percent)
|> Support.AshApi.read!()

Support.User
|> Ash.Query.filter(all_assigned_tickets > 0) # prevent divide by zero
|> Ash.Query.load(:assigned_open_percent)
|> Support.AshApi.read!()
```
to prevent divide by zero

**For #4**, you have if available which could solve for that

```elixir
calculate :percent_open_assignments, :float,
                  expr(if all_assigned_tickets == 0, do: 0, else: active_assigned_tickets / all_assigned_tickets)
```

```elixir
require Ash.Query
Support.User
|> Ash.Query.load(:percent_open_assignments)
|> Support.AshApi.read!()


require Ash.Query
Support.User
|> Ash.Query.filter(all_assigned_tickets > 0) # prevent divide by zero
|> Ash.Query.load(:percent_open_assignments)
|> Support.AshApi.read!()

```

could also have used if `do ... end` style above
an example using fragment (if you can find a way without fragment, that is ideal, because then we can compute the value directly in Elixir)

`expr(active_assigned_tickets / fragment("NULLIF(?, 0)", all_assigned_tickets))`

5) using the `expression` to move the calculation into the data-layer.


**ANSWERS**

Zach Daniel — 13.11.2022 at 10:23 PM



## Query Calculations

**BROKEN - DO NOT USE THIS SECTION!**
This section needs work!

We can do [Query Calculations](https://www.ash-hq.org/docs/module/ash/2.4.2/ash-query-calculation#module-docs) too:

on the fly Query calculations - don't work, I must be overlooking something

```elixir
Support.User
|> Ash.Query.new()
|> Ash.Query.calculate(:both_names, :string, expr(first_name <> " " <> last_name))
|> Ash.Query.calculate(:names_both, :string, concat([:first_name, :last_name], " "))
|> Ash.Query.load([:full_name])
|> Support.AshApi.read!()
```

```elixir

Support.User
|> Ash.Query.new()
|> Ash.Query.calculate(:username, :string, name <> "-")
|> Ash.Query.load([:full_name])
|> Support.AshApi.read!()

# or mixed
Support.User
|> Ash.Query.new()
|> Ash.Query.calculate(:both_names, concat([:first_name, :last_name], " "))
|> Ash.Query.calculate(:names_both, :string, concat([:first_name, :last_name], " "))
|> Ash.Query.load([:full_name])
|> Support.AshApi.read!()
```

Pre-built calculations:

```elixir
# lib/helpdesk/support/resources/user.ex
  calculations do
    # calculate :full_name, :string, expr(first_name <> " " <> last_name)
    calculate :username, :string, expr(name <> "-" <> id)
    calculate :assigned_open_percent, :float, expr(active_assigned_tickets / all_assigned_tickets)
  end
```

USAGE:

```elixir
iex -S mix

require Ash.Query

Support.User
|> Ash.Query.filter(all_assigned_tickets > 0) # prevent divide by zero
|> Ash.Query.filter(assigned_open_percent > 0.25)
|> Ash.Query.sort(:assigned_open_percent)
|> Ash.Query.load(:assigned_open_percent)
|> Support.AshApi.read!()

# try out the username calculation
Support.User
|> Ash.Query.load(:username)
|> Support.AshApi.read!()

# calculations can also be loaded in a separate query afterwards
users = Support.AshApi.read!(Support.User)
Support.AshApi.load!(users, :username)

# we can load multiple calculations and aggregates
users = Support.AshApi.read!(Support.User)
Support.AshApi.load!(users, [:username, :closed_assigned_tickets])
```
