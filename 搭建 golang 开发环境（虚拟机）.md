# 搭建 golang 开发环境（虚拟机）

## 创建目录

```
mkdir ubuntu-17.04-server-cloudimg-amd64-vagrant
mkdir ubuntu-16.04-server-cloudimg-amd64-vagrant
mkdir -p viniciusfs/centos7
mkdir -p samdoran/rhel7
```

## vagrant box add

先下载到本地，后安装

```
wget https://cloud-images.ubuntu.com/releases/zesty/release/ubuntu-16.04-server-cloudimg-amd64-vagrant.box
vagrant box add ubuntu-16.04-server-cloudimg-amd64-vagrant https://cloud-images.ubuntu.com/releases/16.04/release/ubuntu-16.04-server-cloudimg-amd64-vagrant.box


wget https://cloud-images.ubuntu.com/releases/zesty/release/ubuntu-17.04-server-cloudimg-amd64-vagrant.box
vagrant box add ubuntu-17.04-server-cloudimg-amd64-vagrant https://cloud-images.ubuntu.com/releases/17.04/release/ubuntu-17.04-server-cloudimg-amd64-vagrant.box
```

直接网络安装

```
vagrant box add viniciusfs/centos7 https://atlas.hashicorp.com/viniciusfs/boxes/centos7/

vagrant box add samdoran/rhel7 https://vagrantcloud.com/samdoran/boxes/rhel7
```


## vagrant init


```
vagrant init ubuntu-16.04-server-cloudimg-amd64-vagrant
vagrant init ubuntu-17.04-server-cloudimg-amd64-vagrant
vagrant init viniciusfs/centos7
vagrant init samdoran/rhel7
```

该命令会创建 Vagrantfile 文件（默认配置）；若有自己的 Vagrantfile 文件，建议先备份一下；

## 调整 Vagrantfile 配置


```
➜  ubuntu-17.04-server-cloudimg-amd64-vagrant cat Vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box = "ubuntu-17.04-server-cloudimg-amd64-vagrant"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  config.vm.network "private_network", ip: "11.11.11.11"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL
end
```

主要调整

- config.vm.network "private_network", ip: "aa.bb.cc.dd"
- config.vm.synced_folder "../data", "/vagrant_data"


## alias 设置

建议在 ~/.zshrc 中配置几个 alias 简化操作：

```
alias vup='vagrant up'
alias vdown='vagrant halt'
alias vssh='vagrant ssh'
alias vreboot='vagrant reload'
```

## 系统配置

- 测试 `/vagrant_data` 共享目录是否可用
- 设置 root 账户（Ubuntu 安装后默认并没有给 root 用户设置口令，也没有启用 root 帐户，需要自行启用）

```
$ sudo passwd root         # 设置 root 用户密码
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
$
$ su -                    # 切换成 root 用户
Password:
#
```

- 修改 hostname

```
# vi /etc/hostname
```

修改后，需要通过 vreboot 重启虚拟机才能生效；

- 升级系统

```
apt-get -y update
```

- 配置 golang 开发环境

参照[这里](https://github.com/moooofly/dockerfiles/blob/master/go_develop_env/Dockerfile)进行安装；