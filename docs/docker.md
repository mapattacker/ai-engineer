# Docker

Ah... the new world of microservices. Docker is the company that stands at the forefront of modular applications, also known as microservices, where each of them can & usually communicate via APIs. This "container" service is wildly popular, because it is OS-agnostic & resource light, with the only dependency is that docker needs to be installed.

Some facts:

 * All Docker images are linux, usually either alpine or debian
 * Docker is not the only container service available, but the most widely used

## Basics

There are various `nouns` that are important in the Docker world.

 * __Image__: Installed version of an application
 * __Container__: Launched instance of an application from an image
 * __Container Registry__: a hosting platform to store your images. E.g. Docker Container Registry (DCR), AWS Elastic Container Registry (ECR)
 * __Dockerfile__: A file containing instructions to build an image

Also, these are the various `verbs` that are important in the Docker world.

 * __build__: Copy & install necessary packages & scripts to create an image
 * __run__: Launch a container from an image
 * __rm__ or __rmi__: remove a container or image respectively
 * __prune__: clear obsolete images, containers or network


## Dockerfile

The `Dockerfile` is the essence of Docker, where it contains instructions on how to build an image. There are four main instructions:

 * `FROM`: base image to build on, pulled from Docker Container Registry
 * `RUN`: install dependencies
 * `COPY`: copy files & scripts 
 * `CMD` or `ENTRYPOINT`: command to launch the application

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

## Common Commands

### Build

The basic build command is quite straight forward. However, if we have various docker builds in a repo, it is best to name it with and extension representing the function, e.g. `Dockerfile.cpu`. For that, we will need to direct Docker to this specific file name.

```bash
sudo docker build -t <imagename> .
sudo docker build -t <imagename> . -f Dockerfile.cpu
```

### Run

For AI microservice in Docker there are four main run commands to launch the container.
 
| Cmd | Desc |
|-|-|
|  `-d` |  detached mode |
|  `-p 5000:5000` |  Map the Flask port from container to the outside world |
|  `--log-opt max-size=5m --log-opt max-file=5` |  limit the logs stored by Docker, by default it is unlimited |
|  `--restart always` |  in case the server crash & restarts, the container will also restart |
|  `--name <containername>` |  as a rule of thumb, always name the image & container |

```bash
sudo docker run -d -p 5000:5000 --log-opt max-size=5m --log-opt max-file=5 --restart always --name <containername> <imagename>
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