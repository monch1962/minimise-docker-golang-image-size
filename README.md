# Minimising the size of Docker images for Golang code
Steps to minimise the size of your Golang Docker images

## Overview
We're going to go through the exercise of making a minimally-sized Docker image for a Go executable.

Now, we want to run our code inside a Docker container, rather than as a standalone executable file. Docker images wrap around the code they're running, and those Docker images are what needs to execute. Given those Docker images consume resources when running, we want to create Docker images that are as small as possible so they consume as little resource as possible - that way we can run more concurrent instances of our helloworld on each Docker host.

Let's start with a simple `helloworld` app

```
package main
import "fmt"
func main() {
    fmt.Println("hello world")
}
```

## My development environment
These days, virtually all the code I write runs in a cloud. It's been a few years since I've written code that runs on a system I can touch, and even longer since I wrote code intended to run on the machine I code with.

Given this and the fact that I have near-perfect Internet connectivity almost everywhere, I no longer build code on my own systems. A big benefit of this is that I don't have to keep updating tools on my development system - and that's just fine with me. I've tried several options, but the platform I currently use to develop code now is Google Cloud.

Specifically, I spin up Google Cloud Shell instances, which gives me an on-demand Linux VM I can access from within a web browser i.e. from just about any device. That Linux VM is pretty small, but it contains current versions of all the tools I want and Google maintains it for me. Oh, and it's completely free.

## Make sure the code works

We can run this helloworld code quickly to see that does what we expect
```
$ go run helloworld.go
hello world
$
```

Good.

Let's compile it and try it out
```
$ go build helloworld.go
$ ./helloworld
hello world
$
```

## Check the file size of the Golang executable

Now let's see how big those files are. I'm running this on a Google Cloud Shell instance, so if you're on another platform your results might be a bit different. That doesn't matter for the purpose of this exercise

```
$ ls -l *
total 1956
-rwxr-xr-x 1 monch1962 monch1962 1993406 Aug  1 20:41 helloworld
-rw-r--r-- 1 monch1962 monch1962      73 Aug  1 20:32 helloworld.go
$
```

As you can see, the helloworld executable is slightly under 2Mb in size. If you're going to run this file inside a Docker container, that container with the executable file can't be any smaller than 2Mb (i.e. the size of this file). Let's see what happens when we put that file into a container

## Debian container

Now let's create a container image based on Debian. As I write this, the current version of Debian is named 'stretch', and there's a Golang image based on stretch. Let's use that to compile and run our code

```
$ cat Dockerfile
FROM golang:stretch
WORKDIR /
COPY helloworld.go .
RUN go build helloworld.go
CMD ["./helloworld"]
```

Does it work?
```
$ docker build . -t debian-container
...
$ docker run debian-container
hello world
$
```

We've got our helloworld app running inside a container, so are we done here? Let's see how big that container image is...

```
$ docker images | grep debian-container
debian-container     latest              a4c67ca2c404        28 seconds ago      776MB
```

Ouch - our 2Mb executable has ballooned to 776Mb when we wrap it inside a Docker container

## golang-latest container

There's a standard Golang container, so presumably that's what we should be using for Golang code. Let's build a Dockerfile based on that to run our helloworld app

First the Dockerfile

```
FROM golang:latest
WORKDIR /
COPY helloworld.go .
RUN go build helloworld.go
CMD ["./helloworld"]
```

Now let's build it
```
$ docker build . -t golang-latest
$
```

Does it work OK?
```
$ docker run golang-latest
hello world
$
```

Yes it does. We're looking good...

Now, let's look how big that image is
```
$ docker images | grep golang-latest
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
golang-latest       latest              7476c4302e9a        6 seconds ago       816MB
```

Hmm... Our 2Mb executable, wrapped in a container image, turns into 816Mb in size. What the...?

Can we do any better?

## Alpine container

Alpine is a distribution that's designed to let us produce small container images. It contains an absolute minimal set of functionality out of the box, so we're going to have to add the tools we want to that image in order to build our helloworld executable.

## Golang-alpine container

There's a version of the Alpine container that includes the pieces necessary to compile and execute Golang applications. Let's build a Dockerfile for that and see how it goes
```
$ cat Dockerfile
FROM golang:alpine
WORKDIR /
COPY helloworld.go .
RUN go build helloworld.go
CMD ["./helloworld"]
$ docker build . -t golang-alpine
...
$ docker run golang-alpine
hello world
$ docker images | grep golang-alpine
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
golang-alpine       latest              849a6d791860        3 minutes ago       352MB
$
```

OK, we're getting somewhere. Just getting rid of all the unnecessary stuff in a general-purpose base container and going with minimal Alpine has reduced our container size from 816Mb to 352Mb. That's a big improvement, but it's still a whole lot bigger than our 2Mb executable file

What else can we try?

## Multi-stage build

There's a workflow we can use to separate out the build of the Golang executable from having a container able to run it. Let's look at how that would work
```
$ cat Dockerfile
# build stage
FROM golang:alpine as builder
RUN mkdir /build
ADD . /build/
WORKDIR /build
RUN go build -o main .

# final stage
FROM alpine
RUN adduser -S -D -H -h /app appuser
USER appuser
COPY --from=builder /build/main /app/
WORKDIR /app
CMD ["./main"]
$ docker build . -t multistage-alpine
...
$ docker run multistage-alpine
helloworld
$
```

Now that Dockerfile looks quite a bit different to what we've used before, but it seems to have given us a working image. Before we go into the contents of the Dockerfile, let's see how big that image is
```
$ docker images | grep multistage-alpine
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
multistage-alpine   latest              64f614b0c4fd        2 minutes ago       7.58MB
$
```

Now we're kicking goals! We've gone from a 352Mb image file when we were building and running the executable within the one Alpine container, down to 7.58Mb.

Now we've established this approach is giving a much smaller image size, what's with this Dockerfile?

Note that there's 2 distinct sections in the Dockerfile: `build stage` and `final stage`. Our helloworld app is compiled in the build stage, then we spin up a _different_ Alpine image, copy the compiled helloworld from build state into that 'empty' Alpine image and run it.

That second Alpine image is just basic Alpine, plus our executable - no Go compiler, and virtually nothing else

So... can we do even better? Just how small can our Docker image get?

## Multi-stage scratch build

SCRATCH is a base container image that contains ... nothing. Not a file, not even an operating system. Huh?

```
$ cat Dockerfile
# build stage
FROM golang:alpine as builder
RUN mkdir /build
ADD . /build/
WORKDIR /build
RUN go build -o main .
# final stage
FROM scratch
COPY --from=builder /build/main /app/
WORKDIR /app
CMD ["./main"]
$ docker build . -t multistage-scratch
...
$ docker run multistage-scratch
hello world
$ docker images | grep multistage-scratch
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
multistage-scratch   latest              cee55e91289d        20 seconds ago      2MB
```

Now we've shrunk our image even further. It's essentially the same size as the helloworld executable, so we're probably not going to get it any smaller than that.

Awesome. We've got a working helloworld container that's pretty much the same size as the helloworld executable. I can live with this.

But this approach has its limitations...

