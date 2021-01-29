---
title: Initial commit
author: Rapoma
date: 2021-01-27 21:10:00 +0100
categories: [Post, Tutorial]
tags: [sharing, jekyll, github, docker, arm64]
image: /assets/img/posts/post1.png
---

## Intro


In this first post, I share the steps I followed, by making this personal website. There are no particular requirements to be able to follow along. I would recommend you to explore [jekyll](https://jekyllrb.com/), and [Chirpy](https://chirpy.cotes.info) tho. Both provides a quick start guide from zero to hosting your static site on [github](https://github.com/).


## Steps


For development, this method should work on any platforms. I use [docker](https://www.docker.com/) for the local development. You then need to have *docker* and *docker-compose* installed.

You can look at the docker hub the image ```jekyll/jekyll``` by just pulling the latest image (4.0). Last time I checked, there is no image for arm64. I had to build the image from a ```Dockerfile```. Looking into its [repository](https://github.com/envygeeks/jekyll-docker) and did some *massage* on their Dockerfile.

### Downloading the repo

```wget https://github.com/envygeeks/jekyll-docker/trunk/repos/jekyll```

### Customize Dockerfile

```dockerfile
FROM ruby:2.7.1-alpine3.11
COPY copy/all /

#
# EnvVars
# Ruby
#

ENV BUNDLE_HOME=/usr/local/bundle
ENV BUNDLE_APP_CONFIG=/usr/local/bundle
ENV BUNDLE_DISABLE_PLATFORM_WARNINGS=true
ENV BUNDLE_BIN=/usr/local/bundle/bin
ENV GEM_BIN=/usr/gem/bin
ENV GEM_HOME=/usr/gem
ENV RUBYOPT=-W0

#
# EnvVars
# Image
#

ENV JEKYLL_VAR_DIR=/var/jekyll
ENV JEKYLL_DOCKER_TAG=4.0
ENV JEKYLL_VERSION=4.0
ENV JEKYLL_DOCKER_COMMIT='git rev-parse --verify HEAD'
ENV JEKYLL_DOCKER_NAME=myjekyll
ENV JEKYLL_DATA_DIR=/srv/jekyll
ENV JEKYLL_BIN=/usr/jekyll/bin
ENV JEKYLL_ENV=development

#
# EnvVars
# System
#

ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US:en
ENV TZ=America/Chicago
ENV PATH="$JEKYLL_BIN:$PATH"
ENV LC_ALL=en_US.UTF-8
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US


env VERBOSE=false
env FORCE_POLLING=false
env DRAFTS=false

#
# Packages
# User
#


#
# Packages
# Dev
#

RUN apk --no-cache add \
  zlib-dev \
  libffi-dev \
  build-base \
  libxml2-dev \
  imagemagick-dev \
  readline-dev \
  libxslt-dev \
  libffi-dev \
  yaml-dev \
  zlib-dev \
  vips-dev \
  sqlite-dev \
  cmake

#
# Packages
# Main
#

RUN apk --no-cache add \
  linux-headers \
  openjdk8-jre \
  less \
  zlib \
  libxml2 \
  readline \
  libxslt \
  libffi \
  git \
  nodejs \
  tzdata \
  shadow \
  bash \
  su-exec \
  nodejs-npm \
  libressl \
  yarn

#
# Gems
# Update
#

RUN echo "gem: --no-ri --no-rdoc" > ~/.gemrc
RUN unset GEM_HOME && unset GEM_BIN && \
  yes | gem update --system

#
# Gems
# Main
#

RUN unset GEM_HOME && unset GEM_BIN && yes | gem install --force bundler
RUN gem install jekyll -v4.2.0 -- --use-system-libraries

#
# Gems
# User
#


RUN addgroup -Sg 1000 jekyll
RUN adduser  -Su 1000 -G \
  jekyll jekyll

#
# Remove development packages on minimal.
# And on pages.  Gems are unsupported.
#


RUN mkdir -p $JEKYLL_VAR_DIR
RUN mkdir -p $JEKYLL_DATA_DIR
RUN chown -R jekyll:jekyll $JEKYLL_DATA_DIR
RUN chown -R jekyll:jekyll $JEKYLL_VAR_DIR
RUN chown -R jekyll:jekyll $BUNDLE_HOME
RUN rm -rf /home/jekyll/.gem
RUN rm -rf $BUNDLE_HOME/cache
RUN rm -rf $GEM_HOME/cache
RUN rm -rf /root/.gem

# Work around rubygems/rubygems#3572
RUN mkdir -p /usr/gem/cache/bundle
RUN chown -R jekyll:jekyll \
  /usr/gem

CMD ["jekyll", "--help"]
ENTRYPOINT ["/usr/jekyll/bin/entrypoint"]
WORKDIR /srv/jekyll
VOLUME  /srv/jekyll
EXPOSE 35729
EXPOSE 4000

```


### Building the image

```docker build -t myjekyll .```

### Create an application

There are two ways, new project form scratch or fork the repository (this has been detailed in [here](https://chirpy.cotes.info/posts/getting-started/)).

1) brand new from scratch, this would be the case if you don't want to use this theme.

```cd /pathofdev```

```mkdir root```

```docker run --rm --volume="$PWD/root:/srv/jekyll" -it myjekyll jekyll new .```

2) clone the [chripy repository](https://github.com/cotes2020/jekyll-theme-chirpy) and change the name of the directory to ```root``` (optional).

Follow the guide, by running ```tools/init.sh``` additionally remove ```.git``` folder and start to hack. 

### Running the development server

When you are satisfy with the customization or just wanna see if it was working. Create a new file ```docker-compose.yml``` at the same level as ```root``` (inside the ```pathofdev```) with this content

```yml
version: '3.8'

services: 
  web_dev:
    image: myjekyll
    container_name: web_dev
    volumes: 
      - $PWD/root:/srv/jekyll
      - $PWD/root/vendor/bundle:/usr/local/bundle
    ports: 
      - 4000:4000
    command: jekyll serve --watch --incremental


```

Your path tree should look like this

```
.
├── docker-compose.yml
└── root/
```

to run, just ```docker-compose up``` and navigate to [localhost](http://0.0.0.0:4000)
    
## Deploy and tips

### Deployment on Github

The idea here is to follow the guide from [Chirpy](https://chirpy.cotes.info), we proceed the same procedure as the fork method. It uses [Github Flow](https://guides.github.com/introduction/flow/), for an automatic build to a new (existing) branch **gh-pages**. One mistake I did was to not read the file ```.github/workflow/pages-deploy.yml```, in fact the automatic build is executed from a new push master while my branch name was main, if you did the same just change the 5th line 

```yml
name: 'Automatic build'
on:
  push:
    branches:
      - master
```

to 

```yml
name: 'Automatic build'
on:
  push:
    branches:
      - main
``` 

Now git add, commit, push and wait for the change in your settings by pointing the page to the new branch. If you need to add a change always push it on main branch

### Deployment not on Github 
The project is for build.

### Tips

1.  Some modifications (*_config.yml, date format, etc ...*) might not appear on your *jekyll serve*, you might need to rebuild the project, to do so open a new terminal in ```pathtodev/``` and run ```docker-compose exec web_dev jekyll build``` 

2.  Let me know in the comments section if you encounter any problem.








