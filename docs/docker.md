# Docker

Ah... the new world of microservices. Docker is the company that stands at the forefront of modular applications, also known as microservices, where each of them can & usually communicate via APIs. This "container" service is wildly popular, because it is OS-agnostic & resource light, with the only dependency is that docker needs to be installed.

Some facts:

 * All Docker images are linux, usually either alpine or debian
 * Docker is not the only container service available, but the most widely used

## Basics

There are various `nouns` that are important in the Docker world.

| Noun | Desc |
|-|-|
| Image |  Installed version of an application |
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
FROM python:3.7

RUN apt-get update
RUN apt-get install ffmpeg libsm6 libxext6 -y

COPY requirements-serve.txt .
RUN pip install --upgrade pip
RUN pip install -r requirements-serve.txt

COPY . .

ENTRYPOINT [ "python", "-u", "serve_http.py" ]
```

### Intel-GPU

If the host has Nvidia GPU, we should make use of it so that the inference time is much faster; x10 faster for this example. We will need to choose a base image that has CUDA & CUDNN installed so that GPU can be utilised.

```Dockerfile
FROM pytorch/pytorch:1.5.1-cuda10.1-cudnn7-devel

RUN apt-get update
RUN apt-get install ffmpeg libsm6 libxext6 -y

COPY requirements-serve.txt .
RUN pip install --upgrade pip
RUN pip install -r requirements-serve.txt

COPY . .

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
|  `-p 5000:5000` |  Map the Flask port from container to the outside world |
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
sudo docker system prune
```

### Debug

We can check the console logs of the Flask app by opening up docker logs in live mode using `-f`. The logs are stored in `/var/lib/docker/containers/[container-id]/[container-id]-json`.

```bash
sudo docker logs -f container_name
```

At times, we might need to check what is inside the container.

```bash
sudo docker exec -it <container name/id> bash
```

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

### Docker-Compose

When we need to manage multiple containers, it will be easier to set the build & run configurations using Docker-Compose, in a ymal file called `docker-compose.yml`.

The official Docker blog [post](https://www.docker.com/blog/containerized-python-development-part-2/) gives a good introduction on this. 

```yml
version: "3"
services:
    facedetection:
        build: ./facedetection
        container_name: facedetection
        ports:
            - 5001:5000
        restart: always
    maskdetection:
        build: ./maskdetection
        container_name: maskdetection
        ports:
            - 5001:5000
        restart: always
```

When we are ready, we use the commands `docker-compose build` & `docker-compose up -d` to build & launch the containers respectively.