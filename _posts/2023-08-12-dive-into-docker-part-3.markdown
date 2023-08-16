---
layout: post
title:  "Dive Into Docker part 3: Caching and Building Containers"
date:   2023-08-12 00:00:00 +0000
categories: docker
---
Using Docker image layer caching to our advantage when building docker images can significantly speed up the build process. To do this, I first structure Dockerfiles in such a way that easily cached steps happen upfront in the build process and the final few steps are copying in application code.

An example would look like:

```Dockerfile
FROM ruby:3.2.2

WORKDIR /app

COPY package.json .
RUN curl -sL https://deb.nodesource.com/setup_20.x | bash -
RUN apt-get install -y nodejs
RUN npm install

COPY Gemfile Gemfile.lock .
RUN bundle install

COPY . .

CMD ["rails", "server", "-b", "0.0.0.0"]
```
This example first sets a working directory, then copies in the package.json file. This will be cached, if the package.json doesn’t change, so assuming dependencies have not changed, we will get a cache hit, and not have to re-install dependencies each time. Next I’ll copy in the Gemfile and Gemfile.lock, again, assuming they haven’t changed, it will be cache hit, so there is no need to update dependencies. Now, the only thing that should not be a cache hit when creating the resulting image is copying in the application code. This should happen fairly quickly, and result in a really quick image build.

## Why ephemeral environments?

Building Docker images in an environment that is not ephemeral, can quickly lead to a sprawl of tools, and machine specific settings that enable specific builds. To eliminate this, I prefer to use an entirely ephemeral build system. Unfortunately, this means re-pulling and configuring the environment for each run, that’s not all that bad though, especially with Docker Buildkit.

Ephemeral runners also ensure that artifacts from previous builds aren’t hanging around and consuming storage space. This approach also negates the need for the standard cleanup procedures one would use in a typical build pipeline, such as clearing old artifacts, build logs, lingering docker images, docker volumes and networks.

I do like to retain the images from the resulting builds though, especially in the case that we would like to keep the history of images built for compliance reasons. These are typically stored in an image repository, such as ECR, or Nexus

## Docker Layers

What makes up the final docker image is a composition of its layers. Each instruction in a Dockerfile that modifies the filesystem creates a new layer.
```Dockerfile
# syntax=docker/dockerfile:1
FROM golang:alpine
WORKDIR $GOPATH/src/

ARG cache_invalidator

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /go/bin/webserver

ENTRYPOINT ["/go/bin/webserver"]
```
**Quick tip**: By supplying an argument titled cache_invalidator I can easily invalidate the dockerfiles cache by providing it in a build argument, such as `docker build -t golang-caching --build-arg cache_invalidator=$(date -u '+%Y-%m-%dT%H%M:%SZ')` Supplying A datetime in UTC. As long as a new argument is supplied, the cache is invalidated. If the argument matches one that was previously used, it will use that layer as a cache. This is useful, especially, when needing to download new security patches if one of the first steps of the Dockerfile is `apt-get update && apt-get upgrade -y` This can also be a source of frustration if there is a lot of build arguments and you are not sure why the cache is constantly being invalidated 😂😅.

If I build the image from the Dockerfile above using the command `docker build -t golang-caching .` and then inspect the image, it shows a total of 9 layers.

```bash
docker build -t golang-caching .
docker inspect golang-caching | jq '.[].RootFS.Layers' | grep sha | wc -l
       9
```
**Note**: To see the Dockerfile used to build the golang:alpine image we can inspect it in Dockerhub.

The reason the resulting image has 9 layers, is a combination of the 4 layers from the `FROM: golang:alpine` image, and then the 5 from the Dockerfile from above. `WORKDIR` does not initially seem like it would add an additional layer, but under the hood it is effectively creating and changing into that directory.

## Docker build caching on ephemeral hosts

One of the larger challenges in a build system is caching image layers for future builds. This is necessary so that the following builds of an image are significantly sped up. This specifically presents a challenge when using hosts that run a single job, then disappear.
Typically, only builds done on the same machine are fully cached. This is because of how the build cache is built using the build context that was provided at build time. Docker BuildKit solves this problem by allowing us to export the cache to a few different storage backends. Of course, locally or on a shared volume such as an AWS Elastic Filesystem (EFS) mount would be the fastest, but considering price and ease of use, I prefer to cache the layers in a AWS S3 storage bucket.

**Note**: To use a AWS S3 storage bucket to store the cache, object lock must be turned off.

Example of the buildkit command to export the full cache to the s3 backend.

```bash
docker buildx build --push -t <registry>/<image> \
  --cache-to=type=s3,region=us-west-2,bucket=cicd-image-cache,name=${app}-cache,mode=max \
  --cache-from=type=s3,region=us-west-2,bucket=cicd-image-cache,name=${app}-cache \
  --output type=docker \
  . \
  -f "Dockerfile"
```

More options to build with buildkit can be found here.

Converting the output type back to `type=docker` does take a significant amount of time, but is necessary when pushing to AWS’s Elastic Container Repository (ECR) due to the inability to upload the image manifest that is exported by buildkit.

In my last few blog posts regarding docker I will try to go into more specific examples of caching using a multi-stage build. I will then give more examples on Dockerfiles that are uniquely hard to cache.
