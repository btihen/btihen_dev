---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Hugo Academic Content Commands"
subtitle: "Summary of Content Creation Commands"
summary: "A quick summary of the Hugo Academic Theme creation commands"
authors: ["btihen"]
tags: ["Tech", "Hugo", "Static Site", "git", "Academic Theme", "commands"]
categories: ["Technology", "Website"]
date: 2020-05-23T10:39:21+02:00
lastmod: 2020-05-23T10:39:21+02:00
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

## From the Academic Documentation

https://sourcethemes.com/academic/docs/managing-content

## Create a User

```
hugo new --kind authors authors/firstname_lastname
```
**add person's image** (png or jpg)
```
cp picture.jpg content/authors/firstname_lastname/avatar.jpg
```

## Create a Blog

```
hugo new --kind post post/blog_title
```
**images within the article** - add images to the article folder:
```
cp image.jpg content/post/blog_title/article_image.jpg
```
and add it to the content using: `![kanban](example.jpg)` within the article

**add a display image** (png or jpg)
```
cp picture.jpg content/post/blog_title/featured.jpg
```

## Add a Publication Reference

```
hugo new --kind publication publication/publication_title
```

**add a display image** (png or jpg)
```
cp picture.jpg content/publication/publication_title/featured.jpg
```

**add a pdf** (with the same name as the folder) and it will be automatically available
```
cp picture.pdf content/publication/publication_title/publication_title.pdf
```

## Create a Project

```
hugo new --kind project project/project_name
```
**add a display image** (png or jpg)
```
cp picture.jpg content/project/project_name/featured.jpg
```

## Create a Talk

```
hugo new --kind talk talk/my-talk-name
```

**Talk Slides** are a bit more complicated see:
https://sourcethemes.com/academic/docs/managing-content/#create-slides

## Course (Documentation)

This is tricky (copy and rename an existing `course` and adapt it)

`courses` can be renamed and can have multiple folders (courses) within it.

**NOTE:** the `algebra_1` folder cannot have any sub-folders. Within an actual course all materials must be within a FLAT hierarchy.
