---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 6.1 with TailwindCSS 2.0 and AlpineJS"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ['Ruby', "Rails 6.x", "configure", "install", "tailwindcss", "alpinejs"]
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2020-09-10T02:46:07+02:00
lastmod: 2021-08-07T01:46:07+02:00
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
# Intro

TailwindCSS is a very flexible CSS framework and makes it easy to customize unique web pages and animations.

Unfortunately, with Rails its a bit tricky to install and configure with Rails Standards.

* TailwindCSS 2.0 expects PostCSS 8 and Rails Webpacker uses PostCSS 7 (for now)
* TailwindCSS 2.0 expects AlpineJS, React or Vue -- by default Rails uses StimulusJS (although you can additionally install AlpineJS)

# Rails Setup

I am assuming you have followed the Rails setup described at: []()

In the end, I feel like its easier / better to use tailwindcss with AlpineJS since that is how it evolved and lots of Internet resources are available for that.


## Install Tailwind CSS 2.0

### Tailwind CSS 2.0 Install

https://tailwindcss.com/docs

Start by installing the tailwindcss compatible with postcss7 (necessary until rails-webpacker updates to postcss8) -- with or without upgrading webpacker the following should work:
```bash
yarn add tailwindcss@latest postcss@latest autoprefixer@latest

# if you get this error: Error: PostCSS plugin tailwindcss requires PostCSS 8. use:
# yarn add tailwindcss@npm:@tailwindcss/postcss7-compat postcss@^7 autoprefixer@^9
```

now install AlpineJS (its easier to use AlpineJS with tailwind but Stimulus works too - just need to do it all yourself - alpine and stimulus play well together in Rails).  Add alpine turbo drive adapter so that the AlpineJS effects work even AFTER clicking on a link!
```bash
yarn add alpinejs
yarn add alpine-turbo-drive-adapter
```

If you see something like:
```bash
arn add aplinejs
error An unexpected error occurred: “https://registry.yarnpkg.com/aplinejs: Not found”.
```
then check the spelling of the package(s).

Now create the tailwind config file
```bash
npx tailwindcss init
```

now config tailwind:
```javascript
// tailwind.config.js
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
```javascript
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
```bash
mkdir app/javascript/stylesheets
touch app/javascript/stylesheets/application.scss
cat <<EOF >app/javascript/stylesheets/application.scss
/* app/javascript/stylesheets/application.scss */
@import "tailwindcss/base";
@import "tailwindcss/components";
@import "tailwindcss/utilities";

/* Add custom CSS here */
EOF
```

import tailwind into `application.js`
```javascript
/* app/javascript/packs/application.js */
import Rails from "@rails/ujs"
import "@hotwired/turbo-rails"
import * as ActiveStorage from "@rails/activestorage"
import "channels"

// import alpinejs and its necessary rails adaptation
import 'alpine-turbo-drive-adapter'
require("alpinejs")

// import tailwind into javascript
import "../stylesheets/application.scss"

Rails.start()
ActiveStorage.start()

import "controllers"

require("trix")
require("@rails/actiontext")
```

It's great to get samples from https://tailwindui.com (& other places) - USE THE INSPECTOR to copy the HTML (this will copy the AlpineJS settings too) - the standard copy HTML button requires you to add the JS on your own.

create a navbar:
```bash
touch app/views/layouts/_navbar.html.erb
cat <<EOF >app/views/layouts/_navbar.html.erb
<nav x-data="{ open: false }" class="bg-gray-800">
  <!-- NavBar here -->
</nav>
EOF
```

Create a footer:
```ruby
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
```ruby
<!-- app/views/landing/index.html.erb -->
<!-- landing page here -->
```

`application.html.erb` needs to import the javascript stylesheet and the navbar
```ruby
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

You may need to want rails with both (to increase reload speed after changes -- but `bin/rails s` is enough):
```bash
bin/rails s
# runnding the following in a separate window tends to speed CSS / JS recompilation
./bin/webpack-dev-server
```

## Install Tailwind SVG Icons

You can embed the Icon directly into the View - downloading from:

https://heroicons.dev/
https://heroicons.com/

However, you can also use a gem and add flexibility:

https://github.com/bharget/heroicon

In gemfile
```ruby
gem "heroicon"
```

From CLI:
```bash
bundle
rails g heroicon:install
```

Usage:
```ruby
<%= heroicon "search" %>
<%= heroicon "search", variant: :outline %>
<%= heroicon "search", options: { class: "text-primary-500" } %>
```

or
https://github.com/andrewjmead/rails_heroicons/

Gemfile
```ruby
gem 'rails_heroicons', '~> 1.0.1'
```

CLI
```bash
bundle
gem install rails_heroicons
```

Usage:
```ruby
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
```ruby
gem 'inline_svg'
```

CLI:
```bash
bundle
gem install inline_svg
```
USAGE:
```ruby
# Sprockets
inline_svg_tag(file_name, options={})

# Webpacker
inline_svg_pack_tag(file_name, options={})
```


OR without gem:

you can embed the SVG directly into rails using:
https://dev.to/hslzr/using-inline-svgs-with-rails-3khb


## Reference Articles
https://davidteren.medium.com/tailwindcss-2-0-with-rails-6-1-postcss-8-0-9645e235892d
https://web-crunch.com/posts/how-to-install-tailwind-css-2-using-ruby-on-rails
