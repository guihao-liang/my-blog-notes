---
layout: post
title: "Use docker-in-docker in CI/CD"
subtitle: "A brief intro to dind"
published: tr
date: 2020-05-04 16:00:44
bigimg: /img/cute_owl.jpg
tag: ["docker", "gitlab"]
---

## Dind

Recently, I need to upgrade docker version in `gitlab` runner and I get some exposure to the `dind` or `docker-in-docker` service.

```yaml
build_some_thing:
  tags:
    - docker
  stage: build
  services:
    - docker:18.09-dind
  image: docker:18.09
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_DRIVER: overlay2
    DOCKER_CERTS: ""
  script:
    - apk add bash
    - bash -e echo "this is an example"
  artifacts:
    expire_in: 1 day
    paths:
      - target/
```

From [gitlab-ci.yml reference](https://docs.gitlab.com/ee/ci/yaml/#services), `services` is a list of docker services that are linked with a base image of specified `image`. What does this mean?

Base image is from `image` keyword, which runs the CI jobs. `services` are other running docker containers that can be reached (linked) by the network. For example, the `dind` (docker-in-docker) services can be reached by what's set in `DOCKER_HOST`, which is `tcp://docker:2375`, whose `docker` part is the [service name][dind-access-name] for the `dind` service listed under `services` attribute, and it will be substituted with the `dind` service container's __IP__ address. The port 2375 is chosen to receive HTTP requests __without TLS encryption__. If HTTPS is needed, use port [2376][docker-port].

From [gitlab dind reference][dind-reference], one benefit is that each CI job has its completely independent docker engine,

> When using docker-in-docker, each job is in a clean environment without the past history. Concurrent jobs work fine because every build gets its instance of Docker engine so they won’t conflict with each other. But this also means that jobs can be slower because there’s no caching of layers.

In the base image container, when running `docker` command, it will use [`DOCKER_HOST`][docker-daemon] and send requests to the `dind` service to execute the docker commands.

![dind-and-base](/img/docker/dind.jpeg)

As shown above, only gitlab runner has direct access (privileged access) to the docker daemon, `dockerd` in above graph, which is running in the host machine. Thus, this CI/CD container instance can issue docker commands to the host machine to spawn and control new containers.

For `dind` container `C1`, it can issue docker command to its own `dockerd`, e.g., `docker build` or `docker image pull`, without sharing the host `dockerd`. In other words, `dind` provides an isolated and clean docker execution environment. Therefore, `C2` and `C3` uses the docker build context from `C1`. Besides that, `C2` or `C3` cannot send any docker command to intervene `C4`, which can prevent insecure issues such as `docker container stop $(docker ps -a)`.

The obvious drawback is the performance overhead caused by laying a virtualization on top of another virtualization.

## Alternative: bind-mount

As it is mentioned [here][bind-mount], this method can avoid nested docker in order to avoid some runtime overhead.

```bash
sudo gitlab-runner register -n \
  --url https://gitlab.com/ \
  --registration-token REGISTRATION_TOKEN \
  --executor docker \
  --description "My Docker Runner" \
  --docker-image "docker:19.03.8" \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock
```

The gitlab docker instance will shared the `/var/run/docker.sock` from host machine, which is the domain socket listened by docker daemon. What's more, compared with dind approach, it doesn't require the docker container to have privileged access to host docker engine. The runtime layout might look like below:

![image-bind-mount](/img/docker/bind-mount.jpeg)

In above graph, the `gitlab-runner` instance can control other containers or CI/CD jobs through networks or by issuing requests to host docker daemon, e.g., `docker container stop`.

> Notice that it’s using the Docker daemon of the Runner itself, and any containers spawned by Docker commands will be siblings of the Runner rather than children of the Runner.

All the newly spawn containers will have direct access to the host docker daemon so that they can invoke `docker` command within their container. The advantage is that it doesn't introduce extra virtualization overhead.

However, it has some drawbacks, as it is mentioned by [gitlab document][bind-mount]:

1. malicious user can know surrounding running containers: `docker rm -f $(docker ps -a -q)`.

2. jobs may create containers with name conflicts.

3. volume sharing context not based on the container but the host.

As for CI/CD purposes, `dind` doesn't have above drawbacks and it's a better choice since job isolation is the most critical criteria for concurrent jobs to run.

[dind-reference]: https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#making-docker-in-docker-builds-faster-with-docker-layer-caching
[dind-access-name]: https://docs.gitlab.com/ee/ci/docker/using_docker_images.html#accessing-the-services
[bind-mount]: https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-socket-binding
[docker-port]: https://github.com/docker-library/docker/blob/0bab8e3d0ebe6dc4b7a122bb1d0b2e017925c50d/19.03/dind/Dockerfile#L43
[docker-daemon]: https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-socket-option