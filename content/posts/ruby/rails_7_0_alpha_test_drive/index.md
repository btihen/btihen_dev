---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Rails 7.0alpha2 Test-drive"
subtitle: "Installing & creating a new Rails Alpha Code base"
summary: ""
authors: ['btihen']
tags: ['Ruby', 'install', 'initialize']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2021-09-18T21:20:00+02:00
lastmod: 2021-09-18T21:20:00+02:00
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

I thought it would be fun to test the new alpha version of rails - but I always forget how to do this without upgrading an existing projects.

### Discover the Rails Pre-release versions

```bash
gem list rails --remote --prerelease -e
```

--remote - checks the rubygems site - not the locally installed versions
--prerelease - find pre-release versions
-e - use an exact match (many packages have rails in the name).


### Install the Rails Alpha Gems

```bash
gem install rails --version 7.0.0.alpha2
```

this installs a usable local version of rails


### Initailize a new Rails project with a specific version

```bash
rails _7.0.0.alpha2_ new magic_links
```

Now continue as normal!
