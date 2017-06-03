---
author: Luis Morales
date: '2017-06-03T01:00:00+02:00'
linktitle: Are your kubernetes containers alive and ready?
title: Are your kubernetes containers alive and ready?
tags:
- docker
- engineering
- development
- kubernetes
categories:
- kubernetes
- Docker
- Development

---
In my last post [Is your Docker Healthy?]({{< ref "post/docker-health.md" >}}) i talked about the `HEALTHCHECK` instruction in Dockerfile that you could use to know the status of your application running in docker, however kubernetes knows nothing about `HEALTHCHECK`, instead it adds a couple of instructions you can define in your POD called `Readiness` and `Liveness` probes.

What’s a `Liveness` probe you say? kubernetes can run a command inside your container to figure out if your aplication is still working correctly and if it’s not it will restart the container for you automatically, you will know this happens because an event will be created telling you the `Liveness` probe failed and your container got restarted also the kubernetes restart count will get increased by this.

```
NAME                                READY     STATUS    RESTARTS   AGE
deity-2458969935-j8t5w              2/2       Running   4          10d
```

This behavior becomes super handy if you want to autoheal whatever issue that may arise to avoid affecting your service, this probes look very similar to Docker `HEALTHCHECK` instructions and have similar parameters, `periodSeconds` is how often kubernetes will run this check and `initialDelaySeconds` is the time after the container started that kubernetes will wait before running your probe command.

```
livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

if your service has an HTTP health check endpoint (and if your service has HTTP you should have a healthcheck endpoint) you can define that too in your pod spec.

```
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
          - name: Authorization
            value: Bearer xxxxx
      initialDelaySeconds: 5
      periodSeconds: 5
```

TCP Example:

```
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

if you use named ports in your container you can also reference them in your probes

```
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
  path: /healthz
  port: liveness-port
```


In the other hand a `Readiness` probe wont restart the container when they fail, but they will remove the POD from the loadbalancer (service) endpoint list, this is super useful if you want to keep HA, until the POD `Readiness` starts succeeding kubernetes wont sent traffic to it.

Configuring a `Readiness` is the same as a `Liveness`, you can run a CMD, an http check or TCP.

```
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

In addition, all probes have the following parameters

- `initialDelaySeconds` the number of second to wait after the container has started to run the probe.
- `periodSeconds` how often the probe should run.
- `timeoutSeconds` how much to wait before marking the probe as timed out.
- `successThreshold` the number of times the probe needs to succeed before marking the container as healthy.
- `failureThreshold` the number of times the probe needs to fail before marking the container unhealthy.

and HTTP probes expose a few extra ones:

- `host` hostname to connect to, it defaults to container ip.
- `scheme` how to connect to the endpoint HTTP|HTTPS.
- `path` uri to your healthcheck endpoint (/healthz for example).
- `httpHeaders` a list of extra headers to pass when requesting health.
- `port` the port to send the request to.

Using a combinations of `Liveness` and `Readiness` probes means you can have zero downtime deployments inside kubernetes and minimize impact from failures using `Liveness` autoheal capabilities, its also very important to note that `Liveness` probe will honour the [Restart Policy](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) defined in the podspec.

My generic healthcheck in Golang microservices looks like:

```
func (s *service) Health() HealthStatus {  
    
    memStatus := "OK"
    if err := s.memUsage.Health(); err != nil {
        memStatus = fmt.Sprintf("Memory ERROR: %s", err)
    }
    
    diskStatus := "OK"
    if err := s.memUsage.Health(); err != nil {
        diskStatus = fmt.Sprintf("Disk ERROR: %s", err)
    }

    mongoStatus := "OK"
    if err := s.mongo.Health(); err != nil {
        mongoStatus = fmt.Sprintf("Mongo ERROR: %s", err)
    }

    elasticStatus := "OK"
    if err := s.elastic.Health(); err != nil {
        elasticStatus = fmt.Sprintf("Elastic ERROR: %s", err)
    }

    return HealthStatus{
        Memory: memStatus,
        Disk: diskStatus,
        Mongo: mongoStatus,
        Elastic: elasticStatus,
    }
}
```

Off course this varies depending on what the services need to do.