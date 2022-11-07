---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.6 PETAL Stack Setup by Hand"
subtitle: "Simple PETAL Setup"
summary: "The underlying tooling for a PETAL Stack"
authors: ["btihen"]
tags: ["Elixir", "Phoenix 1.6.x", "TailwindCSS", "TailwindCSS 3.x", "AlpineJS", "AlpineJS 3.x", "LiveView", "PETAL", "PETAL Stack", "ASDF"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2022-02-27T01:01:53+02:00
lastmod: 2022-08-12T01:01:53+02:00
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

Once you have established you have the requirements - the download the newest version of Phoenix (go to: https://hexdocs.pm/phoenix/installation.html#phoenix to see the newest version) - at the time of this writing its 1.5.8 - be sure its installed using:
```bash
mix archive.install hex phx_new 1.6.6 --force
```

## create a project with asdf settings

First we will create the folder / project location
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
git commit -m "initial Phoneix install with LiveView"
```

## credo

add credo to mix.exs
```
{:credo, "~> 1.6"}
```

configure credo
```
mix deps.get
mix credo gen.config
```


## install & test Alpine JS
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

let's snapshot:
```bash
git add .
git commit -m "phoenix with alpine js"
```


## Integrating Tailwind into phoenix
https://www.youtube.com/watch?v=vZBHkvTAb2U
https://pragmaticstudio.com/tutorials/adding-tailwind-css-to-phoenix
https://sergiotapia.com/phoenix-160-liveview-esbuild-tailwind-jit-alpinejs-a-brief-tutorial

lets install tailwind:
```bash
cd assets
npm install autoprefixer postcss postcss-import postcss-cli tailwindcss --save-dev
cd ..
```

comment out `import "../css/app.css"` to avoid pipeline compilation conflicts
```js
// assets/js/app.js
// We import the CSS which is extracted to its own file by esbuild.
// Remove this line if you add a your own CSS build pipeline (e.g postcss).
// import "../css/app.css"

```

now create & configure postcss.config.js
```bash
cat <<EOF>assets/postcss.config.js
module.exports = {
  plugins: {
    "postcss-import": {},
    tailwindcss: {},
    autoprefixer: {},
  }
}
EOF
```

now configure tailwind with (to purge *.js, *.eex, *.leex, *.heex files)
```js
cat <<EOF >assets/tailwind.config.js
module.exports = {
  mode: "jit",
  purge: [
    "./js/**/*.js",
    "../lib/*_web/**/*.*ex"
  ],
  theme: {
    extend: {},
  },
  variants: {
    extend: {},
  },
  plugins: [],
};

EOF
```

in `config/dev.exs` update the watcher with:
```elixir
# config/dev.exs
watchers: [
  esbuild: {Esbuild, :install_and_run, [:default, ~w(--sourcemap=inline --watch)]},
  npx: [
    "tailwindcss",
    "--input=css/app.css",
    "--output=../priv/static/assets/app.css",
    "--postcss",
    "--watch",
    cd: Path.expand("../assets", __DIR__)
  ]
]
```


to BUILD ASSETS in Production add in `deps/phoenix/package.json`:
```
"scripts": {
  "deploy": "NODE_ENV=production tailwindcss --postcss --minify --input=css/app.css --output=../priv/static/assets/app.css"
}
```

and change this to:
```javascript
  "scripts": {
    "deploy": "NODE_ENV=production webpack --mode production",
    "watch": "webpack --mode development --watch"
  },
```

we will create a file for our custom styles the `assets/css/default-styles.css` file:
```bash
# assuming you are in the root directory
touch assets/css/default-styles.css
```

Let's also create our a custom component (we will make buttons for a counter to be sure tailwind and aplineJS are playing well together):
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

Now will will configure Phoenix to load Tailwind, our custom-styles and our custom-components -- DO THIS AT THE TOP OF the file `assets/css/app.scss` (@imports must be before all else):
```css
/* Import tailwind - with postcss-import installed */
@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";

/* custom styles - put after base imports! */
@import "./default-styles.css";
/* import custom components */
@import "./components/buttons.css";
/* default phoenix styles - eventually remove */
@import "./phoenix.css";
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

## Test with

```
iex -S mix phx.server
```
Now we should have a dropdown & our colored component buttons on our counter.

now lets snapshot our PETAL setup:
```bash
git add .
git commit -m "PETAL 1.6.x Configured"
```

## Add a custom Font

https://experimentingwithcode.com/custom-fonts-with-phoenix-and-tailwind/

We will download 2 Fonts:
* Noto Sans - (full international support & very readable):
https://google-webfonts-helper.herokuapp.com/fonts/noto-sans?subsets=latin
* Quickens Comic (easy to read and fun) -
https://graphicgoods.net/downloads/quickens-free-font/

### Noto Sans

We will start with Noto - listed on Google fonts from Webfonts Helper

Copy the CSS from the website (prefer modern browswers if possible) - update the Customize folder prefix
```css
mkdir -p assets/vendor/fonts/NotoSans
cat <<EOF>assets/vendor/fonts/NotoSans/noto_sans.css
/* noto-sans-regular - latin-ext_latin */
@font-face {
  font-family: 'Noto Sans';
  font-style: normal;
  font-weight: 400;
  src: local(''),
       url('../fonts/NotoSans/noto-sans-v25-latin-ext_latin-regular.woff2') format('woff2'), /* Chrome 26+, Opera 23+, Firefox 39+ */
       url('../fonts/NotoSans/noto-sans-v25-latin-ext_latin-regular.woff') format('woff'); /* Chrome 6+, Firefox 3.6+, IE 9+, Safari 5.1+ */
}
EOF
```

### Quickens (otf)

Download and convert the font at:

* https://transfonter.org/
* https://onlinefontconverter.com/
* https://www.fontsquirrel.com/tools/webfont-generator



```css
mkdir -p assets/vendor/fonts/Quickens
cat <<EOF>assets/vendor/fonts/Quickens/quickens.css
/* otf not recommended - convert the font at:
  * https://transfonter.org/
  * https://onlinefontconverter.com/
  * https://www.fontsquirrel.com/tools/webfont-generator

@font-face {
  font-family: 'Quickens';
  font-style: normal;
  font-weight: 400;
  src: local(''),
       url('quickens-regular.otf') format('opentype'),
       url('quickens-rough.otf') format('opentype');
} */

@font-face {
  font-family: 'Quickens';
  font-style: normal;
  font-weight: 400;
  src: local(''),
       url('quickens_regular-webfont.woff2') format('woff2'), /* Chrome 26+, Opera 23+, Firefox 39+ */
       url('quickens_regular-webfont.woff') format('woff'); /* Chrome 6+, Firefox 3.6+, IE 9+, Safari 5.1+ */
}

@font-face {
  font-family: 'QuickensRough';
  font-style: normal;
  font-weight: 400;
  src: local(''),
       url('quickens_rough-webfont.woff2') format('woff2'), /* Chrome 26+, Opera 23+, Firefox 39+ */
       url('quickens_rough-webfont.woff') format('woff'); /* Chrome 6+, Firefox 3.6+, IE 9+, Safari 5.1+ */
}
EOF
```

### Add Fonts to HTML

```html
# /lib/fonts_web/templates/layout/root.html.heex
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8"/>
    <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
    <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    <%= csrf_meta_tag() %>
    <%= live_title_tag assigns[:page_title] || "SlackerPetal", suffix: " · Phoenix Framework" %>
    <!-- add font here -->
    <link phx-track-static rel="stylesheet" href={Routes.static_path(@conn, "/assets/vendor/fonts/NotoSans/noto_sans.css")}/>
    <link phx-track-static rel="stylesheet" href={Routes.static_path(@conn, "/assets/vendor/fonts/NotoSerif/noto_serif.css")}/>
    <link phx-track-static rel="stylesheet" href={Routes.static_path(@conn, "/assets/vendor/fonts/Quickens/quickens.css")}/>
    <!-- end custom fonts -->
    <link phx-track-static rel="stylesheet" href={Routes.static_path(@conn, "/assets/app.css")}/>
    <!-- change this from app.js to js/app.js -->
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

OOPS
`(Phoenix.Router.NoRouteError) no route found for GET /assets/vendor/fonts/NotoSerif/noto_serif.css`

### Update the esbuild configuration

update args in

```
# /config/config.exs
# Configure esbuild (the version is required)
config :esbuild,
  version: "0.12.18",
  default: [
    args: ~w(js/app.js vendor/fonts/NotoSans/noto_sans.css vendor/fonts/Quickens/quickens.css --bundle --loader:.woff2=file --loader:.woff=file --target=es2017 --outdir=../priv/static/assets),
    cd: Path.expand("../assets", __DIR__),
    env: %{"NODE_PATH" => Path.expand("../deps", __DIR__)}
  ]
```

be careful `--target=es2017` in also `--target=es2016` in some Phoenix versions, but best to upgrade.

### Update the Tailwind configuration

The final step is to update Tailwind (set NotoSans to the default font & make quickens available)

```js
# /assets/tailwind.config.js
const defaultTheme = require('tailwindcss/defaultTheme')

module.exports = {
  mode: 'jit',
  purge: [
    './js/**/*.js',
    '../lib/*_web/**/*.*ex'
  ],
  theme: {
    extend: {
      fontFamily: {
        sans: ['NotoSans var', ...defaultTheme.fontFamily.sans],
        quicken: ['Quickens']
      },
    },
  },
  variants: {
    extend: {},
  },
  plugins: [],
}
```

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
