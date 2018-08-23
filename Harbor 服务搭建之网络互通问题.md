# Harbor 服务搭建之网络互通问题

staging 和 prod 环境之间网络互通涉及以下两方面配置：

- Security Groups
- VPC peering

## Security Groups

### Amazon EC2 Security Groups for Linux Instances

> 英文地址：[这里](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html)    
> 中文地址：[Linux 实例的 Amazon EC2 个安全组](http://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/using-network-security.html)

安全组基础知识介绍；

### harbor staging SG

> EC2 信息查询：[这里](http://splaydock.llsstaging.com/instances?utf8=%E2%9C%93&dept=all_dept&name=registry&private_ip=&ip=&commit=Search)    
> SG 完整列表：[这里](http://splaydock.llsstaging.com/security_groups)

| Name | Security Group Id | Vpc Id | Description | Inbound |
| -- | -- | -- | -- | -- |
| shared-default | sg-ddd7e9b8 | vpc-a0e20cc4 | Default shared SG for ssh created 2016-08-09 14:05:26	| tcp://22 => 0.0.0.0/0<br>tcp://23333~23433 => 10.1.0.0/16<br>tcp://9100 => 10.1.0.0/16 |
| docker-registry-harbor | sg-d17a68b4 | vpc-a0e20cc4 | docker-registry-harbor created at 2016-10-24 14:50:49 +0800 CST | tcp://80 => 10.1.0.0/16,10.2.0.0/16,10.3.0.0/16,10.4.0.0/16<br>tcp://443 => 10.1.0.0/16,10.2.0.0/16,10.3.0.0/16,10.4.0.0/16<br>tcp://5201 => 10.0.0.0/8 |
| spinnaker-web | sg-27df3d43 | vpc-a0e20cc4 | spinnaker-web created at 2016-12-6 10:50:49 +0800 CST | tcp://80 => 52.80.44.111/32,54.223.229.211/32<br>tcp://443 => 52.80.44.111/32,54.223.229.211/32 |


### harbor prod SG

> EC2 信息：[这里](https://playdock.llsapp.com/instances?utf8=%E2%9C%93&dept=all_dept&name=harbor&private_ip=&ip=&commit=Search)    
> SG 完整列表：[这里](https://playdock.llsapp.com/security_groups)

| Name | Security Group Id | Vpc Id | Description | Inbound |
| -- | -- | -- | -- | -- |
| shared-default | sg-bea396db | vpc-72abbd10 | Default shared SG for ssh created 2016-07-11 13:43:00 +0800 | tcp://22 => 172.1.0.0/16,172.2.0.0/16,172.31.0.0/16<br>tcp://23333~23433 => 172.1.0.0/16,172.2.0.0/16,172.31.0.0/16<br>tcp://9100 => 172.31.0.0/16 |
| harbor | sg-51759d35 | vpc-72abbd10 | harbor created at 2016-12-23 16:38:36 +0800 CST | tcp://80 => 172.31.0.0/16,172.1.0.0/16,172.2.0.0/16,172.3.0.0/16,10.1.1.37/32,10.1.0.13/32,54.223.229.211/32<br>tcp://22 => 172.31.0.0/16<br>tcp://443 => 172.31.0.0/16,172.1.0.0/16,172.2.0.0/16,172.3.0.0/16,10.1.1.37/32,10.1.0.13/32,54.223.229.211/32 |


### 基于 Terraform 生成的安全组信息

以下内容取自 `spiral/platform/harbor/sg.tf` ，可以基于其内容更好的理解上面的规则；

```
# https://www.terraform.io/docs/providers/aws/r/security_group.html
resource "aws_security_group" "harbor" {
  name = "harbor"
  description = "harbor created at 2016-12-23 16:38:36 +0800 CST"
  vpc_id = "vpc-72abbd10"

  # Inbound
  ingress {
    # ssh
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = ["172.31.0.0/16"]
  }

  # Inbound
  ingress {
    # ssh
    from_port = 80
    to_port = 80
    protocol = "tcp"
    # 172.31 is the share cluster
    # 172.1 is prod0 k8s vpc
    # 172.2 is prod1 k8s vpc
    # 172.3 is prod2 k8s vpc
		# 10.1.1.37/32 is old harbor staging to sync replica
		# 10.1.0.13/32 is new harbor staging to sync replica
 		# 54.223.229.211/32 is spinnaker in staging env
    cidr_blocks = ["172.31.0.0/16", "172.1.0.0/16", "172.2.0.0/16", "172.3.0.0/16", "10.1.1.37/32", "10.1.0.13/32", "54.223.229.211/32"]
  }

  # Inbound
  ingress {
    # ssh
    from_port = 443
    to_port = 443
    protocol = "tcp"
    # 172.31 is the share cluster
    # 172.1 is prod0 k8s vpc
    # 172.2 is prod1 k8s vpc
    # 172.3 is prod2 k8s vpc
		# 10.1.1.37/32 is old harbor staging to sync replica
		# 10.1.0.13/32 is new harbor staging to sync replica
 		# 54.223.229.211/32 is spinnaker in staging env
    cidr_blocks = ["172.31.0.0/16", "172.1.0.0/16", "172.2.0.0/16", "172.3.0.0/16", "10.1.1.37/32", "10.1.0.13/32", "54.223.229.211/32"]
  }

  # Outbound
  # All traffic for outbound, it's OK in most cases
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

----------

## VPC peering

**VPC peering**: A VPC peering connection is a networking connection between two VPCs that enables you to route traffic between them using private IPv4 addresses or IPv6 addresses. Instances in either VPC can communicate with each other as if they are within the same network. You can create a VPC peering connection between your own VPCs, or with a VPC in another AWS account. In both cases, the VPCs must be in the same region.

参考：

- [VPC Peering](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-peering.html)
- [What is VPC Peering?](http://docs.aws.amazon.com/AmazonVPC/latest/PeeringGuide/Welcome.html)