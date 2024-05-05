---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 6.1 with TailwindCSS 2.0 and Stimulus 2.0"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ['Ruby', "rails 6", "configure", "install", "tailwind css", "stimulus"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2020-09-10T01:46:07+02:00
lastmod: 2021-03-21T01:46:07+02:00
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
# Intro

TailwindCSS is a very flexible CSS framework and makes it easy to customize unique web pages and animations.

Unfortunately, with Rails its a bit tricky to install and configure with Rails Standards.

* TailwindCSS 2.0 expects PostCSS 8 and Rails Webpacker uses PostCSS 7 (for now)
* TailwindCSS 2.0 expects AlpineJS, React or Vue -- by default Rails uses StimulusJS (although you can additionally install AlpineJS)

# Rails Setup

I am assuming you have followed the Rails setup described at: []()



## Install Tailwind CSS 2.0

### Tailwind CSS 2.0 Install

https://tailwindcss.com/docs


https://davidteren.medium.com/tailwindcss-2-0-with-rails-6-1-postcss-8-0-9645e235892d

https://www.michielsikkes.com/rails-new-app-guide-tailwindcss-webpack-stimulus-postcss/

Try this it looks pretty easy / workable:
https://dev.to/ranjanpurbey/use-tailwind-css-v2-0-with-rails-4j1o

https://davidteren.medium.com/tailwindcss-2-0-with-rails-6-1-postcss-8-0-9645e235892d
```
# would be nice to use postcss8 you need to use a newer rails webpacker for now -- (haven't tried):
yarn remove rails/webpacker
yarn add rails/webpacker#b6c2180

# update gem file with:
gem 'webpacker', github: 'rails/webpacker', ref: 'b6c2180'

# bundle

# uninstall postcss7 versions
yarn remove tailwindcss @tailwindcss/postcss7-compat

# install the postcss8 versions
yarn add tailwindcss@latest postcss@latest autoprefixer@latest  @tailwindcss/forms @tailwindcss/typography @tailwindcss/aspect-ratio
```


https://web-crunch.com/posts/how-to-install-tailwind-css-2-using-ruby-on-rails
```
yarn add tailwindcss@npm:@tailwindcss/postcss7-compat postcss@^7 autoprefixer@^9

# install AlpineJS

yarn add alpinejs

npx tailwindcss init
# or possibly
# npx tailwindcss init --full
```

now config tailwind:
```
# tailwind.config.js
module.exports = {
  purge: [
    './app/**/*/*.html.erb',
    './app/helpers/**/*/*.rb',
    './app/javascript/**/*/*.js',
    './app/javascript/**/*/*.vue',
    './app/javascript/**/*/*.react'
  ],
  darkMode: false, // or 'media' or 'class'
  theme: {
    extend: {},
  },
  variants: {
    extend: {},
  },
  plugins: [
    // not needed here ?
    // require('@tailwindcss/forms'),
  ],
}
```

tell `postcss.config.js` about tailwind:
```
/* postcss.config.js */
module.exports = {
  plugins: [
    require("tailwindcss")("./tailwind.config.js"),
    require("postcss-import"),
    require("postcss-flexbugs-fixes"),
    require("postcss-preset-env")({
      autoprefixer: {
        flexbox: "no-2009",
      },
      stage: 3,
    }),
  ],
}
```

create application.scss
```
mkdir app/javascript/stylesheets
touch app/javascript/stylesheets/application.scss
cat <<EOF >app/javascript/stylesheets/application.scss
/* app/javascript/stylesheets/application.scss */
@import "tailwindcss/base";
@import "tailwindcss/components";

/* Add any custom CSS here */
@import "tailwindcss/utilities";
EOF
```

import tailwind into `application.js`
```
/* app/javascript/packs/application.js */
import Rails from "@rails/ujs"
import "@hotwired/turbo-rails"
import * as ActiveStorage from "@rails/activestorage"
import "channels"

// import tailwind into javascript
import "../stylesheets/application"

Rails.start()
ActiveStorage.start()

import "controllers"

require("trix")
require("@rails/actiontext")
```

It's great to get samples from https://tailwindui.com (& other places) - USE THE INSPECTOR to copy the HTML (this will copy the AlpineJS settings too) - the standard copy HTML button requires you to add the JS on your own.

create a navbar:
```
touch app/views/layouts/_navbar.html.erb
cat <<EOF >app/views/layouts/_navbar.html.erb
<nav x-data="{ open: false }" class="bg-gray-800">
  <!-- NavBar here -->
</nav>
EOF
```

Create a footer:
```
<!-- app/views/layouts/_footer.html.erb -->
<footer class="bg-gray-50" aria-labelledby="footerHeading">
  <h2 id="footerHeading" class="sr-only">Company</h2>
  <div class="max-w-md mx-auto pt-12 px-4 sm:max-w-7xl sm:px-6 lg:pt-16 lg:px-8">
    <div class="xl:grid xl:grid-cols-3 xl:gap-8">
      Some Footer Info
    </div>
    <div class="mt-12 border-t border-gray-200 py-8">
      <p class="text-base text-gray-400 xl:text-center">
        &copy; 2020 Company, Inc. All rights reserved.
      </p>
    </div>
  </div>
</footer>
```

Update the landing page:
```
<!-- app/views/landing/index.html.erb -->
<!-- landing page here -->
```

`application.html.erb` needs to import the javascript stylesheet and the navbar
```
<!-- app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html>

<head>
  <title>Vivers</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>

  <link rel="stylesheet" href="https://rsms.me/inter/inter.css">
  <%= stylesheet_link_tag 'application', media: 'all' %>
  <%= stylesheet_pack_tag 'application', media: 'all' %>
  <%= javascript_pack_tag 'application' %>
  <%= yield :head %>
  <%= turbo_include_tags %>
  <%# stimulus_include_tags %>
</head>

<body>
  <div>
    <div class="pb-32">
      <%= render 'layouts/navbar' %>

      <header class="py-10">
        <div class="max-w-9xl mx-auto px-4 sm:px-6 lg:px-8">
          <p class="notice"><%= notice %></p>
          <p class="alert"><%= alert %></p>
          <h1 class="text-3xl font-bold">
            Dashboard
          </h1>
        </div>
      </header>

    </div>

    <main class="-mt-32">
      <div class="max-w-9xl mx-auto pb-12 px-4 sm:px-6 lg:px-8">

        <%= yield %>

      </div>
    </main>

    <%= render 'layouts/footer' %>
  </div>
</body>

</html>
```

You may need to start rails with both:
```
bin/rails s
# runnding the following in a separate window tends to speed CSS / JS recompilation
./bin/webpack-dev-server
```

## TailwindCSS 2.0 with AlpineJS





## Tailwind CSS 2.0 Animations with Stimulus 2.0

https://minimul.com/lets/watch/ec4V9rFVa1I/animate-tailwindui-components-via-stimulusjs-within-rails

### Also see this

https://dev.to/bobwalsh47hats/creating-a-deployable-rails-6-app-tailwindcss-stimulus-js-and-a-custom-font-3472

### Alerts

HTML: https://github.com/mdjamal/rails-tailwindcss-stimulus-alert-component/blob/master/app/views/home/index.html.erb

This would go into the `header` section of `application.html.erb`
```
<h1 class="semibold text-2xl">Rails Stimulus JS Alerts!</h1>
<p class="mb-8"> Ruby on Rails alerts styled with Tailwind CSS powered with Stimulus JS for close and auto close functions.</p>

<h2 class="mt-4 font-semibold text-lg">Animated (fade out) alerts</h2>
<div class="mt-2 relative h-10">
  <div class="bg-gray-100 border border-dashed border-gray-400 text-gray-700 px-4 py-3 text-center rounded w-full absolute z-0">
    <%= link_to "Reload Alert", root_path, class: "underline text-blue-600 hover:text-blue-700 text-sm lowercase" %>
  </div>
    <div data-controller="alert" data-alert-animation-class="transition duration-500 ease-in-out transform opacity-0" class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded relative z-10" role="alert">
      <strong class="font-bold">Holy smokes!</strong>
      <span class="block sm:inline">Something seriously bad happened</span>
      <span class="absolute top-0 bottom-0 right-0 px-4 py-3"  data-action="click->alert#close">
        <svg class="fill-current h-6 w-6 text-red-500" role="button" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20"><title>Close</title><path d="M14.348 14.849a1.2 1.2 0 0 1-1.697 0L10 11.819l-2.651 3.029a1.2 1.2 0 1 1-1.697-1.697l2.758-3.15-2.759-3.152a1.2 1.2 0 1 1 1.697-1.697L10 8.183l2.651-3.031a1.2 1.2 0 1 1 1.697 1.697l-2.758 3.152 2.758 3.15a1.2 1.2 0 0 1 0 1.698z"/></svg>
      </span>
    </div>
  </div>

  <h2 class="mt-10 font-semibold text-lg">Auto Closing alerts</h2>
  <div class="mt-2 relative h-10">
    <div class="bg-gray-100 border border-dashed border-gray-400 text-gray-700 px-4 py-3 text-center rounded w-full absolute z-0">
      <%= link_to "Reload Alert", root_path, class: "underline text-blue-600 hover:text-blue-700 text-sm lowercase" %>
    </div>
    <div data-controller="alert" data-alert-auto-close="5000" class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded relative" role="alert">
    <strong class="font-bold">Holy cow!</strong>
    <span class="block sm:inline">You are right as always. Closing in T - 5 seconds</span>
    <span class="absolute top-0 bottom-0 right-0 px-4 py-3"  data-action="click->alert#close">
      <svg class="fill-current h-6 w-6 text-green-500" role="button" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20"><title>Close</title><path d="M14.348 14.849a1.2 1.2 0 0 1-1.697 0L10 11.819l-2.651 3.029a1.2 1.2 0 1 1-1.697-1.697l2.758-3.15-2.759-3.152a1.2 1.2 0 1 1 1.697-1.697L10 8.183l2.651-3.031a1.2 1.2 0 1 1 1.697 1.697l-2.758 3.152 2.758 3.15a1.2 1.2 0 0 1 0 1.698z"/></svg>
    </span>
  </div>
</div>
```

Stimulus: https://github.com/mdjamal/rails-tailwindcss-stimulus-alert-component/blob/master/app/javascript/controllers/alert_controller.js

this goes into the `alerts_controller.js`

```
// app/javascript/controllers/alert_controller.js

import { Controller } from "stimulus"

export default class extends Controller {

  connect() {
    // Get animation class from data-alert-animation-class="your animation classess" attr if available else default to class .hidden
    this.animateClasses = (this.data.get('animationClass') || 'hidden').split(' ')
    console.log(this.animateClasses)

    // Auto Close if data-alert-auto-close="10000" attr is set. Time in ms 10000 = 10 seconds
    if (this.data.has("autoClose")) {
      setTimeout(() => this.close(), this.data.get("autoClose"))
    }
  }

  close() {
    if (this.element) {
      // to learn more about the js ... spread operator - https://dev.to/sagar/three-dots---in-javascript-26ci
		  this.element.classList.add(...this.animateClasses) // add the animation class to hide elelment from dom
      setTimeout(() => this.element.remove(), 0.5 * 1000) // remove from dom after 1/2 second
		}
  }
}
```

### Dropdown Menu

JAVASCRIPT for Navbar (with Stimulus), etc:
* https://minimul.com/lets/watch/ec4V9rFVa1I/animate-tailwindui-components-via-stimulusjs-within-rails

Easiest to do with Animate CSS
```
yarn add animate.css
```

Update your `application.css` to use the transitions used by TailwindCSS 2.0
```
# app/javascript/stylesheets/application.scss
/* app/javascript/stylesheets/application.scss */
@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";

/* My Custom Animation CSS for TailwindCSS */

/* Utility classes for animation speeds */
$speeds: 100, 150, 200, 250, 300, 350, 400;
@each $speed in $speeds {
  .speed-#{$speed} {
    animation-duration: #{$speed}ms;
  }
}

// Here we apply TailwindUIs transform recommendation for opening a dropdown
@keyframes dropIn {
  from { @apply transform opacity-0 scale-95; }
  to { @apply transform opacity-100 scale-100; }
}

.dropIn {
  // Just add the animation-name as the speed will be adjusted
  // via the speed-* utility classes made above.
  animation-name: dropIn;
}

// Here we apply TailwindUIs transform recommendation for closing a dropdown
@keyframes dropOut {
  from { @apply transform opacity-100 scale-100; }
  to { @apply transform opacity-0 scale-95; }
}

.dropOut {
  animation-name: dropOut;
}

@keyframes wiggle {
  0% { transform: translate(1px, 0); }
  50% { transform: translate(-1px, 0); }
  100% { transform: translate(1px, 0); }
}

// Make the utility class notification so when an element is hover
// over it will "wiggle".
.hover-wiggle {
  &:hover {
    animation: wiggle 75ms;
    animation-timing-function: linear;
  }
}
```

Make a Stimulus Dropdown Controller:
```
# app/javascript/controllers/dropdowns_controller.js
import { Controller } from "stimulus"

export default class extends Controller {
  static targets = ["menu"]
  static values = { invisible: Boolean }

  connect() {
    this.hide()
  }

  toggle(event) {
    event.preventDefault()
    this.changeBit()
    this.run()
  }

  hide() {
    this.menuTarget.classList.toggle('opacity-0')
    this.menuTarget.classList.toggle('hidden')
  }

  run() {
    let state = []
    if (this.invisibleValue) {
      state.push('dropOut', 'speed-400')
    } else {
      this.hide()
      state.push('dropIn', 'speed-200')
    }
    animateCSS({
      element: this.menuTarget, classes: state, callback: () => {
        if (this.invisibleValue) {
          this.hide()
        }
      }
    })
  }

  changeBit() {
    this.invisibleValue = !this.invisibleValue
  }

  clickedAway(event) {
    if (this.element.contains(event.target) === false && this.invisibleValue === false) {
      this.changeBit()
      this.run()
    }
  }
}
```

To make this controller work we need to make the following dropdown changes to our `_navbar.html.erb`
```
<nav class="bg-gray-800">
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
    <div class="flex items-center justify-between h-16">
      <div class="flex items-center">
        <div class="flex-shrink-0">
          <img class="h-8 w-8" src="https://tailwindui.com/img/logos/workflow-mark-indigo-500.svg" alt="Workflow">
        </div>
        <div class="hidden md:block">
          <div class="ml-10 flex items-baseline space-x-4">
            <!-- Current: "bg-gray-900 text-white", Default: "text-gray-300 hover:bg-gray-700 hover:text-white" -->
            <a href="#" class="bg-gray-900 text-white px-3 py-2 rounded-md text-sm font-medium">Dashboard</a>

            <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium">Team</a>

            <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium">Projects</a>

            <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium">Calendar</a>

            <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium">Reports</a>
          </div>
        </div>
      </div>
      <div class="hidden md:block">
        <div class="ml-4 flex items-center md:ml-6">
          <button class="bg-gray-800 p-1 rounded-full text-gray-400 hover:text-white hover-wiggle focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-gray-800 focus:ring-white">
            <span class="sr-only">View notifications</span>
            <!-- Heroicon name: outline/bell -->
            <svg class="h-6 w-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
            </svg>
          </button>

          <!-- Profile dropdown -->
          <div class="ml-3 relative" data-controller="dropdowns" data-dropdowns-invisible-value="true">
            <div>
              <button type="button"
                      class="max-w-xs bg-gray-800 rounded-full flex items-center text-sm text-white focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-gray-800 focus:ring-white"
                      id="user-menu" aria-expanded="false" aria-haspopup="true"
                      data-action="click->dropdowns#toggle click@window->dropdowns#clickedAway">
                <span class="sr-only">Open user menu</span>
                <img class="h-8 w-8 rounded-full" src="https://images.unsplash.com/photo-1472099645785-5658abf4ff4e?ixlib=rb-1.2.1&ixqx=DVFBnkSYvc&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=facearea&facepad=2&w=256&h=256&q=80" alt="">
              </button>
            </div>

            <!--
              Dropdown menu, show/hide based on menu state.

              Entering: "transition ease-out duration-100"
                From: "transform opacity-0 scale-95"
                To: "transform opacity-100 scale-100"
              Leaving: "transition ease-in duration-75"
                From: "transform opacity-100 scale-100"
                To: "transform opacity-0 scale-95"
            -->
            <div class="origin-top-right absolute right-0 mt-2 w-48 rounded-md shadow-lg py-1 bg-white ring-1 ring-black ring-opacity-5 focus:outline-none"
                role="menu"
                data-dropdowns-target="menu"
                aria-orientation="vertical"
                aria-labelledby="user-menu">
              <a href="#" class="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-100" role="menuitem">
                Your Profile
              </a>
              <a href="#" class="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-100" role="menuitem">
                Settings
              </a>
              <a href="#" class="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-100" role="menuitem">
                Sign out
              </a>
            </div>
          </div>
        </div>
      </div>
      <div class="-mr-2 flex md:hidden">
        <!-- Mobile menu button -->
        <button type="button" class="bg-gray-800 inline-flex items-center justify-center p-2 rounded-md text-gray-400 hover:text-white hover:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-gray-800 focus:ring-white" aria-controls="mobile-menu" aria-expanded="false">
          <span class="sr-only">Open main menu</span>
          <!--
            Heroicon name: outline/menu

            Menu open: "hidden", Menu closed: "block"
          -->
          <svg class="block h-6 w-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16" />
          </svg>
          <!--
            Heroicon name: outline/x

            Menu open: "block", Menu closed: "hidden"
          -->
          <svg class="hidden h-6 w-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
          </svg>
        </button>
      </div>
    </div>
  </div>

  <!-- Mobile menu, show/hide based on menu state. -->
  <div class="md:hidden" id="mobile-menu">
    <div class="px-2 pt-2 pb-3 space-y-1 sm:px-3">
      <!-- Current: "bg-gray-900 text-white", Default: "text-gray-300 hover:bg-gray-700 hover:text-white" -->
      <a href="#" class="bg-gray-900 text-white block px-3 py-2 rounded-md text-base font-medium">Dashboard</a>

      <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white block px-3 py-2 rounded-md text-base font-medium">Team</a>

      <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white block px-3 py-2 rounded-md text-base font-medium">Projects</a>

      <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white block px-3 py-2 rounded-md text-base font-medium">Calendar</a>

      <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white block px-3 py-2 rounded-md text-base font-medium">Reports</a>
    </div>
    <div class="pt-4 pb-3 border-t border-gray-700">
      <div class="flex items-center px-5">
        <div class="flex-shrink-0">
          <img class="h-10 w-10 rounded-full" src="https://images.unsplash.com/photo-1472099645785-5658abf4ff4e?ixlib=rb-1.2.1&ixqx=DVFBnkSYvc&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=facearea&facepad=2&w=256&h=256&q=80" alt="">
        </div>
        <div class="ml-3">
          <div class="text-base font-medium text-white">Tom Cook</div>
          <div class="text-sm font-medium text-gray-400">tom@example.com</div>
        </div>
        <button class="ml-auto bg-gray-800 flex-shrink-0 p-1 rounded-full text-gray-400 hover:text-white focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-gray-800 focus:ring-white">
          <span class="sr-only">View notifications</span>
          <!-- Heroicon name: outline/bell -->
          <svg class="h-6 w-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
          </svg>
        </button>
      </div>
      <div class="mt-3 px-2 space-y-1">
        <a href="#" class="block px-3 py-2 rounded-md text-base font-medium text-gray-400 hover:text-white hover:bg-gray-700">Your Profile</a>

        <a href="#" class="block px-3 py-2 rounded-md text-base font-medium text-gray-400 hover:text-white hover:bg-gray-700">Settings</a>

        <a href="#" class="block px-3 py-2 rounded-md text-base font-medium text-gray-400 hover:text-white hover:bg-gray-700">Sign out</a>
      </div>
    </div>
  </div>
</nav>
```

Now update the _navbar.html.erb to:
```
# app/views/layouts/_navbar.html.erb
<nav class="bg-gray-800">
  <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
    <div class="flex items-center justify-between h-16">
      <div class="flex items-center">
        <div class="flex-shrink-0">
          <img class="h-8 w-8" src="https://tailwindui.com/img/logos/workflow-mark-indigo-500.svg" alt="Workflow">
        </div>
        <div class="hidden md:block">
          <div class="ml-10 flex items-baseline space-x-4">
            <!-- Current: "bg-gray-900 text-white", Default: "text-gray-300 hover:bg-gray-700 hover:text-white" -->
            <a href="#" class="bg-gray-900 text-white px-3 py-2 rounded-md text-sm font-medium">Dashboard</a>

            <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium">Team</a>

            <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium">Projects</a>

            <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium">Calendar</a>

            <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white px-3 py-2 rounded-md text-sm font-medium">Reports</a>
          </div>
        </div>
      </div>
      <div class="hidden md:block">
        <div class="ml-4 flex items-center md:ml-6">
          <button class="bg-gray-800 p-1 rounded-full text-gray-400 hover:text-white hover-wiggle focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-gray-800 focus:ring-white">
            <span class="sr-only">View notifications</span>
            <!-- Heroicon name: outline/bell -->
            <svg class="h-6 w-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
            </svg>
          </button>

          <!-- Profile dropdown -->
          <div class="ml-3 relative" data-controller="dropdowns" data-dropdowns-invisible-value="true">
            <div>
              <button type="button"
                      class="max-w-xs bg-gray-800 rounded-full flex items-center text-sm text-white focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-gray-800 focus:ring-white"
                      id="user-menu" aria-expanded="false" aria-haspopup="true"
                      data-action="click->dropdowns#toggle click@window->dropdowns#clickedAway">
                <span class="sr-only">Open user menu</span>
                <img class="h-8 w-8 rounded-full" src="https://images.unsplash.com/photo-1472099645785-5658abf4ff4e?ixlib=rb-1.2.1&ixqx=DVFBnkSYvc&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=facearea&facepad=2&w=256&h=256&q=80" alt="">
              </button>
            </div>

            <!--
              Dropdown menu, show/hide based on menu state.

              Entering: "transition ease-out duration-100"
                From: "transform opacity-0 scale-95"
                To: "transform opacity-100 scale-100"
              Leaving: "transition ease-in duration-75"
                From: "transform opacity-100 scale-100"
                To: "transform opacity-0 scale-95"
            -->
            <div class="origin-top-right absolute right-0 mt-2 w-48 rounded-md shadow-lg py-1 bg-white ring-1 ring-black ring-opacity-5 focus:outline-none"
                role="menu"
                data-dropdowns-target="menu"
                aria-orientation="vertical"
                aria-labelledby="user-menu">
              <a href="#" class="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-100" role="menuitem">
                Your Profile
              </a>
              <a href="#" class="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-100" role="menuitem">
                Settings
              </a>
              <a href="#" class="block px-4 py-2 text-sm text-gray-700 hover:bg-gray-100" role="menuitem">
                Sign out
              </a>
            </div>
          </div>
        </div>
      </div>
      <div class="-mr-2 flex md:hidden">
        <!-- Mobile menu button -->
        <button type="button" class="bg-gray-800 inline-flex items-center justify-center p-2 rounded-md text-gray-400 hover:text-white hover:bg-gray-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-gray-800 focus:ring-white" aria-controls="mobile-menu" aria-expanded="false">
          <span class="sr-only">Open main menu</span>
          <!--
            Heroicon name: outline/menu

            Menu open: "hidden", Menu closed: "block"
          -->
          <svg class="block h-6 w-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16" />
          </svg>
          <!--
            Heroicon name: outline/x

            Menu open: "block", Menu closed: "hidden"
          -->
          <svg class="hidden h-6 w-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
          </svg>
        </button>
      </div>
    </div>
  </div>

  <!-- Mobile menu, show/hide based on menu state. -->
  <div class="md:hidden" id="mobile-menu">
    <div class="px-2 pt-2 pb-3 space-y-1 sm:px-3">
      <!-- Current: "bg-gray-900 text-white", Default: "text-gray-300 hover:bg-gray-700 hover:text-white" -->
      <a href="#" class="bg-gray-900 text-white block px-3 py-2 rounded-md text-base font-medium">Dashboard</a>

      <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white block px-3 py-2 rounded-md text-base font-medium">Team</a>

      <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white block px-3 py-2 rounded-md text-base font-medium">Projects</a>

      <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white block px-3 py-2 rounded-md text-base font-medium">Calendar</a>

      <a href="#" class="text-gray-300 hover:bg-gray-700 hover:text-white block px-3 py-2 rounded-md text-base font-medium">Reports</a>
    </div>
    <div class="pt-4 pb-3 border-t border-gray-700">
      <div class="flex items-center px-5">
        <div class="flex-shrink-0">
          <img class="h-10 w-10 rounded-full" src="https://images.unsplash.com/photo-1472099645785-5658abf4ff4e?ixlib=rb-1.2.1&ixqx=DVFBnkSYvc&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=facearea&facepad=2&w=256&h=256&q=80" alt="">
        </div>
        <div class="ml-3">
          <div class="text-base font-medium text-white">Tom Cook</div>
          <div class="text-sm font-medium text-gray-400">tom@example.com</div>
        </div>
        <button class="ml-auto bg-gray-800 flex-shrink-0 p-1 rounded-full text-gray-400 hover:text-white focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-gray-800 focus:ring-white">
          <span class="sr-only">View notifications</span>
          <!-- Heroicon name: outline/bell -->
          <svg class="h-6 w-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
          </svg>
        </button>
      </div>
      <div class="mt-3 px-2 space-y-1">
        <a href="#" class="block px-3 py-2 rounded-md text-base font-medium text-gray-400 hover:text-white hover:bg-gray-700">Your Profile</a>

        <a href="#" class="block px-3 py-2 rounded-md text-base font-medium text-gray-400 hover:text-white hover:bg-gray-700">Settings</a>

        <a href="#" class="block px-3 py-2 rounded-md text-base font-medium text-gray-400 hover:text-white hover:bg-gray-700">Sign out</a>
      </div>
    </div>
  </div>
</nav>
```

## Tailwind StimulusJS yarn package

https://github.com/excid3/tailwindcss-stimulus-components
```
yarn add tailwindcss-stimulus-components
```

## Rails 6 / TailwindCSS / StimulusJS Starter kit

https://www.getsjabloon.com/features/ui-components

$179

## other links

* https://github.com/excid3/tailwindcss-stimulus-components
* https://excid3.github.io/tailwindcss-stimulus-components
* https://github.com/mmccall10/el-transition
* https://dev.to/mmccall10/tailwind-enter-leave-transition-effects-with-stimulus-js-5hl7
* https://sebastiandedeyne.com/javascript-framework-diet/enter-leave-transitions/

* https://minimul.com/lets/watch/Wq5b6MfbCp4/animate-your-stimulusjs-components




https://tailwindui.com/ (offical components - paid)

https://www.tailwindtoolbox.com/
https://www.creative-tim.com/blog/web-development/free-tailwind-css-templates-resources/
https://www.creative-tim.com/learning-lab/tailwind-starter-kit/presentation


https://gorails.com/episodes/tailwind-css-framework-with-rails
```
yarn add tailwindcss

```

https://dev.to/bodhish/adding-tailwindcss-to-your-rails-6-project-2nk

https://web-crunch.com/posts/how-to-install-tailwind-css-2-using-ruby-on-rails
https://davidteren.medium.com/rails-6-and-tailwindcss-getting-started-42ba59e45393


## Install Tailwind SVG HeroIcons

You can embed the Icon directly into the View - downloading from:

https://heroicons.dev/
https://heroicons.com/

However, you can also use a gem and add flexibility:

https://github.com/bharget/heroicon

In gemfile
```
gem "heroicon"
```

From CLI:
```
bundle
rails g heroicon:install
```

Usage:
```
<%= heroicon "search" %>
<%= heroicon "search", variant: :outline %>
<%= heroicon "search", options: { class: "text-primary-500" } %>
```

or
https://github.com/andrewjmead/rails_heroicons/

Gemfile
```
gem 'rails_heroicons', '~> 1.0.1'
```

CLI
```
bundle
gem install rails_heroicons
```

Usage:
```
<%= heroicon('user') %>
<%= heroicon('user', class_name: 'icon icon-large') %>
<%= heroicon('user', style: :outline, class_name: 'icon icon-large') %>
```
The classes magically update the SVG embedded using:

### USE SVG Images / Icons in Rails -- HeroIcons or ZondIcons
Downloaded SVG images in Rails:
https://heroicons.com/
https://heroicons.dev/
http://www.zondicons.com/icons.html

OR download the Icons and use the gem:
https://github.com/jamesmartin/inline_svg

Gemfile:
```
gem 'inline_svg'
```

CLI:
```
bundle
gem install inline_svg
```
USAGE:
```
# Sprockets
inline_svg_tag(file_name, options={})

# Webpacker
inline_svg_pack_tag(file_name, options={})
```


OR without gem:

you can embed the SVG directly into rails using:
https://dev.to/hslzr/using-inline-svgs-with-rails-3khb
