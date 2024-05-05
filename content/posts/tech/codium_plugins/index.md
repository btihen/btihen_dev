---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Docker Intro using MailCatcher"
subtitle: "Safely Test email sending in a dev environment"
summary: "Learn to set-up mail catcher for safe email testing with an introduction to Docker"
authors: ["btihen"]
tags: ["Tech", "Codium", "Editor", "Plugin", "Tooling"]
categories: ["Technology", "Code Editor"]
date: 2020-11-03T01:19:09+02:00
lastmod: 2020-11-13T01:19:09+02:00
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
Need to point Codium to MS extions?  Do the following:

### **Like Codium but need a MS Code Plugin?**

https://stackoverflow.com/questions/37143536/no-extensions-found-when-running-visual-studio-code-from-source

open:

`vim /Applications/VSCodium.app/Contents/Resources/app/product.json`

This can be fixed by adding following to product.json:
```
"extensionsGallery": {
    "serviceUrl": "https://marketplace.visualstudio.com/_apis/public/gallery",
    "cacheUrl": "https://vscode.blob.core.windows.net/gallery/index",
    "itemUrl": "https://marketplace.visualstudio.com/items"
}
```
