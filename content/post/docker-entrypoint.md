---
author: Luis Morales
date: '2017-05-03T01:00:00+02:00'
linktitle: Docker Entrypoint
title: Docker Entrypoint
darft: true
tags:
- docker
- engineering
- development
categories:
- Docker
- Development

---
As you should know, Docker runs the entrypoint for your container as `PID 1` this means a lot of different things, one of which is explained in detail [here](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/) and [here](https://www.fpcomplete.com/blog/2016/10/docker-demons-pid1-orphans-zombies-signals)

It's important to know that the Kernel treats `PID 1` in a special way, normally when you send `SIGTERM` to a process and your application does not have registered a handler for `SIGTERM` the kernel will fall back to default behavior (killing the process) however, if your application happens to be `PID 1` the kernel won't fallback to default behavior and sending `SIGTERM` would mean nothing at all.

For this reason is very useful to use some kind of init system, luckily for us the great Engineer at Yelp! have a project i used a lot called [dumb-init](https://github.com/Yelp/dumb-init) this is part of my [base docker image](https://github.com/lacion/Docker-alpine/blob/master/Dockerfile) and becomes the entry point for my dockerfiles

```
FROM lacion/docker-alpine:latest

ARG GIT_COMMIT
ARG VERSION
LABEL REPO="https://github.com/lacion/foo"
LABEL GIT_COMMIT=$GIT_COMMIT
LABEL VERSION=$VERSION

# Because of https://github.com/docker/docker/issues/14914
ENV PATH=$PATH:/opt/foo/bin

WORKDIR /opt/foo/bin

COPY bin/foo /opt/foo/bin/
RUN chmod +x /opt/foo/bin/foo

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["/opt/foo/bin/foo"]
```

one super useful feature of dumb-init is the ability to rewrite signals `--rewrite 15:3` rewriting `SIGTERM` (number 15) to `SIGQUIT` (number 3) for example this is useful in the case of nginx where if you're running in lets say Kubernetes means the scheduler will send a default `SIGTERM` signal but internally we will send the process `SIGQUIT` effectively doing a graceful shutdown.

all of this setup is super important when managing your Kubernetes pods lifecycles as making sure your applications are terminating in a way you need is crucial for running scalable docker containers.