---
author: "Luis Morales"
date: 2017-02-14
tags: [ "docker", "engineering", "development" ]
categories:
  - "Docker"
  - "Development"
linktitle: Tiny Docker Images
title: Tiny Docker Images
---

One of my favorite aspects of docker is the lightweight nature of deployment, having just the bare bones needed to run whatever is needed to make your code run. The problem is that many running containers include more than what is absolutely necessary for the application to run.

Given this i embrace Alpine Linux quickly, it's slim and it's bare bones nature meant i could build and manage my dependencies more granular, and using a build container and then a release container make my final images very small, so learning Alpine quirks was very much worth it.

Here is an example build container i typically use:

```
FROM lacion/docker-alpine:gobuildimage

LABEL app="build-foo"
LABEL REPO="https://github.com/lacion/foo"

ENV GOROOT=/usr/lib/go \
    GOPATH=/gopath \
    GOBIN=/gopath/bin \
    PROJPATH=/gopath/src/github.com/lacion/foo

# Because of https://github.com/docker/docker/issues/14914
ENV PATH=$PATH:$GOROOT/bin:$GOPATH/bin

WORKDIR /gopath/src/github.com/lacion/foo

CMD ["make","build-alpine"]
```

Using a build image means the final image looks very clean (and slim):

```
FROM lacion/docker-alpine:latest

ARG GIT_COMMIT
ARG VERSION
LABEL REPO="https://github.com/lacion/foo"
LABEL GIT_COMMIT=$GIT_COMMIT
LABEL VERSION=$VERSION

# Because of https://github.com/docker/docker/issues/14914
ENV PATH=$PATH:/opt/iothub/bin

WORKDIR /opt/foo/bin

COPY bin/foo /opt/foo/bin/
RUN chmod +x /opt/foo/bin/foobin

CMD /opt/foo/bin/foobin
```

For convenience we can use a [Makefile]({{< ref "post/makefile-is-awesome.md" >}}) to run the whole process to make and push our final container.

Base container images:
```
lacion/docker-alpine   latest              fb569f137457        5 weeks ago         14.1MB
lacion/docker-alpine   gobuildimage        6d3db5cde54c        5 weeks ago         356MB
```
Build Container:
```
lacion/foo           build              812d5b3925d9        About an hour ago   422MB
```

Final Container
```
lacion/foo          local               a57a8903c382        About an hour ago   36.5MB
```

In conclusion adding a small extra step to your build process means you can slim down your instances make it faster to handle and build, with the added benefit that reducing the number of things installed in your container makes management (dependencies) easier and easier to secure.