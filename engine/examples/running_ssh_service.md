---
description: Installing and running an SSHd service on Docker
keywords: docker, example, package installation,  networking
title: Dockerize an SSH service
---

## Build an `eg_sshd` image

### Generate a secure root password for your image

Using a static password for root access is dangerous. Create a random password before proceeding.

### Build the image

The following `Dockerfile` sets up an SSHd service in a container that you
can use to connect to and inspect other container's volumes, or to get
quick access to a test container. Make the following substitutions:

- With `RUN echo 'root:THEPASSWORDYOUCREATED' | chpasswd`, replace "THEPASSWORDYOUCREATED" with the password you've previously generated.
- With `RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config`, use `without-password` instead of `prohibit-password` for Ubuntu 14.04.

```Dockerfile
FROM ubuntu:16.04

RUN apt-get update && apt-get install -y openssh-server
RUN mkdir /var/run/sshd
RUN echo 'root:THEPASSWORDYOUCREATED' | chpasswd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# SSH login fix. Otherwise user is kicked off after login
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

ENV NOTVISIBLE "in users profile"
RUN echo "export VISIBLE=now" >> /etc/profile

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```


Build the image using:

```bash
$ docker build -t eg_sshd .
```
## Run a `test_sshd` container

Then run it. You can then use `docker port` to find out what host port
the container's port 22 is mapped to:

```bash
$ docker run -d -P --name test_sshd eg_sshd
$ docker port test_sshd 22

0.0.0.0:49154
```

And now you can ssh as `root` on the container's IP address (you can find it
with `docker inspect`) or on port `49154` of the Docker daemon's host IP address
(`ip address` or `ifconfig` can tell you that) or `localhost` if on the
Docker daemon host:

```bash
$ ssh root@192.168.1.2 -p 49154
# or
$ ssh root@localhost -p 49154
# The password is ``screencast``.
root@f38c87f2a42d:/#
```

## Environment variables

Using the `sshd` daemon to spawn shells makes it complicated to pass environment
variables to the user's shell via the normal Docker mechanisms, as `sshd` scrubs
the environment before it starts the shell.

If you're setting values in the `Dockerfile` using `ENV`, you need to push them
to a shell initialization file like the `/etc/profile` example in the `Dockerfile`
above.

If you need to pass`docker run -e ENV=value` values, you need to write a
short script to do the same before you start `sshd -D` and then replace the
`CMD` with that script.

## Clean up

Finally, clean up after your test by stopping and removing the
container, and then removing the image.

```bash
$ docker container stop test_sshd
$ docker container rm test_sshd
$ docker image rm eg_sshd
```
