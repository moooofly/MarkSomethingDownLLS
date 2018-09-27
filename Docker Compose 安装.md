# Docker Compose 安装

> 参考：[Install Docker Compose](https://docs.docker.com/compose/install/)

## Prerequisites

Docker Compose relies on **Docker Engine** for any meaningful work, so make sure you have Docker Engine installed either locally or remote, depending on your setup.

- On desktop systems like **Docker for Mac** and **Windows**, Docker Compose is included as part of those desktop installs.
- On **Linux** systems, first install the Docker for your OS as described on the Get Docker page, then come back here for instructions on installing Compose on Linux systems.

## Install Compose on Linux systems

On Linux, you can download the Docker [Compose binary from the Compose repository release page on GitHub](https://github.com/docker/compose/releases).

step by step instructions:

```
# Use the latest Compose release number in the download command.
$ sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

# Apply executable permissions to the binary
$ sudo chmod +x /usr/local/bin/docker-compose

# Optionally, install command completion for the bash and zsh shell.
# https://docs.docker.com/compose/completion/

# Test the installation
docker-compose --version
```

## 脚本

Ref: https://github.com/moooofly/scaffolding/blob/master/docker-compose_setup.sh