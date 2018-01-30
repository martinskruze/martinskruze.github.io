---
layout: post
title:  docker for eight-year-olds
date:   2018-01-30 12:58:12 +0200
categories: cheatsheets
---
my try for Feynman's learning method: `https://medium.com/taking-note/learning-from-the-feynman-technique-5373014ad230`

## basics
- `container` is virtual machine - it is isolated and can contain os or anything that it needs to run
- `containers` are based on `images`
- `image` is like a template that you can copy, extend or build your own (images are created from Dockerfiles), it can contain other images and a lot of extra stuff
- `containers` are identified by `container's friendly-name` or `container's-id` (found with `docker ps` command)
  - `container's-id` looks as short or long hash string (both works to identify container)
- `container` is stateless (it has no persistent data by default), but data can be saved from creation to creation or accessed from multiple containers or changed using volumes (-v option when creating container)

## general things
- you can execute commands on running containers with: `docker exec {friendly-name or container-id} {command}`
  - get to bash in running container: `docker exec -i -t {friendly-name or container-id} /bin/bash`
- you can add option `-it` and interact with container, e.g. access it's bash `docker run -it ubuntu bash` - this will run `ubuntu` and then access it's bash shell

## look at things
- search image libraries: `docker search {name}` from `hub.docker.com`
- look at containers running in background: `docker ps`
- look at container's details: `docker inspect {friendly-name or container-id}`
- look at container's log files `docker inspect {friendly-name or container-id}`
- look at container's accessible ports from outside:
  - specific port: `docker port {friendly-name or container-id} {needed-port-inside}`
  - all available ports from outside container: `docker port {friendly-name or container-id}`

## create containers
- create container and run it from image: `docker run {options} {image-name}`
  - image 'redis' detached (not on command line): `docker run -d redis`
  - image 'redis' in latest version: `docker run -d redis:latest`
  - image 'redis' in specific version: `docker run -d redis:3.2`
  - image 'redis' detached in latest version with name: 'redisTitle' `docker run -d --name redisTitle redis:latest`
  - image 'redis' same as previously, but exposing port to outside of specific container: `docker run -d --name redisTitle -p 6379:6379 redis:latest`
    - `-p` can be given: 2 ports `{port-that-you-want-to-access-from-outside}:{port-of-containers-inside}`)
    - `-p` can be given: 1 port `{port-of-containers-inside}`, which will become accessible from outside from randomly allocated port (you can find this port by `docker ps` or `docker port {friendly-name or container-id} {needed-port-inside}`)
  - image 'redis' with data that stays from creation to creation and can be accessed from different locations (volume): `docker run -d -v {folder-location-outside-container}:{folder-location-inside-of} redis`
    - you can use `$PWD` as placeholder for current directory on outside e.g.: `docker run -d -v "$PWD/data":/data redis`

## create images
- you can create docker images by using Dckerfiles with command `docker build {options} {directory-with-Dockerfile}`, for example `docker build -t webserver-image:v1 .` will build image called `webserver-image` with tag `v1` in current (`.`) folder.
  - if you will user the command `docker build -t webserver-image:v1 .` with Dockerfile from first example and any index.html in current folder, you will be able to run container with `docker run -d -p 80:80 webserver-image:v1` - which will lunch single page webserver with index.html file from docker image's creation.

#### composing Dockerfile:
- docker file below will create image which will be based on `nginx` with `alpine` os (Linux distro's image), then will copy everything (in this case there should be index.html in current container) from current location to `/usr/share/nginx/html` - this will make webserver from single page:
```Dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
```
- you can use version numbers for docker base images and you can use commmands:
  - `RUN {command}` - to run command as from command prompt, but you should add commands as array of strings containing `command`, `arguments tag`, `argument` (arguments tag and argument can be repeated multiple times), for example `nginx -g daemon off;` will be `CMD ["nginx", "-g", "daemon off;"]`
  - `COPY {source} {destination}` - copy files from the directory containing the Dockerfile to the container's image. This is extremely useful for source code and assets that you want to be deployed inside your container.
  - `EXPOSE {port number}` - expose any port `xxx`, ports `xxx xxy xxz` or port range `xxx-xxy`
- the effect of Dockerfile below is same as before. We should create image `docker build -t my-nginx-image:latest .` (with extra index.html file in Dockerfile's location folder) then we could `docker run -d -p 80:80 my-nginx-image:latest` to create container of this image - and we have web server with our index.html displayed:
```Dockerfile
FROM nginx:1.11-alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

# for later tests:
- ? can you lounch simple web server from volume html page?
