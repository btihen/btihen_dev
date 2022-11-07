---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.6 Umbrella to include a GenServer"
subtitle: "Running a GenServer inside Phoenix"
summary: ""
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "GenServer", "Umbrella"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2022-01-10T01:01:53+02:00
lastmod: 2022-01-10T01:01:53+02:00
featured: false
draft: true

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

## Create the Project

```
mix phx.new feenix_live --live --umbrella
cd feenix_live_umbrella
```

## Resources

- https://www.elixircryptobot.com/
- Hands-on Elixir & OTP: Cryptocurrency trading bot: YouTube Channel, https://www.youtube.com/playlist?list=PLxsE19GnjC5Nv1CbeKOiS5YqGqw35aZFJ
