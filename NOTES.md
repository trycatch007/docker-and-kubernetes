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
docker system prune -a
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

The convention for the `tag` is:

`<your docker id>/<project name>:<version>`

The `:<version>` is really the tag.  If you don't explicitly specify it it defaults to `:latest`

One last interesting thing: it is possible to manually create an image out of a running container that you have setup/configured manually. It's not a good practice to do that, so don't, but it's interesting to know that you can:
```
docker commit -c 'CMD ["redis-server"]' <container id> <tag>
```

`'CMD ["redis-server"]'` is the image startup command you want to use.


### Making Real Projects with Docker

See code here: `./Course/04 - Making Real Projects with Docker/simpleweb`

`alpine` starting image doesn't include node etc.

`node` image also exists and already includes node and npm. It has several different tags, including `alpine` = as small as compact as possible. To use that particular image add this to Dockerfile:

```
FROM node:alpine
```


It seems that it's important to set the image WORKDIR right upfront for stuff to work these days.  This sets the default image directory () that commands assume when copying files etc.  In this case we're saying our app will live in `/usr/app` directory:
```
WORKDIR /usr/app
```

DISCREPANCY:

In the course lesson 44 Stephen lectures about missing package.json when building an image from the Dockerfile.  He shows docker throwing an error.  I don't see that error - docker build succeeds.

But the problem is still there.  I didn't get an error but examining the /usr/app directory in the image shows that package.json file IS there, but it's EMPTY!  It's not the same package.json file that is on my local filesystem! So I still need the step to copy it to the image.

To do that, COPY files from the `build context` to inside of the container.
- The first argument `./` is the path to folder to copy from on *your machine* relative to the `build context` (specified on `docker build <context>`)
- The second argument `./` is a place to copy stuff to inside *the container*
```
COPY ./ ./
```

So now `docker run` starts the container successfully, and starts running `node index.js` (the startup command) successfully.  We see a console log informing that it's "Listening on port 8080", BUT when pointing our local web browser to it, it doesn't work.

---

SIDE NOTE: 

On Windows Home with Docker Toolbox, you can't point your browser to `localhost:8080`.  That doesn't work apparently.  You need to figure out the actual IP address of the `docker-machine`.  To do that:
```
docker-machine ip
```

I believe this is a Windows Home only issue though.  I don't even have `docker-machine` command on my Manjaro Linux.
---

The running container has it's own ISOLATED ports that can receive traffic, but by default they are not connected. We have to setup an explicit *port mapping*.  It's very similar to port-forwarding on a home router.

NOTE that this limitation is ONLY for *incoming* requests.  The container *CAN* by default make any requests from the inside to the outside word!

To open up the ports, we do not change the Dockerfile.  Instead we need to provide extra arguments when *starting* the container up, the `-p INCOMING_LOCAL_HOST_PORT:PORT_INSIDE_CONTAINER`. For example, to forward port 8080 from our local machine network to port 8080 of the container, you would start the container like so
```
docker run -p 8080:8080 <image id>
```

Finally, now pointing your local browser to `localhost:8080` hits the server running inside the container.

NOTE that the local machine port and container ports DO NOT have to be identical. So you could do the following and point your local browser to `localhost:5000` instead.
```
docker run -p 5000:8080 <image id>
```

NEXT PROBLEM is with local development workflow.  We have the container started and can hit the server with the local web browser.  However, when we make a change to the server code (index.js) and save, the browser doesn't see that change!

This is because we take a one-time *snapshot* of our files when creating the image, with the COPY command, and the container doesn't automagically update when we make a change to our local filesystem.

There is a way to accomplish that with a configuration, but that will be covered in the sub-sequent section.  For now, you must rebuild your container after every local change.

There is a small problem with this approach though: everything after the COPY command in the Dockerfile must be rebuild from scratch because it changed and invalidated the image cache for the following commands.  This is inefficient.  

More specifically it has to `RUN npm install` and re-install ALL the dependencies from scratch even though we didn't change them - only changed the index.js file.  This could be a lengthy process! It's not ideal.

To fix that, we can split the COPY command in the Dockerfile into two separate steps.

Note that `RUN npm install` cares ONLY about the package.json file - it doesn't need the rest of the files.  So we can first just copy that file before, than execute the RUN command, and finally copy the rest of the files afterwards:
```
COPY ./package.json ./
RUN npm install
COPY ./ ./
```

This moves the image cache invalidation AFTER `RUN npm install` which is expensive to invalidate.  The image cache still gets invalidated, but afterwards and is not that bad anymore.  But if we make a change to package.json at that point it will still invalidate and re-install all packages.

The lesson here is to pay attention to the log output of `docker build` and look out for "Using cache".  Then maybe re-arranging the Dockerfile COMMANDS to optimize the image caching.

It's also a good idea to COPY the *bare minimum* for each successive step.

### Docker Compose with Multiple Local Containers

We're building here a new project that will consist of two containers:
1. A web app that displays some data
2. Redis

They are put in separate containers so that we can keep a SINGLE instance of Redis, but multiple web app containers to scale as necessary, each connecting to the same instance of Redis.

The code for this project lives at: `./Course/05 - Docker Compose with Multiple Local Containers`

When the node app first starts up, it is trying to connect to the Redis server.  But each, the node app and the Redis server, run in their own isolated containers and cannot by default talk to each other.

We have two options:
- use `docker` CLI to setup a network between the containers
- use `docker-compose`

Setting up the network with `docker` is a real pain to do.
It involves running a handful of commands each time we start up.
It could be scripted, but no one ever goes with this approach.

Using `docker-compose` is much easier.  
It's a separate CLI (that gets installed with `docker` so you should already have it?)

Advantages:

- `docker-compose` really exists to prevent you from writing a lot of separate commands with the docker CLI and providing all the details as program arguments (specifying ports, etc)

- used to start up multiple Docker containers at the same time with some form of networking

- automates some of the long-winded arguments we were passing to `docker run`


To make use of docker-compose, we essentially take the same commands we were running before with the `docker` CLI, but we're going to encode these commands in a configuration file:

`docker-compose.yml`

That file instructs how to create one or more docker containers.
Each container can be build from either:
- an image (including those stored in Docker Hub)
- a Dockerfile on your machine

In addition, it has a section for mapping network ports.

In docker-compose terminology, `services` word essentially boils down to meaning a `container`. It's more like a *type* of container. In the yml file we setup `services` to be docker `containers`.

Here is an example for the project we're working with:
```
version: '3'
services:
  redis-server:
    image: 'redis'
  node-app:
    build: .
    ports:
      - "4001:8081"
```

Notice that we didn't have to configure a "network".  Just by specifying those two "services" in the same yml file, `docker-compose` is going to automatically create both of these containers on essentially the same network and they can thus communicate with each other in any way that they please.  Basically they share the ports on the same network.

All containers within that same network can communicate with each other and we don't have to do any additional configuration to make that happen.
We ONLY have to punch in `port` holes to services that we want to make incoming requests to from *OUTSIDE* of that virtual network they live in. The node app for example.

But how does one service address servers running in another container? Typically outside of `docker-compose` world the host would be some sort of URL with an IP address or domain name.  Services in the `docker-compose` world however MUST address each other by the *name* assigned to them in the `docker-compose.yml` file.

For example, when specifying a `host` configuration for the redis server, instead of:

`host: 'https://my-redis-server.com'`

we must use the name of the service that is running redis, in our case:

`host: 'redis-server'`

node.js, express, etc. have no idea what 'redis-server' means.  Redis just takes it and assumes it will work as any other URL. Docker performs the translation magic when it sees Redis attempting to connect to 'redis-server' and forwards it to the appropriate service at runtime.

Once we have the `docker-compose.yml` configured, to start the containers specified:
```
docker-compose up
```

The above doesn't automatically re-build from Dockerfile however. To do that we need an addition `--build` argument:
```
docker-compose up --build
```

`docker-compose` looks for `docker-compose.yml` file in the current directory.



How to deal with containers that crash or hang?

- restart it automatically

To do that, we can specify a "Restart Policy" inside of our `docker-compose.yml` file. There are four different policies we can use:

- no
- always
- on-failure
- unless-stopped

For example,

```
services:
  node-app:
    restart: always
```

### Creating a Production-Grade Workflow
[skipped]

### Continuous Integration and Deployment with AWS
[skipped]

### Building a Multi-Container Application
[skipped]

### "Dockerizing" Multiple Services
[skipped]

### A Continuous Integration Workflow for Multiple Images
[skipped]

### Multi-Container Deployments to AWS
[skipped]

### Onwards to Kubernetes!

A `Cluster` in a world of Kubernetes is an assembly of:
- a `master` that controls what each `node` does
- and one or more `node`s

A `Node` is a Virtual Machine OR a physical computer that runs some number of `container`s

A `Master` controls what each `Node` is running at any given time.

Developers interact with the Kubernetes `Cluster` by reaching out to the `Master`.  We give `Master` some set of directions for example: "hey, please run five containers using the 'client worker' image". `Master` takes those directions and relays them to the `Nodes`.

`Load Balancer` lives outside of the `Cluster`. It relayes outside network requests to the `Node`s

Kubernetes is a system for:

- running many different containers
- different types of containers
- different number of containers
- over several different computers or VMs
- good for scaling up our application with different containers and quantities independently


Working with Kubernates:

- Development
   - `minikube`

- Production (managed solutions)
   - Azure (AKS)
   - Amazon (EKS)
   - Google (GKE)
   - Digital Ocean
   - Do it yourself

`minikube` is a command line tool used to:

  - setup a Kubernetes `cluster` on your local machine
  - managing the Node itself (Node = VM)
  - it creates one `Node` that runs the `Container`s

`kubectl` is another command line tool used to:

  - interact with the `Cluster` in general
  - manage the the `Node`s and `Container`s
  - used for both `minikube` as well as production solutions

Kubernetes expects all the images to be already built.  There is no option to have Kubernetes build it for us as `docker-compose` could.  The images must be build during some outside step.

Kubernetes uses *multiple* configuration files. Unlike `docker-compose` where each service was a `container`, with Kubernetes we have one configuration file to define an `object`.

And `object` is not necessarily going to be a `container`. 

In Kubernetes, we have to manually setup a vast majority of our networking, including internal networking between `objects`.  It is a far more involved process vs `docker-compose`. [Networking setup is done is a separate config file???]

Since I've skipped building the multi-client in the lectures above, I'm using Stephen's image instead in the lecture:

image: stephengrider/multi-client

In this section we're creating two configuration files that define two different `kind`s, or types, of `object`s:
```
kind Pod
kind Service
```

We will feed those configuration files to `kubectl` tool to configure our `minikube` `cluster`.  It will create two `object`s one for each file.

An `object` is a *thing* that exists inside out Kubernetes cluster.
- Pod
- Service
- StatefulSet
- ReplicaController

The `apiVersion` specified in the configuration file defines a different set of `object`s we can use in that file.

`apiVersion: v1`
- componentStatus
- configMap
- Endpoints
- Event
- Namespace
- Pod
- Service
- [more...]

`apiVersion: apps/v1`
- ControllerRevision
- StatefulSet
- [more...]


A `Pod` is used to run a `container`.
`Node` > `Pod` > `Container`

A `Pod` is a *grouping* of `container`s with a very common purpose. Its a group of containers that MUST be deployed together or they won't work otherwise.  Typically we will just run one `container`.

In Kubernetes world there is no such thing as "just creating a container" on a `cluster`.  
It must be in a `Pod`. 
A `Pod` is the *smallest* thing you can deploy to run a single `container`.
A `Pod` *must* run one or more `container`s inside of it. 



A `Service` is used to setup networking.
There are four sub-types:
- ClusterIP
- NodePort
- LoadBalancer
- Ingress

The way this is specified in a configuration file:
```
apiVersion: v1
kind: Service
spec:
  type: NodePort
```

The purpose of a `NodePort` `Service` is to expose a `Container` to the outside world. To allow you as a developer to open up your web browser to access that container. It is *ONLY* good for development purposes. It is not used in production environments. 

QUESTION: should I have two different sets of k8s cofig files for `dev` and `prod`?

Every single Node/VM has a single `kube-proxy`.  It is Node/VM's one, only, single window to the outside world.  Every incoming request first flows through this `kube-proxy`.  It inspects the request and decides how to route it inside the Node.  It routes it to a `Service` which in turn has port mapping to some `Container`. 

`Service` of subtype `NodePort` uses a `selector` section to specify which `object`s to route the request to. 

Instead of using `object` or `Pod` names for this, it uses `labels`.  You first apply some arbitrary label name on your component, and than use that same label name in the `selector` section of the configuration file.

In the `Pod` config file, the Pod is labeled in the `metadata` section:
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: web
```

`component: web` is a `key: value` definition. So you can for example apply the `component` key with different `value` on different `container`s 

`NodePort` `Service` defines three different kinds of ports:
- port - a port that another Pod *inside* our cluster can use to connect to the Pod we're defining
- targetPort - a port inside our Pod that we want to open up the traffic to
- nodePort - is what the web browser would use (30000-32767)


How to point your browser to a container running in your cluster?

First we need to find out the IP address of the cluster, even when using minikube locally.  The Node/VM it created has its own IP address.  To find out what it is:
```
minikube ip
```

That's the IP address you must use.  There is also the `nodePort` that has been configured that must also be appended. You can get that from your configuration file, or run
```
kubectl get services
```

and look for the port in the PORT(S) column - it's the second one after the colon. (31515 in this case).

Something to notice, when we start `minikube` we end up with a Node/VM which is running another copy of Docker - separate from you local machine's Docker.  Stephen somehow seems to interact from his machine with the Docker running inside Kubernetes cluster but hasn't explained yet how to do that.  Reason I say that is because when I do `docker ps --all` I *don't* see the containers running in the `cluster` as he is showing.

Other important takeaways:

- Kubernetes is a system to deploy containerized apps
- `Nodes` are individual machines (or VMs) that run Docker containers (in Pods)
- `Masters` are machines (or VMs) with a set of programs to manage `Nodes`
- Kubernetes doesn't build our images - it gets them from somewhere else (hub)
- `Master` decides *where* (which `Node`) to run each container - each `Node` can run a dissimilar set of containers (although we DO have the ability to control it via config files if we want to)
- To deploy something, we update the desired state of the `Master` with a config file
- The `Master` works *constantly* to meet your desired state (ie. restarts crashed containers)


Discussion about two different ways of approaching deployments:

- Imperative Deployments: "do exactly these steps to arrive at this container setup"
- Declarative Deployments: "our container setup should look like this, make it happen"


### Maintaining Sets of Containers with Deployments

Our new goal for this section is to update previously deployed Pod with `multi-client` image to use the `multi-worker` image instead.  The idea is to learn how to do that *declaratively* via the config files.

`Master` uses the `name` and `kind` values from the configuration file to figure out which `object` to update.  You can change all other values in the file, but not those two, otherwise you're basically creating a new `object`.

How can we inspect a `Pod` to verify which containers it is running? To verify that it is running our new image? We need to get detailed information about our Pod.

To get detailed information about an `object` inside of our `cluster`:
```
kubectl describe <object type> <object name>
kubectl describe pod client-pod
```

I verified with this command that indeed the `container` running in our `Pod` is running and using the new image.

Next we try to change the value of the containerPort, say to 9999 and `cubectl apply` it again. BUT THAT FAILS! kubectl spits out an error message instead with some info about *which* properties can be updated and apparently containerPort is not one of them.

This is a limitation of a `Pod` kind.  We can use another `object` kind if we want to be able to change *any* of the config values: `Deployment`

A `Deployment` is a Kubernetes `object` that is meant to maintain a *set* of *identical* `Pods` ensuring that they have the correct config and that the right number exists.  It ensures the the `Pods` are running the correct configuration, and is always in a runnable state.

It really is very similar in nature as `Pod` in practice. We can use either `Pods` or `Deployments` to run `containers`, however `Pods` are really only used during development and rarely in production.  A `Deployment` is good for both dev and prod.  It allows changing all of the Pod configuration.

In reality, we just used `Pods` to learn about them, and instead use `Deployment`s from now on.

A `Deployment` object has attached to it a `Pod Template` used to create the `Pod`s. Any config changes we actually make, change this `Pod Template`, not the `Pod` configuration itself.  `Deployment` will than either update the `Pod` configuration if it can, or kill and spin up a new one with updated configuration.  It constantly watches the Pods, their state, configuration, and restarts them when they crash.

The `Deployment` is actually talking to the `Master` to coordinate creation of `Pods`.

Each `Pod` gets its own internal IP address assigned to it in a `cluster`.
It can change however every time it restarts, updates, etc.
To find out what that IP address is:
```
kubectl get pods -o wide
```

A `Service` doesn't route incoming requests using `Pod`'s IP address however.  Instead it uses the `selector` specified in the configuration file.  So even though `Pods` IP address changes it can still find it.

My own observation: a `Service` still routes to a `Pod` even if that `Pod` is created via a `Deployment`.

So now we deleted the old Pod with:
```
kubectl delete -f <config-file>
```

We created a `Deployment` with a template for `Pods` and applied that instead.

We can finally change the `containerPort` now in the `Deployment` file.  It works! But we can see that column `AGE` in `kubectl get pods` shows *seconds*.  What this means is that the `Deployment` deleted the old container and re-created a new one with the updated IP address.

To verify that the container is running with the new port:
```
kubectl get pods
```

Note the name of the pod from above and:
```
kubectl describe pods client-deployment-cb665f48d-qsl5w
```

Look for "Port:" in the output.

When we change the `replicas` in a `Deployment` configuration file, it will create multiple copies of the `Pod`.  Each one with a different IP address, but exposing the same port.

We were also able to update the `Deployment` to a new `image` name and see that succeed.

But how do we re-create our `Pod`s with the new version of the same image?
Surprisingly that is very challenging.  There is not a very good solution around it.

Why is this challenging?

Because *nothing* changes in the `Deployment` file.  There is nothing in that file that versions the image.  Since `kubectl` doesn't see any changes it rejects the file with "unchanged" message.  It does not update the containers with a new image version from Docker Hub 

Workarounds? (none are that good, but sort of work)

- manually delete pods to get the deployment to recreate them with the latest version (but that seems silly and a bad idea)

- tag build images with a real version number when building and specify that version in the config file (better).  It adds an extra step in the production deployment process, and is not really friendly - but it could work.  We are not allowed to use environment variables in Kubernetes config files so we would need some custom templating process. Also, building and tagging of the docker images would typically occur in some CI environment, but we would have to update and `git commit` k8s configs with the new version *afterwards* which is a big pain in the rear.

- use an imperative command to update the image version the deployment should use after the image is built and tagged.  So when we build our image we still append a version number, but rather then updating our configuration files with new version, we issue a imperative command afterwards to update Kubernetes `Pods` in a cluster. (this is what Stephen in going with - it is pretty reasonable).

To run a `kubectl` command forcing the deployment to use the new image version (imperative command to update an image):
```
kubectl set image <object_type>/<object name> <container_name> = <new image to use>
kubectl set image deployment/client-deployment client=stephengrider/multi-client:v5
```

The building, version tagging of an image, and the imperative command to update the deployment will need to be automated/scripted in the CI. 

Multiple docker installations:

This goes back to hooking up into the Docker running in the minikube instead your local machine.

There is a way to re-configure your local machine's `docker-client` (CLI) to point to the `docker-server` of the minikube `Node`/VM.  This is done on per-terminal basis.

To configure your local copy of docker CLI to communicate with a copy of docker-server inside the Node/VM of minikube (temporary per terminal window only)
```
eval $(minikube -p minikube docker-env)
```

To see what this command is doing, you can run this by itself:
```
minikube docker-env
```

The output is:
```
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.49.2:2376"
export DOCKER_CERT_PATH="/home/tom/.minikube/certs"
export MINIKUBE_ACTIVE_DOCKERD="minikube"

# To point your shell to minikube's docker-daemon, run:
# eval $(minikube -p minikube docker-env)
```

It essentially just sets up some new envirnment variables.
The DOCKER_HOST is the IP address of you minikube.

Why would we want to do this? Why mess with Docker in the Node?

- use all the same debugging techniques we learned with Docker CLI (but many of these commands are actually available through `kubectl`)
- manually kill containers to test Kubernetes ability to 'self-heal'
- delete cached images in the node


### A Multi-Container App with Kubernetes

Introducing a new kind of `Service` - a `ClusterIP`

Recall that we use a `Service` any time we want to setup some kind of network for an object.

A `NodePort` `Service` we used previously was used to "expose a set of pods to the outside world.  That is NOT something you want to do in production - it's for dev purposes only!

A `ClusterIP` is more restricted form of networking.  It exposes a set of pods to **other objects in the cluster** but **NOT** to the outside world.

We will later need to hook it up to another `Ingress` `Service` which does expose the cluster to the outside world.

In the yaml file configuring `ClusterIP`:
- `port` is the exposed port number that OTHER `objects` would use to connect to multi-client
- `targetPort` is the port that multi-client is exposing and listening on for requests

We created a bunch of `ClusterIP` and `Deployment` files first for each pod: client, server, worker, redis, postgres.

**What is a PVC and why do we need it for postgres?**

PVC = Persistent *Volume* Claim

- postgres writes data to a `file system` (hard drive of sorts)
- docker container has its own `file system` baked by the image
- if that container restarts, it starts with fresh `file system` from the image loosing any data it saved
- so we need something outside of the container where to save data
- we need a `volume`

- a `volume` can reside on a host machine (the machine that runs the docker engine) completely outside of the container/cluster

- in this case when the container restarts, it re-points to the `volume` and thus has all the previous data


`Volume` means two different things depending on the context:

- in generic docker container terminology, it means some type of mechanism that allows a container to access a filesystem outside itself

- in Kubernetes, it is an `object` that allows a container to store data at the `Pod` level

As a matter of fact, Kubernates has three different kinds of volumes:

- `Volume`
- `Persistent Volume`
- `Persistent Volume Claim`

We don't want the first one, `Volume`, for data that needs to last. It's **not exactly** the same thing as Docker volume!!!

For postgres we want either:

- `Persistent Volume`
- `Persistent Volume Claim`

A `Volume` belongs to a `Pod` and can be accessed by any container running inside that Pod.  If the `container` restarts, it still gets access to the `Volume` in the `Pod`.  However, if the `Pod` itself ever dies - so does the `Volume` and all the data it holds!!!  So that is why we are not using a `Volume`.

A `Persistent Volume` on the other hand creates some type of durable storage that is not tied to a specific `Container` OR a specific `Pod`.  It is outside of the `Pod`.  If `Pod` crashes and restarts, `Persistent Volume` sticks around and the new `Pod` can connect to it.  It has a lifecycle not connected to a `Pod`.

A `Persistent Volume Claim` is an **advertizement** (like the Billboard). It is **NOT** and actual volume. It can't store anything, it's just an advertizement that says: "here are the different storage options that you have to store data inside your cluster".  We, the developers, write out this advertizement in a config file to inform the cluster what's available.

Kubernetes than may have a bunch of `Statically provisioned Persistent Volume`s that are ready to go and hand out to that claim.  Those are something that we very specifically created ahead of time.  There is another option that could be created on the fly: `Dynamically provisioned Persistent Volume`.  It's not created ahead of time.  It's only create when you ask for it when creating the Pod.  Any of the Pods inside the cluster an choose from volumes advertized in the `Persistent Volume Claim`.

A `PersistentVolumeClaim` has three different types of `access modes`:

- `ReadWriteOnce` - can be used by a **single Node**
- `ReadOnlyMany` - **multiple nodes** can **read** from this
- `ReadWriteMany` = can be **read** and **written** to by **many** nodes

Some options we may need to use in `Persistent Volume Claim` config file:

```
kubectl get storageclass
```

shows all the different options on your computer that Kubernetes has for creating a persistent volume.

On a Cloud Provider, you may have a "billion" options available. Each provider is going to have their own options.  Google Cloud service has a "Persistent Disk", etc:

- Google Cloud Persistent Disk
- Azure File
- Azure Disk
- AWS Block Store
- etc.

You can see some of the other options here:

https://kubernetes.io/docs/concepts/storage/storage-classes/

So, inside the `Persistent Storage Volume` configuration file, it is possibly to explicitly specify `storageClassName` - but it's best in most cases to leave it out and let Kubernetes use the default that the Cloud provider has.

For minikube, the default is to create a slice of your hard-drive for the persistent volume. In the cloud it will be something else - whatever the default is.

Q: how would I GROW this volumes if needed for my database?
Q: how can I backup a PVC

ENVIRONMENT VARIABLES

- constant values - these are super easy to setup

- constant values that do not need to change overtime, but a URLs of sorts. These are HOST variables - these need to be used in other containers.  For those, all we have to do is provide the **name of the ClusterIP Service** as the HOST name. The `name` from the yaml configuration file under `metadata` of the ClusterIP config file.  For example: HOST_NAME=http://redis-cluster-ip-service.  In reality, there is no 'http://' there - that's just an example to make it clear it is the host name.  In practice it is:  `HOST_NAME=redis-cluster-ip-service`

- Things like PGPASSWORD are a little different and slightly harder to setup in order to keep them safe and secure.  We wouldn't want to store its value in plain text in a config file.  We manage these as "secrets" or "secret variables" inside a Kubernetes cluster.

To specify regular non-secret environment variables that are OK to go in clear text in configuration file, we add this section to the definition of a cluster in the configuration file
```
env:
  - name: REDIS_HOST
    value: redis-cluster-ip-service
```

The `value` here is the value of the `name` specified in the ClusterIP Service configuration file.

But how to we define a secret like PGPASSWORD? We use a new type of Kubernetes `object` called a `Secret` (like `Pod`, `Deployment`, etc.)

You would use it to securely store one or more pieces of information inside of your cluster:

- db password
- an API key
- an SSH key
- maybe a PGUSER and PGDATABASE could go in here as well

How do we create a `Secret`.  Typically we use a configuration file for each `object` to create it, but not in this case. We run an imperative command to create it. Why? Because we don't want to save the config file with the secret in it ;)

This means that we must create these secrets manually in our local environment, and in the **production** environment as well!

To create a `Secret` imperatively:
```
kubectl create secret generic <secret_name> --from-literal key=value
kubectl create secret generic pgpassword --from-literal PGPASSWORD=12345asdf
```
where 
- `generi` is a type of secret
- `<secret_name>` is a name of secret for later reference in a pod config
- `--from-literal` means we are going to add the secret information into this command, as opposed to from a file
- `key=value` is the Key-value pair of the secret information

You can verify with:
```
kubectl get secrets
```

That shows you the name of the secret and in the DATA column the number of key/value pairs it holds (apparently it can hold multiple?)

The next step is to 
- configure the deployment file where you want to use this secret, in this case the server-deployment.yaml (so that it can use it to create a connection to the database).
- configure the postgres-deployment.yaml to tell it what its password should be (override the images default password to use our password when it creates a db)

Note, for the postgres image, the environment variable change recently from PGPASSWORD to POSTGRES_PASSWORD, so it should look like this when configuring it:
```
  env:
    - name: POSTGRES_PASSWORD
```

Now, to get access to this password, to configure your container's environment variable with the value of one of the KEYs inside that secret:
```
env:
  - name: PGPASSWORD # name of the environment variable
    valueFrom:
      secretKeyRef:
        name: pgpassword # name of the secret itself
        key: PGPASSWORD
```

You might experience an error message when applying config files:
```
cannot convert int64 to string
```

This is because ALL environment `value` fields MUST be strings.  You may get this for example when you specify your port values as integers.  Simply enclose those values in single quotes to make them strings instead.

All that's left now is punching a hole in the cluster to expose it to incoming requests - a topic for next section!

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

Create and run a container from an image with port forwarding (`-p`)
```
docker run -p INCOMING_LOCAL_HOST_PORT:PORT_INSIDE_CONTAINER <image name>
docker run -p 8080:8080 <image name|id>
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

Get output of the main running process from the point the container started:
```
docker logs <container id>
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

Create an image out of a running container 
that you maybe setup manually via sh
(don't use this very often - just interesting):
```
docker commit -c 'CMD ["redis-server"]' <container id>
```

Push a docker image to Docker Hub:
```
docker push <image name>
```

To configure your local copy of docker CLI to communicate with a copy of docker-server inside the Node/VM of minikube
```
eval $(minikube docker-env)
```

### Dockerfile INSTRUCTIONS

Base your image on some previously defined image
```
FROM <image name>:tag
```  

Set a working directory. Any following command will be executed relative to this path in the container. This prevents copying files to the root `/` for example and copies them to the specified work directory:
```
WORKDIR /usr/app
```

Download and install a dependency
```
RUN apk add --update redis
```

Tell the image what to do when it starts as a container
```
CMD ["redis-server"]
```

Copy files from `build context` to inside of the container.
- The first argument `./` is the path to folder to copy from on *your machine* relative to the `build context` (specified on `docker build <context>`)
- The second argument `./` is a place to copy stuff to inside *the container*
```
COPY ./ ./
```

### Docker Compose CLI examples

The very special configuration file
```
docker-compose.yml
```

Specify the version
```
version: '3'
```

Specify services (aka containers)
```
services: 
```

Specify that we want a `redis-server` container build from the `redis` image
```
services:
  redis-server:
    image: 'redis'
```

Specify that we want a `node-app` container build from a Dockerfile in the local directory
```
services:
  node-app:
    build: .
```

Specify port mapping.  Same convention as in `docker` CLI: first is the local machine port : second is the port of the container
```
services:
  node-app:
    build: .
    ports:
      - "4001:8081"
```

To specify a restart policy (one of: `no`, `always`, `on-failure`, `unless-stopped`)
```
services:
  node-app:
    restart: always
```



Once we have the `docker-compose.yml` configured, to start the containers specified in that file, `cd` to where `docker-compose.yml` is defined and:
```
docker-compose up
```

The above doesn't automatically re-build from Dockerfile however. To do that we need an additional `--build` argument:
```
docker-compose up --build
```

To launch all the containers in the *background* add `-d` flag
```
docker-compose up -d
```

To stop all the containers:
```
docker-compose down
```

Show status of running containers:
```
docker-compose ps
```

### Kubernetes CLI examples

Start local Kubernetes cluster
```
minikube start
```

Stop local Kubernetes cluster
```
minikube stop
```

Check minikube status
```
minikube status
```

Get info about your cluster
```
kubectl cluster-info
```

When running minikube cluster in development, you must use its Node/VM IP address to access its containers.  To find out what that IP address is:
```
minikube ip
```

Feed a config file to `kubectl`
```
kubectl apply -f <filename>
```

Print out the status of the `object`s in a `cluster`
(to make sure they were successfully created for example)
```
kubectl get pods
kubectl get deployments
kubectl get services
```

To get a little more information append `-o wide` to `kubectl get`.
This is how you can find out the IP address assigned to a `Pod` for example:
```
kubectl get pods -o wide
```

To get detailed information about an `object` inside of our `cluster`:
```
kubectl describe <object type> <object name>
kubectl describe pod client-pod
```

To delete/remove and existing object, use the original configuration file that was used to create it when running this command. Note that this is an *imperative* update!
```
kubectl delete -f <config file>
kubectl delete deployment client-deployment
```

To run a `kubectl` command forcing the deployment to use the new image version (imperative command to update an image):
```
kubectl set image <object_type>/<object name> <container_name> = <new image to use>
kubectl set image deployment/client-deployment client=stephengrider/multi-client:v5
```

Get output of the main running process from the point the container started:
```
kubectl get pods
kubectl logs <pod name>
```

To shell into a container
```
kubectl get pods
kubectl exec -it <pod name> sh
```

To removed all cached images in minikube Node/VM
```
minikube docker-env
eval $(minikube -p minikube docker-env)
docker system prune -a
```

Show all the different options on your computer that Kubernetes has for creating a persistent volume
```
kubectl get storageclass
kubectl describe storageclass
```

To show all persistent volumes claims (this is the **advertizement** - you can get it if you want to)
```
kubectl get pvc
```

To show all persistent volumes (this is the actual **instance** of storage that meats the advertizement)
```
kubectl get pv
```

To create a `Secret` imperatively:
```
kubectl create secret generic <secret_name> --from-literal key=value
kubectl create secret generic pgpassword --from-literal PGPASSWORD=12345asdf
```
where 
- `generic` is a type of secret (some arbitrary number of key/value pairs together); other options are: `docker-registry` and `tls`
- `<secret_name>` is a name of secret for later reference in a pod config
- `--from-literal` means we are going to add the secret information into this command, as opposed to from a file
- `key=value` is the Key-value pair of the secret information


To examine secrets in your cluster:
```
kubectl get secrets
```


--- MY OWN ADDITIONS ---------------------------------

To see the Node's memory utilization:
```
kube-capacity --util --node-labels agentpool=ml100prod 
```

### Kubernetes on AZURE

### Kubernetes on Digital Ocean

## Q & A

### What is an *unnamed docker volume* ?


### What is a *named docker volume* ?

Example of running `nextcloud` container with a *named docker volume* `nextcloud` (where -v nextcloud: is a directory on your host?)
```
docker run -d -v nextcloud:/var/www/html nextcloud
```

Example of running a `postgresql` container with a *named docker volume* `db` that holds PostgreSQL Data (where -v db: is a directory on your host?)
```
docker run -d -v db:/var/lib/postgresql/data postgres
```

### What is a *mounted host directory* ?