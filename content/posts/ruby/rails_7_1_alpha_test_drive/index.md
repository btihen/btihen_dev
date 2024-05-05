---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.1.0.alpha2 (main) Test-drive"
subtitle: "Installing & creating a new Rails 7.1 from Main"
summary: ""
authors: ['btihen']
tags: ['Ruby', 'install', 'initialize']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2023-03-04T01:20:00+02:00
lastmod: 2023-07-05T01:20:00+02:00
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

## Install Rails Gem

To install rails you first need to install the rails gem with:
```
gem install rails
```

I thought it would be fun to test the new alpha version of rails - but I always forget how to do this without upgrading an existing projects.

## Discover the Pre-release versions

```bash
gem list rails --remote --prerelease -e
```

`--remote` - checks the rubygems site - not the locally installed versions
`--prerelease` - find pre-release versions
`-e` - use an exact match (many packages have rails in the name).


If you find a package version you want the you install using:

```bash
gem install rails --version 7.0.0.alpha2

# or (also works to install older versions of rails)

rails _7.0.0.alpha2_ new magic_links
```

## Install main branch (an unpackaged version)

to install rails using `main` do:
```
rails new rails71 --main
```

## Favorite switches for rails new:

`--skip-test` - since I like using `rspec` ;)

```
rails new rails71 --javascript=esbuild --css=tailwind --database=postgresql --skip-bootsnap --skip-test

# see all options with:

rails new --h
```

Now continue as normal!

I like install:

* rspec
* flowbite
