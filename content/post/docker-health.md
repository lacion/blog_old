---
author: Luis Morales
date: '2017-05-28T01:00:00+02:00'
linktitle: Is your Docker Healthy?
title: Is your Docker Healthy?
tags:
- docker
- engineering
- development
categories:
- Docker
- Development

---

Since version [1.12 of Docker](https://blog.docker.com/2016/06/docker-1-12-built-in-orchestration/) we now have a new Dockerfile syntax to make our containers aware of the health of our running application using [HEALTHCHECK instruction in a Dockerfile](https://docs.docker.com/engine/reference/builder/#healthcheck) this is perfect for a container-specific way to determine readiness.

Apart from exit codes, Docker doesn’t know much about the internal workings of containerized applications. When `docker run` is invoked from the command-line, it often starts a single process specified using the CMD instruction in a Dockerfile. That determines the result of the `STATUS` column in the `docker ps` command.

```
$ docker ps -a
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS                    PORTS                    NAMES
2aec3088ea60        xxxxxx/xxxxxx:26c89cd                ""                       5 days ago          Exited (1) 5 days ago                              elastic_noyce
5dcf21d2cab6        xxxxxx/xxxxxx:26c89cd                ""                       5 days ago          Exited (1) 5 days ago                              infallible_clarke
d9625497e99c        quay.io/coreos/clair-git:latest      "/clair -config /c..."   5 days ago          Exited (137) 5 days ago                            clair_clair
1a748ee579d7        postgres:latest                      "docker-entrypoint..."   5 days ago          Exited (0) 5 days ago                              clair_postgres
cbcb5909800a        keyvanfatehi/sinopia:latest          "/opt/sinopia/star..."   7 weeks ago         Up 2 days                 0.0.0.0:4873->4873/tcp   sinopia
```

This indicates that some containers are running while others have exited, either successfully or with an error `Exited (1)`. Unfortunately, even applications that show a status of “Up 2 days” may serve 500 errors or could be stuck in an infinite loop. Adding HEALTHCHECK to the container Dockerfile address the issue.

Health are a single command. They run inside the container and if they exit with code 0, the container is reported as healthy, and if theey exit with code 1, the container is marked as unhealthy, interval `--interval=DURATION (default: 30s)`, timeout `--timeout=DURATION (default: 30s)` and number of retries `--timeout=DURATION (default: 30s)` can be spesified control how healthchecks are executed. There can only be one HEALTHCHECK instruction in a Dockerfile. If you list more than one then only the last HEALTHCHECK will take effect.

A very basic example of an HTTP endpoint healthcheck could look like:

```
HEALTHCHECK --interval=1m --timeout=5s CMD curl -f http://localhost/healthz || exit 1
```

Any output text that the command writes TO stdout or stderr will be stored in the health status and can be accesed using `docker inspect`. When the health status of a container changes, a health_status event is generated with the new status.

It is very important to note here that [kubernetes does not currently support the native Docker HEALTHCHECK](https://github.com/kubernetes/kubernetes/issues/25829) and will not report the status of the container to the api, if you want to check health on Kubernetes refer to the docs on [Configuring Liveness and Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)