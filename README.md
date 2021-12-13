# https://github.com/mainnika/ghost-docker

This is a fork of the official repo of the [Docker "Official Image"](https://github.com/docker-library/official-images#what-are-official-images) for [`ghost`](https://hub.docker.com/_/ghost/).

The image in this repository is based on Red Hat Universal Base Image 8 ["registry.access.redhat.com/ubi8/ubi"](https://catalog.redhat.com/software/containers/ubi8/ubi/5c359854d70cc534b3a3784e).

To make this image manually please specify `ghost` and `ghost-cli` versions using build-args:
```
$ DOCKER_BUILDKIT=1 docker build --build-arg GHOST_VERSION=4.27.2 --build-arg GHOST_CLI_VERSION=1.18.1 -t ghost .
```

# get container image using registry

> see available tags at https://github.com/mainnika/ghost-docker/pkgs/container/ghost

```
$ docker pull ghcr.io/mainnika/ghost:4-ubi
```
