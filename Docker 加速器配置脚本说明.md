# Docker 加速器配置脚本说明

DaoCloud 提供的用于帮助配置 Docker 加速器（Registry Mirror）的脚本

```
#!/usr/bin/env bash

# 该脚本用于配置 Docker 加速器
# 取自: https://get.daocloud.io/daotools/set_mirror.sh
# 用法: curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://99a63370.m.daocloud.io

set -e

# 需要一个入参
if [ -z "$1" ]
then
    echo 'Error: Registry-mirror url required.'
    exit 1
fi

MIRROR_URL=$1
lsb_dist=''

# 判定命令是否存在的实现
command_exists() {
    command -v "$@" > /dev/null 2>&1
}
if command_exists lsb_release; then
    # e.g. Ubuntu
    lsb_dist="$(lsb_release -si)"
    # e.g. 14.04
    lsb_version="$(lsb_release -rs)"
fi

# 若 lsb_release 命令不存在，则直接读取系统中相应的配置文件内容
if [ -z "$lsb_dist" ] && [ -r /etc/lsb-release ]; then
    # 这里调用了 source 命令
    # 1. 不会新开一个 sub-shell 来执行被调用的脚本，而是在同一个 shell 中执行
    # 2. 被调用的脚本中声明的变量和环境变量，都可以在主脚本中得到和使用
    # 由此可知，当 source 命令成功执行后，$DISTRIB_ID 和 $DISTRIB_RELEASE 被加载到当前主 shell 中
    lsb_dist="$(. /etc/lsb-release && echo "$DISTRIB_ID")"
    lsb_version="$(. /etc/lsb-release && echo "$DISTRIB_RELEASE")"
fi
if [ -z "$lsb_dist" ] && [ -r /etc/debian_version ]; then
    lsb_dist='debian'
fi
if [ -z "$lsb_dist" ] && [ -r /etc/fedora-release ]; then
    lsb_dist='fedora'
fi
if [ -z "$lsb_dist" ] && [ -r /etc/os-release ]; then
    lsb_dist="$(. /etc/os-release && echo "$ID")"
fi
if [ -z "$lsb_dist" ] && [ -r /etc/centos-release ]; then
    lsb_dist="$(cat /etc/*-release | head -n1 | cut -d " " -f1)"
fi
if [ -z "$lsb_dist" ] && [ -r /etc/redhat-release ]; then
    lsb_dist="$(cat /etc/*-release | head -n1 | cut -d " " -f1)"
fi


lsb_dist="$(echo $lsb_dist | cut -d " " -f1)"

# 版本号切割
docker_version="$(docker -v | awk '{print $3}')"
docker_major_version="$(echo $docker_version| cut -d "." -f1)"
docker_minor_version="$(echo $docker_version| cut -d "." -f2)"

# 这里正则使用的很风骚
lsb_dist="$(echo "$lsb_dist" | tr '[:upper:]' '[:lower:]')"

# 注意该函数中的 sudo 使用
set_daemon_json_file(){
    # Docker 版本在 1.12 或更高，支持使用 /etc/docker/daemon.json 文件
    DOCKER_DAEMON_JSON_FILE="/etc/docker/daemon.json"
    if sudo test -f ${DOCKER_DAEMON_JSON_FILE}
    then
        # 若之前就存在该文件，则进行 registry-mirrors 相关配置调整
        sudo cp  ${DOCKER_DAEMON_JSON_FILE} "${DOCKER_DAEMON_JSON_FILE}.bak"
        # 此处使用 -q 静默输出，使用 tee 命令创建修改后的文件
        if sudo grep -q registry-mirrors "${DOCKER_DAEMON_JSON_FILE}.bak";then
            sudo cat "${DOCKER_DAEMON_JSON_FILE}.bak" | sed -n "1h;1"'!'"H;\${g;s|\"registry-mirrors\":\s*\[[^]]*\]|\"registry-mirrors\": [\"${MIRROR_URL}\"]|g;p;}" | sudo tee ${DOCKER_DAEMON_JSON_FILE}
        else
            sudo cat "${DOCKER_DAEMON_JSON_FILE}.bak" | sed -n "s|{|{\"registry-mirrors\": [\"${MIRROR_URL}\"],|g;p;" | sudo tee ${DOCKER_DAEMON_JSON_FILE}
        fi
    else
        # 若之前不存在该文件，则创建该文件并添加 registry-mirrors 相关配置
        sudo mkdir -p "/etc/docker"
        sudo echo "{\"registry-mirrors\": [\"${MIRROR_URL}\"]}" | sudo tee ${DOCKER_DAEMON_JSON_FILE}
    fi
}


# 用于判定 docker 版本是否满足 1.12 或更高这个条件
can_set_json(){
	if [ "$docker_major_version" -eq 1 ] && [ "$docker_minor_version" -lt 12 ]
	then
		echo "docker version < 1.12"
		return 0
	else
		echo "docker version >= 1.12"
		return 1
	fi
}

# 针对不同发行版为 Docker Service 配置 Registry Mirror
set_mirror(){
    # docker 版本低于 1.9 不 ok
    if [ "$docker_major_version" -eq 1 ] && [ "$docker_minor_version" -lt 9 ]
        then
            echo "please upgrade your docker to v1.9 or later"
            exit 1
    fi

    case "$lsb_dist" in
        centos)
        # 此处使用 > /dev/null 静默输出
        if grep "CentOS release 6" /etc/redhat-release > /dev/null
        then
            DOCKER_SERVICE_FILE="/etc/sysconfig/docker"
            sudo cp ${DOCKER_SERVICE_FILE} "${DOCKER_SERVICE_FILE}.bak"
            # 直接使用 -i 选项原地修改文件
            sudo sed -i "s|other_args=\"|other_args=\"--registry-mirror='${MIRROR_URL}'|g" ${DOCKER_SERVICE_FILE}
            sudo sed -i "s|OPTIONS='|OPTIONS='--registry-mirror='${MIRROR_URL}'|g" ${DOCKER_SERVICE_FILE}
            echo "Success."
            echo "You need to restart docker to take effect: sudo service docker restart"
            exit 0
        fi
        if grep "CentOS Linux release 7" /etc/redhat-release > /dev/null
        then
            if can_set_json; then
                # 不能使用 /etc/docker/daemon.json 的情况
                DOCKER_SERVICE_FILE="/lib/systemd/system/docker.service"
                sudo cp ${DOCKER_SERVICE_FILE} "${DOCKER_SERVICE_FILE}.bak"
                sudo sed -i "s|\(ExecStart=/usr/bin/docker[^ ]* daemon\)|\1 --registry-mirror="${MIRROR_URL}"|g" ${DOCKER_SERVICE_FILE}
                # 注意该命令的执行
                sudo systemctl daemon-reload
            else
                # 能使用 /etc/docker/daemon.json 的情况
                set_daemon_json_file
            fi
            echo "Success."
            echo "You need to restart docker to take effect: sudo systemctl restart docker "
            exit 0
        else
            echo "Error: Set mirror failed, please set registry-mirror manually please."
            exit 1
        fi
    ;;
        fedora)
        if grep "Fedora release" /etc/fedora-release > /dev/null
        then
            if can_set_json; then
            DOCKER_SERVICE_FILE="/lib/systemd/system/docker.service"
            sudo cp ${DOCKER_SERVICE_FILE} "${DOCKER_SERVICE_FILE}.bak"
            sudo sed -i "s|\(ExecStart=/usr/bin/docker[^ ]* daemon\)|\1 --registry-mirror="${MIRROR_URL}"|g" ${DOCKER_SERVICE_FILE}
            sudo systemctl daemon-reload
            else
                set_daemon_json_file
            fi
            echo "Success."
            echo "You need to restart docker to take effect: sudo systemctl restart docker"
            exit 0
        else
            echo "Error: Set mirror failed, please set registry-mirror manually please."
            exit 1
        fi
    ;;
        ubuntu)
        # 这里又尝试了 `xxx` 的使用
        v1=`echo ${lsb_version} | cut -d "." -f1`
        if [ "$v1" -ge 16 ]; then
            # >= ubuntu 16.xx
            if can_set_json; then
                # 不能使用 /etc/docker/daemon.json 的情况
                DOCKER_SERVICE_FILE="/lib/systemd/system/docker.service"
                sudo cp ${DOCKER_SERVICE_FILE} "${DOCKER_SERVICE_FILE}.bak"
                sudo sed -i "s|\(ExecStart=/usr/bin/docker[^ ]* daemon -H fd://$\)|\1 --registry-mirror="${MIRROR_URL}"|g" ${DOCKER_SERVICE_FILE}
                sudo systemctl daemon-reload
            else
                # 能使用 /etc/docker/daemon.json 的情况
                set_daemon_json_file
            fi
            echo "Success."
            echo "You need to restart docker to take effect: sudo systemctl restart docker.service"
            exit 0
        else
            # < ubuntu 16.xx
            if can_set_json; then
                DOCKER_SERVICE_FILE="/etc/default/docker"
                sudo cp ${DOCKER_SERVICE_FILE} "${DOCKER_SERVICE_FILE}.bak"
                if grep "registry-mirror" ${DOCKER_SERVICE_FILE} > /dev/null
                then
                    sudo sed -i -u -E "s#--registry-mirror='?((http|https)://)?[a-zA-Z0-9.]+'?#--registry-mirror='${MIRROR_URL}'#g" ${DOCKER_SERVICE_FILE}
                else
                    echo 'DOCKER_OPTS="$DOCKER_OPTS --registry-mirror='${MIRROR_URL}'"' >> ${DOCKER_SERVICE_FILE}
                    echo ${MIRROR_URL}
                fi
            else
                set_daemon_json_file
            fi
        fi
        echo "Success."
        echo "You need to restart docker to take effect: sudo service docker restart"
        exit 0
    ;;
        debian)
        if can_set_json; then
            DOCKER_SERVICE_FILE="/etc/default/docker"
            sudo cp ${DOCKER_SERVICE_FILE} "${DOCKER_SERVICE_FILE}.bak"
            if grep "registry-mirror" ${DOCKER_SERVICE_FILE} > /dev/null
            then
                sudo sed -i -u -E "s#--registry-mirror='?((http|https)://)?[a-zA-Z0-9.]+'?#--registry-mirror='${MIRROR_URL}'#g" ${DOCKER_SERVICE_FILE}
            else
                echo 'DOCKER_OPTS="$DOCKER_OPTS --registry-mirror='${MIRROR_URL}'"' >> ${DOCKER_SERVICE_FILE}
                echo ${MIRROR_URL}
            fi
        else
            set_daemon_json_file
        fi
        echo "Success."
        echo "You need to restart docker to take effect: sudo service docker restart"
        exit 0
    ;;
        arch)
        if grep "Arch Linux" /etc/os-release > /dev/null
        then
            if can_set_json; then
                DOCKER_SERVICE_FILE="/lib/systemd/system/docker.service"
                sudo cp ${DOCKER_SERVICE_FILE} "${DOCKER_SERVICE_FILE}.bak"
                sudo sed -i "s|\(ExecStart=/usr/bin/docker[^ ]* daemon -H fd://\)|\1 --registry-mirror="${MIRROR_URL}"|g" ${DOCKER_SERVICE_FILE}
                sudo systemctl daemon-reload
            else
                set_daemon_json_file
            fi
            echo "Success."
            echo "You need to restart docker to take effect: sudo systemctl restart docker"
            exit 0
        else
            echo "Error: Set mirror failed, please set registry-mirror manually please."
            exit 1
        fi
    ;;
        suse)
        if grep "openSUSE Leap" /etc/os-release > /dev/null
        then
            if can_set_json; then
            DOCKER_SERVICE_FILE="/usr/lib/systemd/system/docker.service"
            sudo cp ${DOCKER_SERVICE_FILE} "${DOCKER_SERVICE_FILE}.bak"
            sudo sed -i "s|\(^ExecStart=/usr/bin/docker daemon -H fd://\)|\1 --registry-mirror="${MIRROR_URL}"|g" ${DOCKER_SERVICE_FILE}
            sudo systemctl daemon-reload
            else
                set_daemon_json_file
            fi
            echo "Success."
            echo "You need to restart docker to take effect: sudo systemctl restart docker"
            exit 0
        else
            echo "Error: Set mirror failed, please set registry-mirror manually please."
            exit 1
        fi
    esac
    echo "Error: Unsupported OS, please set registry-mirror manually."
    exit 1
}
set_mirror

```

## 从该脚本中可以学到

- 脚本入参判定：`-z "$1"`
- 命令是否存在的判定：`command -v`
- 针对 lsb_release 命令输出内容的解析，以及在 lsb_release 不存在时的处理策略；
- shell 中 source 命令的使用；
- cut 命令使用；
- sudo 使用场景；
- Docker 版本和 /etc/docker/daemon.json 文件使用的关系（以 1.12 为分界）；
- 针对不同发行版为 Docker Service 配置 Registry Mirror 的操作方式；


## 核心价值内容

- docker 版本低于 1.9 就没法使用 Registry Mirror 了；
- CentOS release 6
    - docker 服务配置文件：/etc/sysconfig/docker
    - 服务重启命令：sudo service docker restart
- CentOS Linux release 7
    - docker version < 1.12
        - docker 服务配置文件：/lib/systemd/system/docker.service
        - sudo systemctl daemon-reload
        - sudo systemctl restart docker
    - docker version >= 1.12
        - docker 服务配置文件：创建或修改 /etc/docker/daemon.json
        - sudo systemctl restart docker
- ubuntu
    - ubuntu version >= 16.xx
        - docker version < 1.12
            - docker 服务配置文件：/lib/systemd/system/docker.service
            - sudo systemctl daemon-reload
            - sudo systemctl restart docker.service
        - docker version >= 1.12
            - docker 服务配置文件：创建或修改 /etc/docker/daemon.json
            - sudo systemctl restart docker
    - ubuntu version < 16.xx
        - docker version < 1.12
            - docker 服务配置文件：/etc/default/docker
            - sudo service docker restart
        - docker version >= 1.12
            - docker 服务配置文件：创建或修改 /etc/docker/daemon.json
            - sudo service docker restart




