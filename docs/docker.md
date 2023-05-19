# Docker

Ah... the new world of microservices. Docker is the company that stands at the forefront of modular applications, also known as microservices, where each of them can & usually communicate via APIs. This "container" service is wildly popular, because it is OS-agnostic & resource light, with the only dependency is that docker needs to be installed.

Some facts:

 * All Docker images are linux, usually either alpine or debian
 * alpine installations will need `apk`, while debian uses `apt`
 * Docker is not the only container service available, but the most widely used

## Installation

You may visit Docker's official website for installation in various OS and architectures. Below is for ubuntu.

```bash
# Uninstall old versions
sudo apt-get remove docker docker-engine docker.io containerd runc

sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker’s official GPG key:
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# set up the repository:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install latest Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

## Basics

There are various `nouns` that are important in the Docker world.

| Noun | Desc |
|-|-|
| Image | Installed version of an application |
| Container |  Launched instance of an application from an image |
| Container Registry |  a hosting platform to store your images. E.g. Docker Container Registry (DCR), AWS Elastic Container Registry (ECR) |
| Dockerfile |  A file containing instructions to build an image |

Also, these are the various `verbs` that are important in the Docker world.

| Verb | Desc |
|-|-|
| build |  Copy & install necessary packages & scripts to create an image |
| run |  Launch a container from an image |
| rm or rmi |  remove a container or image respectively |
| prune |  clear obsolete images, containers or network |

## Dockerfile

The `Dockerfile` is the essence of Docker, where it contains instructions on how to build an image. There are four main instructions:

| CMD | Desc |
|-|-|
|  `FROM` |  base image to build on, pulled from Docker Container Registry |
|  `RUN` |  install dependencies |
|  `COPY` |  copy files & scripts  |
|  `CMD` or `ENTRYPOINT` |  command to launch the application |

Each line in the Dockerfile is cached in memory by sequence, so that a rebuilding of image will not need to start from the beginning but the last line where there are changes. Therefore it is always important to always (copy and) install the dependencies first before copying over the rest of the source codes, as shown below.

```Dockerfile
FROM python:3.7-slim

RUN apt-get update && apt-get -y upgrade \
    && apt-get install ffmpeg libsm6 libxext6 -y \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements-serve.txt .
RUN pip3 install --no-cache-dir --upgrade pip~=22.3.1 \
    && pip install --no-cache-dir -r requirements-serve.txt

COPY . /app

ENTRYPOINT [ "python", "-u", "serve_http.py" ]
```

### Intel-GPU

If the host has Nvidia GPU, we should make use of it so that the inference time is much faster; x10 faster for this example. We will need to choose a base image that has CUDA & CUDNN installed so that GPU can be utilised.

```Dockerfile
FROM pytorch/pytorch:1.5.1-cuda10.1-cudnn7-devel

RUN apt-get update && apt-get -y upgrade \
    && apt-get install ffmpeg libsm6 libxext6 -y \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements-serve.txt .
RUN pip3 install --no-cache-dir --upgrade pip~=22.3.1 \
    && pip install --no-cache-dir -r requirements-serve.txt

COPY . /app

ENTRYPOINT [ "python", "-u", "serve_http.py" ]
```

The run command is:

```bash
sudo docker run --gpus all --ipc=host -d -p 5000:5000 --name  <containername> <imagename>
```

### ARM-GPU

The ARM architecture requires a little more effort; the below installation is for Nvidia Jetson Series Kit.

```Dockerfile
# From https://ngc.nvidia.com/catalog/containers/nvidia:l4t-pytorch 
# it contains Pytorch v1.5 and torchvision v0.6.0
FROM nvcr.io/nvidia/l4t-pytorch:r32.4.3-pth1.6-py3

ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get -y update && apt-get -y upgrade
RUN apt-get install -y wget python3-setuptools python3-pip libfreetype6-dev

# Install OpenCV; from https://github.com/JetsonHacksNano/buildOpenCV
RUN apt-get -y install qt5-default
COPY ./build/OpenCV-4.1.1-dirty-aarch64.sh .
RUN ./OpenCV-4.1.1-dirty-aarch64.sh --prefix=/usr/local/ --skip-license && ldconfig

# Install other Python libraries required by Module
COPY requirements.txt .
RUN pip3 install -r requirements-serve.txt

# Copy Python source codes
COPY . .

RUN apt-get clean && rm -rf /var/lib/apt/lists/* && rm OpenCV-4.1.1-dirty-aarch64.sh

ENTRYPOINT [ "python3", "-u", "serve_http.py" ]
```

The run command is:

```bash
sudo docker run -d -p 5000:5000 --runtime nvidia --name <containername> <imagename>
```

## Common Commands

### Build

The basic build command is quite straight forward. However, if we have various docker builds in a repo, it is best to name it with and extension representing the function, e.g. `Dockerfile.cpu`. For that, we will need to direct Docker to this specific file name.

```bash
sudo docker build -t <imagename> .
sudo docker build -t <imagename> . -f Dockerfile.cpu
```

### Run

For an AI microservice in Docker, there are five main run commands to launch the container.
 
| Cmd | Desc |
|-|-|
|  `-d` |  detached mode |
|  `-p 5000:5000` |  OS system-port:container-port |
|  `--log-opt max-size=5m --log-opt max-file=5` |  limit the logs stored by Docker, by default it is unlimited |
|  `--restart always` |  in case the server crash & restarts, the container will also restart |
|  `--name <containername>` |  as a rule of thumb, always name the image & container |

The full command is as such.

```bash
sudo docker run -d -p 5000:5000 --log-opt max-size=5m --log-opt max-file=5 --restart always --name <containername> <imagename>
```

We can also stop, start or restart the container if required.

```bash
sudo docker stop <container-name/id>
sudo docker start <container-name/id>
sudo docker restart <container-name/id>
```

### Check Status

This checks the statuses of each running container, or all containers.

```bash
sudo docker ps
sudo docker ps -a
```

Shows resource usage for each running container.

```bash
sudo docker stats
```

### Clean

To manually remove a container or image, used the below commands. Add `-f` to forcefully remove if the container is still running.

```bash
sudo docker rm <container-id/name>
sudo docker rmi <image-id/name>
```

Each iteration of rebuilding and relaunching of containers will create a lot of intermediate images, and stopped containers. These can occupy a lot of space, so using `prune` can remove all these obsolete items at one go.

```bash
sudo docker image prune
sudo docker container prune
sudo docker volume prune
sudo docker network prune
sudo docker system prune

# force, without the need to reply to yes|no
sudo docker system prune -f
```

### Debug

We can check the console logs of the Flask app by opening up docker logs in live mode using `-f`. The logs are stored in `/var/lib/docker/containers/[container-id]/[container-id]-json`.

```bash
sudo docker logs <container_name> -f
sudo docker logs <container_name> -f --tail 20
```

At times, we might need to check what is inside the container.

```bash
sudo docker exec -it <container name/id> bash
```

We can also run specific commands by defining a new entrypoint. the `--rm` will remove the container after command is completed.

```bash
docker run --rm --entrypoint=/bin/sh <image_name> -c "ls -la"
```

Sometimes, we might want to copy file(s) out to examine them. We can do so with the following commands.

```bash
docker cp <container_id>:/path/to/file /path/on/host
docker cp <container_id>:/path/to/directory /path/on/host

```

### Storage

By design, docker containers do not store persistent data. Thus, any data written in containers will not be available once the container is removed. There are three options to persist data, bind mount, volume, or tmpfs mount.

More from [4sysops](https://4sysops.com/archives/introduction-to-docker-bind-mounts-and-volumes/), and [docker](https://docs.docker.com/storage/)

Bind mount provides a direct connection of a local folder's file system to the container's system. We can easily swap a file within the local folder and it will be immediately reflected within the container. This is helpful when we need to change a new model after training.

```bash
docker run \
    -p 5000:5000 \
    --mount type=bind,source=/Users/jake/Desktop/data,target=/data,readonly \
    --name <containername> <imagename>
```

Volume mount is the preferred mechanism for updating a file from a container into the file system. The volume folder is stored in the local filesystem managed by docker in `/var/lib/docker/volumes`.

```bash
docker run \
    -p 5000:5000 \
    --mount type=volume,source=<volumename>,target=/data \
    --name <containername> <imagename>
```

| Cmd | Desc |
|-|-|
| `docker volume inspect volume_name` | inspect the volume; view mounted directory in docker |
| `docker volume ls` | view all volumes in docker |
| `docker rm volume` | delete volume |


### Ports

Arguably one of the more confusing commands. Docker port commands involve two types

 - Expose: opening a port in a container to communicate with other containers
 - Bind: linking a port in a container to communicate with the host

Each container is a separate environment on its own, so you can have multiple containers having the same port.

The same cannot be said for the latter. Since you can only bind to one specific host port, else you will receive an error of duplicate ports.


### Network

For different containers to communicate to each other, we need to set a fixed network as the IP will change with each instance. We can do so by creating a network and setting it in when the container is launched.

```bash
docker network create <name>
docker run -d --network <name> --name=<containera> <imagea>
docker run -d --network <name> --name=<containerb> <imageb>

# list networks
docker network ls
```

Alternatively, we can launch the containers together using docker-compose, and they will automatically be in the same network.

For sending REST-APIs between docker containers in the same network, the IP address will be `http://host.docker.internal` for Mac & Windows, and `http://172.17.0.1` for Linux.

## Security Patches

We can improve our container security by running the os updates. More information is described [here](https://pythonspeed.com/articles/security-updates-in-docker/).

```Dockerfile
FROM debian:buster
RUN apt-get update && apt-get -y upgrade
```

## Optimize Image Size

There are various ways to reduce the image size being built.

### Slim Build

For python base image, we have many options to choose the python various and build type. As a rule-of-thumb, we can use the slim build as defined below. It has a lot less libraries, and can reduce the image by more than 500Mb. Most of the time the container can run well, though some libraries like opencv can only work with the full image.

Note that the alpine build is the smallest, but more often then not, you will find that a lot of python library dependencies does not work well here. 

```Dockerfile
FROM python:3.8-slim
```

### Disable Cache

By default, all the python libraries installed are cached. This refers to installation files(.whl, etc) or source files (.tar.gz, etc) to avoid re-download when not expired. However, this is usually not required when building your image.

```Dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

Below shows the image size being reduced. The bottom most is the full python image, with cache. The middle is the python slim image, with cache. The top most is the python slim image with no cache.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/dockersize-optimization.PNG?raw=true" />
  <figcaption></a></figcaption>
</figure>

## Multi-Stage

We can split the Dockerfile building by defining various stages. This is very useful when we want to build to a certain stage; for e.g., for testing we only need the dependencies, and we can use the command `docker build --target <stage-name> -t <image-name> .`


```Dockerfile
FROM python:3.9-slim AS build

# os patches
RUN apt-get update && apt-get -y upgrade \
    # for gh-actions path-filter
    && apt-get -y install git \
    && rm -rf /var/lib/apt/lists/*

FROM build AS install

# install python requirements
COPY requirements-test.txt .
RUN pip3 install --no-cache-dir --upgrade pip \
    && pip3 install --no-cache-dir -r requirements-test.txt
```

A second benefit is to reduce the image size, and improving security by ditching unwanted programs or libraries from former stages.


## Linting

Linters are very useful to ensure that your image build is well optimized. One of the more popular linters is called [hadolint](https://github.com/hadolint/hadolint). Below is an example output after scanning using the command `hadolint <Dockerfile-name>`.

```bash
Dockerfile:4 DL3009 info: Delete the apt-get lists after installing something
Dockerfile:5 DL3008 warning: Pin versions in apt get install. Instead of `apt-get install <package>` use `apt-get install <package>=<version>`
Dockerfile:5 DL3015 info: Avoid additional packages by specifying `--no-install-recommends`
Dockerfile:5 DL3059 info: Multiple consecutive `RUN` instructions. Consider consolidation.
Dockerfile:6 DL3059 info: Multiple consecutive `RUN` instructions. Consider consolidation.
Dockerfile:6 DL3015 info: Avoid additional packages by specifying `--no-install-recommends`
Dockerfile:6 DL3008 warning: Pin versions in apt get install. Instead of `apt-get install <package>` use `apt-get install <package>=<version>`
Dockerfile:9 DL3045 warning: `COPY` to a relative destination without `WORKDIR` set.
Dockerfile:10 DL3013 warning: Pin versions in pip. Instead of `pip install <package>` use `pip install <package>==<version>` or `pip install --requirement <requirements file>`
Dockerfile:10 DL3042 warning: Avoid use of cache directory with pip. Use `pip install --no-cache-dir <package>`
Dockerfile:11 DL3059 info: Multiple consecutive `RUN` instructions. Consider consolidation.
Dockerfile:18 DL3009 info: Delete the apt-get lists after installing something
```

## Buildx

Docker introduced a new cli feature called `buildx` that makes it possible and  easy to build and publish Docker images that work on multiple CPU architectures. This is very important due to the prominence of Apple M1 (ARM64), where you need to build to x64 (amd64); as well as build ARM images to leverage cheaper ARM cloud instances (can be up to 30% cheaper than x64).

You can see the various supported architectures below.

```bash
docker buildx ls
# linux/amd64, linux/riscv64, linux/ppc64le, linux/s390x, 
# linux/386, linux/arm64/v8, linux/arm/v7, linux/arm/v6
```

To do that, we can use the `buildx build --platform linux/<architecture>` command. For ARM, we can omit the version by using `linux/arm64`.

```bash
docker buildx build --platform linux/amd64 -t sassy19a/dummy-flask .
```

In `docker-compose` this can be done by specifying the `platform` key. 

```yaml
version: '2.4'

services:
  testbuild:
    build: .../testbuild
    image: testbuild
    platform: linux/arm64/v8
```

## Docker-Compose

When we need to manage multiple containers, it will be easier to set the build & run configurations using Docker-Compose, in a ymal file called `docker-compose.yml`. We need to install it first using `sudo apt  install docker-compose`.

The official Docker blog [post](https://www.docker.com/blog/containerized-python-development-part-2/) gives a good introduction on this. 

```yaml
version: "3.5"
services:
    facedetection:
        build: 
            context: ./project
            dockerfile: Dockerfile-api
        # if Dockerfile name is not changed, can just use below
        # build: ./project
        container_name: facedetection
        ports:
            - 5001:5000
        logging:
            options:
                max-size: "10m"
                max-file: "5"
        deploy:
          resources:
            limits:
              cpus: '0.001'
              memory: 50M
        volumes:
            - type: bind
              source: /Users/jake/Desktop/source
              target: /model
        restart: unless-stopped
    maskdetection:
        build: ./maskdetection
        container_name: maskdetection
        ports:
            - 5001:5000
        logging:
            options:
                max-size: "10m"
                max-file: "5"
        environment:
            - 'api_url={"asc":"http://172.17.0.1:5001/product-association",
                        "sml":"http://172.17.0.1:5002/product-similarity",
                        "trd":"http://172.17.0.1:5003/product-trending",
                        "psn":"http://172.17.0.1:5004/product-personalised"}'
            - 'resultSize=10'
        restart: unless-stopped
```

We can simplify repeated code using anchors, but each anchor must have the prefix of `x-`.


```yaml
version: "3.5"

x-common: &common
  logging:
      options:
          max-size: "10m"
          max-file: "5"
  restart: unless-stopped

services:
    facedetection:
        build: 
            context: ./project
            dockerfile: Dockerfile-api
        # if Dockerfile name is not changed, can just use below
        # build: ./project
        container_name: facedetection
        ports:
            - 5001:5000
        deploy:
          resources:
            limits:
              cpus: '0.001'
              memory: 50M
        volumes:
            - type: bind
              source: /Users/jake/Desktop/source
              target: /model
        <<: *common
    maskdetection:
        build: ./maskdetection
        container_name: maskdetection
        ports:
            - 5001:5000
        environment:
            - 'api_url={"asc":"http://172.17.0.1:5001/product-association",
                        "sml":"http://172.17.0.1:5002/product-similarity",
                        "trd":"http://172.17.0.1:5003/product-trending",
                        "psn":"http://172.17.0.1:5004/product-personalised"}'
            - 'resultSize=10'
        <<: *common
```



The commands follows docker commands closely, with some of the more important ones as follows.

| Cmd | Desc |
|-|-|
| `docker-compose build` | build images |
| `docker-compose pull` | pull image from a registry; must input key `image` |
| `docker-compose up` | run containers |
| `docker-compose up servicename` | run specific containers |
| `docker-compose up -d` | run containers in detached mode |
| `docker-compose ps` | view containers' statuses |
| `docker-compose stop` | stop containers |
| `docker-compose start` | start containers |
| `docker-compose down` | remove all containers |
| `docker-compose down --rmi all --volumes` | remove all containers, images, and volumes |

## Docker Dashboard

Docker in Windows & Mac comes by default a docker dashboard, which gives you a easy GUI to see and manage your images and containers, rather than within the commandline. 

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/docker-dashboard.png?raw=true" />
  <figcaption></a></figcaption>
</figure>

However, this is lacking in Linux. A great free alternative (with more features) is [Portainer](https://www.portainer.io/). We just need to launch it using docker with the following commands, and the web-based GUI will be accessible via `localhost:9000`. After creating a user account, the rest of it is pretty intuitive.

```bash
sudo docker volume create portainer_data
sudo docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
```

The home page, showing the local docker connection, and some summary statistics. 

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/portainer1.png?raw=true" />
  <figcaption></a></figcaption>
</figure>

On clicking that, an overview of the local docker is shown.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/portainer2.png?raw=true" />
  <figcaption></a></figcaption>
</figure>

Entering the container panel, we have various options to control our containers.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/portainer3.png?raw=true" />
  <figcaption></a></figcaption>
</figure>

We can even go into the container, by clicking the console link.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/portainer4.png?raw=true" />
  <figcaption></a></figcaption>
</figure>


## Blue-Green Deployment

Normal docker or docker-compose deployments requires downtime during redeployment, using `docker stop/start <container>` or `docker restart <container>` commands.

To achieve a zero-downtime deployment, we can use a neat trick with nginx reload functionality, together with docker-compose scale bumping of versions. The details of this blue-green deployment are well explained in these two articles [[1](https://www.tines.com/blog/simple-zero-downtime-deploys-with-nginx-and-docker-compose), [2](https://dev.to/wassimbj/deploy-your-docker-containers-with-zero-downtime-o3f)].


This is the `docker-compose` file which contains the api service, as well as nginx. They are tied to the same network, so that they can communicate with each other. Note that the network is not really required to explictly defined, as by default if they are launched together in a compose file, they will be within the same network.

An essential part of this file is that we must not define the container name of the api service.

```yaml
version: "3.2"
services:
  api:
    build:
      context: .
    networks:
      - bluegreen
  nginx:
    image: nginx
    container_name: nginx
    volumes:
      - ./nginx-conf.d:/etc/nginx/conf.d
    ports:
      - 8080:80
    networks:
      - bluegreen
    depends_on:
      - api

networks:
  bluegreen:
   name: bluegreen
```

This is the nginx config file which is stored at the root location of `nginx-conf.d/bluegreen.conf`. The most important thing of this file is that the proxy link to the api service is using the service name itself.

```
server {
    listen 80;
    location / {
            proxy_pass http://api:5000;
    }
}

```

This is the bash script for redeployment. In essence, it will first build/pull the first image into docker. Then, it will run another new container of the api service, with the service name being bumped to a new version.

Nginx will then reload to start routing to the new container. After the old container is destroyed, nginx will reload again to stop routing to the old container.

```bash
# define container service to redeploy
service_name=api
service_port=5000


# either build new image or pull image
docker-compose build $service_name
# docker-compose pull $service_name


reload_nginx() {  
  docker exec nginx /usr/sbin/nginx -s reload  
}

zero_downtime_deploy() {  
  
  old_container_id=$(docker ps -f name=$service_name -q | tail -n1)

  # bring a new container online, running new code  
  # (nginx continues routing to the old container only)  
  docker-compose up -d --no-deps --scale $service_name=2 --no-recreate $service_name

  # wait for new container to be available  
  new_container_id=$(docker ps -f name=$service_name -q | head -n1)

  # start routing requests to the new container (as well as the old)  
  reload_nginx

  # take the old container offline  
  docker stop $old_container_id
  docker rm $old_container_id

  docker-compose up -d --no-deps --scale $service_name=1 --no-recreate $service_name

  # stop routing requests to the old container  
  reload_nginx  
}

zero_downtime_deploy
```

Quite a ingenious way to achieve a zero-downtime deployment I must say.