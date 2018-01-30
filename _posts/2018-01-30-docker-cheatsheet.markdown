---
layout: post
title:  docker for eight-year-olds
date:   2018-01-30 12:58:12 +0200
categories: cheatsheets
---
my try for Feynman's learning method: `https://medium.com/taking-note/learning-from-the-feynman-technique-5373014ad230`

## basics
- `container` is virtual machine - with mini OS and all things that it needs to run
- `containers` are based on `images`
- `image` is like a template that you can copy, extend or build your own, it can contain other images and a lot of extra stuff
- `containers` are identified by `container's friendly-name` or `container's-id` (found with `docker ps` command)
- `container's-id` looks as short or long hash string (both works)


## general things
- search image libraries: `docker search {name}` from `hub.docker.com`
- create container and run it from image: `docker run {options} {image-name}`
  - image 'redis' detached (not on command line): `docker run -d redis`
  - image 'redis' in latest version: `docker run -d redis:latest`
  - image 'redis' in specific version: `docker run -d redis:3.2`
  - image 'redis' detached in latest version with name: 'redisTitle' `docker run -d --name redisTitle redis:latest`
  - image 'redis' same as previously, but exposing port to outside of specific container: `docker run -d --name redisTitle -p 6379:6379 redis:latest`
    - `-p` can be given: 2 ports `{port-that-you-want-to-access-from-outside}:{port-of-containers-inside}`)
    - `-p` can be given: 1 port `{port-of-containers-inside}`, which will become accessible from outside from randomly allocated port (you can find this port by `docker ps` or `docker port {friendly-name or container-id} {needed-port-inside}`)


## look at things
- look at containers running in background: `docker ps`
- look at container's details: `docker inspect {friendly-name or container-id}`
- look at container's log files `docker inspect {friendly-name or container-id}`
- look at container's accessible ports from outside:
  - specific port: `docker port {friendly-name or container-id} {needed-port-inside}`
  - all available ports from outside container: `docker port {friendly-name or container-id}`
