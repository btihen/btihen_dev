---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Hugo Website using Github"
subtitle: "Using the Academic Theme"
summary: "hugo web (with the Academic Theme) and using git submodules and github to publish a free website"
authors: ["btihen"]
tags: ["Tech", "Hugo", "Static Site", "git", "submodules"]
categories: ["Technology", "Website"]
date: 2020-05-16T10:39:21+02:00
lastmod: 2021-08-07T01:39:21+02:00
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

### step 0: install hugo

```bash
brew install hugo
```

### step 1: create a **public** `username_website` repo

I'll assume your github account is `username` I think this repo needs to be publicly readable (not 100% sure)

### step 2: clone the academic hugo locally

```bash
git clone https://github.com/sourcethemes/academic-kickstart.git username_website
cd academic_website
git submodule update --init --recursive  # without this the site won't start correctly
```

be sure you have many files within: **`themes/academic`**

### step 3: Update .gitignore & public folder

1. update `.gitignore` remove the line with `public`
2. be sure there is no `public` folder (yet), if there is remove it and all its contents.

### step 4: point this repo to your `username_website` repo

I have found the easiest way to overwrite the source `origin` repo is to do this by hand.

Currently your `.git/config` file will currently look like (notice the url referencing: `git://github.com/sourcethemes/academic-kickstart.git` - this is what we need to update):

```toml
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[remote "origin"]
	url = git://github.com/sourcethemes/academic-kickstart.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
```

change the origin url by hand or by using sed:
```bash
sed -i.bak -e 's/https:\/\/github.com\/sourcethemes\/academic-kickstart.git/git@github.com:username\/username_website.git/' .git/config
```

when your `.git/config` file is correct it will look like:
```toml
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[remote "origin"]
	url = git@github.com:username/username_website.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
```

now you can push this local repo to your github repo using:

```bash
git push -u origin --all
# git init
# git add .
# git commit -m "Initial commit"
# git push -u origin master
```

### step 5: configure website basics

#### 5A: Set your site name:

in `config/_default/config.toml`

find the `title` attribute and set it to `username` (or whatever is appropriate)

#### 5B: Pick a themes

from https://sourcethemes.com/academic/themes/

in `config/_default/config.toml`

find the `theme` attribute and set it to your favorite theme color (or leave it as is)

#### 5C: site logo & favicon

Put your image files into assets/images:
* `logo.png` (the logo on your webpage) file and
* `icon.png` (the favicon - icon in the webtab)

You can go to `https://www.namecheap.com/logo-maker` and make a logo

#### 5D: menu items

in `config/_default/menus.toml`

remove any items you won't use.  In my case this file now looks like:

```toml
[[main]]
  name = "Posts"
  url = "#posts"
  weight = 20

[[main]]
  name = "About"
  url = "#about"
  weight = 50

[[main]]
  name = "Contact"
  url = "#contact"
  weight = 60
```

These will also be the sections on the home page that will be enabled and configured.

The larger the weight the further to the **right** the item will be shown.

### step 6: configure site parameters

You may want to read through all the params - but the ones listed here are enough to get started.

* **site_type** -- in the file: `config/_default/params.toml`: be sure to configure the `site_type` variable
* **configure 'contact details'**
  - if you choose not to add an email, then be sure to set the variable `email_form=0` on the `content/home/contact.md` file!
  - if you choose not to enter an address and coordinates the in the `[map]` section set the `engine=0` to avoid problems.
* **configure social details** -- optional
* **Regional Settings** -- NOTE: The date display settings seems to have a bug -- so I don't recommend adjusting that.

### step 7: configure your homepage

At this point I suggest starting `hugo server` so you can watch your edits.

Now go into the folder `content/home` and we will adjust or disable the files in this folder.
* disable with: `active=false`
* enable with: `active=true`
* oder with: `weight=20` the bigger the number the further down on the page is show (I suggest you use the same weights used in the menu)

* **`contact.md`** - review and see if changes are desired.
* **`accomplishments.md`** - (and all other home page sections you decide not to display) change `active=true` to **`active=false`**

#### 7A: `about` page

I prefer to use the `about` page when it is a person's site and the `people` page when the site is about a group effort.  So in this case:

```bash
hugo new --kind authors authors/author_name
```

**`content/home/about.md`**

- change the title to whatever you like: biography, about, etc...
- change the variable `author` to match the name you used to generate you profile above, ie:
```yml
author = "author_name"
```

**`content/authors/author_name/_index.md`**

- Adjust the file so the information is accurate
- below the `---` toward the end of the file, add your own free text to the about page.

**`content/authors/author_name/avatar.jpeg`** (png, jpg, etc also work)

- add an attractive image to the folder `content/authors/author_name/` and name it: **`avatar.jpg`**

#### 7B: `people` (or Team) page

**disable `content/home/about.md`**
- Mark the `active` variable as `false`:
```yaml
active=false
```

**enable `content/home/people.md`**
- set `active=true`
- create sub-group names:
```toml
[content]
  user_groups = ["Educators", "Researchers"]
```

or alternatively, use an empty string to create a team without sub-teams:
```toml
[content]
  user_groups = [""]
```

**Create the people (authors)**
```bash
hugo new --kind authors authors/person_name
```

**`content/authors/person_name/_index.md`**

- add one (or more) `user_group` to the person's profile using the `user_groups` variable:
```toml
user_groups = ["Educators"]
```

if you used an empty string in `people.md` add:
```toml
user_groups = [""]
```
- Edit this file so that the information is accurate
- below the `---` toward the end of the file, add your own free text to the about page.

**`content/authors/person_name/avatar.jpeg`** (png, jpg, etc also work)

- add an attractive image to the folder `content/authors/person_name/` and name it: **`avatar.jpg`**


### step 8: Test publish to `username.github.io`

When you site is good enough to publish then its time to follow the following steps (these MUST be done in order to prevent problems!)

#### 8A: public folder (non-existent)

The first time you do setup for publishing it is important this folder doesn't exist yet and that `public` isn't listed in the .gitignore` file

#### 8B: git snapshot

**(DO NOT YET GENERATE your website)**

Create your git snapshot (very important at this point since the next steps are tricky)
```bash
git add .
git commit -m "First draft of homepage"
git push
```

#### 8C: make second github repo **`username.github.io`**

Now make a second **public** repo (CLICK THE BOX TO INCLUDE A **README** and/or a **LISENCE** file!) on github called **`username.github.io`**, this MUST be exactly: `username.github.io` for this to work!

Double check your repo is not empty, but has a **README** and/or a **LISENCE** file.

NOW go to github repo **settings** and click on **manage access** and be sure you have permission to at administer (or at least write to this repo) -- probably not so click the **`invite teams or people`** button and add yourself as an admin (an other as needed).


#### 8D: clone **`username.github.io`** to public (within your Hugo project)

now go back into your website code (root folder) and type:
```bash
git clone https://github.com/username/username.github.io.git public
```
if you see: `warning: You appear to have cloned an empty repository.` -- go back to the repo and create a README file!

#### 8E: check your permissions

enter you public folder and create an `index.html` file and put in very simple html code: `<h1>Hello username.github.io</h1>`

```bash
cd public
touch index.html
echo '<h1>Hello username.github.io</h1>' >> index.html
```

now check this in and push it to github.
```bash
git add .
git commit -m "test webpage"
git push
```

At this point you should see a bunch of message and toward the end you should see a line with:
```bash
To github.com:username/username.github.io.git
```

If instead you get the error:
```bash
remote: Permission to btihen/challenges.github.io.git denied to btihen.
fatal: unable to access 'https://github.com/btihen/challenges.github.io.git/': The requested URL returned error: 403
```

go back and check your site permissions.

If site permissions aren't a problem do the following:

re-create your website repo `username.github.io.git` outside the webcode project.
```bash
git clone git@github.com:username/username.github.io.git
cd username.github.io
echo '<h1>Hello username.github.io - v1</h1>' >> index.html
git add index.html
git commit -m "update readme and test permissions"
git push
```

assuming this works then move this repo into the hugo repo:
```
rm -rf username_website/public
mv username.github.io username_website/public
cd username_website/public
echo '<h1>Hello username.github.io - v2</h1>' >> index.html
git commit -am "update readme and test permissions within hugo project"
git push
```


#### 8F: check the website

Wait a few minutes and go to the website `https://username.github.io` and be sure you see your newly published html page.


### step 9: configure `public` as a **submodule**

Now add the username.github.io repo as a submodule to your website code repo using.  This allows nested projects without confusing git.

First be sure you are in the hugo root and not the public folder and type:
```bash
cd public
git submodule add -b master https://github.com/username/username.github.io.git public`
```

now in `.git/modules` you might see a folder called `public` (with a bunch of stuff in it) if not simply edit your `.git/config` so that after:
```toml
[submodule "themes/academic"]
  path = themes/academic
  url = https://github.com/gcushen/hugo-academic.git
```

you see:
```toml
[submodule "public"]
  path = public
  url = https://github.com/username/username.github.io.git
  branch = master
```

You can add it by hand or with:
```toml
cat <<"EOF" >> git/config
[submodule "public"]
  path = public
  url = https://github.com/username/username.github.io.git
  branch = master
EOF
```

### step 10: publish your new Hugo webpage:

Now to publish the Hugo site you prepared do the following:
```bash
hugo -d public
cd public
git add .
git commit -m "first webpage content"
git push
# toward the end you should see: `To github.com:username/username.github.io.git`
cd ..
```
Follow this proceedure every time you update your site.

NOTE: BE SURE NOT TO delete the folder `public/.git/` or you will need to reconfigure your public submodule.

now go back to `https://username.github.io` and you should see your hugo site!

(This might take a few minutes -- up to a half-hour -- to publish)
