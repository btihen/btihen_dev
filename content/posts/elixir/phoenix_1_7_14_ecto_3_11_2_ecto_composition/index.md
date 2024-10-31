---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Phoenix / Ecto - Ecto Query Composition"
subtitle: "Building upon Existing Queries"
# Summary for listings and search engines
summary: "Building upon an existing query based on various conditions"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "Ecto", "Fuzzy", "Postgres"]
categories: ["Code", "Elixir Language", "Phoenix Framework", "PostgreSQL"]
date: 2024-07-07T01:01:53+02:00
lastmod: 2024-07-07T01:01:53+02:00
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
