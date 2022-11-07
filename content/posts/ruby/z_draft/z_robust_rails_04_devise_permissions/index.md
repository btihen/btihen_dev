---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Robust_rails_04_devise_permissions"
subtitle: ""
summary: ""
authors: ["btihen"]
tags: ['Ruby']
categories: ["Code", "Ruby Language", "Rails Framework"]
date: 2020-07-10T21:11:10+02:00
lastmod: 2020-07-10T21:11:10+02:00
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
Add controller info to template - ease testing of multiple pages with same url too
Add permissions to app_controllers (with simple and complex exceptions)

### allow faculty into kids with devise

https://stackoverflow.com/questions/18520030/how-to-skip-before-filter-authenticate-user-if-current-admin-user?lq=1

before_filter :authenticate_user!, unless: :current_admin_user

### friendly URLs
https://gist.github.com/jcasimir/1209730
https://github.com/norman/friendly_id
