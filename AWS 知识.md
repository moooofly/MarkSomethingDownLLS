# AWS 知识

## 资料

- [AWS 知识中心](https://phab.llsapp.com/w/engineer/ops/aws/)
- [AWS 101](https://phab.llsapp.com/w/engineer/ops/aws/101/)
- [Amazon Web Services in Plain English](https://www.expeditedssl.com/aws-in-plain-english) -- 一句话解释 AWS 概念
- [open-guides/og-aws](https://github.com/open-guides/og-aws) -- 非官方指南
- [EC2 & RDS 实例信息查询表](https://www.ec2instances.info/)
- [EC2 Instance Types](https://www.amazonaws.cn/en/ec2/details/#ec2instancetypes)
- [AWS IAM](https://phab.llsapp.com/w/engineer/ops/aws/iam/)
- [AWS IAM Policy Examples](https://phab.llsapp.com/w/engineer/ops/aws/iam/policy-examples/)
- [Complete AWS IAM Reference](https://iam.cloudonaut.io/)
- [可在 IAM 策略中使用的 AWS 服务操作和条件上下文键](http://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/reference_policies_actionsconditions.html)
- [IAM identities (Users, Groups, and Roles)](https://docs.aws.amazon.com/IAM/latest/UserGuide/id.html)
- [所有 IAM policy 列表](http://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_actionsconditions.html)
- [AWS Provider](https://www.terraform.io/docs/providers/aws/index.html) -- Terraform 的资源
- [AWS 申请机器, Redis, RDS 等资源](https://phab.llsapp.com/w/engineer/ops/aws/apply/)
- [AWS Security Group 规范](https://phab.llsapp.com/w/engineer/ops/aws/sg/)
- [aws 安全组规范（讨论）](https://phab.llsapp.com/T14706)
- [staging 安全组查询](http://playdock.llsstaging.com/security_groups)
- [production 安全组查询](https://playdock.llsapp.com/security_groups)
- [AWS 已知的坑](https://phab.llsapp.com/w/engineer/ops/aws/pitfalls/)
- [AWS dev 账号相关](https://phab.llsapp.com/w/engineer/ops/aws/dev-resources/)
- [AWS staging 账号相关](https://phab.llsapp.com/w/engineer/ops/aws/staging-resources/)
- [LLS AWS 网络拓扑图](https://phab.llsapp.com/w/engineer/ops/awsarc/)
- [aws_s3_bucket](https://www.terraform.io/docs/providers/aws/r/s3_bucket.html)
- [AWS Tags](https://phab.llsapp.com/w/engineer/ops/aws/tags/)
- [What are some recommended best practices for tagging my Amazon EC2 resources?](https://amazonaws-china.com/cn/premiumsupport/knowledge-center/ec2-resource-tags/) -- tags 命名规范
- [如何查看 disk (instance storage) 以及挂载](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html)
- [如何扩充volume的容量](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-modify-volume.html)
- [调整卷大小后扩展 Linux 文件系统](https://docs.amazonaws.cn/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html)
- [Best EC2 setup for redis server [closed]](https://stackoverflow.com/questions/11765502/best-ec2-setup-for-redis-server)
- [如何连上elasticache的redis](http://docs.aws.amazon.com/AmazonElastiCache/latest/UserGuide/WhatIs.html)
- [Redis as the primary data store? WTF?!](https://muut.com/blog/technology/redis-as-primary-datastore-wtf.html)
- [VPC Peering](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-peering.html)
- [What is VPC Peering?](http://docs.aws.amazon.com/AmazonVPC/latest/PeeringGuide/Welcome.html)
- [AWS VPN](https://phab.llsapp.com/w/engineer/ops/aws/vpn/)
- [What Is Amazon VPC?](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html)
- [Terraform](https://www.terraform.io/intro/) -- a tool for building, changing, and versioning infrastructure safely and efficiently


## 概念梳理

- **AMI**: An Amazon Machine Image is simply a packaged-up environment that includes all the necessary bits to set up and boot your instance. Your AMIs are your unit of deployment. You might have just one AMI or you might compose your system out of several building block AMIs (e.g., webservers, appservers, and databases). 

- **AWS**: Since early 2006, AWS has provided companies of all sizes an information technology services platform in the cloud. With AWS, software developers can easily requisition compute, storage, database, and other Internet-based services to power their applications. Developers have the flexibility to choose any development platform or programming environment that makes sense for the problems they’re trying to solve. Since developers only pay for what they use, with no upfront capital costs and low operating costs, AWS is the most cost-effective way to deliver computational resources, stored data, and other applications to end users.

- **IAM**: AWS Identity and Access Management (IAM) enables you to securely control access to AWS services and resources for your users. Using IAM you can create and manage AWS users and groups and use permissions to allow and deny their permissions to AWS resources.

- **VPC peering**: A VPC peering connection is a networking connection between two VPCs that enables you to route traffic between them using private IPv4 addresses or IPv6 addresses. Instances in either VPC can communicate with each other as if they are within the same network. You can create a VPC peering connection between your own VPCs, or with a VPC in another AWS account. In both cases, the VPCs must be in the same region. 

----------

名词解释 - 1

- PBS: Persistent block storage
- **实例**存储（即 EC2 实例存储）作为根卷，也叫**本地存储**；
- **EBS** 存储作为根卷，也叫**网盘**，独立于 EC2 实例存储；
- aws 物理机：
- 云主机：云主机基于 AMI 镜像创建；
- AMI 镜像决定了作为根卷的是实例存储，还是 EBS 存储：
    - AMI 镜像放在 EBS 中，则据其创建的云主机的根卷就放在 EBS 中；
	- AMI 镜像放在 AWS S3 中，则据其创建的云主机的根卷就放在实例存储（物理主机的本地磁盘）中；
- 本地存储和云主机放在同一个 aws 物理机上，而 EBS 存储和云主机放在不同 aws 物理机上，且 EBS 放在专门的存储主机上而且有备份；
- 衡量 EBS 有两个重要指标：
    - IOPS：每秒 IO 次数
	- 吞吐量：每秒输出的字节数


----------


名词解释 - 2

- **ARN**：Amazon Resource Names，资源标识符，以 'arn:' 开头，在很多服务中使用，特别是 IAM policy 。比如 "arn:aws-cn:s3:::warehouse-tmp" ；
- **本地盘**/**Instance Store**（创建 instance 时会看到）/**Ephemeral instance**: are physically attached to the host computer. The data on an instance store persists only during the lifetime of the instance.（C4 没有本地盘）
- **EBS**：非本地盘，可以随意挂载在 ec2 实例上，并可以脱离 ec2 而存在。创建时的 Root 自动为 EBS ；
- **EIP**：可以跟 instance 绑定，来获得一个固定 IP 。instance stop 再 start 后，public IP 会变化。
- **AZ**：Availability Zone ，同一 region（地区）的不同物理机房（目前 beijing 有 `cn-north-1a`/`cn-north-1b` 两个 AZ）
- **RI**：Reserved instance ，预付一年买的instance，会自动和相同 region，instance type 和 platform 的 instance 绑定，有折扣。如果使用大于三个月，则建议购买，省钱。
- **Security group**：安全组，限定了属于这个安全组的机器有哪些协议和接口可以让什么 source 访问（一般是 ip、instance、或安全组；选安全组时，对该安全组下的所有机器都适用），有 inbound、outbound，一般设置 inbound 。 为了安全性，端口尽量只对内开放
- **IAM**: Identity and Access Management 控制账号、API key、机器对 AWS API 的访问权限。
- **ELB**: Elastic Load Balancing，类似于 HA，做负载均衡。


## AWS 产品

> 地址：
> 
> - 中文：https://www.amazonaws.cn/products/?nc1=f_dr
> - 英文：https://www.amazonaws.cn/en/products/?nc1=h_ls

- 计算
    - Amazon EC2
    - Amazon EC2 Container Registry
    - Amazon EC2 Container Service
    - AWS Lambda
    - Amazon Virtual Private Cloud (VPC)
    - AWS Elastic Beanstalk
    - Auto Scaling
    - Elastic Load Balancing
- 存储
    - Amazon S3
    - Amazon Elastic Block Store (EBS)
    - Amazon Glacier
    - AWS Storage Gateway
- 数据库
    - Amazon Relational Database Service (RDS)
    - Amazon DynamoDB
    - Amazon ElastiCache
    - Amazon Redshift
- 网络和内容分发
    - CloudFront
    - AWS Direct Connect
- 移动服务
    - Amazon API Gateway
- 开发人员工具
    - AWS CodeDeploy
- 管理工具
    - Amazon CloudWatch
    - Amazon EC2 Systems Manager
    - AWS CloudFormation
    - AWS CloudTrail
    - AWS Config
    - AWS 管理控制台
- 安全性、身份与合规性
    - AWS Identity and Access Management (IAM)
    - Amazon Cognito
- 分析
    - Amazon Elastic MapReduce
    - Amazon Kinesis Streams
- 应用程序服务
    - Amazon API Gateway
    - Amazon Simple Workflow (SWF)
- 消息发送
    - Amazon Simple Queue Service (SQS)
    - Amazon Simple Notification Service (SNS)
- 物联网
    - AWS IoT 平台
- 支持
    - AWS Premium Support
                                                    

## EC2

- Elastic Compute Cloud (EC2), Amazon EC2 presents a true virtual computing environment, allowing you to use web service interfaces to launch instances with a variety of operating systems, load them with your custom application environment, manage your network’s access permissions, and run your image using as many or few systems as you desire.

### EC2 提供的 features

#### EBS

- Elastic Block Store (EBS), 
    - Amazon EBS volumes are network-attached, and persist independently from the life of an instance. 
    - Amazon EBS volumes are highly available, highly reliable volumes that can be leveraged as an Amazon EC2 instance’s boot partition or attached to a running Amazon EC2 instance as a standard block device. 
    - Amazon EBS volumes offer greatly improved durability over local Amazon EC2 instance stores, as Amazon EBS volumes are automatically replicated on the backend (in a single Availability Zone).
    - For those wanting even more durability, Amazon EBS provides the ability to create point-in-time consistent snapshots of your volumes that are then stored in Amazon S3, and automatically replicated across multiple Availability Zones.
    -  Amazon EBS provides two volume types: **Standard volumes** and **Provisioned IOPS volumes**.

#### EBS-Optimized Instances

#### EIP

- Elastic IP (EIP) Addresses are static IP addresses designed for dynamic cloud computing. 

#### VPC

- Amazon Virtual Private Cloud (Amazon VPC) lets you provision a **logically isolated section** of the Amazon Web Services (AWS) Cloud where you can launch AWS resources in a virtual network that you define.
- Amazon VPC is the networking layer for Amazon EC2. 
- A virtual private cloud (VPC) is a virtual network dedicated to your AWS account. It is logically isolated from other virtual networks in the AWS Cloud. 
- You can configure your VPC by modifying its IP address range, create **subnets**, and configure route tables, network gateways, and security settings.
- A **subnet** is a range of IP addresses in your VPC. You can launch AWS resources into a specified subnet.
- To protect the AWS resources in each subnet, you can use multiple layers of security, including **security groups** and **network access control lists (ACL)**. 

#### Amazon CloudWatch

#### AS

- Auto Scaling (AS) allows you to automatically scale your Amazon EC2 capacity up or down according to conditions you define.

#### ELB

- Elastic Load Balancing (ELB) automatically distributes incoming application traffic across multiple Amazon EC2 instances.

#### High Performance Computing (HPC) Clusters

#### VM Import/Export

#### Enhanced Networking

#### Auto Recovery


----------


## 线上环境申请操作

```
➜  GitLab_liulishuo git clone git@git.llsapp.com:ops/rasen.git
➜  GitLab_liulishuo cd rasen
➜  rasen git:(master) git remote show
origin
➜  rasen git:(master)
➜  rasen git:(master) git remote add my git@git.llsapp.com:fei.sun/rasen.git
➜  rasen git:(master) ./bin/terraform-generator harbor-test-service
From git.llsapp.com:ops/rasen
 * branch            master     -> FETCH_HEAD
请先对 AWS(https://phab.llsapp.com/w/engineer/aws/ ) 和 Terraform(https://www.terraform.io/intro/ ) 有一定了解
你需要回答一些问题来生成配置([] 里的是默认值，直接回车即可，否则需要输入):
团队名(backend, analytics, algorithm, platform) 通用就是 platform: backend
你的名字（slack 的 username）: fei.sun
你想创建的资源（1. EC2 2. 安全组 3. ElastiCache(Redis), 可以考虑用公共的 dev-redis 4. ELB 0. 结束）: 1
以下是 EC2 的配置：
数量 [1]:
安全组 id(逗号分隔，如 sg-1234,sg-5678，请在 http://playdock.llsstaging.com/security_groups 查询。如果需要添加新安全组，直接写名字，并在 EC2 添加之后添加同名字的安全组，如 new-name,sg-1234。必须包含一个 shared- 安全组。) [sg-ddd7e9b8]:
镜像 ID (docker: ami-5906d034) [ami-5b06d036]:
实例类型(https://www.amazonaws.cn/en/ec2/details/#ec2instancetypes 中国区现有 t2/m3/c3/c4/r3/i2 系列): m3.large
root 硬盘大小(GB) [8]:
角色/用途是什么？(web/rpc 等) []: harbor
环境是？ [staging]:
是否需要 EIP (yes/no) [no]:
是否需要 pagerduty 报警 (yes/no)[no]:
临时使用？(yes/no) [no]:
是否需要购买 RI（0. 需要购买 2. 不确定 3. 不需要购买）[0]: 2
你想创建的资源（1. EC2 2. 安全组 3. ElastiCache(Redis), 可以考虑用公共的 dev-redis 4. ELB 0. 结束）: 0
在 ./backend/harbor-test-service 目录下创建配置文件。 请:
1. 在注释/Terraform文档的帮助下，查看并自定义生成的配置
2. 提交代码，并 push 到一个新的分支
3. 访问 http://playdock.llsapp.com/terraforms/new?path=backend/harbor-test-service
4. 选择刚刚提交的分支、path，点击 Plan 进行预览。
5. 预览没问题后在 Playdock 页面上提交 MR。
➜  rasen git:(master) ✗
```

https://git.llsapp.com/ops/rasen/merge_requests/395

```
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: .terraform/terraform.tfstate

Outputs:

login to harbor-test-service = ssh deployer@52.80.122.184 or @10.1.14.69
```


----------

## 申请用于升级 harbor 版本的环境

```
➜  GitLab_liulishuo cd rasen
➜  rasen git:(master) git branch
  harbor-test-service
* master
➜  rasen git:(master)
➜  rasen git:(master) git remote -v
my	git@git.llsapp.com:fei.sun/rasen.git (fetch)
my	git@git.llsapp.com:fei.sun/rasen.git (push)
origin	git@git.llsapp.com:ops/rasen.git (fetch)
origin	git@git.llsapp.com:ops/rasen.git (push)
➜  rasen git:(master)
➜  rasen git:(master) git pull origin
...
➜  rasen git:(master) ./bin/terraform-generator harbor-version-upgrade-service
From git.llsapp.com:ops/rasen
 * branch            master     -> FETCH_HEAD
请先对 AWS(https://phab.llsapp.com/w/engineer/aws/ ) 和 Terraform(https://www.terraform.io/intro/ ) 有一定了解
你需要回答一些问题来生成配置([] 里的是默认值，直接回车即可，否则需要输入):
团队名(backend, analytics, algorithm, platform) 通用就是 platform: platform
你的名字（slack 的 username）: fei.sun
你想创建的资源（1. EC2 2. 安全组 3. ElastiCache(Redis), 可以考虑用公共的 dev-redis 4. ELB 0. 结束）: 1
以下是 EC2 的配置：
数量 [1]:
安全组 id(逗号分隔，如 sg-1234,sg-5678，请在 http://playdock.llsstaging.com/security_groups 查询。如果需要添加新安全组，直接写名字，并在 EC2 添加之后添加同名字的安全组，如 new-name,sg-1234。必须包含一个 shared- 安全组。) [sg-ddd7e9b8]:
镜像 ID (docker: ami-5906d034) [ami-5b06d036]:
实例类型(https://www.amazonaws.cn/en/ec2/details/#ec2instancetypes 中国区现有 t2/m3/c3/c4/r3/i2 系列): m4.large
root 硬盘大小(GB) [8]: 100
角色/用途是什么？(web/rpc 等) []: harbor
环境是？ [staging]:
是否需要 EIP (yes/no) [no]:
是否需要 pagerduty 报警 (yes/no)[no]:
临时使用？(yes/no) [no]:
是否需要购买 RI（0. 需要购买 2. 不确定 3. 不需要购买）[0]: 2
你想创建的资源（1. EC2 2. 安全组 3. ElastiCache(Redis), 可以考虑用公共的 dev-redis 4. ELB 0. 结束）: 0
在 ./platform/harbor-version-upgrade-service 目录下创建配置文件。 请:
1. 在注释/Terraform文档的帮助下，查看并自定义生成的配置
2. 提交代码，并 push 到一个新的分支
3. 访问 http://playdock.llsapp.com/terraforms/new?path=platform/harbor-version-upgrade-service
4. 选择刚刚提交的分支、path，点击 Plan 进行预览。
5. 预览没问题后在 Playdock 页面上提交 MR。
➜  rasen git:(master) ✗
➜  rasen git:(master) ✗ gst
On branch master
Your branch is up-to-date with 'origin/master'.
Untracked files:
  (use "git add <file>..." to include in what will be committed)

	platform/harbor-version-upgrade-service/

nothing added to commit but untracked files present (use "git add" to track)
➜  rasen git:(master) ✗ ll platform/harbor-version-upgrade-service/
total 16
-rw-r--r--  1 sunfei  staff   129B 11 22 15:48 default.tf
-rw-r--r--  1 sunfei  staff   2.2K 11 22 15:48 ec2.tf
➜  rasen git:(master) ✗
➜  rasen git:(master) ✗
➜  rasen git:(master) ✗ git add .
➜  rasen git:(master) ✗ git commit -m "harbor-version-upgrade-servic"
[master 7974c23] harbor-version-upgrade-servic
 2 files changed, 79 insertions(+)
 create mode 100644 platform/harbor-version-upgrade-service/default.tf
 create mode 100644 platform/harbor-version-upgrade-service/ec2.tf
➜  rasen git:(master)
➜  rasen git:(master) git checkout -b harbor-version-upgrade-service
Switched to a new branch 'harbor-version-upgrade-service'
➜  rasen git:(harbor-version-upgrade-service)
➜  rasen git:(harbor-version-upgrade-service)
➜  rasen git:(harbor-version-upgrade-service) git branch
  harbor-test-service
* harbor-version-upgrade-service
  master
➜  rasen git:(harbor-version-upgrade-service)
➜  rasen git:(harbor-version-upgrade-service) git push my
Counting objects: 193, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (93/93), done.
Writing objects: 100% (193/193), 21.81 KiB | 10.90 MiB/s, done.
Total 193 (delta 115), reused 173 (delta 98)
remote: Resolving deltas: 100% (115/115), completed with 14 local objects.
remote:
remote: To create a merge request for harbor-version-upgrade-service, visit:
remote:   https://git.llsapp.com/fei.sun/rasen/merge_requests/new?merge_request%5Bsource_branch%5D=harbor-version-upgrade-service
remote:
To git.llsapp.com:fei.sun/rasen.git
 * [new branch]      harbor-version-upgrade-service -> harbor-version-upgrade-service
➜  rasen git:(harbor-version-upgrade-service)
➜  rasen git:(harbor-version-upgrade-service)
➜  rasen git:(harbor-version-upgrade-service) git diff HEAD origin/master --name-only
➜  rasen git:(harbor-version-upgrade-service)
```

步骤梳理（针对 staging 环境，非初次使用）：

- 进入 rasen 的 master 分支；
- 更新 master 分支到最新（git pull）；
- 基于 `./bin/terraform-generator NAME_OF_YOUR_SERVICE` 生成配置文件； 
- 按需修改生成的配置文件（一般情况下无需修改，除非之前配置有误）；
- `git add` + `git commit` + `git checkout -b <new_branch>` + `git push my`（推送到自己的 repo）
- 访问 http://playdock.llsstaging.com/terraforms/new ，此时将能看到自己刚推送的 branch ，按照提示信息执行 `git diff HEAD origin/master --name-only` ，并输出结果（path）贴入文本框，之后点击 **Plan** 按钮；
- 当 Job 在 playdock 成 passed 后（http://playdock.llsstaging.com/jobs/5582），即完成“预览”，之后点击 `Gitlab Auth for MR` 按钮授权 MR 提交，而不要直接到 Gitlab 页面上操作（之后还需要回到 http://playdock.llsstaging.com/jobs/5582 页面上点击 `Create Merge Request`）；
- 等待有权限的同学合并 MR ；
- MR 合并完成后，可以在 plan 结果页右上角打开 `View Apply Job` 页面（http://playdock.llsstaging.com/jobs/5584）查看实际执行结果；并获取访问服务器的信息 "login to harbor-service-new = ssh deployer@54.222.208.221 or @10.1.0.13"

