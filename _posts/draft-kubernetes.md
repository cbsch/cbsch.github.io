---
title:  "Learning Kubernetes"
tags: [kubernetes, docker]
tags: [kubernetes, docker]
author: Christopher Berg Schwanstrøm
---

# Containers

Before talking about Kubernetes, we need to understand what containers are. It's easy to think of them as some sort of virtualization technology, but that's not what containers are. A running container is in essence just an extremely isolated process or processes running on its host. This means that the process has no access to anything on the host operating system (by default), no network, no file system, nothing.

The container itself includes a full filesystem for a functional OS, typically extremely stripped down to just meet the requirements of the application that should run on it, and that is all that it sees. The containers filesystem is read-only, so the running container cannot write any data by default.

To make the container do useful work like persisting data or exposing a service on the network the operator needs to specifically make those things available to the container.

This can for example look like this:

```bash
docker run -p 2345:5432 -v /var/data:/var/lib/postgresql/data postgres
```

Here we tell the container runtime to expose port `5432` of the container on port `2345` of the host operating system. We also say that anything the container writes to `/var/lib/postgresql/data` should end up on `/var/data` on the host.

https://www.youtube.com/watch?v=8fi7uSYlOdc