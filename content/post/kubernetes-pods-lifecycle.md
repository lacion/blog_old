---
author: Luis Morales
date: '2017-05-01T01:00:00+02:00'
linktitle: Kuberetes pods lifecycle
title: Kuberetes pods lifecycle
tags:
- kubernetes
- docker
- engineering
- development
categories:
- Kubernetes
- Docker
- Development

---
Kubernetes pods can be terminated any time, due to an auto-scaling policy or when rolling out an update. In most of such cases, you will probably want to control how to shutdown your application running inside the containers within the pods.

***Kubernetes Pod Termination***

When kubernetes terminates a pod a number of things happen:

* A `SIGTERM` signal is sent to the main process in each container, and a “grace period” countdown starts.
* if a pod has a `preStop hook` its invoked inside the container.
* If a container doesn’t terminate within the grace period, a `SIGKILL` signal will be sent and the container.

> By default, all deletes are graceful within 30 seconds. The kubectl delete command supports the --grace-period=seconds option which
> allows a user to override the default and specify their own value. The value 0 force deletes the pod. In kubectl version >= 1.5, you
> must specify an additional flag --force along with --grace-period=0 in order to perform force deletions.

you can find more detailed info on pod termination in the [kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods)

***Enter Kubernetes Lifecycle Hooks***

Kubernetes allow us to run commands just after the container has started, as well as before it stops, this allow us to initialize and cleanup our containers effectively.

the 2 hooks we have available are:

`PostStart`:

> This hook executes immediately after a container is created. However, there is no guarantee that the hook will execute before the container ENTRYPOINT. No parameters are passed to the handler.

`PreStop`:

> This hook is called immediately before a container is terminated. It is blocking, meaning it is synchronous, so it must complete before the call to delete the container can be sent. No parameters are passed to the handler.

you can find more detailed info on container lifecycle and troubleshooting them in the [kubernetes documentation](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)

***Getting our Hands Dirty***

one example where `preStop` hooks work really well is when deploying consul to kubernetes, when you stop a consul node you need to leave the cluster before you terminate the pod if not you will need to adjust the cluster manually.

Here is an example `PetSet` (`StatefulSet`) for running a consul cluster:
```
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: consul
spec:
  serviceName: consul
  replicas: 3
  template:
    metadata:
      labels:
        app: consul
    spec:
      terminationGracePeriodSeconds: 10
      securityContext:
        fsGroup: 1000
      containers:
        - name: consul
          image: "consul:0.8.1"
          lifecycle:
            preStop:
              exec:
                command: ["consul","leave"]
          env:
            ...
          args:
            ...
          volumeMounts:
            ...
          ports:
            ...
      volumes:
        ...
  volumeClaimTemplates:
    ...
```

There is a couple of important things in here.
* terminationGracePeriodSeconds: 10: this is the time kubernetes will wait before sending `SIGKILL` to forcefully terminate the container if it did not stopped by itself.
* `lifecycle: preStop: exec: command: ["consul","leave"]` will make sure the node leaves the cluster cleanly.

this hooks are essential if you wan to remove pods cleanly in your cluster to minimize impact, like draining connections before shutdown.

Even though im not using `PostStart` scripts here, with some shell wizzardy we could talk to the kubernetes api to gather info on the current consul deployed pods to join them them making this hole setup fully automated.