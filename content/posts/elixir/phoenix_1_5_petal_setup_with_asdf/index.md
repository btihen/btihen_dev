---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.5 PETAL Stack Setup - w/ asdf"
subtitle: "Phoenix, Elixir, TailwindCSS, AlpineJS, LiveView - PETAL Stack"
summary: "Create a modern Phoenix SPA with tremendous flexibility"
authors: ["btihen"]
tags: ["Elixir", "TailwindCSS", "AlpineJS", "LiveView", "PETAL Stack"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2021-04-10T01:01:53+02:00
lastmod: 2021-08-07T01:01:53+02:00
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

- https://thinkingelixir.com/install-elixir-using-asdf/
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
asdf plugin-add nodejs
asdf plugin-add Postgres
```

Now you need to install the desired versions (usually the newest) - currently:
```bash
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
```bash
mix archive.install hex phx_new 1.5.8
```

## create / config a project

First we will creat the folder / project location
```bash
mkdir fenix
```

Now we will tell it which software to use:
```bash
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
```bash
mix phx.new fenix --live
cd fenix
mix ecto.create
mix phx.server
```

assuming all is good lets configure git:
```bash
git init
git add .
git commit -m "initial Phoneix install with LiveView"
```


## install & test Alpine JS
https://underjord.io/getting-started-with-petal.html
https://dockyard.com/blog/2020/12/21/optimizing-user-experience-with-liveview

```bash
cd assets
npm install alpinejs
```

Now change `app.js` is to require our new setup:

```bash
# assets/js/app.js
// .. after the app.scss import add:
import Alpine from "alpinejs";
```

still in `assets/js/app.js` find`:
`let liveSocket = new LiveSocket("/live", Socket, {params: {_csrf_token: csrfToken}})`

and change to:
```elixir
let liveSocket =
    new LiveSocket("/live",
                    Socket,
                    { params: {_csrf_token: csrfToken},
                      dom: {
                        onBeforeElUpdated(from, to){
                          if(from.__x){ Alpine.clone(from.__x, to) }
                        }
                    } }
                  )
```

TEST by adding to the end of: `lib/fenix_web/live/page_live.html.leex`
```elixir
<section>
  <h2>Alpine JS Installed</h2>
  <div x-data="{name:''}">
    <label for="name">Name:</label>
    <input id="name" type="text" x-model="name" />
    <p><br><b><em>Output:</em></b> <span x-text="name"></p>
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
https://pragmaticstudio.com/tutorials/adding-tailwind-css-to-phoenix
https://fullstackphoenix.com/tutorials/get-started-with-tailwind-in-phoenix

This source gives several options - here we install with `postcss-import` (for components from the beginning):

```javascript
cd assets
npm install tailwindcss postcss autoprefixer postcss-loader@4.2  postcss-import --save-dev

touch postcss.config.js
cat <<EOF > postcss.config.js
module.exports = {
  plugins: {
    "postcss-import": {},
    tailwindcss: {},
    autoprefixer: {}
  }
}
EOF
```

now open: `assets/webpack.config.js` and find:
```javascript
{
	test: /\.[s]?css$/,
	use: [
		MiniCssExtractPlugin.loader,
		'css-loader',
		'sass-loader',
	],
}
```

change too (add `'postcss-loader'` between `'css-loader'` & `'sass-loader'`):
```css
{
	test: /\.[s]?css$/,
	use: [
		MiniCssExtractPlugin.loader,
		'css-loader',
		'postcss-loader',
		'sass-loader',
	],
}
```

now initialize tailwind config with:
```bash
npx tailwindcss init
```

this creates the file `tailwind.config.js` we will replace the `purge: [],` section with:
```javascript
purge: [
    '../lib/**/*.ex',
    '../lib/**/*.leex',
    '../lib/**/*.eex',
    './js/**/*.js'
  ],
```

Now the fill will look like:
```javascript
module.exports = {
  purge: [
    '../lib/**/*.ex',
    '../lib/**/*.leex',
    '../lib/**/*.eex',
    './js/**/*.js'
  ],
  darkMode: false, // or 'media' or 'class'
  theme: {
    extend: {},
  },
  variants: {
    extend: {},
  },
  plugins: [],
}
```


now in `assets/package.json` find:
```javascript
  "scripts": {
    "deploy": "webpack --mode production",
    "watch": "webpack --mode development --watch"
  },
```

and change this to:
```javascript
  "scripts": {
    "deploy": "NODE_ENV=production webpack --mode production",
    "watch": "webpack --mode development --watch"
  },
```

we will create a file for our custom styles the `assets/css/custom-style.css` file:
```bash
# assuming you are still in the assets directory on the cli
touch css/custom-styles.css
```

Let's also create our a custom component (we will make buttons for a counter to be sure tailwind and aplineJS are playing well together):
```css
# assuming you are still in the assets directory on the cli
mkdir css/components
touch css/components/buttons.css
cat <<EOF > css/components/buttons.css
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
@import "./custom-styles.css";

/* import custom components */
@import "./components/buttons.css";

/* default phoenix styles - eventually remove */
@import "./phoenix.css";
```


add a test html from tailwind to the end of: `lib/fenix_web/live/page_live.html.leex`
```elixir
<section class="grid grid-cols-1 gap-4">
  <div>
    <h2 class="text-red-500 text-5xl font-bold text-center">Tailwind CSS with AlpineJS</h2>
    <p class="mt-5 font-bold text-center">Red Title with Colored Counter Buttons</p>
  </div>
  <div class="mt-10 flex justify-center" x-data="{ count: 0 }">
    <button class="btn-redish" x-on:click="count--">Decrement</button>
    <code>count: </code><code x-text="count"></code>
    <button class="btn-greenish" x-on:click="count++">Increment</button>
  </div>
</section>
```
Now when we start the server with `mix phx.server` we should have a centered / red title and colored buttons on our counter.

now lets snapshot our PETAL setup:
```bash
git add .
git commit -m "Tailwind installed"
```


## Resources

- https://www.youtube.com/watch?v=o4Prej0wIZA
- http://blog.pthompson.org/alpine-js-and-liveview
- https://pragmaticstudio.com/tutorials/adding-tailwind-css-to-phoenix
- https://fullstackphoenix.com/tutorials/get-started-with-tailwind-in-phoenix
- https://fullstackphoenix.com/tutorials/combine-phoenix-liveview-with-alpine-js
- https://medium.com/mindvalley-technology/how-to-add-tailwindcss-to-your-phoenix-project-e2250ad31ace
- https://thinkingelixir.com/podcast-episodes/021-tailwind-css-alpine-js-and-liveview-with-patrick-thompson/
