---
layout: post
title: What is eBPF
date: 2024-12-07 00:00:00 +0000
categories: SRE, DevOps, telemetry, Observability
---

### Intro

The goal of this post is to introduce what eBPF is and give an example as to why we should care about it. 
At the end, I will share my `Dockerfile` that you can use to work on eBPF programs, on a Macbook with a m series chip. 

If you don't plan on sticking around, please at least read the word of caution regarding eBPF, it's only magic, if we decide not to understand it!


### A Word Of Caution Regarding eBPF

I also want to provide an upfront warning. Just because a tool markets itself as using "eBPF" does not mean that it is performant. All tools should be measured and understood before being using in an environment where latency matters. The more middleware we add to an application, the longer calls take. The same is true about running tools in the kernel space.

When running a program on Kubernetes, or anywhere really, we need to think about security. Do we trust this program to have access to read everything we do..
We should always provision with the least amount of access as possible. Please do understand [Pod Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) before implementing any of these solutions. 

Do also read through the [eBPF Security Threat Model](https://www.linuxfoundation.org/hubfs/eBPF/ControlPlane%20%E2%80%94%20eBPF%20Security%20Threat%20Model.pdf?utm_source=the+new+stack&utm_medium=referral&utm_content=inline-mention&utm_campaign=tns+platform) written by Jack Kelly, James Callaghan, and Andrew Martin. It is a great primer, and really helps you understand eBPF.


## What is eBPF
eBPF stand for Extended Berkley Packet Filter. It came out as available in the linux kernel 3.18 and basically extended the existing Berkley Packet Filter. The existing BPF implementation was used to only filter and capture network packets. This tool lived in the kernel space making it fast, and not as accessible for the user.

eBPF made it possible for users to configure small programs to run on a lot of different hooks, such as:
- System Calls
- Kernel Functions
- Network Events
This basically means that for most events, eBPF can be configured to run a program in the kernel space. These programs can modify the events, or they can simply record the events, which we see often in the observability space. 

eBPF extended the Linux Kernel to the user, allowing us to configure low level programs to collect, record, and alter data.

## Why run programs in the kernel space?

The programs ran here operate in a sandboxed environment where their execution is verified prior to execution. This helps ensure that the program is not going to crash the kernel. These programs are closer to the source that is handling the events, which means handling the events here will provide much better latency and response times. eBPF programs can also be loaded without restarting a node or the kernel, meaning programs can be dynamically added to the kernel space while the system is running.

eBPF really shines due to it's low level of execution and it being triggered on tons of hooks into the kernel. This makes it an ideal candidate to provide low level observability into how a system is performing, it's networking calls, and the security for the system. Some of the core things that eBPF can monitor are:
- System Calls
- Filesystem Activity
- Processes
- Security Auditing
- Network Traffic

We can think of eBPF as a magical tool that essentially provides us with x-ray vision into the linux host. 

### Why use eBPF?

Well, monitoring of course! Observability, at the lowest level. There is less latency and we can observe events at the lowest level. This is incredible when we are looking at it from a security perspective. We can mitigate and observe filesystem modification, processes, system calls and network traffic as it operates on the host.

It can provide packet filtering and the ability to proactively drop packets that could otherwise  be malicious. Other eBPF programs give us deep application insights all the way to the system layer. 

Tools like [Cilium](https://cilium.io/use-cases/cni/) are replacing Kube-Proxy and use eBPF instead of iptables or nftables for super efficient load balancing. Keep in mind though, iptables lives in the userspace and nftables, it's replacement, also lives in the kernel space. It will be interesting to see some performance tests comparing eBPF to efficient use of nftables for load balancing. Although, I think we will likely always get "more" when using eBPF, just due to how we can monitor it and export the data to other services.

Tools like [parca] (https://www.parca.dev/) enable us to quickly find out if we are introducing a lot of latency to our application by understanding how our application is using the hosts memory, CPU, and IO. It helps bring to light inefficient system calls, like file.Open(), or maybe we are processing a byte array slowly in memory, and that is causing our latency for an application call to be 100ms higher then it needed to be. 



Within the next couple posts, I will add some tutorials to showcase how to build and run eBPF programs. Below is the Dockerfile that I am using. I'll re-post it, once I write up the tutorial though!
### Dockerfile for eBPF development


```
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    build-essential \
    clang \
    llvm \
    libelf-dev \
    linux-headers-generic \
    pkg-config \
    git \
    curl \
    vim \
    libbpf-dev \
    ca-certificates && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

ENV GO_VERSION=1.21.1
RUN curl -LO https://golang.org/dl/go${GO_VERSION}.linux-amd64.tar.gz && \
    tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz && \
    rm go${GO_VERSION}.linux-amd64.tar.gz
ENV PATH="/usr/local/go/bin:${PATH}"
ENV GOARCH=arm64

RUN ln -s /usr/include/asm-generic /usr/include/asm
```

We do have to run this as privileged so it will have access to the necessary system resources.
```
docker build . -t ebpf-dev-container
docker run --privileged --rm -it -v $(pwd):/app -w /app ebpf-dev-container
```
