---
title: Sagemath development environment
author: Rapoma
date: 2021-01-29 02:21:00 +0100
categories: [Post, Tutorial]
tags: [mathematics, sagemath, sharing, docker, arm64]
image: https://www.sagemath.org/pix/sage_logo_new_l_hc_edgy-nq8.png
---

## Intro
 
In this post, I share the steps I followed, by making a develpment enviroment for [sage](https://www.sagemath.org/). There are no particular requirements to be able to follow along. I would recommend you to explore the [documentation](https://doc.sagemath.org/) for installing sage from source.

[Sagemath](https://www.sagemath.org/) is a free open source, covering several discipline in Mathematics. It uses [python](https://www.python.org/) as an interpreter for several library written in (C, C++, etc) such as [PARI](http://pari.math.u-bordeaux.fr/), [GAP](https://www.gap-system.org/) and so much more.
Sage is one of the important software that student (undergraduate, graduate), researcher in Mathematics needed to know, especially in applied field. It has its limit but at least we have (should) some reference to our result, what I mean is that let assume in your research you have invented a fast algorithm for computing [Galois Group](https://encyclopediaofmath.org/index.php?title=Galois_group), since Sage will run forever on higher degree more than 6. As well, the last time I checked Pari could do up to degree 11. So that your algorithm should match those results. 


## Steps

The method I am using here should work on any platform. I use [docker](https://www.docker.com/) for the local development. You then need to have *docker* and *docker-compose* installed.

You can look at the docker hub the image ```sagemath/sagemath``` by just pulling the latest. Last time I checked, there is no image for arm64. I had to build the image from a ```Dockerfile```. Looking into its [repository](https://github.com/sagemath/sage/tree/develop/docker) and did some *massage* on their Dockerfile.

### Downloading the source

This step is optional, since I could have done it inside docker (maybe because my network was kinda of slow).

Go to the [download page](https://www.sagemath.org/download.html) and choose the closest mirror to your location.

For example in linux computer would be ```wget http://sage.mirror.garr.it/mirrors/sage/src/sage-9.2.tar.gz``` 


### Customize Dockerfile

```dockerfile
FROM ubuntu:20.04

ENV LC_ALL C.UTF-8
ENV LANG C.UTF-8
ENV SHELL /bin/bash

RUN apt-get update \
    && apt-get upgrade -y \
    && DEBIAN_FRONTEND=noninteractive TZ=Europe/Italy apt-get install -y bc binutils bzip2 ca-certificates cliquer curl \
                    eclib-tools fflas-ffpack flintqs g++ g++ gcc gcc gfan gfortran \
                    git glpk-utils gmp-ecm lcalc libatomic-ops-dev libboost-dev \
                    libbraiding-dev libbrial-dev libbrial-groebner-dev libbz2-dev \
                    libcdd-dev libcdd-tools libcliquer-dev libcurl4-openssl-dev \
                    libec-dev libecm-dev libffi-dev libflint-arb-dev libflint-dev \
                    libfreetype6-dev libgc-dev libgd-dev libgf2x-dev libgiac-dev \
                    libgivaro-dev libglpk-dev libgmp-dev libgsl-dev libiml-dev \
                    liblfunction-dev liblrcalc-dev liblzma-dev libm4rie-dev libmpc-dev \
                    libmpfi-dev libmpfr-dev libncurses5-dev libntl-dev libopenblas-dev \
                    libpari-dev libpcre3-dev libplanarity-dev libppl-dev libpython3-dev \
                    libreadline-dev librw-dev libsqlite3-dev libsuitesparse-dev \
                    libsymmetrica2-dev libz-dev libzmq3-dev libzn-poly-dev m4 make \
                    nauty palp pari-doc pari-elldata pari-galdata pari-galpol pari-gp2c \
                    pari-seadata patch perl pkg-config planarity ppl-dev python3 \
                    python3-dev python3-distutils r-base-dev r-cran-lattice sqlite3 \
                    sympow tachyon tar xcas xz-utils yasm \
                    dvipng texlive ffmpeg default-jdk pandoc libavdevice-dev \
    && apt-get autoremove \
    && apt-get autoclean \
    && rm -r /var/lib/apt/lists/*


ARG SAGE_ROOT=/home/sage/sage
RUN ln -s "$SAGE_ROOT/sage" /usr/bin/sage

ARG HOME=/home/sage
RUN mkdir -p /etc/sudoers.d
RUN adduser --quiet --shell /bin/bash --gecos "Sage user,101,," --disabled-password --home "$HOME" sage \
    && echo "sage ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/01-sage \
    && chmod 0440 /etc/sudoers.d/01-sage


USER sage
WORKDIR ${HOME}

COPY sage-9.2.tar.gz sage-9.2.tar.gz
RUN tar -xvf sage-9.2.tar.gz
RUN mv sage-9.2 sage
RUN cd sage && ./configure && make -j4
RUN rm -rf /home/sage/sage-9.2.tar.gz

RUN sage -pip install terminado "notebook>=5" "ipykernel>=4.6"
RUN sage -i gap_jupyter singular_jupyter pari_jupyter

ENV LD_PRELOAD /usr/lib/aarch64-linux-gnu/libgomp.so.1

COPY entrypoint.sh /usr/local/bin/sage-entrypoint

ENTRYPOINT [ "/usr/local/bin/sage-entrypoint" ]
CMD [ "bash" ]
```

💡 *notices:* the number of core used by make

Save this Dockerfile at the same level as the downloaded source ```sage-9.2.tar.gz```.

Next in the same directory create file ```entrypoint.sh``` and add the follwing 

```shell
#!/bin/bash
if [ x"$1" = x"sage-jupyter" ]; then
    # If "sage-jupyter" is given as a first argument, we start a jupyter notebook
    # with reasonable default parameters for running it inside a container.
    shift
    exec sage -n jupyter --no-browser --ip='0.0.0.0' --port=8888 "$@"
else
    exec sage -sh -c "$*"
fi
```
this is actually the original entrypoint from sagemath docker official images [here](https://github.com/sagemath/sage/blob/develop/docker/entrypoint.sh). To make sure, change the access permission of it to be executable by ```chmod +x entrypoint.sh```.

### Building the image

```docker build -t mysage .```

this will take a while, so be patient

### Test the image

For testing our image, we can start it by 
```docker run -it mysage sage```
you will be redirect in a sage interpreter, try to run some basic command.

![preview](/assets/img/posts/post2preview1.png)


## Tips

1. Why would I recommend using this even you are running on a platform that you can download the binary directly? because some of the package are not fully available using ```sage --package list```. For instance, I personally struggling to install [database_kohel](http://mirrors.mit.edu/sage/spkg/upstream/database_kohel/database_kohel-20160724.tar.gz).

2. The image is big, around 18GB. Well as I said if you pull the image from docker hub ```sagemath/sagemath```, you might have the same issue when checking ```sage --package list```, or even gave you a complain since its has removed the build directory. 

The image tagged ```dev``` gives you this tho but the problem is that it recompile every-time you run the container. 

3. If you use [jupyter](https://jupyter.org/), use the command ```sage-jupyter```, everything should go smoothly. here is a simple ```docker-compose.yml``` that can be helpful when working on notebook.

```yml
version: "3.8"

services: 
  sagemath:
    image: mysage
    container_name: sagemath
    volumes: 
      - $PWD/notebook:/home/sage/notebook
    ports: 
      - 8888:8888
    command: sage-jupyter
```

4. Happy hacking 😃, let me know in the comment if you encounter problem by following this steps.  