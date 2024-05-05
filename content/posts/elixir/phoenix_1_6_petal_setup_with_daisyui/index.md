---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.6 Easy PETAL Stack with DaisyUI"
subtitle: "Simple PETAL Setup"
summary: "Adding a Tailwind CSS Framework to Phoenix"
authors: ["btihen"]
tags: ["Elixir", "Phoenix 1.6.x", "TailwindCSS", "TailwindCSS 3.x", "AlpineJS", "AlpineJS 3.x", "LiveView", "PETAL", "PETAL Stack", "ASDF", "CSS", "DaisyUI"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2022-04-09T01:01:53+02:00
lastmod: 2022-04-11T01:01:53+02:00
featured: false
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
I have been enjoying the tools associated with Elixir and exploring the frontend. LiveView helps make that more intuitive and when that isn't enough, AlpineJS is a lightweight JS tool with a similar syntax as Vue.

## Install asdf - and required software

- https://www.cogini.com/blog/using-asdf-with-elixir-and-phoenix/
- https://carlyleec.medium.com/create-an-elixir-phoenix-app-with-asdf-e918649b4d58

On a Mac I used Homebrew:
```bash
brew install asdf
echo -e '\n. $(brew --prefix asdf)/asdf.sh' >> ~/.bash_profile
echo -e '\n. $(brew --prefix asdf)/etc/bash_completion.d/asdf.bash' >> ~/.bash_profile
source ~/.bash_profile  # (or open a new terminal)
```

Now you can install asdf software packages:
```bash
asdf plugin-add erlang
asdf plugin-add elixir
asdf plugin-add Postgres
```

Now you need to install the desired versions (usually the newest) - currently:
```bash
asdf list all erlang
asdf install erlang 24.3.3
asdf global erlang 24.3.3


# note the elixir version otp must match the erlang version!
asdf list all elixir
asdf install elixir 1.13.4-otp-24
asdf global elixir 1.13.4-otp-24

# asdf install elixir 1.11.4-otp-24
# if you mismatch elixir with erlang you will get errors like:
# {"init terminating in do_boot",{undef,[{elixir,start_cli,[],[]},{init,start_em,1,[]},{init,do_boot,3,[]}]}}

asdf list all Postgres
asdf install Postgres 14.2 # Slow! (maybe just use homebrew's version)
```

## Get the newest Elixir tools

```
mix local.rebar --force
mix local.hex --force
```

## Get the newest Phoenix Hex Package

Once you have established you have the requirements - the download the newest version of Phoenix (go to: https://hexdocs.pm/phoenix/installation.html#phoenix to see the newest version) - at the time of this writing its 1.5.8 - be sure its installed using:
```bash
mix archive.install hex phx_new 1.6.6 --force
```

## create a project with asdf settings

First we will create the folder / project location
```bash
mkdir petal
```

Now we will tell it which software to use :
```bash
touch petal/.tool-versions
cat <<EOF >>petal/.tool-versions
erlang 24.3.3
elixir 1.13.3-otp-24
EOF
```

## Create a new Phoenix Project

https://carlyleec.medium.com/create-an-elixir-phoenix-app-with-asdf-e918649b4d58

Now you can simply do:
```bash
mix phx.new petal --live
cd petal
mix ecto.create
```

assuming all is good lets configure git:
```bash
git init
git add .
git commit -m "initial Phoneix install with LiveView"
```

## credo

add credo to mix.exs (optional but often useful)
```elixir
{:credo, "~> 1.6"}
```

configure credo
```bash
mix deps.get
mix credo gen.config
```

## Tailwind & DaisyUI

Add to mix.exs
```elixir
{:phx_gen_tailwind, "~> 0.1", only: :dev, runtime: false}
```

configure TailwindCSS
```
mix deps.get
mix phx.gen.tailwind
```

## Test TailwindCSS

add to the top of: `lib/slacker_gen_tail_web/templates/page/index.html.heex`
```css
<div class="text-green-500 text-5xl text-center">Large Centered Green TailwindCSS</div>
```

start phoenix `mix -S phx.server`

## Add DaisyUI

https://daisyui.com/docs/install/
https://elixirforum.com/t/how-to-get-daisyui-and-phoenix-to-work/46612/8 (also explains what to do when using/deploying to Fly.io)!

first we need to initialize npm (otherwise DaisyUI won't load / integrate with esbuild)
```bash
cd assets
npm init
# just using the defaults seems to work
npm install daisyui
```

Now configure Tailwind (`assets/tailwind.config.js`) to use daisyui

```javascript
// assets/tailwind.config.js
module.exports = {
  mode: 'jit',
  purge: [
    './js/**/*.js',
    '../lib/*_web/**/*.*ex'
  ],
  theme: {
  },
  variants: {
    extend: {},
  },
  plugins: [require("daisyui")],
}
```

## Test DaisyUI

now lets add a DaisyUI Button to the default page `lib/fare_web/templates/page/index.html.heex`:
```html
<button class="btn btn-primary">Button</button>
```

ideally when we start phoenix:
```
iex -S mix phx.server
```

we should now see in the log:

```
warn - You have enabled the JIT engine which is currently in preview.
warn - Preview features are not covered by semver, may introduce breaking changes, and can change at any time.

Rebuilding...

🌼 daisyUI components 2.13.6  https://github.com/saadeghi/daisyui
  ✔︎ Including:  base, components, themes[29], utilities
```

now when we look at the landing page we should see a rounded blue button!


## install & test Alpine JS

see the article on [phoenix installer (easiest)](https://btihen.me/post_elixir_phoenix/phoenix_1_6_petal_w_gen_n_custom_fonts/) or [by hand](https://btihen.me/post_elixir_phoenix/phoenix_1_6_petal_setup_with_asdf/) to install aplinejs

## See Adding fonts for more options

see the article on adding a [custom font](https://btihen.me/post_elixir_phoenix/phoenix_1_6_petal_w_gen_n_custom_fonts/) for font information


## Tailwind & DaisyUI with Fly.io

https://github.com/phoenixframework/tailwind
https://elixirforum.com/t/how-to-get-daisyui-and-phoenix-to-work/46612/8
https://elixirforum.com/t/tailwind-not-working-in-production-no-styles-just-plain-html/45192

## Resources (1.6.x)

- https://www.youtube.com/watch?v=vZBHkvTAb2U
- https://carlyleec.medium.com/create-an-elixir-phoenix-app-with-asdf-e918649b4d58
- https://sergiotapia.com/phoenix-160-liveview-esbuild-tailwind-jit-alpinejs-a-brief-tutorial
- https://pragmaticstudio.com/tutorials/adding-tailwind-css-to-phoenix
- https://thinkingelixir.com/petal-stack-in-elixir/
- https://larainfo.com/blogs/build-simple-count-app-using-apline-js-with-tailwind-css
- https://fullstackphoenix.com/tutorials/combine-phoenix-liveview-with-alpine-js
- https://www.cogini.com/blog/using-asdf-with-elixir-and-phoenix/
- https://carlyleec.medium.com/create-an-elixir-phoenix-app-with-asdf-e918649b4d58
- https://tailwindcss.com/
- https://github.com/alpinejs/alpine
- https://github.com/tailwindlabs/tailwindcss
- https://experimentingwithcode.com/custom-fonts-with-phoenix-and-tailwind/
- https://fullstackphoenix.com/tutorials/add-tailwind-html-generators-in-phoenix
- https://elixirforum.com/t/how-do-i-use-a-custom-font-with-phoenix-1-6-and-esbuild/43791/16


## Older Resources
- https://www.youtube.com/watch?v=o4Prej0wIZA
- http://blog.pthompson.org/alpine-js-and-liveview
- https://pragmaticstudio.com/tutorials/adding-tailwind-css-to-phoenix
- https://fullstackphoenix.com/tutorials/get-started-with-tailwind-in-phoenix
- https://fullstackphoenix.com/tutorials/combine-phoenix-liveview-with-alpine-js
- https://medium.com/mindvalley-technology/how-to-add-tailwindcss-to-your-phoenix-project-e2250ad31ace
- https://thinkingelixir.com/podcast-episodes/021-tailwind-css-alpine-js-and-liveview-with-patrick-thompson/
