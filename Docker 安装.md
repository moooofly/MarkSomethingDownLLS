# Docker 安装

NOTE: last update 2019-10-24

> 参考：[Get Docker CE for Ubuntu](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/)

## OS requirements of Docker CE

To install Docker CE, you need the 64-bit version of one of these Ubuntu versions:

- Disco 19.04
- Cosmic 18.10
- Bionic 18.04 (LTS)
- Xenial 16.04 (LTS)

## Uninstall old versions

Older versions of Docker were called `docker` or `docker-engine`. If these are installed, uninstall them:

```
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

The Docker CE package is now called `docker-ce`.

The contents of `/var/lib/docker/`, including images, containers, volumes, and networks, are preserved. The **Docker Engine - Community** package is now called `docker-ce`.


> **If you need to use `aufs`**
>
> Docker CE now uses the `overlay2` storage driver by default, and it is recommended that you use it instead of `aufs`. If you need to use `aufs`, you will need to do additional preparation.
>
> **XENIAL 16.04 AND NEWER**
>
> For Ubuntu 16.04 and higher, the Linux kernel includes support for **OverlayFS**, and Docker CE will use the `overlay2` storage driver by default. If you need to use `aufs` instead, you need to configure it manually.

## Install Docker CE

You can install Docker CE in different ways, depending on your needs:

- Most users set up Docker’s repositories and install from them, for ease of installation and upgrade tasks. This is the recommended approach.
- Some users download the DEB package and install it manually and manage upgrades completely manually. This is useful in situations such as installing Docker on air-gapped systems with no access to the internet.
- In testing and development environments, some users choose to use automated convenience scripts to install Docker.

> 推荐第一种

Before you install Docker CE for the first time on a new host machine, you need to set up the Docker repository. Afterward, you can install and update Docker from the repository.

构建 Docker repository ；

```
# Update the apt package index
$ sudo apt-get update

# Install packages to allow apt to use a repository over HTTPS
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

# Add Docker’s official GPG key
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Verify that you now have the key with the fingerprint 9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88, by searching for the last 8 characters of the fingerprint.
$ sudo apt-key fingerprint 0EBFCD88

# Use the following command to set up the stable repository.
# for amd64 only
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

安装 Docker CE ；

```
# Update the apt package index.
$ sudo apt-get update

# Install the latest version of Docker CE, or go to the next step to install a specific version. Any existing installation of Docker is replaced.
$ sudo apt-get install docker-ce docker-ce-cli containerd.io

# On production systems, you should install a specific version of Docker CE instead of always using the latest. This output is truncated. List the available versions.
$ apt-cache madison docker-ce
The contents of the list depend upon which repositories are enabled. Choose a specific version to install.
$ sudo apt-get install docker-ce=<VERSION>

# Verify that Docker CE is installed correctly by running the hello-world image.
$ sudo docker run hello-world
```

## 脚本

Ref: https://github.com/moooofly/scaffolding/blob/master/docker_setup.sh
