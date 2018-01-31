---
layout: post
title:  docker for eight-year-olds
date:   2018-01-30 12:58:12 +0200
categories: cheatsheets
---
my try for Feynman's learning method: `https://medium.com/taking-note/learning-from-the-feynman-technique-5373014ad230`

## basics
- `container` is virtual machine - it is isolated and can contain os or anything that it needs to run
- `containers` are based on `images`, which are based on other images + instructions
- `image` is like a template that you can copy, extend or build your own (images are created from Dockerfiles), it can contain other images and a lot of extra stuff
- `containers` are identified by `container's friendly-name` or `container's-id` (found with `docker ps` command)
  - `container's-id` looks as short or long hash string (both works to identify container)
- `container` is stateless (it has no persistent data by default), but data can be saved from creation to creation or accessed from multiple containers or changed using volumes (-v option when creating container) or you can make special Data container to also be persistent.

## naming guidelines
- all tools should prefix their keys with the reverse DNS notation of a domain controlled by the author. For example, `com.katacoda.some-label`
- if you're creating labels for CLI use, then they should follow the DNS notation making it easier for users to type
- keys should only consist of lower-cased alphanumeric characters, dots and dashes (for example,`[a-z0-9-.]`)

## general things
- you can execute commands on running containers with: `docker exec {friendly-name or container-id} {command}`
  - get to bash in running container: `docker exec -i -t {friendly-name or container-id} /bin/bash`
  - look at nginx containers setup: `docker exec nginx cat /etc/nginx/conf.d/default.conf`
- you can add option `-it` and interact with container, e.g. access it's bash `docker run -it ubuntu bash` - this will run `ubuntu` and then access it's bash shell
- you can copy things to container with `docker cp {file_to_copy_from_outside} {friendly-name or container-id}:{folder_to_copy_to_inside_container}/` e.g. `docker cp config.conf dataContainer:/config/`

## look at things
- search image libraries: `docker search {name}` from `hub.docker.com`
- look at containers running in background: `docker ps`
- look at all containers - running and not running `docker ps -a`
- look at container's details: `docker inspect {friendly-name or container-id}`
- look at some containers logs: `docker logs {friendly-name or container-id}`
- look at container's accessible ports from outside:
  - specific port: `docker port {friendly-name or container-id} {needed-port-inside}`
  - all available ports from outside container: `docker port {friendly-name or container-id}`
- look at all active networks `docker network ls`
- inspect specific network `docker network inspect {network's name}`
- you can see details of launched by docker-compose containers with `docker-compose ps`
- you can see logs from launched by docker-compose containers with `docker-compose logs`
- you can see containers stats (metrics) with `docker stats {friendly-name or container-id}` or just `docker stats`


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
  - you can add environmental variables with `-e`: `-e NODE_ENV=production`
- you can set up automatic restart policies for containers with additional option `--restart=on-failure:3` which will restart 3 times on failure before stopping or `--restart=always` if you want for container to allways try to restart if failed
- you can add labels for container in options:
  - single label with `-l` option e.g. `docker run -l user=12345 -d redis`
  - labels from label file with `--labelfile` option, e.g. `docker run --label-file=labels -d redis` (each new label in new line of text file)
  - you can see the labels of container with filtering json response from `inspect` e.g. `docker inspect -f "{\{json .Config.Labels }\}" redis`
  - you can filter containers with `--filter` using labels e.g. `docker ps --filter "label=user=scrapbook"`

## link containers or network them
- you can simply link containers (used for simple database setups):
  - initially create container you want to link to `docker run -d --name {friendly name} {base image name}` e.g. `docker run -d --name redis-server redis`
  - afterwards `docker run --link {\{name or id}:{alias} or {name or id}\} alpine` e.g. `docker run --link redis-server:redis alpine`
- you can link multiple containers also with `Docker Networks` which is usually the preferred way to do it - it is more elastic of creation/destruction of containers:
  - create network with `docker network create {network-name}` e.g. `docker network create backend-network`
  - run some container and add it to network: `docker run -d --name={new container name} --net={network-name} {base image}` e.g. `docker run -d --name=redis --net=backend-network redis` and container's name in network will be `redis.backend-network`
  - you can add containers to existing network with: `docker network connect --alias {containers-alias-in-network} {network's-name} {container-that-you-want-to-add}` e.g. `docker network connect --alias db frontend-network2 redis`
  - you can disconnect container from network: `docker network disconnect {networks name} containers name` e.g. `docker network disconnect frontend-network redis`

## Load Balance containers (concept)
- The Service Discovery pattern is where the application uses a third party system to identify the location of the target service. For example, if our application wanted to talk to a database, it would first ask an API what the IP address of the database is. This pattern allows you to quickly reconfigure and scale your architectures with improved fault tolerance than fixed locations.
Example:
  - create 'proxy container' and add docker sock to it - this will listen to port 80 requests and later delegate the job to some other containers: `docker run -d -p 80:80 -e DEFAULT_HOST=proxy.example -v /var/run/docker.sock:/tmp/docker.sock:ro --name nginx jwilder/nginx-proxy` (note: docker sock is with `:ro` flag or read-only)
  - create one container in the proxy which can receive tasks from proxy: `docker run -d -p 80 -e VIRTUAL_HOST=proxy.example katacoda/docker-http-server` (you can lounch multiple servers and the load will be balanced automatically)

## orchestrate container creation - Docker Compose
- this happens in `docker-compose.yml` file with default format:
```yml
container_name:
  property: value
    - or options
```
- you can launch containers with `up` command e.g. `docker-compose up -d` `-d` will detach the containers (similar as run)
- you can scale up or down amount of servers with `scale`, e.g. scale containers from image web to total number of 3: `docker-compose scale web=3`
- you can stop all docker-compose containers with `docker-compose stop`
- you can remove all docker-compose containers with `docker-compose stop`

#### docker-compose example
```yml
web:
  build: .
  links:
    - redis
  ports:
    - "3000"
    - "3001"
    - "8000"

redis:
  image: redis:alpine
  volumes:
    - /var/redis/data:/data
```

## create images
- you can create docker images by using Dckerfiles with command `docker build {options} {directory-with-Dockerfile}`, for example `docker build -t webserver-image:v1 .` will build image called `webserver-image` with tag `v1` in current (`.`) folder.
  - if you will user the command `docker build -t webserver-image:v1 .` with Dockerfile from first example and any index.html in current folder, you will be able to run container with `docker run -d -p 80:80 webserver-image:v1` - which will lunch single page webserver with index.html file from docker image's creation.
- to ensure persistence (data or databse) you can run thing called `Data Container` (also uses volumes):
  - you can create DataContainer images with `docker create -v {folder_for_persistance} --name {friendly name} {container_image - with data}` e.g. `docker create -v /config --name dataContainer busybox`
  - you can run contaners using this images volumes: `docker run --volumes-from {data-containers-base-name} {images-to-use-data-name}` e.g. `docker run --volumes-from dataContainer ubuntu`
  - you can backup all data from data container `docker export {data containers name} > {filename}.tar` e.g. `docker export dataContainer > dataContainer.tar`
  - you can import from file `docker import {filename}.tar` e.g. `docker import dataContainer.tar`
- you can add files to ignore when building image with `.dockerignore` - text file that you add names of files that need to be ignored (for example - all git directories)
- you can add labels for images in options:
  - single label with `-l` option
  - labels from label file with `--labelfile` option
  - you can see the labels from image with filtering json response e.g. `docker inspect -f "{\{json .ContainerConfig.Labels }\}" redis`
  - you can filter docker images with `--filter` e.g. `docker images --filter "label=vendor=Katacoda"`

### composing Dockerfile:
You can add instructions in **Shell** and **Exec** forms:
- **Shell** form: `{instruction} {command}` e.g. `RUN apt-get install python3`
- **Exec** form: ``{instruction} {array of strings - command and all paras as separate strings}` e.g.  `RUN ["apt-get", "install", "python3"] ` (this form is prefered for `CMD` and `ENTRYPOINT`)

You have these commands (+ some others) in Docekrfile:
- `FROM {image name and version}` this is the base image from which usually you create your image
- `RUN {command}` - to run command as from command prompt (usually for installing packages) - *tip*: if installing write `apt-get update` and `apt-get install` in single line or `install` will `update` also e.g. `RUN apt-get update && apt-get install -y git` will install `git` and run `update` only once (`-y` mens that installing will be already provided with "yes" answer for all prompts).
- `CMD {array of strings - command and all paras as separate strings}` - this list of commands will be executed only when you run container without specifying a command. If Docker container runs with a command, the default command will be ignored. Example of `CMD`: `nginx -g daemon off;` will be `CMD ["nginx", "-g", "daemon off;"]` Choose CMD if you need to provide a default command and/or arguments that can be overwritten from command line when docker container runs.
- `ENTRYPOINT {array of strings - command and all paras as separate strings}` instruction allows you to configure a container that will run as an executable. It looks similar to CMD, because it also allows you to specify a command with parameters. The difference is ENTRYPOINT command and parameters are not ignored when Docker container runs with command line parameters. Prefer ENTRYPOINT to CMD when building executable Docker image and you need a command always to be executed. Additionally use CMD if you need to provide extra default arguments that could be overwritten from command line when docker container run
- `COPY {source} {destination}` - copy files from the directory containing the Dockerfile to the container's image. This is extremely useful for source code and assets that you want to be deployed inside your container.
- `EXPOSE {port number}` - expose any port `xxx`, ports `xxx xxy xxz` or port range `xxx-xxy`
- `ENV {environmental variables}`
- `WORKDIR {directory}` - all future commands are executed from the directory relative to our application (inside the container that will be created from this image)

### Dockerfile examples:
- docker file below will create image which will be based on `nginx` with `alpine` os (Linux distro's image), then will copy everything (in this case there should be index.html in current container) from current location to `/usr/share/nginx/html` - this will make webserver from single page:
```Dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
```
- the effect of Dockerfile below is same as before. We should create image `docker build -t my-nginx-image:latest .` (with extra index.html file in Dockerfile's location folder) then we could `docker run -d -p 80:80 my-nginx-image:latest` to create container of this image - and we have web server with our index.html displayed:
```Dockerfile
FROM nginx:1.11-alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
- create container that will run node.js app in the same folder as Dockerfile is in (to create image: `docker build -t my-nodejs-app .`, and afterwards to create container from this image: `docker run -d --name my-running-app -p 3000:3000 my-nodejs-app`):
```Dockerfile
FROM node:7-alpine
RUN mkdir -p /src/app
WORKDIR /src/app
COPY package.json /src/app/package.json
RUN npm install
COPY . /src/app
EXPOSE 3000
CMD [ "npm", "start" ]
```

### composing Dockerfile.multi
- you can build multifile same as ordinary dockerfile, just add `-f` option for file e.g. `docker build -f Dockerfile.multi -t golang-app .`

#### Dockerfile.multi example:
```Dockerfile
# First Stage
FROM golang:1.6-alpine

RUN mkdir /app
ADD . /app/
WORKDIR /app
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Second Stage
FROM alpine
EXPOSE 80
CMD ["/app"]

# Copy from first stage
COPY --from=0 /app/main /app

```

## references:
- [interactive courses](https://katacoda.com/courses)
- [Docker RUN vs CMD vs ENTRYPOINT](http://goinbigdata.com/docker-run-vs-cmd-vs-entrypoint/)
