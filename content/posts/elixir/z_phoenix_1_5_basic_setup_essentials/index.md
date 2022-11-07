---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.5 LiveView Setup Essentials"
subtitle: "Phoenix, Elixir, Testing"
summary: "Create a modern Phoenix SPA with tremendous flexibility"
authors: ["btihen"]
tags: ["Elixir", "TailwindCSS", "AlpineJS", "LiveView", "PETAL Stack"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2021-05-06T01:01:53+02:00
lastmod: 2021-05-06T01:01:53+02:00
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
I have been enjoying the tools associated with Elixir and exploring the frontend. LiveView helps make that more intuitive and when that isn't enough, AlpineJS is a lightweight JS tool with a similar syntax as Vue.

## Install asdf - and required software

- https://thinkingelixir.com/install-elixir-using-asdf/
- https://www.cogini.com/blog/using-asdf-with-elixir-and-phoenix/
- https://carlyleec.medium.com/create-an-elixir-phoenix-app-with-asdf-e918649b4d58

On a Mac I used Homebrew:
```
brew install asdf
echo -e '\n. $(brew --prefix asdf)/asdf.sh' >> ~/.bash_profile
echo -e '\n. $(brew --prefix asdf)/etc/bash_completion.d/asdf.bash' >> ~/.bash_profile
source ~/.bash_profile  # (or open a new terminal)
```

Now you can install asdf software packages:
```
asdf plugin-add erlang
asdf plugin-add elixir
asdf plugin-add nodejs
asdf plugin-add Postgres
```

Now you need to install the desired versions (usually the newest) - currently:
```
asdf list all erlang
asdf install erlang 23.3.1

# note the elixir version otp must match the erlang version!
asdf list all elixir
asdf install elixir 1.11.4-otp-23

# asdf install elixir 1.11.4-otp-24
# if you mismatch elixir with erlang you will get errors like:
# {"init terminating in do_boot",{undef,[{elixir,start_cli,[],[]},{init,start_em,1,[]},{init,do_boot,3,[]}]}}

asdf list all nodejs
asdf install nodejs lts-fermium

asdf list all Postgres
asdf install Postgres 13.2
```

## Get the newest Phoenix Hex Package

Once you have established you have the requrements - the download the newest version of Phoenix (go to: https://hexdocs.pm/phoenix/installation.html#phoenix to see the newest version) - at the time of this writing its 1.5.8 - be sure its installed using:
```
mix archive.install hex phx_new 1.5.8
```

## create / config a project

First we will creat the folder / project location
```
mkdir fenix
```

Now we will tell it which software to use:
```
touch fenix/.tool-versions
cat <<EOF >>fenix/.tool-versions
erlang 23.3.1
elixir 1.11.4-otp-23
Postgres 13.2
nodejs lts-Fermium
EOF
```

## Create a new Phoenix Project

https://carlyleec.medium.com/create-an-elixir-phoenix-app-with-asdf-e918649b4d58

Now you can simply do:
```
mix phx.new fenix --live
cd fenix
mix ecto.create
mix phx.server
```

assuming all is good lets configure git:
```
git init
git add .
git commit -m "initial Phoneix install with LiveView"
```



## Setup Email sending

- https://hexdocs.pm/bamboo/readme.html
- https://elixircasts.io/sending-email-with-bamboo-part-1
- https://elixircasts.io/sending-email-with-bamboo-part-2
- https://www.kabisa.nl/tech/real-world-phoenix-lets-send-some-emails/


## Testing

- https://github.com/dashbitco/mox
- https://hex.pm/packages/phoenix_integration
-


## Add Test helpers

  wallaby / hound - browser testing
  {:faker, "~> 0.16", only: :test},
  {:ex_machina, "~> 2.7.0", only: :test},
  {:phoenix_integration, "~> 0.8.2"}
  mox - mock testing external apis
  moch
  FakerElixir

  CSS matching:
  https://github.com/philss/floki#supported-selectors


## Auth Users

Setup Authentication (with POW) - see:
Setup Authentication (with auth.gen) - see:
Setup Resource Authorization - see:


## Resources

- https://www.youtube.com/watch?v=o4Prej0wIZA
- http://blog.pthompson.org/alpine-js-and-liveview
- https://pragmaticstudio.com/tutorials/adding-tailwind-css-to-phoenix
- https://fullstackphoenix.com/tutorials/get-started-with-tailwind-in-phoenix
- https://fullstackphoenix.com/tutorials/combine-phoenix-liveview-with-alpine-js
- https://medium.com/mindvalley-technology/how-to-add-tailwindcss-to-your-phoenix-project-e2250ad31ace
- https://thinkingelixir.com/podcast-episodes/021-tailwind-css-alpine-js-and-liveview-with-patrick-thompson/
