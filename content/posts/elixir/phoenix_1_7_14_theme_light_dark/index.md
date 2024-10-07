---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Phoenix - Migrating to a Theme"
subtitle: "Relatively easily migrate the Core Components to a Theme and allow light and dark mode"
# Summary for listings and search engines
summary: "Replacing given colors with named colors and allow dark-mode settings"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "LiveView", "Tailwind Theme"]
categories: ["Code", "Elixir Language", "Phoenix Framework", "Frontend", "Theme"]
date: 2024-10-06T01:01:53+02:00
lastmod: 2024-10-76T01:01:53+02:00
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

I've been been looking to add a theme to Phoenix - in this way simplifying the core components and dark-mode.

Here is my current solution.

## Configure Tailwind Color Variables

### using CSS & HSL

Define Colors in CSS using HSL (Hue, Saturation, Lightness)

```scss
// assets/css/app.css
:root {
  --border-standard: 220 13% 91%; /* Light mode: a very light blue-gray */
  --background-primary: 240 25% 97%; /* Light mode: almost white */
  --text-standard: 220 39% 11%; /* Light mode: dark blue-gray */
}

.dark {
  --border-standard: 220 13% 30%; /* Dark mode: much darker blue-gray */
  --background-primary: 220 27% 13%; /* Dark mode: very dark */
  --text-standard: 210 20% 98%; /* Dark mode: nearly white text */
}
```

You can use the HSL color picker to find the values you want. To convert from RGB to HSL, you can use an online converter like [this one](https://www.rapidtables.com/convert/color/rgb-to-hsl.html).

Reference in Tailwind with hsl(var(--...)):

In tailwind.config.js, you are telling Tailwind to use the CSS variables and then combining them with hsl() to make them usable:

```js
// config/tailwind.config.js
Copy code
extend: {
  colors: {
    'border-standard': 'hsl(var(--border-standard))', // Tailwind uses this to generate utility classes
  },
},
```

### using Tailwind colors

In order to use the existing tailwind, but allow light and dark mode.

We start by adding the tailwind color module by adding:
`const colors = require('tailwindcss/colors');`

and within the the theme colors area we define colors found in the existing components - ie:
`'neutral-strongest': colors.zinc[900],`

Then to simplify light / dark mode, we add the following plugin - which allows the color definitions to be appropriate for light and dark:
```js
 // add custom color palette that auto switches between light and dark
    function ({ addBase, theme }) {
      const lightModeColors = {
        '--color-neutral': theme('colors.neutral-strong'),
        '--color-neutral': theme('colors.neutral-accented'),
        '--color-neutral': theme('colors.neutral'),
        '--color-neutral': theme('colors.neutral-muted'),
        '--color-neutral': theme('colors.neutral-faint'),
        '--color-neutral': theme('colors.neutral-strong-reverse'),
        '--color-neutral': theme('colors.neutral-accented-reverse'),
        '--color-neutral': theme('colors.neutral-reverse'),
        '--color-neutral': theme('colors.neutral-muted-reverse'),
        '--color-neutral': theme('colors.neutral-faint-reverse'),
        '--color-primary': theme('colors.primary'),
      };

      const darkModeColors = {
        '--color-neutral': theme('colors.neutral-strong-dark'),
        '--color-neutral': theme('colors.neutral-accented-dark'),
        '--color-neutral': theme('colors.neutral-dark'),
        '--color-neutral': theme('colors.neutral-muted-dark'),
        '--color-neutral': theme('colors.neutral-faint-dark'),
        '--color-neutral': theme('colors.neutral-strong-reverse-dark'),
        '--color-neutral': theme('colors.neutral-accented-reverse-dark'),
        '--color-neutral': theme('colors.neutral-reverse-dark'),
        '--color-neutral': theme('colors.neutral-muted-reverse-dark'),
        '--color-neutral': theme('colors.neutral-faint-reverse-dark'),
        '--color-primary': theme('colors.primary-dark'),
      };
```

```js
// See the Tailwind configuration guide for advanced usage
// https://tailwindcss.com/docs/configuration

const plugin = require("tailwindcss/plugin")
const fs = require("fs")
const path = require("path")
// needed for using tailwind colors
const colors = require('tailwindcss/colors');

module.exports = {
  content: [
    "./js/**/*.js",
    "../lib/phx_theme_web.ex",
    "../lib/phx_theme_web/**/*.*ex"
  ],
  theme: {
    extend: {
      colors: {
        brand: "#FD4F00",
        //
        // neutral colors (light)
        'neutral-strongest': colors.zinc[900],
        'neutral-heavy': colors.zinc[800],
        'neutral-accented': colors.zinc[700],
        'neutral': colors.zinc[600],
        'neutral-central': colors.zinc[500],
        'neutral-muted': colors.zinc[400],
        'neutral-soft': colors.zinc[300],
        'neutral-subtle': colors.zinc[200],
        'neutral-dim': colors.zinc[100],
        'neutral-faint': colors.zinc[50],
        // 'neutral-strong-reverse': colors.zinc[100],
        // 'neutral-accented-reverse': colors.zinc[200],
        // 'neutral-reverse': colors.zinc[400],
        // 'neutral-muted-reverse': colors.zinc[400],
        // 'neutral-faint-reverse': colors.zinc[900],
        //
        // neutral colors (dark)
        'neutral-strongest-dark': colors.zinc[100],
        'neutral-heavy-dark': colors.zinc[200],
        'neutral-accented-dark': colors.zinc[300],
        'neutral-dark': colors.zinc[400],
        'neutral-central-dark': colors.zinc[500],
        'neutral-muted-dark': colors.zinc[600],
        'neutral-soft-dark': colors.zinc[700],
        'neutral-subtle-dark': colors.zinc[800],
        'neutral-dim-dark': colors.zinc[900],
        'neutral-faint-dark': colors.zinc[950],
        //
        // 'neutral-strong-reverse-dark': colors.zinc[100],
        // 'neutral-accented-reverse-dark': colors.zinc[800],
        // 'neutral-reverse-dark': colors.zinc[700],
        // 'neutral-muted-reverse-dark': colors.zinc[600],
        // 'neutral-faint-reverse-dark': colors.zinc[100],
        //
        // colors (light)
        'primary': colors.blue[500],
        // colors (dark)
        'primary-dark': colors.blue[300],
      }
    },
  },
  plugins: [
    require("@tailwindcss/forms"),
    // Allows prefixing tailwind classes with LiveView classes to add rules
    // only when LiveView classes are applied, for example:
    //
    //     <div class="phx-click-loading:animate-ping">
    //
    plugin(({addVariant}) => addVariant("phx-click-loading", [".phx-click-loading&", ".phx-click-loading &"])),
    plugin(({addVariant}) => addVariant("phx-submit-loading", [".phx-submit-loading&", ".phx-submit-loading &"])),
    plugin(({addVariant}) => addVariant("phx-change-loading", [".phx-change-loading&", ".phx-change-loading &"])),

    // Embeds Heroicons (https://heroicons.com) into your app.css bundle
    // See your `CoreComponents.icon/1` for more information.
    //
    plugin(function({matchComponents, theme}) {
      let iconsDir = path.join(__dirname, "../deps/heroicons/optimized")
      let values = {}
      let icons = [
        ["", "/24/outline"],
        ["-solid", "/24/solid"],
        ["-mini", "/20/solid"],
        ["-micro", "/16/solid"]
      ]
      icons.forEach(([suffix, dir]) => {
        fs.readdirSync(path.join(iconsDir, dir)).forEach(file => {
          let name = path.basename(file, ".svg") + suffix
          values[name] = {name, fullPath: path.join(iconsDir, dir, file)}
        })
      })
      matchComponents({
        "hero": ({name, fullPath}) => {
          let content = fs.readFileSync(fullPath).toString().replace(/\r?\n|\r/g, "")
          let size = theme("spacing.6")
          if (name.endsWith("-mini")) {
            size = theme("spacing.5")
          } else if (name.endsWith("-micro")) {
            size = theme("spacing.4")
          }
          return {
            [`--hero-${name}`]: `url('data:image/svg+xml;utf8,${content}')`,
            "-webkit-mask": `var(--hero-${name})`,
            "mask": `var(--hero-${name})`,
            "mask-repeat": "no-repeat",
            "background-color": "currentColor",
            "vertical-align": "middle",
            "display": "inline-block",
            "width": size,
            "height": size
          }
        }
      }, {values})
    }),
    // add custom color palette that auto switches between light and dark
    function ({ addBase, theme }) {
      const lightModeColors = {
        '--color-neutral': theme('colors.neutral-strong'),
        '--color-neutral': theme('colors.neutral-accented'),
        '--color-neutral': theme('colors.neutral'),
        '--color-neutral': theme('colors.neutral-muted'),
        '--color-neutral': theme('colors.neutral-faint'),
        '--color-neutral': theme('colors.neutral-strong-reverse'),
        '--color-neutral': theme('colors.neutral-accented-reverse'),
        '--color-neutral': theme('colors.neutral-reverse'),
        '--color-neutral': theme('colors.neutral-muted-reverse'),
        '--color-neutral': theme('colors.neutral-faint-reverse'),
        '--color-primary': theme('colors.primary'),
      };

      const darkModeColors = {
        '--color-neutral': theme('colors.neutral-strong-dark'),
        '--color-neutral': theme('colors.neutral-accented-dark'),
        '--color-neutral': theme('colors.neutral-dark'),
        '--color-neutral': theme('colors.neutral-muted-dark'),
        '--color-neutral': theme('colors.neutral-faint-dark'),
        '--color-neutral': theme('colors.neutral-strong-reverse-dark'),
        '--color-neutral': theme('colors.neutral-accented-reverse-dark'),
        '--color-neutral': theme('colors.neutral-reverse-dark'),
        '--color-neutral': theme('colors.neutral-muted-reverse-dark'),
        '--color-neutral': theme('colors.neutral-faint-reverse-dark'),
        '--color-primary': theme('colors.primary-dark'),
      };

      // Add CSS variables to :root for light mode
      addBase({
        ':root': lightModeColors,
      });

      // Add CSS variables to .dark for dark mode
      addBase({
        '.dark': darkModeColors,
      });
    }
  ]
}
```

## Alternative Solution

Define Colors in CSS (`assets/css/app.css`) using HSL colors - this is simpler and easier to manage, but requires using HSL 0 which is not as intuitive (for me).

```css
/* at the end of assets/css/app.css (or in another CSS file) */
:root {
  --standard-border: 220 13% 91%; /* Light mode: a very light blue-gray */
  --background-primary: 240 25% 97%; /* Light mode: almost white */
  --text-standard: 220 39% 11%; /* Light mode: dark blue-gray */
}

.dark {
  --standard-border: 220 13% 30%; /* Dark mode: much darker blue-gray */
  --background-primary: 220 27% 13%; /* Dark mode: very dark */
  --standard-text: 210 20% 98%; /* Dark mode: nearly white text */
}
```
Reference in Tailwind with hsl(var(--...)):

In `tailwind.config.js`, you are telling Tailwind to use the CSS variables and then combining them with hsl() to make them usable:

```js
// tailwind.config.js
extend: {
  colors: {
    'standard-border': 'hsl(var(--border-standard))', // Tailwind uses this to generate utility classes
  },
},
```

Usage in HTML:

You can now use the Tailwind utility classes like border-border-standard, and they will adapt based on the mode:

```html
Copy code
<div class="border border-standard-border p-4">
  <h1 class="text-standard-text">Hello, World!</h1>
</div>
```

## Rename in-line colors

now we can rename colors in the components like:
`zinc-600` to `neutral`
and we can go through the entire list and do this.

## Colors:

Independent of what Phoenix has done it seems to make sense to define colors similar to (and these are all defined for both light `:root` and dark `.dark`) - this allow us to avoid using both:
`text-neutral dark:text-neutral`
and just use:
`text-neutral`
for both (the plugin we wrote handles light and dark mode colors)

**Colors**

* primary
* secondary
* success
* danger
* warning
* info
* neutral

* white
* black

**Neutral Shades** - possibly other colors if needed

* strongest
* accented
* standard
* muted
* faintest

## add dark-mode

See the article:

* [automatic light/dark mode](https://btihen.dev/posts/elixir/phoenix_1_7_11_liveview_1_0_0_system_dark_toggle/)
* [manual light/dark mode](https://btihen.dev/posts/elixir/phoenix_1_7_11_liveview_1_0_0_manual_dark_toggle/)

**TLDR** Example JavaScript to toggle dark mode:

```js
// Toggles the 'dark' class on the root HTML element
const toggleDarkMode = () => {
  document.documentElement.classList.toggle('dark');
};
```
Storing User Preference:

You can store the user’s preference (light or dark) in localStorage and use JavaScript to apply it on page load.

```js
const userPrefersDark = localStorage.getItem('theme') === 'dark';
if (userPrefersDark) {
  document.documentElement.classList.add('dark');
}

const toggleDarkMode = () => {
  const htmlElement = document.documentElement;
  htmlElement.classList.toggle('dark');
  localStorage.setItem('theme', htmlElement.classList.contains('dark') ? 'dark' : 'light');
};
```

## Conclusion

It would be cool if Phoenix did this from the start!
