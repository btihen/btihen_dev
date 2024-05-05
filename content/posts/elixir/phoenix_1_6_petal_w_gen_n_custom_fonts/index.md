---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.6 Easy PETAL Stack Setup (w/ Custom Font)"
subtitle: "Simple PETAL Setup (Phoenix, Elixir, TailwindCSS, AlpineJS, LiveView)"
summary: "Create a modern webapp with tremendous flexibility"
authors: ["btihen"]
tags: ["Elixir", "Phoenix 1.6.x", "TailwindCSS", "TailwindCSS 3.x", "AlpineJS", "AlpineJS 3.x", "LiveView", "PETAL", "PETAL Stack", "Custom Font"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2022-03-04T01:01:53+02:00
lastmod: 2022-08-12T01:01:53+02:00
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
asdf install erlang 24.2.2
asdf global erlang 24.2.2


# note the elixir version otp must match the erlang version!
asdf list all elixir
asdf install elixir 1.13.3-otp-24
asdf global elixir 1.13.3-otp-24

# asdf install elixir 1.11.4-otp-24
# if you mismatch elixir with erlang you will get errors like:
# {"init terminating in do_boot",{undef,[{elixir,start_cli,[],[]},{init,start_em,1,[]},{init,do_boot,3,[]}]}}

asdf list all Postgres
asdf install Postgres 14.2
```

## Get the newest Elixir tools

```
mix local.rebar --force
mix local.hex --force
```

## Get the newest Phoenix Hex Package

Once you have established you have the requrements - the download the newest version of Phoenix (go to: https://hexdocs.pm/phoenix/installation.html#phoenix to see the newest version) - at the time of this writing its 1.5.8 - be sure its installed using:
```bash
mix archive.install hex phx_new 1.6.6 --force
```

## create a project with asdf settings

First we will creat the folder / project location
```bash
mkdir petal
```

Now we will tell it which software to use:
```bash
touch petal/.tool-versions
cat <<EOF >>petal/.tool-versions
erlang 24.2.2
elixir 1.13.3-otp-24
Postgres 14.2
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
git commit -m "initial Phoenix install with LiveView"
```

## credo

add credo to mix.exs
```
{:credo, "~> 1.6"},
{:phx_gen_tailwind, "~> 0.1.3", only: :dev, runtime: false}
```

configure Credo & TailwindCSS
```
mix deps.get
mix credo gen.config
mix phx.gen.tailwind
```

## Test TailwindCSS

add to the top of: `lib/slacker_gen_tail_web/templates/page/index.html.heex`
```css
<div class="text-green-500 text-5xl text-center">Large Centered Green TailwindCSS</div>
```

start phoenix `mix -S phx.server`



## install AlpineJS
https://www.youtube.com/watch?v=vZBHkvTAb2U
https://sergiotapia.com/phoenix-160-liveview-esbuild-tailwind-jit-alpinejs-a-brief-tutorial


```bash
cd assets
npm install alpinejs
cd ..
```

Now change `app.js` is to require our new setup:

```js
# assets/js/app.js
// .. after the app.scss import add:
import Alpine from "alpinejs";
```

still in `assets/js/app.js` find:
```js
let csrfToken = document.querySelector("meta[name='csrf-token']").getAttribute("content")
let liveSocket = new LiveSocket("/live", Socket, {params: {_csrf_token: csrfToken}})
```

and change to:
```js
// after all the imports
window.Alpine = Alpine;
Alpine.start();
let hooks = {};
let csrfToken = document.querySelector("meta[name='csrf-token']").getAttribute("content")
let liveSocket = new LiveSocket("/live", Socket, {
  params: { _csrf_token: csrfToken },
  hooks: hooks,
  dom: {
    onBeforeElUpdated(from, to) {
      if (from._x_dataStack) {
        window.Alpine.clone(from, to);
      }
    },
  },
});
```

now change the loading of `assets/app.js` in the root file to load `assets/js/app.js` to load AlpineJS into Phoenix you do that with in `lib/fonts_web/templates/layout/root.html.heex` by changing:

```
    <!-- change this from app.js to js/app.js -->
    <script defer phx-track-static type="text/javascript" src={Routes.static_path(@conn, "/assets/app.js")}></script>
```
to:
```
    <!-- change this from app.js to js/app.js -->
    <script defer phx-track-static type="text/javascript" src={Routes.static_path(@conn, "/assets/js/app.js")}></script>
```

so now `lib/fonts_web/templates/layout/root.html.heex` should look like:
```html
<!-- lib/fonts_web/templates/layout/root.html.heex -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8"/>
    <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <%= csrf_meta_tag() %>
    <%= live_title_tag assigns[:page_title] || "SlackerPetal", suffix: " · Phoenix Framework" %>
    <link phx-track-static rel="stylesheet" href={Routes.static_path(@conn, "/assets/app.css")}/>
    <!-- change the `javascript` static path from `/assets/app.js` to `/assets/app.js` to load AlpineJS -->
    <%# <script defer phx-track-static type="text/javascript" src={Routes.static_path(@conn, "/assets/app.js")}></script> %>
    <script defer phx-track-static type="text/javascript" src={Routes.static_path(@conn, "/assets/js/app.js")}></script>
  </head>
  <body>
    <header>
      <section class="container">
        <nav>
          <ul>
            <li><a href="https://hexdocs.pm/phoenix/overview.html">Get Started</a></li>
            <%= if function_exported?(Routes, :live_dashboard_path, 2) do %>
              <li><%= link "LiveDashboard", to: Routes.live_dashboard_path(@conn, :home) %></li>
            <% end %>
          </ul>
        </nav>
        <a href="https://phoenixframework.org/" class="phx-logo">
          <img src={Routes.static_path(@conn, "/images/phoenix.png")} alt="Phoenix Framework Logo"/>
        </a>
      </section>
    </header>
    <%= @inner_content %>
  </body>
</html>
```


## Test AlpineJS

TEST by adding to the end of: `lib/petal_web/live/page_live.html.leex`
```elixir
<section>
  <h2>Alpine JS Installed</h2>
  <div x-data="{name:''}">
    <label for="name">Name:</label>
    <input id="name" type="text" x-model="name" />
    <p><br><b><em>Output:</em></b> <span x-text="name"></span></p>
  </div>
</section>
```

test with:
`mix phx.server`

when typing the name should appear below!


## Add CSS Components to TailwindCSS

we will create a file for our default CSS styles the: `assets/css/default.css` file:
```bash
# assuming you are in the root directory
touch assets/css/default.css
```
Put your default site CSS here.


Let's also create some custom CSS components:
```css
# assuming you are still in the assets directory on the cli
mkdir assets/css/components
touch assets/css/components/buttons.css
cat <<EOF >assets/css/components/buttons.css
@layer components {
  .btn-redish {
    @apply bg-red-300 hover:bg-red-600 text-blue-800 font-bold py-2 px-4 rounded;
  }
  .btn-greenish {
    @apply bg-green-300 hover:bg-green-600 text-blue-800 font-bold py-2 px-4 rounded;
  }
}
EOF
```

Let's add our components (& Default CSS to our config)
```css
/* Import tailwind - with postcss-import installed */
@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";

/* custom styles - put after base imports! */
@import "./default.css";
/* import custom components */
@import "./components/buttons.css";
/* default phoenix styles - replaced bby default.css file */
/* @import "./phoenix.css"; */
```

add a test html from tailwind to the end of: `lib/petal_web/live/page_live.html.leex`
```elixir
<section class="grid grid-cols-1 gap-4">
	<!-- tailwind text -->
  <div>
    <h2 class="text-red-500 text-5xl font-bold text-center">Tailwind CSS with Alpine JS Dropdown</h2>
  </div>
  <!-- alpinejs dropdown test -->
	<div x-data="{ open: false }" class="relative text-left">
  	<button
  					@click="open = !open"
  					@keydown.escape.window="open = false"
  					@click.away="open = false"
  					class="flex items-center h-8 pl-3 pr-2 border border-black focus:outline-none">
  			<span class="text-sm leading-none">
  					Options
  			</span>
  			<svg class="w-4 h-4 mt-px ml-2" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
  					<path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
  			</svg>
  	</button>
  	<div
  					x-cloak
  					x-show="open"
  					x-transition:enter="transition ease-out duration-100"
  					x-transition:enter-start="transform opacity-0 scale-95"
  					x-transition:enter-end="transform opacity-100 scale-100"
  					x-transition:leave="transition ease-in duration-75"
  					x-transition:leave-start="transform opacity-100 scale-100"
  					x-transition:leave-end="transform opacity-0 scale-95"
  					class="absolute flex flex-col w-40 mt-1 border border-black shadow-xs">
  			<a class="flex items-center h-8 px-3 text-sm hover:bg-gray-200" href="#">Settings</a>
  			<a class="flex items-center h-8 px-3 text-sm hover:bg-gray-200" href="#">Support</a>
  			<a class="flex items-center h-8 px-3 text-sm hover:bg-gray-200" href="#">Sign Out</a>
  	</div>
	</div>

  <!-- alpinejs counter test -->
  <div>
    <p class="mt-5 font-bold text-center">Counter with Component Buttons</p>
  </div>
  <!--
    If you want a box around the counter use:
    <div class="flex items-center justify-center h-screen bg-gray-200">
  -->
  <div class="mt-10 flex justify-center" x-data="{ count: 0 }">
    <button class="btn-redish" x-on:click="count--">Decrement</button>
    <code>count: </code><code x-text="count"></code>
    <button class="btn-greenish" x-on:click="count++">Increment</button>
  </div>
</section>
```

Start phoenix with:
```
iex -S mix phx.server
```
Now we should have a dropdown & our colored component buttons on our counter.


## Add a custom Font

https://experimentingwithcode.com/custom-fonts-with-phoenix-and-tailwind/

Find a font at: https://fonts.google.com/ (or any other source of fonts)

I will use `handlee` as it is very distinctive - to make the setup easier I will download `handlee` from:
https://google-webfonts-helper.herokuapp.com/fonts/handlee?subsets=latin

I will put the fonts in the folder: `assets/fonts/handlee`
```bash
mkdir -p assets/fonts/handlee
touch assets/fonts/handlee/handless.css
cat <<EOF>>assets/fonts/handlee/handless.css
/* handlee-regular - latin - copied from: https://google-webfonts-helper.herokuapp.com/fonts/handlee?subsets=latin */
/* be sure the font-path is local to where the fonts are */
@font-face {
  font-family: 'Handlee';
  font-style: normal;
  font-weight: 400;
  src: local(''),
       url('handlee-v12-latin-regular.woff2') format('woff2'),
       url('handlee-v12-latin-regular.woff') format('woff');
}
EOF
```

Now lets setup `esbuild` to manage the CSS and fonts - the default in `config/config.exs` is:
```elixir
# config/config.exs
config :esbuild,
  version: "0.14.0",
  default: [
    args: ~w(js/app.js --bundle --target=es2017 --outdir=../priv/static/assets --external:/fonts/* --external:/images/*),
    cd: Path.expand("../assets", __DIR__),
    env: %{"NODE_PATH" => Path.expand("../deps", __DIR__)}
  ]
```

We will need to add our FontCSS path `fonts/handlee/handlee.css` to the build System along with the font types to load:`--loader:.woff2=file --loader:.woff=file`, so now it should look like:
```elixir
config :esbuild,
  version: "0.14.0",
  default: [
    args: ~w(js/app.js fonts/handlee/handlee.css --bundle --loader:.woff2=file --loader:.woff=file --target=es2017 --outdir=../priv/static/assets --external:/fonts/* --external:/images/*),
    cd: Path.expand("../assets", __DIR__),
    env: %{"NODE_PATH" => Path.expand("../deps", __DIR__)}
  ]
```

Now you NEED to restart phoenix once a config file changes!

Assuming everything is working you will see:
* the fonts files in: `priv/static/assets/`
* the fonts css in: `priv/static/assets/fonts/handlee/handlee.css`

this is critical - if this isn't working you MUST fix it!


### Integrate the font into CSS (& TailwindCSS)

Now you will need to tell TailwindCSS about the font so you can use: `font-handprint` as a CSS class.

Your initial `assets/tailwind.config.js` file will look like:
```javascript
module.exports = {
  mode: 'jit',
  purge: [
    './js/**/*.js',
    '../lib/*_web/**/*.*ex'
  ],
  theme: {
    extend: {},
  },
  variants: {
    extend: {},
  },
  plugins: [],
}
```

Your define you font in the `theme` section with:
```javascript
module.exports = {
  mode: 'jit',
  purge: [
    './js/**/*.js',
    '../lib/*_web/**/*.*ex'
  ],
  theme: {
    extend: {
      fontFamily: {
        handprint: ['Handlee']
      },
    },
  },
  variants: {
    extend: {},
  },
  plugins: [],
}
```

Working in Switzerland it is convenient to use very multi-lingual fonts such as `Noto` and redefine the sans and serif fonts - in fact, my goto fonts and setup are:
```javascript
module.exports = {
  mode: 'jit',
  purge: [
    './js/**/*.js',
    '../lib/*_web/**/*.*ex'
  ],
  theme: {
    extend: {
      fontFamily: {
        comic: ['ComicNeue'],
        hand: ['Handless'],
        sans: ['NotoSans', 'sans-serif'],
        serif: ['NotoSerif', 'serif']
      },
    },
  },
  variants: {
    extend: {},
  },
  plugins: [],
}
```


### Integrate the Fonts Into Phoenix

in the root.html.heex file add the static route `/assets/fonts/Hanlee/hanlee.css` before the `/assets/app.css`.

This is done with the following html route:
```elixir
    <!-- add font before app.css -->
    <link phx-track-static rel="stylesheet" href={Routes.static_path(@conn, "/assets/fonts/Hanlee/hanlee.css")}/>
    <link phx-track-static rel="stylesheet" href={Routes.static_path(@conn, "/assets/app.css")}/>
```


So now `lib/fonts_web/templates/layout/root.html.heex` would look like:

```html
<!-- lib/fonts_web/templates/layout/root.html.heex -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8"/>
    <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <%= csrf_meta_tag() %>
    <%= live_title_tag assigns[:page_title] || "SlackerPetal", suffix: " · Phoenix Framework" %>
    <!-- add font here BEFORE `/assets/app.css` -->
    <link phx-track-static rel="stylesheet" href={Routes.static_path(@conn, "/assets/fonts/Hanlee/hanlee.css")}/>
    <link phx-track-static rel="stylesheet" href={Routes.static_path(@conn, "/assets/app.css")}/>
    <!-- change the `javascript` static path from `/assets/app.js` to `/assets/app.js` to load AlpineJS -->
    <%# <script defer phx-track-static type="text/javascript" src={Routes.static_path(@conn, "/assets/app.js")}></script> %>
    <script defer phx-track-static type="text/javascript" src={Routes.static_path(@conn, "/assets/js/app.js")}></script>
  </head>
  <body>
    <header>
      <section class="container">
        <nav>
          <ul>
            <li><a href="https://hexdocs.pm/phoenix/overview.html">Get Started</a></li>
            <%= if function_exported?(Routes, :live_dashboard_path, 2) do %>
              <li><%= link "LiveDashboard", to: Routes.live_dashboard_path(@conn, :home) %></li>
            <% end %>
          </ul>
        </nav>
        <a href="https://phoenixframework.org/" class="phx-logo">
          <img src={Routes.static_path(@conn, "/images/phoenix.png")} alt="Phoenix Framework Logo"/>
        </a>
      </section>
    </header>
    <%= @inner_content %>
  </body>
</html>
```


### Test you new font

Now add to the file `lib/slacker_gen_tail_web/templates/page/index.html.heex` to the following html:
```html
<p class="text-red-500 text-5xl text-center font-handprint">Hand Printed</p>
```

Ideally you now see your new font displayed within the "Hand Printed" text

NOTE:
if you get an error like:
`(Phoenix.Router.NoRouteError) no route found for GET /assets/fonts/Handlee/handlee.css`

Check your esbundle config and that `priv/static/assets/fonts/handlee/handlee.css` is being copied (Static routes always point to: `priv/static/`)!



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
