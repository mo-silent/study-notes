---
title: 在 AWS EKS 部署 ALB
slug: eks-deploy-alb
categories:
  - EKS
tags:
  - AWS
halo:
  site: https://blog.silentmo.cn
  name: 6d00fb3d-26c5-493b-9c42-c75ded6d4688
  publish: true
---
> [!TIP]
>
> 文档编写时间：2021-12-02

## 一、前置条件

在AWS EKS中`service`默认的`LoadBalance`模式是`CLB`。

如果需要使用`ALB`或`NLB`则需要安装`AWS Load Balancer Controller`

安装AWS负载均衡器控制器前需要为集群创建`IAM  OIDC`提供商

## 二、创建IAM  OIDC提供商

1. 确定集群是否拥有现有 IAM OIDC 提供商。

   查看集群的 OIDC 提供商 URL。

   ```shell
   aws eks describe-cluster --name <cluster_name> --query "cluster.identity.oidc.issuer" --output text
   ```

   输出示例：

   ```shell
   https://oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E
   ```

   列出您账户中的所有 OIDC 提供商。将 `<EXAMPLED539D4633E53DE1B716D3041E>`（包括 `<>`）替换为上一个命令返回的值。

   ```shell
   aws iam list-open-id-connect-providers | grep <EXAMPLED539D4633E53DE1B716D3041E>
   ```

   输出示例

   ```shell
   "Arn": "arn:aws-cn:iam::111122223333:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
   ```

   如果上一个命令返回了输出，则表示您的集群已经有提供商。如果没有返回输出，则您必须创建 IAM OIDC 提供商。

2. 使用以下命令为您的集群创建 IAM OIDC 身份提供商。将 `<cluster_name>`（包括 `<>`）替换为您自己的值。

   ```shell
   eksctl utils associate-iam-oidc-provider --cluster <cluster_name> --approve
   ```

## 三、安装`AWS Load Balancer Controller`

1. 下载AWS负载均衡器控制器的 IAM 策略，该策略允许负载均衡器代表您调用 AWS API。您可以查看 GitHub 上的[策略文档](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json)。

   ```shell
   curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json
   ```

2. 使用上一步中下载的策略创建一个 IAM 策略。

   ```shell
   aws iam create-policy \
       --policy-name AWSLoadBalancerControllerIAMPolicy \
       --policy-document file://iam_policy.json
   ```

   记下返回的策略 ARN。

3. 将 `my_cluster` 替换为您的集群的名称，并将 `111122223333` 替换为您的账户 ID，然后运行命令。

   ```shell
   eksctl create iamserviceaccount \
     --cluster=my_cluster \
     --namespace=kube-system \
     --name=aws-load-balancer-controller \
     --attach-policy-arn=arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
     --override-existing-serviceaccounts \
     --approve          
   ```

4. 使用 Helm V3 或更高版本来安装AWS负载均衡器控制器

   a. 安装 `TargetGroupBinding` 自定义资源定义。

   ```shell
   kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"
   ```

   b. 添加 `eks-charts` 存储库。

   ```shell
   helm repo add eks https://aws.github.io/eks-charts
   ```

   c. 更新您的本地存储库，以确保您拥有最新的图表。

   ```shell
   helm repo update
   ```

   d. 安装AWS负载均衡器控制器。

   **重要**

   如果要将控制器部署到[被限制访问 Amazon EC2 实例元数据服务 (IMDS)](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/best-practices-security.html#restrict-ec2-credential-access) 的 Amazon EC2 节点，或者部署到 Fargate 节点，则需要在运行的命令中添加以下标志：

   - `--set region=region-code`
   - `--set vpcId=vpc-xxxxxxxx`

   ```shell
   helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
     --set clusterName=cluster-name \
     --set serviceAccount.create=false \
     --set serviceAccount.name=aws-load-balancer-controller \
     -n kube-system
   ```

5. 验证控制器是否已安装。

   ```shell
   kubectl get deployment -n kube-system aws-load-balancer-controller
   ```

   输出

   ```shell
   NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
   aws-load-balancer-controller   2/2     2            2           84s
   ```

## 四、应用中部署ALB

```shell
https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/annotations/
```

## 参考

[1] [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/)
