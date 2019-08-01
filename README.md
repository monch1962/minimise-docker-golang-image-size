# minimise-docker-golang-image-size
Steps to minimise the size of your Golang Docker images

## Overview
We're going to go through the exercise of making a minimally-sized Docker image for a Go executable.

Let's start with a simple `helloworld` app

```
package main
import "fmt"
func main() {
    fmt.Println("hello world")
}
```

We can run this quickly to see that does what we expect
```
$ go run helloworld.go
hello world
$
```

Yep!

Let's compile it and try it out
```
$ go build helloworld.go
$ ./helloworld
hello world
$
```

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

Now let's create a container image based on Debian

```
$ cd debian
$ docker build . -t debian
$
```

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
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
golang-latest       latest              7476c4302e9a        6 seconds ago       816MB
```

Hmm... Our 2Mb executable, wrapped in a container image, turns into 816Mb in size. What the...?

Can we do any better?

