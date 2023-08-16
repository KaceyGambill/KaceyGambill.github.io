---
layout: post
title:  "Dive Into Docker part 1: How to use Docker Images"
date:   2023-07-30 00:00:00 +0000
categories: docker
---
In this short series, I will explore several areas of Docker. Initially I will start by providing a brief overview of using and creating Docker images, followed by an explanation as to why they are necessary when running production grade software. Then I will cover how to efficiently build and cache docker images, specifically when building on ephemeral hosts, this will include using buildkit with docker. After that, I will cover the different types of Docker image registries.

To get started with Docker, we first need to install docker. Once it is installed, it is pretty easy to get going.

We can run the default test docker image suggested by the installation guide with docker pull hello-world this will start by checking to see if the image is stored locally, if not, it will then pull the image. After pulling it, I can go ahead and inspect it to see what it is going to do, in this example I am looking for what CMD is going to run.

```bash
docker pull hello-world
docker inspect hello-world | jq '.[].ContainerConfig.Cmd'
[  "/bin/sh",  "-c",  "#(nop) ",  "CMD [\"/hello\"]"]
```

Looking at the CMD instruction I can see that it is going to run a binary named ‘hello’. Additionally, we can check to see how many layers this image contains, while not especially useful, it is interesting.

```bash
docker inspect hello-world | jq '.[].RootFS.Layers'
[
  "sha256:a7866053acacfefb68912a8916b67d6847c12b51949c6b8a5580c6609c08ae45"
]
```

I can run this via: docker run hello-world which will return with:

```bash
>Hello from Docker!
>This message shows that your installation appears to be working correctly.
```

Containers give us a way to isolate what is almost a group of processes with kernel namespace isolation so that the group of processes can run predictably on many different machines. This can be helpful when building, or deploying software.

Consider the scenario of creating a CI/CD pipeline for a compact Go application.

Firstly, we must ensure the build process is not affected by packages or caches from a developers local machine, thus maintaining determinism in our build.

Secondly, we desire that anyone utilizing this application can operate it uniformly, irrespective of the host machines operating system.

To accomplish this, a common approach could be to directly start authoring a Dockerfile, potentially making educated guesses at the build steps. However, an easier and more effective approach might be to start within a Docker image, carry out the build process, and then translate those steps into Dockerfile instructions.

```bash
docker run -d -v $(pwd):/go/src --name golang-builder golang:alpine tail -f /dev/null

docker exec -it golang-builder shcd src/ && ls
go.mod   main.go# here we can see our files in the container / directory that we bind mounted them to. 
# now we can build the binary, that we will eventually build inside of a multi-stage docker containerCGO_ENABLED=0 GOOS=linux go build -o /go/bin/webserverls /go/bin
webserver
```

Okay, now I know that the command worked and that I was able to build the webserver. I am ready to convert this to a Dockerfile

```Dockerfile
FROM golang:alpine AS BUILDER
WORKDIR $GOPATH/src/
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /go/bin/webserver

FROM scratch
COPY --from=BUILDER /go/bin/webserver .
ENTRYPOINT ["/webserver"]
```

Now I can build and run the resulting image.

`docker build -t golang-webserver .`

```
docker run -d -p 3000:8080 --name golang-webserver golang-webserver# now that the docker container is running we can go ahead and curl it from our local machine
curl localhost:3000/
Hello, World!# to see logs for the docker container we can use
docker logs golang-webserver
2023/07/28 21:33:22 User hit the / route
2023/07/28 21:33:23 User hit the / route
```

Now to quickly cover the flags that were used in the previous commands.

```bash
-t # tags the image
-d # runs the container in detached mode
-p # exposes a port to the host machine, host machine being the first, argument separated by the colon
-v # mounts a local volume on the docker container
```

Note, if pushing to an image repository tag the image with the URL of the repository. If pushing to AWSs Elastic Container Registry (ECR) it would look like

```bash
docker build -t 333111888777.dkr.ecr.us-west-2.amazonaws.com/mirror:postgres-12-alpine .
docker push 333111888777.dkr.ecr.us-west-2.amazonaws.com/mirror:postgres-12-alpine
# or in the example above with the golang-webserver
# this would tag our image with the new tag, then push it to the desired repository
docker build -t golang-webserver .
docker tag golang-webserver 333111888777.dkr.ecr.us-west-2.amazonaws.com/webserver:golang-webserver-v1.0.0
docker push golang-webserver 333111888777.dkr.ecr.us-west-2.amazonaws.com/webserver:golang-webserver-v1.0.0
```

In my next post I will dive deeper into efficiently building containers, focusing on minimizing the layers and implementing a multi-stage build.

Below are some helpful commands when dealing with docker

```bash
docker ps # lists containers
docker ps -a # lists all containers to include ones that have stopped
docker ps -q # lists only the container IDs, especially useful when stopping all running containers:
docker stop <container_id> # example: docker stop 9917e6e86ff7
docker rm golang-webserver # removing a container after use
docker tag # adds a new tag to a container
docker run # pulls and runs a container
docker inspect # shows details about a container in json
docker image ls # lists images
docker logs <container_id> # example docker logs 9917e6e86ff7
# docker logs, -f follows, and -n50 displays the last 50 lines, useful if there is already a lot of output
docker exec -it golang-webserver sh # exec's into container so that you can explore
docker system df # check how much space the docker system is using and if there is reclaimable space due to dangling images
docker system prune --all --force # prune all unused docker images and networks
```



Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
