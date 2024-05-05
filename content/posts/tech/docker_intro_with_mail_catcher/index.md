---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Docker Intro using MailCatcher"
subtitle: "Safely Test email sending in a dev environment"
summary: "Learn to set-up mail catcher for safe email testing with an introduction to Docker"
authors: ["btihen"]
tags: ["Tech", "Docker", "email", "Testing"]
categories: ["Technology", "Docker"]
date: 2020-05-12T21:19:09+02:00
lastmod: 2020-05-23T21:19:09+02:00
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
### **Intro**

It is often helpful to be able to test email sending from an application during development or testing (or even to make testing emails on a staging server possible).

To do this follow these instructions for a safe convenient way to test and inspect emails sent from an application.

### **SETUP**

First we need to get the repo (or at least the docker file)
```
# get the mailcatcher repo
git clone git@github.com:sj26/mailcatcher.git

# go into mailcather repo
cd mailcatcher

# configure to use the newest `released` gem version of mailcatcher
sed -i.bu1 's/FROM ruby:2.5/FROM ruby:2.6/' Dockerfile
sed -i.bu2 's/ARG VERSION=0.6.5/ARG VERSION=0.7.1/' Dockerfile
```

The Dockerfile should now look like (which is actually all that is actually needed):
```
FROM ruby:2.6
MAINTAINER Samuel Cochran <sj26@sj26.com>

ARG VERSION=0.7.1

RUN gem install mailcatcher -v $VERSION

EXPOSE 1025 1080

ENTRYPOINT ["mailcatcher", "--foreground"]
CMD ["--ip", "0.0.0.0"]
```

### **BUILD IMAGE**

Now you can download the docker image and install the gems into it with:
```
# -t adds repository:tag info -- the '.' at the end is important:
docker build -t btihen/ruby/mailcatcher:ruby_2.6 .
# ...
# should end with something like
# Successfully built 21e0de2bdd68

# now tag it as the **lasted** image with:
docker build -t btihen/ruby/mailcatcher:latest .
```

now you can see your list of docker images (you should see the starting image/container we just created):
```
docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
btihen/ruby/mailcatcher    latest              21e0de2bdd68        8 minutes ago       870MB
btihen/ruby/mailcatcher    ruby_2.6            21e0de2bdd68        8 minutes ago       870MB
ruby                       2.6                 a98425292e84        2 weeks ago         843MB
```

### **BUILD CONTAINER**

Now start the docker image using the build image id (`-d` allows it to run in the backgroud, `-p 1025:1025 -p 1080:1080` opens a connection on ports 1025 & 1080 from localhost to the docker image):

```
# build a container so we can test our image
docker run -d -p 1025:1025 -p 1080:1080 --name mailcatcher btihen/ruby/mailcatcher:latest

# or if you like ids better
docker run -d -p 1025:1025 -p 1080:1080 --name mailcatcher 21e0de2bdd68

# if you forgot the image-id you can list the images with:
docker images
```

### **TESTING (http & smtp)**

now you should be able to go to `http://localhost:1080` and see the mailcatcher webpage.

now lets test the smtp side from the cli using these instructions: `https://www.shellhacks.com/send-email-smtp-server-command-line/`
```
# connect to the mail server
$ telnet localhost 1025
# or
$ telnet 127.0.0.1 1025
220 smtp.domain.ext ESMTP Sendmail ?version-number?; ?date+time+gmtoffset?

# declare yourself (IP or DNS)
> HELO local.domain.name
250 smtp.domain.ext Hello local.domain.name [xxx.xxx.xxx.xxx], pleased to meet you

# declare who the email is from:
> MAIL FROM: test@local.domain.name
250 2.1.0 sender@adress.ext... Sender ok

# declare who should get the email:
> RCPT TO: recipient@adress.ext
250 2.1.5 recipient@adress.ext... Recipient ok

# setup the DATA transmission:
 > DATA
354 Enter mail, end with "." on a line by itself

# type a subject two returns and a message ending with '.' (on its own line):
SUBJECT: Test message

Hello,
this is a TEST message,
please don't reply.
Thank you.
.

# end the connection
> QUIT
```
Now check the mail has arrived in mailcatcher at `localhost:1080`

Assuming you see the email sent - you can be sure your image & container is setup properly.


### **STOPPING (exited) CONTAINER**

When we are done with mailcatcher we can stop the docker process:
```
docker ps -a
docker kill mailcatcher
```

### **STARTING BUILT (but exited) CONTAINERS**
To restart mailcatcher at a later date simply type:

`docker start mailcatcher`


### **SHARING IMAGES (once they work)**

```
# login to the Azure Container Repository
docker login btihen -u username -p xxxxxxxxxxx

# upload the new image
docker push btihen/ruby/mailcatcher
```

### **RETRIEVING SHARED IMAGE**

```
az acr login --name username
az acr repository list --name username --output table

# getting the image
docker pull btihen/ruby/image_name
```

**containerize the image**
```
# these are the default local ports - adjust to your needs
docker run -d -p 1025:1025 -p 1080:1080 --name mailcatcher btihen/ruby/mailcatcher:latest
```

**start the container**
```
docker start mailcatcher
```


### **LISTING Repo IMAGES**

**One-time install**
```
# if needed install the azure cli
brew update && brew install azure-cli

# the following may also be needed:
brew update && brew install python3 && brew upgrade python3
brew link --overwrite python3
```

**Retrieve the image list**
```
# login with the azure-cli
az acr login --name username

# list the images
az acr repository list --name username --output table
```

### **REMOVING CONTAINERS**
when we no longer need mailcatcher we can remove it with (`-a` lists running and stopped containers):
```
docker ps -a
docker rm mailcatcher
```

**REMOVING IMAGES**
To fully clean up and remove (images -- after the containers are removed):
```
$ docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
btihen/ruby/mailcatcher  ruby_2.5            21e0de2bdd68        25 minutes ago      870MB
ruby                     2.5                 a98425292e84        2 weeks ago         843MB

$ docker image rm 21e0de2bdd68

$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ruby                2.5                 a98425292e84        2 weeks ago         843MB
```
