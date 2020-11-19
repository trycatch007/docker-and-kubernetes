# Notes

## Course Sections

### Dive Into Docker!

#### Install docker
- [How to Install Docker on Manjaro](https://manjaro.site/how-to-install-docker-on-manjaro-18-0/)

- Install `docker-compose` using Manjaro's Package Manager (search for it)

- Create an account on [Docker Hub](https://hub.docker.com/)

- Login your dev machine to the above account: 

  `$ docker login`

- Test your setup to see if it can download an image from Docker Hub and start a container to run it:

  `$ docker run hello-world`

#### What is a container?
  - related OS concepts (specific to Linux)
      - `Namespacing` - *isolate* resources per process
      - `Control Groups` (cgroups) - limit the *amount* of resources used per process
  - A `container` takes the docker `image` which includes a file system 
     snapshot of some programs configured in that image and the image's `startup command` and applies the above OS concepts to it: `Namespacing` and `Control Groups`
  - A `container` is a running process along with a subset of physical    
  resources (HD, RAM, CPU, Network, etc.) that are allocated on your computer
    to that process specifically.
  - Because `Namespacing` and `Control Groups` are Linux-only concepts, docker on Windows and MacOS historically used a Virtual Machine to run Linux.  The `containers` were than created inside this VM where a Linux Kernel was in charge of managing the `Namespacing` and `Control Groups`
  - It may no longer be true that docker is running VM in Windows/Mac (I haven't checked), see: https://www.hanselman.com/blog/docker-and-linux-containers-on-windows-with-or-without-hyperv-virtual-machines
  - To check what OS docker is running on the Server, run the following command and look under Server > Engine > OS/Arch
  
    `$ docker version`

  - On Linux docker applies these concepts natively without running a VM

### Manipulating Containers with the Docker Client

Basic create and run a container from an image
```
docker run <image name>
```

Above with default command override! (`echo hi there` or `ls` in the examples). 

Note that the `<command>` must be an executable that is included in the image file system (and copied over to the running container)
```
docker run <image name> <command>
docker run busybox echo hi there
docker run busybox ls
```

Some containers just start up, run an executable, and immediately shut down in which case you won't see them here (the examples ran so far).  You would need a process that keeps running a long time.  Here is an example you can run in a separate terminal:
```
docker run busybox ping google.com
```

Create container without running it right away. CLI prints out a container ID for you which can be used to later run it.
```
docker create <image name> <command>
docker create hello-world
docker create busybox ping google.com
```

Run a container that is already created on your machine (using the default startup command copied from the image).  

The `-a` tells the CLI to print out any output from the executable to your terminal (default is not to). This is a different default from `docker run` which by default prints out output to your terminal.

It is *not* possible to supply a different `<command>` to a container that is already created (as opposed to using `docker run`).  It gets *fixed* at creation time and is always used when the container is started thereafter. You *can* however supply it to `docker create` above.
```
docker start -a <container id>
```

You can also start a container without the `-a` in which case nothing will be printed out to your console.  The output of the executable is not lost however. You can retrieve all of it from the time the container process started with the subsequent `docker logs` command like so:
```
docker logs <container id>
```

To get an ID of the running container,
list all of the *running* containers that are currently on your machine. 
```
docker ps 
```

List *all* of the containers that we have ever created (not just those that are running, but those are also included).
```
docker ps --all
```

Container lifecycle (listed with the `ps --all` in STATUS column):
- Created
- Up
- Exited

You can also stop a container that is 'Up'.  This sends a hardware signal to the process running inside that container, SIGTERM, which tells the process to shut down on its own time (when it's ready, giving it a chance to do any cleanup like deleting temp files, closing sockets, releasing resources, etc).
Docker gives that 10 seconds and if it hasn't stopped at that point it automatically issues the `kill` command (see below).
```
docker stop <container id>
```

If that doesn't work, you can also kill it with SIGKILL (process doesn't have a chance to clean up and shuts down right away)
```
docker kill <container id>
```

At some point you will probably want to delete all of the `Exited` containers on your machine to free up some space.  This is the command, but also note that it will delete few other things including the cached images you have already downloaded (so next time you will need to re-download - not a big deal).
```
docker system prune
```

#### Multi-Command Containers

Without docker with `redis-server` started, you can use `redis-cli` to interact with the running server.

Let's try that with docker.

Start up the dockerized `redis-server`
```
docker run redis
```

So far so good, but how can we peek into it with dockerized `redis-cli`?
You can't use your local `redis-cli` to look into the container.

Somehow, we need to get inside the container that is already running the server, and execute `redis-cli` there.

To execute an additional command inside a running container:
```
docker exec -it <container id> <command>
```

`-it` allows us to provide/type input directly into the container (like in a terminal etc.) The `i` specifies that we want to attach the terminal to STDIN of the `<command>`.  The `t` makes the text show up pretty/formatted (it's doing a little more but that's the effect).  You could specify just `-i` to see what happens (no indents, no auto-completes, etc.)

For example, to get the `redis-cli` running inside our `redis-server` container:

```
docker ps --all # to get the ID and verify container is Up
docker exec -it <container id> redis-cli
```

If you find yourself trying to `exec` a command inside a running container and instead it just kicks you out and terminates, it could be because you forgot the `-it` argument (and thus had no way to type anything in the container so the process just terminated).

One *very common* usecase for `docker exec` is to get shell access, or terminal access to your running container, to run commands inside of your container.  Very powerful for debugging:
```
docker exec -it <container id> sh
```

#### Container Isolation

Between any two containers started from the same image, they do *NOT* automatically share their file system with each other.

You can verify that by starting two `busybox` constainers with `sh` and creating a file in one.  The other will not have that file.


### Building Custom Images Through Docker Server

#### Creating Docker Images

Creating docker images consists of creating a file named `Dockerfile` that typically:

- specifies an existing base image
- runs some commands to install additional programs
- specifies a command to run on container startup

Here is an example for one that builds a redis-server (save it in a file named `Dockerfile`):

```
FROM alpine

# Download and install a dependency
RUN apk add --update redis

# Tell the image what to do when it starts as a container
CMD ["redis-server"]

```

The capitalized words in above Dockerfile are called `instructions`:

- FROM - specifies the base image for this image (alpine)
- RUN - a command to run that prepares the image in some way (install stuff)
- CMD - sets the default startup program for the container and its arguments without actually executing it right away (not until the container gets started)


To build an image out of the Dockerfile, or to re-build it after changing Dockerfile:

```
docker build .
```

The `.` specifies the `build context` (another word for directory in docker lingo?)

`build context` is essentially the set of files and folders that belong to our project that we want to encapsulate or wrap inside our container.

Once build, you can run your image in a container:
```
docker run <image id>
```

A more convenient way is to `tag` and image when building it to give it a readable name that you can later use to run it:
```
docker build -t thomasmarkiewicz/redis:latest .
docker run thomasmarkiewicz/redis:latest
```

The convension for the `tag` is:

`<your docker id>/<project name>:<version>`



### Making Real Projects with Docker

### Docker Compose with Multiple Local Containers

### Creating a Production-Grade Workflow

### Continuous Integration and Deployment with AWS

### Building a Multi-Container Application

### "Dockerizing" Multiple Services

### A Continuous Integration Workflow for Multiple Images

### Multi-Container Deployments to AWS

### Onwards to Kubernetes!

### Maintaining Sets of Containers with Deployments

### A Multi-Container App with Kubernetes

### Handling Traffic with Ingress Controllers

### Kubernetes Production Deployment

### HTTPS Setup with Kubernetes

### Local Development with Skaffold


## Extras

### Docker CLI examples

Login to Docker Hub
```
docker login
```

Test local setup
```
docker run hello-world
```

Check what OS docker is running on the Server. On Windows and MacOS it is likely linux actually (running in a Virtual Machine).  To see, run the following command and look under Server > Engine > OS/Arch
```
docker version
```

List all running containers to get their ID
```
docker ps
```

List all containers to check if they are on your machine or their STATUS
```
docker ps --all
```

Delete stopped containers to free up space on your machine
```
docker system prune
```

Create and run a container from an image
```
docker run <image name>
```

Create and run a container from an image booting straight into `sh` *instead of* executing the default startup program:
```
docker run -it <image name> sh
docker run -it busybox sh
```

Create a container from an image without running it
```
docker create <image name> <optional startup command>
```

Start a container that is created but stopped
```
docker start -a <container id>
```

Stop a container (SIGTERM followed by SIGKILL after 10 seconds)
```
docker stop <container id>
```

Kill a container right away
```
docker kill <container id>
```

Get shell/terminal access to your running container:
```
docker exec -it <container id> sh
```

Or in general run another process in an already running container:
```
docker exec -it <container id> <command>
```

Build an image from a `Dockerfile`, or re-build it after changing Dockerfile:

```
docker build .
```

Tag an image when building it:
```
docker build -t thomasmarkiewicz/redis:latest .
```


### Kubernetes CLI examples

### Kubernetes on AZURE

### Kubernetes on Digital Ocean
