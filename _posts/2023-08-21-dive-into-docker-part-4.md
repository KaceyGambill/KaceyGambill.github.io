---
layout: post
title:  "Dive Into Docker part 4: Inspecting Docker Image layers"
date:   2023-08-12 00:00:00 +0000
categories: docker
---
This post is going to be shorter. I'd like to highlight a tool that I really 
enjoy working with called "[Dive](https://github.com/wagoodman/dive)" 
It is an essential tool when working to build and optimize docker containers.

Dive is a an essential tool when building or inspecting Dockerfiles. This tool can help pinpoint exactly what is contained in each layer of the Dockerfile. Specifically it 
quickly combes through the Dockerfile and tries to show wasted space.

![dive image wasted space](/img/image-details-1.png)

## Installing Dive
[installation-instructions](https://github.com/wagoodman/dive#installation)
My preferred way to install Dive, if using a mac, is to use brew: -- `brew install dive`

## Using Dive
I prefer to use dive during local development of Docker containers. To get started
I typically just run: `dive image-name` if the image is not found locally this 
will take care of pulling the image from the remote repository. 
**Note**: tmux keybindings will get in the way, I usually detach from tmux or
open another terminal session before using `dive`

Running `dive ruby:3.2.0`
![dive ruby:3.2.0](/img/dive-ruby-3-2-0.png)
It first pulls the image if it is not found locally, and then we are presented
with "Layers", "Layer Details", "Image Details" and "Current Layer Contents".

Press <Tab> to move between views.
In each view, it presents us with a few more hotkeys that we can use to further
inspect this image. 

Looking at the "Layers" tab, it presents us with either "layer changes" or
"aggregated changes" on the right-hand side.
You can press either <ctrl+l> or <ctrl+a> to switch between these two. 

Before moving to the "Layer Contents" view, I like to pick through the various
"Layer Details" right below "Layers"
![dive ruby:3.2.0, layer details](/img/layer-details-1.png)
Here it shows the command that was run to generate that layer. 
On the right-hand side of the screen we can see "Current Layer Contents", this
includes the details of the files that were added, removed, permissions on those
files and how much space these files are taking up. If we <tab> over to that 
view, it presents a few new options:
- <space> - collapse single dir
- <ctrl+space> - collapse all dir's
- <ctrl+a> Added
- <ctrl+r> Removed
- <ctrl+m> modified
- <ctrl+u> unmodified
- <ctrl+b> attributes
- <ctrl+p> wrap

I prefer to start out by collapsing all dir's and then start digging into the
layers that are showing the largest increase in file space. 

![dive ruby:3.2.0, current layer details](/img/current-layer-contents-1.png)


## Using Dive in a Continuous integration Pipeline

Running Dive with `CI=true` is one of the most effective ways to quickly find
wasted space. Example: `CI=true dive ruby:3.2.0` This also is something that could be plugged into a docker image
pipeline to ensure that a ridiculous amount of assets or image space is not
wasted.

Full output here: 
```bash
CI=true dive ruby:3.2.0
  Using default CI config
Image Source: docker://ruby:3.2.0
Fetching image... (this can take a while for large images)
Analyzing image...
  efficiency: 98.8316 %
  wastedBytes: 11616315 bytes (12 MB)
  userWastedPercent: 1.6002 %
Inefficient Files:
Count  Wasted Space  File Path
    6        5.0 MB  /var/cache/debconf/templates.dat
    4        3.2 MB  /var/cache/debconf/templates.dat-old
    6        1.2 MB  /var/lib/dpkg/status
    6        1.2 MB  /var/lib/dpkg/status-old
    5        376 kB  /var/log/dpkg.log
    5        194 kB  /var/log/apt/term.log
    6         95 kB  /etc/ld.so.cache
    6         86 kB  /var/cache/debconf/config.dat
    6         71 kB  /var/lib/apt/extended_states
    5         54 kB  /var/cache/ldconfig/aux-cache
    5         52 kB  /var/log/apt/eipp.log.xz
    4         42 kB  /var/cache/debconf/config.dat-old
    5         36 kB  /var/log/apt/history.log
    4         26 kB  /var/log/alternatives.log
    2         903 B  /etc/group
    2         892 B  /etc/group-
    2         756 B  /etc/gshadow
    2           0 B  /etc/.pwd.lock
    6           0 B  /tmp
    5           0 B  /var/cache/apt/archives/partial
    3           0 B  /var/lib/dpkg/triggers/Unincorp
    6           0 B  /var/lib/dpkg/lock-frontend
    5           0 B  /var/cache/apt/archives/lock
    6           0 B  /var/lib/dpkg/lock
    6           0 B  /var/cache/debconf/passwords.dat
    5           0 B  /var/lib/apt/lists
    2           0 B  /usr/src
    6           0 B  /var/lib/dpkg/triggers/Lock
    6           0 B  /var/lib/dpkg/updates
Results:
  PASS: highestUserWastedPercent
  SKIP: highestWastedBytes: rule disabled
  PASS: lowestEfficiency
Result:PASS [Total:3] [Passed:2] [Failed:0] [Warn:0] [Skipped:1]
```
With this particular image we could go through and remove those files, but in 
this case it does not take up a significant amount of room, so it is 
unnecessary.
[more configuration options](https://github.com/wagoodman/dive#ci-integration)


## Dealing with Sensitive Data

Do not pass sensitive details through build-arg's and environment variables
into dockerfiles during image creation. Simply inspecting the resulting docker
image layers will expose these secrets.

If a Dockerfile needs sensitive data, pass it using buildx secrets mounts.

This can be done either with a file, containing the secret value, or an environment
variable containing the secret. 

First Create a file, named `build_key` with the value 
`xyz:xyz`

Next add this to the Dockerfile to access the secret.
```Dockerfile
RUN --mount=type=secret,id=build_key
# to access the secret:
RUN echo "using build_key: $(cat /run/secrets/build_key)" # note this is an example

```
Finally when running the docker build with buildx we use the secret:
`docker buildx build --secret id=build_key,src=build_key .`

If we were to use an environment variable containing the secret the command to
build would look like:
```bash
build_key=xyz:xyz
docker buildx build --secret id=build_key,env=build_key
```


### Example of Secrets Leaking

This is a rough example, because we would likely never need to add the db
connection string at build time, but there are a few apps that require a
build_license when installing packages, or a way to authenticate to a remote 
GitHub  server..

```Dockerfile
FROM ubuntu

ARG build_license \
    postgres_db_string

ENV build_license=$build_license \
    postgres_db_string=$postgres_db_string

COPY . .

CMD echo "secret_sauce: $secret_sauce" \
 && echo "build_license: $build_license" \
 && echo "postgres_db_string: $postgres_db_string"

```
To see these details in an image, all that is needed is the image to exist
locally, then run: `docker save <image-name> -o <image.tar>` then from inspecting
the tar archive with `vim` I can see the layer contents.
```
" tar.vim version v32
" Browsing tarfile /Users/kaceygambill/personal/ubuntu-mount/blog/4/test.tar
" Select a file with cursor and press ENTER

444f68a42c829ead4bff4566c6554c761e2075c92d2eef50cbb9152fde8b13cc/
444f68a42c829ead4bff4566c6554c761e2075c92d2eef50cbb9152fde8b13cc/VERSION
444f68a42c829ead4bff4566c6554c761e2075c92d2eef50cbb9152fde8b13cc/json
444f68a42c829ead4bff4566c6554c761e2075c92d2eef50cbb9152fde8b13cc/layer.tar
a93a4c1e4d72d16b55e6aae767bb48e862a4ad8a43ab33107f8d5dfdc749912b.json
ee72d37eae4759eeaadd189b4341c0418faa7662ebc5089ddb528b4640e08c2f/
ee72d37eae4759eeaadd189b4341c0418faa7662ebc5089ddb528b4640e08c2f/VERSION
ee72d37eae4759eeaadd189b4341c0418faa7662ebc5089ddb528b4640e08c2f/json
ee72d37eae4759eeaadd189b4341c0418faa7662ebc5089ddb528b4640e08c2f/layer.tar
manifest.json
repositories
```
Looking at any one of those .json files gives us more details about the layer.
Expanding `444f68a42c829ead4bff4566c6554c761e2075c92d2eef50cbb9152fde8b13cc/json`
I can see a JSON object that includes the sensitive data.


If you haven't checked out [Dive](https://github.com/wagoodman/dive), I'd highly
suggest checking it out and implementing it as a check in a CI/CD pipeline!
