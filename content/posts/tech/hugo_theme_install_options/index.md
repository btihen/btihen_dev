---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Hugo installing Themes - Options"
subtitle: "Theming Options, modules, git-submodules, or theme copy"
summary: "Options - choosing what's best for you"
authors: ["btihen"]
tags: ["Tech", "Hugo", "Static Site", "theme"]
categories: ["Technology", "Website"]
date: 2023-03-04T01:39:21+02:00
lastmod: 2021-03-05T01:39:21+02:00
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

## Getting Started

### step 0: install hugo & node

```bash
brew install hugo node
```

### step 1: create a new blank hugo site

```bash
hugo new site yourproject
```

### step 2: find a theme & fork (or directly clone it)

A good place to start is at: https://themes.gohugo.io/

## Install The Theme

A few interesting themes (to me)
* [Webslides](https://themes.gohugo.io/themes/hugo-webslides/)
* [Academic](https://themes.gohugo.io/themes/starter-hugo-academic/)
* [DocuAPI](https://themes.gohugo.io/themes/docuapi/) - API Docs with Code samples
* [Hero](https://themes.gohugo.io/themes/hugo-hero-theme/)
* [newbee](https://themes.gohugo.io/themes/newbee/)
* [travelify](https://themes.gohugo.io/themes/hugo-travelify-theme/)
* [port-hugo](https://themes.gohugo.io/themes/port-hugo/)
* [luna](https://themes.gohugo.io/themes/hugo-theme-luna/)
* [fresh](https://themes.gohugo.io/themes/hugo-fresh/)


### A - Hugo Modules

The officially preferred way. Some themes only allow this - ie: `https://github.com/wowchemy/starter-hugo-academic`

first update the `config.yaml` file with:

```yaml
module:
  imports:
  - path: "github.com/willfaught/paige"
```

Install the module:

```bash
cd yourproject
hugo mod init github.com/youraccount/yourproject
hugo mod get github.com/willfaught/paige
```

Get theme updates:
```bash
cd yourproject
hugo mod get -u
```

NOTE: I have had problems with some themes as they evolve their theme.

### B - Git subtree

[Subtrees Explained](https://www.atlassian.com/git/tutorials/git-subtree) - (Easier than Git Submodule)

Example `config.yaml`:
```
theme: "paige"
```
Install:
```
$ cd yourproject
$ git subtree add --prefix themes/paige --squash https://github.com/willfaught/paige master
```

Update:
```
$ cd yourproject
$ git subtree pull --prefix themes/paige --squash https://github.com/willfaught/paige master
```

### C - Git submodule

Example `config.yaml`:

```
theme: "paige"
```

Install:
```
$ cd yourproject
$ git submodule add https://github.com/willfaught/paige themes/paige
$ git submodule update --init --recursive --remote
```

Update:
```
$ cd yourproject
$ git submodule update --recursive --remote
```

NOTE: this turns out to be complex with CloudCannon and other web editors

### D - copy the theme (and remove git config)

Example `config.yaml`:
```
theme: "paige"
```

Install:
```
$ cd yourproject
$ git clone https://github.com/willfaught/paige themes/paige
$ rm -rf themes/paige/.git
```

Update:
```
$ cd yourproject
$ rm -rf themes/paige
$ git clone https://github.com/willfaught/paige themes/paige
$ rm -rf themes/paige/.git
```

## Running

```
$ cd yourproject
$ hugo mod npm pack
$ npm install
$ hugo server -D
```

## Resource

* [Paige Theme Instructions](https://github.com/willfaught/paige)
