# Docker Execution, Networking, and Dockerfiles

Today I continued learning Docker, mostly around three areas:

- executing commands inside running containers
- how containers communicate through Docker networks
- building my own images with Dockerfiles

I had already used Docker commands before, but these exercises helped connect several pieces together.

The biggest thing that became clearer is that a container is not just something I start and forget about.

I can enter it, inspect it, execute commands inside it, control what it can communicate with, and define exactly how its environment should be built.

---

## Executing Commands Inside Containers

The first section was about interacting with containers that are already running.

### Docker Help

Docker has built-in help for commands.

Instead of always searching for documentation, I can inspect commands directly from the CLI.

For example:

```bash
docker --help
docker exec --help
```

This is useful because Docker has a lot of commands and options, and I don't need to memorize all of them.

I mostly need to remember how to discover them.

---

## `docker exec`

`docker exec` lets me execute a command inside an already running container.

Conceptually:

```text
Host
  |
  | docker exec
  v
Running Container
  |
  └── execute another command
```

For example:

```bash
docker exec <container> <command>
```

The important distinction for me is that `docker exec` does not create another container.

It runs another process inside the existing container.

So if a web server is already running inside a container, I can still execute another command inside that same container to inspect what is happening.

---

## Inspecting Networking with `netstat`

One exercise used `exec` together with `netstat`.

Something along the lines of:

```bash
docker exec <container> netstat ...
```

This helped connect Docker commands with normal Linux troubleshooting tools.

Docker gives me a way to enter or execute something inside the container, but once I am there, many of the tools and concepts are still normal Linux concepts.

For example, I can inspect:

- listening ports
- network connections
- processes
- files
- environment variables

This is useful when a container is running but the application inside it is not behaving the way I expect.

---

## Live Shell

Instead of executing one command at a time, I can also open an interactive shell inside a running container.

For example:

```bash
docker exec -it <container> sh
```

or, if Bash exists in the image:

```bash
docker exec -it <container> bash
```

The `-it` part gives me an interactive terminal.

The mental model is basically:

```text
docker exec command
    ↓
run one command inside container

docker exec -it ... sh
    ↓
open a shell inside container
    ↓
inspect/debug interactively
```

This seems especially useful for debugging.

If an application behaves differently inside Docker than it does on my host machine, I can inspect the environment from the container's point of view.

---

# Docker Networking

The next section was about networking.

This was probably the most interesting part because it showed that containers are isolated, but that isolation can be controlled.

---

## Offline Container

A container does not necessarily need network access.

Docker can start a container without a network:

```bash
docker run --network none ...
```

With the `none` network, the container is essentially isolated from normal external/container networking.

This helped me realize that network access is something Docker gives to a container rather than something I should automatically assume exists.

Conceptually:

```text
Normal container

Container
    |
Docker Network
    |
Other Containers / Internet


--network none

Container

(no Docker network connection)
```

---

## Breaking the Network

The exercises also demonstrated what happens when networking between containers is removed or incorrectly configured.

This is useful because an application can be perfectly healthy by itself while still being unusable because it cannot reach another service.

For example:

```text
Load Balancer
      |
      X   network problem
      |
Application Server
```

The application server might still be running.

The load balancer might still be running.

But the system as a whole is broken because the communication path between them is broken.

That is an important distinction:

```text
container running
!=
application architecture working
```

---

## Application Servers and Load Balancers

The networking labs introduced multiple containers working together.

Something like:

```text
                ┌── Application Server 1
Client
  |
  v
Load Balancer ──┼── Application Server 2
                |
                └── Application Server 3
```

Instead of exposing every application server directly to the user, a load balancer can receive traffic and forward requests to backend application servers.

This made Docker networking feel more practical.

The network is not just about giving containers internet access.

It is also how different parts of an application architecture discover and communicate with each other.

---

## Custom Docker Networks

Docker allows me to create my own networks.

For example:

```bash
docker network create <network-name>
```

Containers can then be attached to that network.

Conceptually:

```text
Docker Host
|
└── custom-network
    |
    ├── load-balancer
    ├── app-server-1
    └── app-server-2
```

Containers on the same Docker network can communicate with each other.

This is much cleaner than thinking about containers as completely independent processes with manually managed IP addresses.

---

## Container Names and Service Communication

One networking idea that started making more sense is that container IP addresses should usually not be treated like permanent addresses.

Containers can be recreated.

Their IP addresses can change.

On a Docker network, using container/service names gives us a better abstraction.

Instead of thinking:

```text
connect to 172.x.x.x
```

I can think more like:

```text
connect to app-server
```

Then Docker's networking layer handles the underlying container address.

This matters when configuring things such as a load balancer.

The application architecture can describe services by name instead of depending on temporary container IP addresses.

---

## Configuring the Load Balancer

The final networking exercise connected the earlier concepts.

The load balancer needs to know where the backend application servers are.

But the load balancer also needs network connectivity to those containers.

So configuration alone is not enough.

Both things need to agree:

```text
Load Balancer Configuration
            +
Docker Network Connectivity
            |
            v
Working communication
```

This made me think about containerized systems in layers:

```text
Application configuration
        ↓
Container
        ↓
Docker network
        ↓
Other containers
```

A problem at any layer can make the application appear broken.

---

# Dockerfiles

The last section moved from running existing images to creating my own.

A Dockerfile is basically a recipe for building an image.

```text
Dockerfile
    |
    | docker build
    v
Docker Image
    |
    | docker run
    v
Container
```

That relationship is simple, but important:

```text
Dockerfile → Image → Container
```

The Dockerfile is not the container.

The image is not the running process.

The container is an instance created from the image.

---

## Basic Dockerfile Instructions

A Dockerfile can describe things such as:

```dockerfile
FROM ...
WORKDIR ...
COPY ...
RUN ...
ENV ...
CMD ...
```

Each instruction has a different purpose.

### `FROM`

Defines the base image.

```dockerfile
FROM ubuntu
```

or:

```dockerfile
FROM python:3
```

Instead of building an operating system and runtime completely from scratch, I can start from an existing image.

---

### `WORKDIR`

Sets the working directory inside the image/container.

```dockerfile
WORKDIR /app
```

Commands after that can operate relative to `/app`.

---

### `COPY`

Copies files from the build context into the image.

```dockerfile
COPY . /app
```

This is how my source code or configuration becomes part of the image.

---

### `RUN`

Executes commands while building the image.

For example:

```dockerfile
RUN apt-get update
```

or:

```dockerfile
RUN pip install -r requirements.txt
```

An important distinction is:

```text
RUN
→ happens while building the image

CMD
→ happens when a container starts
```

That distinction is easy to miss at first.

---

### `ENV`

Defines environment variables:

```dockerfile
ENV SOME_VARIABLE=value
```

This gives applications configuration inside the container environment.

It also helped reinforce that containers have their own environment separate from my host machine.

---

### `CMD`

Defines the default command executed when a container starts.

For example:

```dockerfile
CMD ["python", "app.py"]
```

So the image contains the application and its dependencies, while `CMD` describes what it should normally execute when started.

---

# Building a Server Image

The server exercises connected Dockerfiles with the container workflow I had already learned.

Instead of doing something manually inside a container every time:

```text
start container
↓
install something
↓
copy files
↓
configure server
↓
run server
```

I can describe those steps in a Dockerfile:

```text
Dockerfile
↓
docker build
↓
reproducible image
↓
docker run
```

That is a much better model.

If the container is deleted, I don't need to remember all the manual steps I performed inside it.

I rebuild the image from the Dockerfile.

---

## Dockerizing Python

The Python exercises showed the same concept with an application runtime.

A Python application may require:

- Python itself
- source files
- dependencies
- environment configuration
- a command that starts the program

A Dockerfile can package those requirements together.

Conceptually:

```dockerfile
FROM python:...
WORKDIR /app
COPY ...
RUN ...
CMD ...
```

Then:

```bash
docker build -t my-python-app .
docker run my-python-app
```

The exact application is less important than the pattern.

The image contains everything necessary to provide the application's runtime environment.

---

## Dockerizing Python Error

One of the exercises intentionally involved an error while dockerizing the Python application.

I think this was useful because building a container successfully does not automatically mean the application will successfully run.

There are different stages where things can fail:

```text
Dockerfile
    ↓
Build image
    ↓
Start container
    ↓
Start application
    ↓
Application actually works
```

An error at each stage means something different.

For example:

- Dockerfile syntax problem
- dependency installation failure
- missing file
- incorrect working directory
- wrong command
- missing environment variable
- application runtime error
- networking problem

So debugging Docker is often about first identifying **which layer is actually failing**.

---

# What Clicked

Before these exercises, I mostly thought about Docker like this:

```text
image
  ↓
docker run
  ↓
container
```

That model is correct, but incomplete.

Now I see more of the lifecycle:

```text
                    Dockerfile
                        |
                        v
                   docker build
                        |
                        v
                      Image
                        |
                        v
                    Container
                   /         \
                  /           \
        docker exec         Docker Network
             |                   |
             v                   v
       inspect/debug        other services
```

A Docker container is simultaneously:

- a process execution environment
- a filesystem environment
- a network endpoint
- an instance of an image
- something I can inspect and debug while it is running

Another thing that clicked is that Docker is still built around concepts I already know.

Inside Docker there are still:

```text
processes
ports
network interfaces
files
environment variables
Linux commands
application processes
```

Docker mostly gives me isolation and tooling around those concepts.

The Dockerfile then makes that environment reproducible.

Instead of configuring a server manually and hoping I remember the steps later, I describe the environment as code and rebuild it.

And Docker networking lets multiple isolated environments become part of the same application architecture when they need to communicate.

The overall mental model I have now is:

```text
Dockerfile
    ↓
Image
    ↓
Container
    ├── processes
    ├── filesystem
    ├── environment
    └── network
          ↓
    other containers
```

I think this is the point where Docker is starting to feel less like a collection of commands and more like one connected system.