---
layout: post
title:  "Dive Into Docker part 2: Building Docker Images"
date:   2023-08-06 00:00:00 +0000
categories: docker
---
In the next installment of this series, I’ll jump into building Docker images. The final images I typically work with hover around 500MB, but it’s crucial to minimize this size as much as possible.

The size of the image becomes particularly important when deploying 15–20 of these images to a platform like ECS, which would equate to 15–20, 500MB images. If I take 20 * 500MB, it is about 10GB of data.

If you’ve configured VPC Endpoints, it can provide significant cost savings. Without them, you could be looking at a data transfer of around 10GB to deploy a single application.
Given that most businesses operate with multiple, non-monolithic applications or microservices, and considering the frequency of deployments, this can quickly escalate into a substantial data transfer cost.

As I mentioned in the first post of this series, I will cover how to efficiently build docker images. I’ll start by giving examples of good practices when writing Dockerfiles.

The goal is to minimize the amount of layers in the Dockerfile, while also, caching as many steps as possible to ensure quicker builds. To do this, we need to put any build instructions that are not usually cached near the end of the Dockerfile.

All common packages and setup steps should be near the beginning of the Dockerfile.

I’ve updated the golang example from the previous post to include an external library, specifically my favorite golang logging library. So that now we have some outside dependencies. The steps to build can be a little more refined.

Example:
```Dockerfile
# syntax=docker/dockerfile:1
FROM golang:alpine AS BUILDER
WORKDIR $GOPATH/src/

COPY go.mod go.sum .
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /go/bin/webserver


FROM scratch
COPY --from=BUILDER /go/bin/webserver .
ENTRYPOINT ["/webserver"]
```
With this, I am only rebuilding the dependencies when go.mod or go.sum changes. If those files stay the same, they stay cached as a docker layer.

In this example, I will not necessarily see a build speed increase, but in a larger application, splitting out copying dependencies and copying application code can lead to large speed increases.

To trim this down further, I am passing the current build directory using `.` as our build context. `docker build -t golang-webserver .` In this directory, there are a few files that do not help us build the application, such as `Dockerfile` , `README.md` , `sonar-project.properties` , `web_test.go` .
```bash
tree
.
├── Dockerfile
├── README.md
├── go.mod
├── go.sum
├── main.go
├── sonar-project.properties
└── web
    ├── web.go
    └── web_test.go
```
When building this file, I only care about `go.mod` , `go.sum` , `main.go` , and `web/web.go` . So I’ll create a `.dockerignore` file, this acts very similarly to a `.gitignore` file.

```bash
cat .dockerignore

Dockerfile
README.md
**/*_test.go
sonar-project.properties
```
Now inside of my docker container the only files that will exist after running COPY . . are:
```bash
tree
.
├── go.mod
├── go.sum
├── main.go
└── web
    └── web.go
```
I find that using a `.dockerignore` file is much more maintainable instead of copying each file / directory needed into the Dockerfile.

Even though I am only using a multi-stage build and I am only copying over the actual binary to be ran, I still prefer to keep each layer of each image to be as tidy and small as possible.

That concludes this post! In the next post of this series I will describe in more detail how to take advantage of caching docker builds to improve the build time of subsequent builds.

I will also try to follow up with some more examples of multi-stage builds using other languages other than golang. I understand that this becomes more difficult when it is necessary to have more than the single statically linked binary, like in this example.
