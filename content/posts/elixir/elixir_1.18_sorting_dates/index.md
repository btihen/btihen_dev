---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Elixir - Enum and Dates"
subtitle: "Properly sorting & min / max dates without passing a type created unexpected results"
summary: "Enum.sort(Date), Enum.min(Date), Enum.max(Date) - needs `Date` to work as expected"
authors: ["btihen"]
tags: ["Elixir", "Enum", "Dates", "Sort", "Min", "Max"]
categories: ["Code", "Elixir Language", "Enumerators", "Dates"]
date: 2025-10-19T01:01:53+02:00
lastmod: 2025-10-19T01:01:53+02:00
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

## Context

Recently we had a little bug in our code we were looking for a date-range (minimum begin-date and maximum end-date).  We realized `Enum.sort, Enum.min, Enum.max, Enum.sort_by, Enum.min_by, Enum.max_by` are not reliable with dates.

for example the following causes problems - as you can see different dates will sort in unexpected ways and differently depending upon the dates:
```elixir
dates_reverse = [~D[2023-03-28],~D[2024-05-18],~D[2025-07-08]]
dates_unclear = [~D[2017-02-28],~D[2019-02-01],~D[2022-05-01]]

Enum.sort(dates_reverse)
[~D[2025-07-08], ~D[2024-05-18], ~D[2023-03-28]]
Enum.sort(dates_unclear)
[~D[2019-02-01], ~D[2022-05-01], ~D[2017-02-28]]

Enum.min(dates_reverse)
~D[2025-07-08]
Enum.min(dates_unclear)
~D[2019-02-01]

Enum.max(dates_reverse)
~D[2023-03-28]
Enum.max(dates_unclear)
~D[2017-02-28]
```

However, as of Elixir 1.10 you can pass a type `Date` parameter, thus the following works reliably as expected:
```elixir
dates_reverse = [~D[2023-03-28],~D[2024-05-18],~D[2025-07-08]]
dates_unclear = [~D[2017-02-28],~D[2019-02-01],~D[2022-05-01]]

dates_reverse |> Enum.sort(Date)
[~D[2023-03-28], ~D[2024-05-18], ~D[2025-07-08]]
dates_unclear |> Enum.sort(Date)
[~D[2017-02-28], ~D[2019-02-01], ~D[2022-05-01]]

dates_reverse |> Enum.min(Date)
~D[2023-03-28]
dates_unclear |> Enum.min(Date)
~D[2017-02-28]

dates_reverse |> Enum.max(Date)
~D[2025-07-08]
dates_unclear |> Enum.max(Date)
~D[2022-05-01]
```

## Resources

* https://medium.com/@steven.cole.elliott/sorting-dates-in-elixir-59592a0c33a5
* https://significa.co/blog/elixir-sort-lists-by-date
