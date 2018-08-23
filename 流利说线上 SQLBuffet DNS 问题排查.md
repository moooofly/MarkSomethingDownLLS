# 流利说线上 SQLBuffet DNS 问题排查

## 问题描述

> https://phab.llsapp.com/T51997

错误信息摘录：

```
[2018-03-28 14:16:11,194: ERROR/Worker-5] got error in task_id: 67c9492e-324f-11e8-a657-0286059f4ff3, sub_query_id: 0, part_id: 13, error: {
    "errorCode": 16777219,
    "message": "Error opening Hive split s3://warehouse/hive/llsdw.db/dw_lls_user_persona_daily/data_date=2018-03-09/000000_0 (offset=0, length=6972369): Unable to execute HTTP request: warehouse.s3.cn-north-1.amazonaws.com.cn",
    "errorType": "EXTERNAL",
    "failureInfo": {
        ...
        "cause": {
            "suppressed": [],
            "cause": {
                "suppressed": [],
                "message": "warehouse.s3.cn-north-1.amazonaws.com.cn",
                "type": "java.net.UnknownHostException",
                "stack": [
                    "java.net.InetAddress.getAllByName0(InetAddress.java:1280)",
                    "java.net.InetAddress.getAllByName(InetAddress.java:1192)",
                    "java.net.InetAddress.getAllByName(InetAddress.java:1126)",
                    "com.amazonaws.SystemDefaultDnsResolver.resolve(SystemDefaultDnsResolver.java:27)",
                    "com.amazonaws.http.DelegatingDnsResolver.resolve(DelegatingDnsResolver.java:38)",
                    "org.apache.http.impl.conn.DefaultHttpClientConnectionOperator.connect(DefaultHttpClientConnectionOperator.java:111)",
                    ...
                ]
            },
            "message": "Unable to execute HTTP request: warehouse.s3.cn-north-1.amazonaws.com.cn",
            "type": "com.amazonaws.AmazonClientException",
            ...
        },
        "message": "Error opening Hive split s3://warehouse/hive/llsdw.db/dw_lls_user_persona_daily/data_date=2018-03-09/000000_0 (offset=0, length=6972369): Unable to execute HTTP request: warehouse.s3.cn-north-1.amazonaws.com.cn",
        "type": "com.facebook.presto.spi.PrestoException",
        ...
    },
    "errorName": "HIVE_CANNOT_OPEN_SPLIT"
}
```

问题以前出现过, 海涛的总结如下：

- 所有的节点都归属相同的 autoscaling group ；
- DNS 解析错误随机出现在不同节点上；
- 之前 case 关闭了, 是因为第二天启动的 autoscaling 节点又没有这个问题了；

## 问题复现

- 测试查询: http://data.llsapp.com:8989/query/query_result/query_id/62914f8a-325c-11e8-848e-0286059f4ff3
- 点击“重新编辑SQL”
- 点击“执行SQL”
- 点击“查看 presto 任务”
- 查看异常栈信息和执行结果为 FAILED 的机器

## 问题排查

由于 autoscaling group 中的机器每天都会被销毁，第二天再被重建；因此对应的机器 IP 地址每天都会发生变化；因此需要通过如下脚本进行处理

- 添加公钥和进行抓包的脚本（存在一些问题）

```
#!/bin/bash

function ssh_server() {
    ssh -t -o StrictHostKeyChecking=no  -i ~/ec2-test-key.pem $*
}

function add_script(){
    for server in `cat /tmp/presto.hosts`
    do
        echo "to server $server"
        ssh_server ec2-user@$server "echo 'sudo killall tcpdump' >/tmp/s3_dns.sh"
        ssh_server ec2-user@$server "echo 'sudo nohup tcpdump -i eth0 -s0 port 53 -w /tmp/s3.$server.pcap &' >>/tmp/s3_dns.sh "
        echo "done to $server"
    done
}


function start_script(){
    for server in `cat /tmp/presto.hosts`
    do
        echo "to server $server"
         ssh -t -o StrictHostKeyChecking=no  -i ~/ec2-test-key.pem ec2-user@$server <<EOF
            sudo nohup tcpdump -i eth0 -s0 port 53 -w /tmp/s3.$server.pcap &
EOF
        echo "end $server"
    done
}

#add_script
#start_script
function add_key(){
    for server in `cat /tmp/presto.hosts`
    do
        echo $server
        ssh_server ec2-user@$server 'echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCwG5BGVxsIbgQV4AOWxjy6Pg89bvvkXU9gDBHOufA6iI2il4cyh0+gOHuethbqc5WGu14FKIbY+Q497xz7B5/WjnOovGa8dgY6z5V63fzgDT755n/eK8D1FsWJJgENj5jx5YUUkl7NhHKj5JkMLbGB3cJ4Ml7t932WUpeij+884bncXcH7CA1AmTOm32mY3LhO5ehgeLzouRJNdPqOs9u9UETCGVicv6xDBkrXrYhJ6Ohg2CzDchKNWyJGhcR5cR8VNbY1fzCOwYY/pGYMate8RSNGWurHw6IM3dUa1X0Jq17j+AJe0pyNVXRNOJW/KtWgKzfVlo5UtG3f53AAnmlcopItXLn+TJvfygMA00SE9nH3wu2n2OLAhZE5s8mFvfHmleZquN9mJH/u0gj9JBLcpQBjkofEewbRIaVlLRX1OWqKliJxmXGLhvd/b1mO4YGUD88cLgsOH919T8nCH/lrK65RPyQl5v0fUC+UmSnNfhU+FFHvvr3DK3VtcJjjMCUofJOJFzMtj6Zv73Yua1pTMBfRWbtCm3OaCHVjCB4C2IyMSin7Pq3HXsaCrLpNVboAGd9pltFXZUWVqrJVj44SH3Lw8xykHd8HcH3TlSlxFtuQNOjWFIRLN24tnM/nCD1WgYJ/+Nx4L65VOkciXwzEbT7Gv3maxOK23HNPFaJ0HQ== fei.sun@liulishuo.com" >>/home/ec2-user/.ssh/authorized_keys'
    done

}
```

- `get_autoscaling_nodes.sh` 用于获取当前 autoscaling group 中所有节点的地址

```
#!/bin/bash

function get_auto_scaling_nodes(){
    local group_name=$1
    for instance_id in $(aws autoscaling describe-auto-scaling-instances  --query "AutoScalingInstances[?AutoScalingGroupName==\`$group_name\`].InstanceId" --output text)
    do
        aws ec2 describe-instances --instance-ids $instance_id --query Reservations[].Instances[].PrivateIpAddress --output text
    done
}


get_auto_scaling_nodes "presto-0-common"
get_auto_scaling_nodes "presto-0-spot"
```

> 由于 aws 权限的问题，目前没有办法正常使用上述脚本进行相关处理，只能让有权限的人执行后，将当前机器列表提供出来

机器登录命令

```
ssh ec2-user@<目标机器ip>
```


----------


安装完 awscli 后，需要先进行 configure ，在配置的时候需要 aws 的 key ，但目前没有给我；


```
➜  Shell ./get_autoscaling_nodes.sh
You must specify a region. You can also configure your region by running "aws configure".
You must specify a region. You can also configure your region by running "aws configure".
➜  Shell
➜  Shell
➜  Shell aws configure
AWS Access Key ID [None]:
AWS Secret Access Key [None]:
Default region name [None]: cn-north-1
Default output format [None]:
➜  Shell
➜  Shell
➜  Shell
➜  Shell ./get_autoscaling_nodes.sh
Unable to locate credentials. You can configure credentials by running "aws configure".
Unable to locate credentials. You can configure credentials by running "aws configure".
➜  Shell
```

权限相关说明：https://phab.llsapp.com/w/engineer/ops/aws/iam/


----------


## 问题结论

> 问题排查当天的机器列表：
> 
> ```
> 172.31.7.236
> 172.31.3.250
> 172.31.14.109
> 172.31.14.5
> 172.31.10.7
> 172.31.0.135
> 172.31.8.251
> 172.31.6.157
> 172.31.9.122
> 172.31.6.29
> 172.31.10.115
> ```


经过两天的排查分析，目前可以确定的信息如下：

- 进行了三次测试，其中
    - 2:05 左右，只看到 1 个 failed ，对应 172.31.14.5
    - 2:08 左右，看到 37 个 failed，其中
        - 172.31.6.157 共 12 个
        - 172.31.3.250 共 7 个
        - 172.31.8.251 共 6 个
        - 172.31.10.115 共 12 个
    - 2:11 左右，看到 3 个 failed，其中
        - 172.31.14.109 共 2 个
        - 172.31.6.157 共 1 个
- 三次测试过程均通过 `sudo tcpdump -i eth0 -n -s 0 port 53 -w ip-172-31-xx-xx.pcap` 进行了抓包，其中
    - 全部 11 台主机上的抓包中，**均未发现任何 dns 查询相关的 error**
    - 仅在 `ip-172-31-0-135.pcap` 中 14:12:04 左右出现 `25.779ms` 的 AAAA DNS 查询，在 `ip-172-31-14-5.pcap` 中 14:06:13 左右出现 `251.349ms` 的 A DNS 查询；
    - 其他查询最高耗时在 `4ms~6ms` 左右；

当前结论：

- 从抓包上看，确实没有看到任何 dns 相关 error ，所以 aws support 的回复邮件没毛病；但针对 dns 查询“慢”的情况（两次），可以和其进一步沟通；
- 代码层面可能也值得怀疑一下：
    - dns 查询 latency 大的情况就发生了两次，而 failed 出现了非常多次，且时间点上并不完全对应，因此，有理由怀疑导致 failed 的原因可能有多种；
    - 三次实验中 error message 的内容并不完全一样，stack 信息也会略有差异，似乎虽然均和 dns 解析相关，但出错位置有好几处，因此，有理由怀疑 dns 查询失败可能是“果”，真正的原因另有其他（例如系统资源不足等问题？）


------

手撸了一下第二次试验中 172.31.6.157 上出现的 failed 的信息，可以和抓包文件 `ip-172-31-6-157.pcap` 对照查看

| ID | Host | State | Pending splits | Running splits | Blocked splits | Completed splits | Rows | Rows/s | Bytes | Bytes/s | Elapsed | CPU Time | Buffered |
| -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| 166.5 | 172.31.6.157 | FAILED | 0 | 1 | 0 | 0| 0.00 | 0.00 | 0 | 0 | 33.09s | 4.12ms | 0 |
| 159.5 | 172.31.6.157 | FAILED | 0 | 1 | 0 | 0 | 0.00 | 0.00 | 0 | 0 | 33.41s | 3.55ms | 0 |
| 123.6 | 172.31.6.157 | FAILED | 0 | 2 | 0 | 1 | 203K | 6.19K | 6.74M | 211K | 32.74s | 215.84ms | 590K |
| 84.4 | 172.31.6.157 | FAILED | 0 | 2 | 0 | 0 | 1.02K | 30.8 | 5.78M | 178K | 33.27s | 71.62ms | 0 |
| 83.6 | 172.31.6.157 | FAILED | 0 | 2 | 0 | 0 | 0.00 | 0.00 | 0 | 0 | 32.85s | 3.79ms | 0 |
| 150.6 | 172.31.6.157 | FAILED | 0 | 1 | 0 | 0 | 0.00 | 0.00 | 0 | 0 | 33.17s | 4.46ms | 0 |
| 114.6 | 172.31.6.157 | FAILED | 0 | 4 | 0 | 0 | 1.17M | 35.4K | 48.3M | 1.46M | 32.96s | 1.20s | 0 |
| 115.6 | 172.31.6.157 | FAILED | 0 | 1 | 0 | 0 | 0.00 | 0.00 | 0 | 0 | 33.04s | 3.71ms | 0 |
| 66.4 | 172.31.6.157 | FAILED | 0 | 1 | 0 | 0 | 0.00 | 0.00 | 0 | 0 | 32.88s | 6.11ms | 0 |
| 63.6 | 172.31.6.157 | FAILED | 0 | 3 | 0 | 0 | 93.0 | 2.83 | 5.33K | 166 | 32.87s | 12.51ms | 0 |
| 155.6 | 172.31.6.157 | FAILED | 0 | 1 | 0 | 0 | 0.00 | 0.00 | 0 | 0 | 33.51s | 4.47ms | 0 |
| 86.4 | 172.31.6.157 | FAILED | 0 | 1 | 0 | 0 | 0.00 | 0.00 | 0 | 0 | 32.23s | 4.81ms | 0 |


------

提供一些截图供参考：

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/2nd_err_msg.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/3nd_err_msg.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ip-172-31-14-109_ts140613_4ms.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ip-172-31-6-157_ts140736_4ms.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ip-172-31-14-5_ts140613_200ms.png)


----------


## 其他


可以直接基于 DNS 协议的 reply code 过滤出有 error 信息的包；

```
tshark -r ip-172-31-0-135.pcap -Y dns.qry.type==1,28 -Y dns.flags.rcode!=0 -d udp.port==53,dns
```

其中

- `-r <infile>`: set the filename to read from (- to read from stdin)
- `-Y <display filter>`: packet displaY filter in Wireshark display filter syntax
- `-d <layer_type>==<selector>,<decode_as_protocol> ...`: "Decode As", see the man page for details Example: tcp.port==8888,http
- `dns.qry.type`
    - 1 对应 A 查询
    - 28 对应 AAAA 查询；
- `dns.flags.rcode`
    - 0 对应 No error
    - 1 对应 Format error
    - 2 对应 Server failure
    - 3 对应 No such name
    - 4 对应 Not implemented
    - 5 对应 Refused
    - 6 对应 Name exists
    - 7 对应 RRset exists
    - 8 对应 RRset dose not exists
    - 9 对应 Not authoritative
    - 10 对应 Name out of zone
- `dns.flags.response`
    - 0 对应 DNS queries
    - 1 对应 DNS responses



----------


## [Installing the AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)


- The primary distribution method for the AWS CLI on Linux, Windows, and macOS is `pip`
-  you can install the AWS CLI with the following command: `pip install awscli --upgrade --user`, The `--upgrade` option tells pip to upgrade any requirements that are already installed. The `--user` option tells pip to install the program to a subdirectory of your user directory to avoid modifying libraries used by your operating system.
- Verify that the AWS CLI installed correctly by running `aws --version`.
- To update to the latest version of the AWS CLI, run the installation command again: `pip install awscli --upgrade --user`.
- If you need to uninstall the AWS CLI, use `pip uninstall`.


## [Install the AWS Command Line Interface on macOS](https://docs.aws.amazon.com/cli/latest/userguide/cli-install-macos.html#awscli-install-osx-path)

Install Python, `pip`, and the AWS CLI on macOS

```
$ curl -O https://bootstrap.pypa.io/get-pip.py
$ python3 get-pip.py --user
$ pip3 install awscli --upgrade --user
$ aws --version
```

Adding the AWS CLI Executable to your Command Line Path

After installing with `pip`, you may need to add the aws executable to your OS's PATH environment variable. The location of the executable depends on where Python is installed.

```
/Users/sunfei/Library/Python/2.7/bin/aws
```
