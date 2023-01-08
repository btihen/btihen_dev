---
# Title, summary, and page position.
linktitle: '04-Calculations'
summary: Ash Belongs To relationships
weight: 5
icon: book-reader
icon_pack: fas

# Page metadata.
title: '04-Calculations'
date: '2022-11-04T00:00:00Z'
type: book # Do not modify.
---

[Calculations](https://hexdocs.pm/ash/Ash.Calculation.html) have two aspects.  Those defined within a:

* [Resource](https://hexdocs.pm/ash/Ash.Resource.Calculation.html#expr_calc/1) - these use the [Resource DSL](https://hexdocs.pm/ash/Ash.Resource.Dsl.html#module-calculations) for forming the Calculations.
* [Query]().

**NOTE:** A few points still need research and experimentation

Before we get started, let's create some test data - with and without middle_names:

```elixir
iex -S mix phx.server

# Create a Customer and a technician
customer = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_customer, %{first_name: "Ratna", last_name: "Sönam", email: "ratna@example.com"}
    )
  |> Support.AshApi.create!()
)
technician = (
  Support.User
  |> Ash.Changeset.for_create(
      :new_employee, %{first_name: "Nyima", middle_name: "Druk",
                       last_name: "Sönam", email: "nyima@example.com",
                       department_name: "Tech Support"}
    )
  |> Support.AshApi.create!()
)
```

We will use these two accounts for testing in this section.

## Resource Calculations

Calculations allow us to extend our resources from the [Guides](https://ash-hq.org/docs/guides/ash/2.4.2/topics/expressions#portability) we can see the example of how to combine attributes and create a calculated field called `:full_name`:

```elixir
# lib/support/resources/user.ex
  # ...
  calculations do
    calculate :full_name, :string, expr(first_name <> " " <> last_name)
  end
  # ...
```

**Example 1**

Shortly we will see how this works but for now it might be interesting to see how Ash represents this:

```elixir
recompile()

Support.User
  |> Ash.Query.load([:full_name])
  # #Ash.Query<
  #   resource: Support.User,
  #   calculations: %{
  #     full_name: #Ash.Resource.Calculation.Expression<[expr: first_name <> " " <> last_name]> - %{context: %{}}
  #   }
```

I am still exploring, but I believe if you follow the [Expressions](https://hexdocs.pm/ash/expressions.html) Docs you will be able to build the calculations you need.

**Note:** Filters use expressions too (so probably useful to spend time exploring these).

**Example 2**

We can also see from the Docs that there's a [built in calculation](https://hexdocs.pm/ash/Ash.Resource.Calculation.Builtins.html) allowing us to accomplish the same thing with:

```elixir
# lib/support/resources/user.ex
  # ...
  calculations do
    # we can call a built-in calculation `concat`
    calculate :full_name, :string, concat([:first_name, :last_name], " ")
  end
  # ...
```

In this case the internal representation looks like:

```elixir
recompile()

Support.User
  |> Ash.Query.load([:full_name])
  #Ash.Query<
  #   resource: Support.User,
  #   calculations: %{
  #     full_name: #Ash.Resource.Calculation.Concat<[keys: [:first_name, :last_name], separator: " "]> - %{context: %{}}
  #   }
```

### Retrieving Calculations

As we have seen above to 'view' the calculation can **retrieve** the calculated resource with the 'load' function with the format:

```elixir
recompile()

require Ash.Query

# we can retrieve the calculated resource with the 'load' function :
Support.User
|> Ash.Query.load([:full_name])
|> Support.AshApi.read!()

# you should get something like:
# [
#   #Support.User<
#     full_name: "Nyima Sönam",
#     ...
#     calculations: %{},
#     ...
#   >,
# ...
# ]
```

The calculated attribute `full_name` should now be visible. We will cover the `calculations: %{}` field shortly.

### Calculations used within Queries

Calculated results once loaded can be used within the query - for example we can sort with it:

```elixir
Support.User
|> Ash.Query.load([:full_name])
|> Ash.Query.sort(full_name: :asc)
|> Support.AshApi.read!()
```

or within a filter Query:

```elixir
Support.User
|> Ash.Query.load([:full_name])
|> Ash.Query.filter(full_name == "Marpo Sönam")
|> Support.AshApi.read!()
```


The be sure to filter it with:

```elixir
require Ash.Query

Support.User
|> Ash.Query.filter(all_assigned_tickets > 0) # prevent divide by zero
|> Ash.Query.filter(assigned_open_percent > 0.25)
|> Ash.Query.load(:assigned_open_percent)
|> Support.AshApi.read!()
```

## Complexer Resource Calculations

WORK IN PROGRESS

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

**Second Try**

We need to carefully study [Ash Expressions](https://ash-hq.org/docs/guides/ash/latest/topics/expressions) - which isn't as flexible as elixir - as it is designed to work on multiple Data Layers.

```elixir
# lib/support/resources/user.ex
  # ...
  calculations do
    calculate :with_middle_name, :string,
              expr(if exists(middle_name),
                      do: first_name <> " " <> middle_name <> " " <> last_name,
                      else: first_name <> " " <> last_name
                  )
    end
  end
  # ...
```

when try to call these we get errors such as:

```elixir
recompile()

require Ash.Query

Support.User
|> Ash.Query.new()
|> Ash.Query.load(:with_middle_name)
|> Support.AshApi.read!()

# Ash.Query.load([:with_middle_name])
# warning: the following fields are unknown when raising Ash.Error.Query.NoSuchFunction: [resource: Support.User]. Please make sure to only give known fields when raising or redefine Ash.Error.Query.NoSuchFunction.exception/1 to discard unknown fields. Future Elixir versions will raise on unknown fields given to raise/2
```

## Custom Calculation Extensions

A good approach to complex calculations is to write a custom calculation!

For more complex calculations we can define our own calculations for [example](https://hexdocs.pm/ash/Ash.Resource.Dsl.html#module-calculations):

To build a reusable Resource [Custom Resource Calculation](https://hexdocs.pm/ash/calculations.html#module-calculations) we can do something like the following:
```elixir
# lib/support/calculations/formal_name.ex
defmodule Support.Calculation.UserFormalName do
  use Ash.Calculation
  # records is a list of all the records being processed
  # opts is a list of keys - which we could use to dynamically build the calculation
  # context is a map of any additional information needed to build the calculation
  @impl true
  def calculate(records, _opts, _context) do
    # each record is calculated individually
    # (implement the `expression` callback function to move to the DataLayer_
    Enum.map(records, fn record ->
      last_join_first(record) # the actual calculation
    end)
  end

  defp last_join_first(record) do
    record.last_name  <> ", " <> ([record.first_name, record.middle_name]
                                    |> Enum.reject(&is_nil/1)
                                    |> Enum.reject(fn string -> string == "" end)
                                    |> Enum.join(" ")
                                  )
  end

  # You can implement this callback to make this calculation possible in the data layer
  # *and* in elixir. Ash expressions are already executable in Elixir or in the data layer,
  # but this gives you fine grain control over how it is done
  # @impl true
  # def expression(opts, context) do
  # end
end

# we will add it to the User Resource with:
# lib/support/resources/user.ex
defmodule Support.User do
  # ...
  calculations do
    calculate :full_name, :string, concat([:first_name, :last_name], " ") # built-in calculator
    calculate :formal_name, :string, {Support.Calculation.UserFormalName} # our custom calculator
  end
  # ...
end
```

**TEST**

```elixir
recompile()

# call the Resource Calculation
Support.User
|> Ash.Query.new()
|> Ash.Query.load([:formal_name])
|> Support.AshApi.read!()

# now you should see something like:
#  #Support.User<
#    formal_name: "Customer, Friendly5",
#    full_name: "Friendly5 Customer",
```

## Query Calculation (submitting in-line expressions)

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

## Query Calculation (accessing Custom Calculator)

If we want to use the custom calculation within a query (and say give it another name) we can do the following:

```elixir
Support.User
|> Ash.Query.new()
|> Ash.Query.calculate(:legal_name, {Support.Calculation.UserFormalName, []}, :string)
|> Support.AshApi.read!()

# now you should see something like:
# [
#   #Support.User<
#     ...
#     id: "d949be4e-dd0f-484c-8bf2-e81e1b462b57",
#     email: "friendly5@customer.com",
#     first_name: "Friendly5",
#     ...
#     aggregates: %{},
#     calculations: %{legal_name: "Customer, Friendly5"},
#     ...
#   >
# ]
```

### Generic Custom Calculator

Now let's explore a calculator that will work with more than one Resource (or can be used in various ways)

Let's write a Generic Calculator - that does the same as before, but this time we can choose which attributes will be used and we will include a check in `init` that we are using the options correctly.

```elixir
# lib/support/calculations/first_separator_rest.ex
defmodule Support.Calculation.FirstSeparatorRest do
  use Ash.Calculation

  # Optional callback - verifies options has a list of keys (as atoms)
  @impl true
  def init(options) do
    key_list = options[:keys]
    if is_list(key_list) && (Enum.count(key_list) > 1) && Enum.all?(key_list, &is_atom/1) do
      {:ok, options}
    else
      {:error, "Expected a `keys` option with at least two values to build a formal name"}
    end
  end

  @impl true
  def calculate(records, options, context) do
    # each record must be calculated individually
    # DataLayer API calculations are done with the `expression` callback function
    Enum.map(records, fn record ->
      first_separator_rest(record, options, context) # the actual calculation
    end)
  end

  defp first_separator_rest(record, [keys: key_list] = _options, context) do
    record_values = Enum.map(key_list, fn key ->
                      # fetch the value from the record based on the key and ensure it is a string
                      to_string(Map.get(record, key))
                    end)
    [first_value | rest_values] = record_values
    rest_values_list = if is_list(rest_values), do: rest_values, else: [rest_values]

    # in case no separator was given use ", "
    separator = context[:separator] || ", "
    first_value  <> separator <> (rest_values_list
                                    |> Enum.reject(&is_nil/1) # remove nil values
                                    |> Enum.reject(fn string -> string == "" end) # remove empty strings
                                    |> Enum.join(" ") # join the remaining values into a string with a space separator
                                  )
  end
  # TODO: not sure how this works yet
  # You can implement this callback to make this calculation possible in the data layer
  # *and* in elixir. Ash expressions are already executable in Elixir or in the data layer,
  # but this gives you fine grain control over how it is done
  # @impl true
  # def expression(opts, context) do
  # end
end
```

now we can update our User Resource with:

```elixir

  calculations do
    # calculate :full_name, :string, expr(first_name <> " " <> last_name)
    calculate :full_name, :string, concat([:first_name, :last_name], " ")
    # ...
    # custom calculations can be called with options
    calculate :last_comma_first_name, :string,
              {Support.Calculation.FirstSeparatorRest, [keys: [:last_name, :first_name]]}
    calculate :last_separator_all_names, :string,
              {Support.Calculation.FirstSeparatorRest,
               [keys: [:last_name, :first_name, :middle_name]]} do
      # not sure how to use this in a resource calculation
      argument :separator, :string, constraints: [allow_empty?: true, trim?: false]
    end
  end
```

To send the `separator` parameter - our load would look like:

```elixir
Support.User
|> Ash.Query.new()
|> Ash.Query.load(:last_separator_all_names, %{separator: " ~ "})
|> Support.AshApi.read!()
```

To use the custom calculation as a query call it looks like (notice we can now order the keys in whatever order we want and submit our own separator):

```elixir
Support.User
|> Ash.Query.new()
|> Ash.Query.calculate(:with_middle_name,
                       {Support.Calculation.FirstSeparatorRest,
                         keys: [:first_name, :middle_name, :last_name]},
                       :string, %{separator: ": "})
|> Support.AshApi.read!()
```

**TEST**

So lets create some

Now that we have some users (with and without middle_names):
```elixir
Support.User
|> Ash.Query.new()
|> Ash.Query.load([:last_comma_first_name, :last_separator_names])
|> Ash.Query.load(:last_separator_all_names, %{separator: " ~ "})
|> Ash.Query.calculate(:dash_name,
                       {Support.Calculation.FirstSeparatorRest, keys: [:first_name, :last_name]},
                      :string, %{separator: " - "})
|> Ash.Query.calculate(:with_middle_name,
                       {Support.Calculation.FirstSeparatorRest, keys: [:last_name, :first_name, :middle_name]},
                      :string, %{separator: " -- "})
|> Support.AshApi.read!()

# now we should get something like:
# [
#   #Support.User<
#     last_separator_names: "Sönam, Nyima Druk",
#     last_comma_first_name: "Sönam, Nyima",
#     ...
#     id: "424ef4aa-0bbc-4427-ad59-07a4032b5c8e",
#     email: "nyima@example.com",
#     first_name: "Nyima",
#     middle_name: "Druk",
#     last_name: "Sönam",
#     ...
#     aggregates: %{},
#     calculations: %{
#       dash_name: "Nyima - Sönam",
#       with_middle_name: "Sönam -- Nyima Druk"
#     },
#     ...
#   >
# ]
```
