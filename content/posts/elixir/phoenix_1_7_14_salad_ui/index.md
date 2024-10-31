---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Phoenix / Salad UI - Basics"
subtitle: "Exploring Salad UI in a Phoenix App"
# Summary for listings and search engines
summary: "Installation of SaladUI and creating a navbar in a Phoenix App"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "Salad UI"]
categories: ["Code", "Elixir Language", "Phoenix Framework", "Frontend"]
date: 2024-10-02T01:01:53+02:00
lastmod: 2024-10-02T01:01:53+02:00
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

We recently build an app and quickly realized the usefulness of a UI Framework.  We chose [Salad UI](https://salad.run/) because it is based on [Tailwind](https://tailwindcss.com/) the now default CSS system in Phoenix.  We also like the idea that it creates components that are consistent with Phoenix's architecture.

## Getting Started

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

# create a new phoenix project
mix phx.new salad_ui

cd salad_ui
mix ecto.create

# run phoenix
iex -S mix phx.server
```

## Install Salad UI

[Salad UI Demo & Docs](https://salad-storybook.fly.dev/welcome?tab=variations&variation_id=menu), however, I find the [SaladUI Hex Docs](https://hexdocs.pm/salad_ui/readme.html) or its [Repo](https://github.com/bluzky/salad_ui) better for installation instructions.

1. Install `salad_ui` hex package:

  * Add `salad_ui` in `mix.exs`:
  ```elixir
  def deps do
    [
      ...,
      {:salad_ui, "~> 0.4.2"}
    ]
  end
  ```
  * run: `mix deps.get`


2. Now go to: https://ui.shadcn.com/themes

  * click the `customize` button
  * after choosing a theme click the `copy code` button.
  * paste this code into the file: `assets/css/app.css`- I chose to use:
```scss
// assets/css/app.css
@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 224 71.4% 4.1%;
    --card: 0 0% 100%;
    --card-foreground: 224 71.4% 4.1%;
    --popover: 0 0% 100%;
    --popover-foreground: 224 71.4% 4.1%;
    --primary: 262.1 83.3% 57.8%;
    --primary-foreground: 210 20% 98%;
    --secondary: 220 14.3% 95.9%;
    --secondary-foreground: 220.9 39.3% 11%;
    --muted: 220 14.3% 95.9%;
    --muted-foreground: 220 8.9% 46.1%;
    --accent: 220 14.3% 95.9%;
    --accent-foreground: 220.9 39.3% 11%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 20% 98%;
    --border: 220 13% 91%;
    --input: 220 13% 91%;
    --ring: 262.1 83.3% 57.8%;
    --radius: 0.5rem;
    --chart-1: 12 76% 61%;
    --chart-2: 173 58% 39%;
    --chart-3: 197 37% 24%;
    --chart-4: 43 74% 66%;
    --chart-5: 27 87% 67%;
  }

  .dark {
    --background: 224 71.4% 4.1%;
    --foreground: 210 20% 98%;
    --card: 224 71.4% 4.1%;
    --card-foreground: 210 20% 98%;
    --popover: 224 71.4% 4.1%;
    --popover-foreground: 210 20% 98%;
    --primary: 263.4 70% 50.4%;
    --primary-foreground: 210 20% 98%;
    --secondary: 215 27.9% 16.9%;
    --secondary-foreground: 210 20% 98%;
    --muted: 215 27.9% 16.9%;
    --muted-foreground: 217.9 10.6% 64.9%;
    --accent: 215 27.9% 16.9%;
    --accent-foreground: 210 20% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 20% 98%;
    --border: 215 27.9% 16.9%;
    --input: 215 27.9% 16.9%;
    --ring: 263.4 70% 50.4%;
    --chart-1: 220 70% 50%;
    --chart-2: 160 60% 45%;
    --chart-3: 30 80% 55%;
    --chart-4: 280 65% 60%;
    --chart-5: 340 75% 55%;
  }
}
```


3. Create the file: `assets/tailwind.colors.json` and paste the following into:
```json
{
  "accent": {
    "DEFAULT": "hsl(var(--accent))",
    "foreground": "hsl(var(--accent-foreground))"
  },
  "background": "hsl(var(--background))",
  "border": "hsl(var(--border))",
  "card": {
    "DEFAULT": "hsl(var(--card))",
    "foreground": "hsl(var(--card-foreground))"
  },
  "destructive": {
    "DEFAULT": "hsl(var(--destructive))",
    "foreground": "hsl(var(--destructive-foreground))"
  },
  "foreground": "hsl(var(--foreground))",
  "input": "hsl(var(--input))",
  "muted": {
    "DEFAULT": "hsl(var(--muted))",
    "foreground": "hsl(var(--muted-foreground))"
  },
  "popover": {
    "DEFAULT": "hsl(var(--popover))",
    "foreground": "hsl(var(--popover-foreground))"
  },
  "primary": {
    "DEFAULT": "hsl(var(--primary))",
    "foreground": "hsl(var(--primary-foreground))"
  },
  "ring": "hsl(var(--ring))",
  "secondary": {
    "DEFAULT": "hsl(var(--secondary))",
    "foreground": "hsl(var(--secondary-foreground))"
  }
}
```
Ideally copy this from the docs in case of updates.


4. Now in `assets/tailwind.config.js` add the following into the correct parts of this file:
```js
module.exports = {
  content: [
      "../deps/salad_ui/lib/**/*.ex",
    ],
  theme: {
    extend: {
      colors: require("./tailwind.colors.json"),
    },
  },
  plugins: [
    require("@tailwindcss/typography"),
    require("tailwindcss-animate"),
    ...
  ]
}
```
be sure tailwind-animate is installed:
```bash
cd assets
npm i -D tailwindcss-animate
# or yarn
yarn add -D tailwindcss-animate
```

5. Add the colors file to `config/config.exs`
```elixir
config :tails, colors_file: Path.join(File.cwd!(), "assets/tailwind.colors.json")
```

6. add the following to `assets/js/app.js`
```js
window.addEventListener("phx:js-exec", ({ detail }) => {
  document.querySelectorAll(detail.to).forEach((el) => {
    liveSocket.execJS(el, el.getAttribute(detail.attr));
  });
});
```
you can close an opening sheet like this.
```elixir
  @impl true
  def handle_event("update", params, socket) do
    # your logic
    {:noreply, push_event(socket, "js-exec", %{to: "#my-sheet", attr: "phx-hide-sheet"})}
  end
  ```
  We will demonstrate this in the code samples.

7. SKIP!
To make dark and light mode work correctly, add following to `assets/css/app.css`
```css
body {
  @apply bg-background text-foreground;
}
```

8. If borders don't work properly add `@apply border-border !important;` to `assets/css/app.css`
```css
@layer base {
  @apply border-border !important;
  :root {
    ...
  }
}
```

# Add login

```bash
mix phx.gen.auth Accounts User users --web Auth
mix deps.get
mix ecto.migrate
```

now you should see `register` and `login` links in the top right.

# SaladUI Navbar
