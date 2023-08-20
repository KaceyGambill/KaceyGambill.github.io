---
layout: post
title:  "Dive Into Docker part 4: Inspecting Docker Image layers"
date:   2023-08-12 00:00:00 +0000
categories: docker
---
This post is going to be shorter. I'd like to highlight a tool that I really enjoy working with called "[Dive](https://github.com/wagoodman/dive)" I think it is an essential tool when working to build and optimize docker containers.

Dive is a an essential tool when building or inspecting Dockerfiles. This tool can help pinpoint exactly what is contained in each layer of the Dockerfile. Specifically it 
quickly combes through the Dockerfile and tries to show wasted space.

## Installing Dive
[installation-instructions](https://github.com/wagoodman/dive#installation)
My preferred way to install Dive, if using a mac, is to use brew: -- `brew install dive`

## Using Dive
I prefer to use dive during local development of Docker containers. To get started
I typically just run: `dive image-name` if the image is not found locally this 
will take care of pulling the image from the remote repository. 
**Note**: tmux keybindings will get in the way, I usually detach from tmux or
open another terminal session before using `dive`

![dive ruby:3.2.0](/img/dive-ruby-3-2-0.png)
