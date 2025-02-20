## AWS EKS 部署要点以及控制台与eksctl创建的差异

上一篇文章描述如何使用eksctl创建AWS EKS

这篇文章，主要是说明创建AWS EKS所要注意的要点，以及控制台与eksctl的差异

## 一、EKS部署要点

### 1.1 IAM角色（控制台方式，eksctl会自动创建角色）

在控制台创建AWS EKS需要两种角色权限，一个为Cluster，一个为Node。

其中Node分两种，一种为EC2模式的，一种为Fargate模式

在IAM控制台选择创建角色，选择EKS案例，分别创建Cluster案例和Nodegroup案例




### 1.2 网络

在控制台创建的节点组所选子网，可选的有三种

第一种是公有子网，第二种是带有NAT Gateway路由的私有子网

第三种，是纯私有子网，只有内网路由。

第三种子网的集群为私有集群，但纯私有子网必须要有以下VPC Endpoint（参考文档[1]）

- Interface endpoints for ECR (both ecr.api and ecr.dkr) to pull container images (AWS CNI plugin etc)
- A gateway endpoint for S3 to pull the actual image layers
- An interface endpoint for EC2 required by the aws-cloud-provider integration
- An interface endpoint for STS to support Fargate and IAM Roles for Services Accounts (IRSA)
- An interface endpoint for CloudWatch logging (logs) if CloudWatch logging is enabled

### 1.3 节点AMI

使用AWS EKS，节点必须是Amazon EKS 优化的 AMI（参考[2]）

如果需要自定义镜像，需要执行Amazon EKS优化AMI生成脚本[3]

### 1.4 集群身份

创建 Amazon EKS 集群时，将在控制层面的集群 RBAC 配置中自动为创建集群的 IAM 实体用户或角色（例如，[联合身份用户](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers.html)）授予 `system:masters` 权限。

集群的默认masters权限为创建者，初始创建的集群，除了创建者其他IAM实体无法操作集群。

因此请确保跟踪最初创建了集群的 IAM 实体。

要授予其他 AWS 用户或角色与您的集群进行交互的能力，必须编辑 Kubernetes 内的 `aws-auth` ConfigMap。

### 1.5 节点自动扩展(AutoScaling)

在AWS EKS中的托管节点组，虽然使用的了AutoScaling组的方式来创建节点，但是初始的AutoScaling并没有任何扩缩规则。

在AWS EKS中节点需要自动扩缩需要在集群中安装和配置 cluster-autoscaler。

## 二、控制台与eksctl的差异

### 2.1 节点组

在AWS EKS控制台创建的节点组皆为托管节点组，能够在控制台显示查看。

而eksctl创建的节点有托管节点组和自行管理节点组，自行管理节点组不在AWS EKS界面显示。

自信管理节点组还可以通过Cloudformation创建，eksctl本质就是创建一个Cloudformation堆栈实现对集群的管理。

### 2.2 插件

在AWS EKS控制台创建的集群中Amazon VPC CNI、CoreDNS、kube-proxy插件皆为托管插件，由AWS管理更新。

而使用eksctl创建的集群中这三个插件为非托管插件，虽然是AWS开源的插件，但不由AWS管理更新，需要用户自行管理。

### 2.3 选项

在AWS EKS控制台创建的集群可选集群属性简单，而eksctl方式创建的集群可选属性多。

但很多属性会导致集群的控制难度加大，与AWS 其他集成服务兼容方式复杂。

因此在使用eksctl创建集群时，一般会选择最小原则，即只配置对应的节点组和一般的集群属性。

IAM和OIDC等属性不做配置，集群创建后续添加。

### 2.4 节点组存储

在AWS EKS控制台创建的节点组可以通过启动模板来配置数据卷，但不会自动挂载，需要编写bash脚本文件

而使用eksctl创建的节点组，只能修改根卷大小，无法创建数据卷。

## 参考

**[1] eksctl**：https://eksctl.io/usage/eks-private-cluster/#configuring-private-access-to-additional-aws-services

**[2] Amazon EKS优化AMI**：https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/eks-optimized-amis.html

**[3] 优化AMI脚本**：https://github.com/awslabs/amazon-eks-ami