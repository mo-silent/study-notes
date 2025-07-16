---
title: AWS EKS中部署Cluster Autoscaler
slug: eks-depoly-ca
categories:
  - EKS
tags:
  - AWS
halo:
  site: https://blog.silentmo.cn
  name: 743650a2-1c20-4196-bd57-d0bc1277f4b4
  publish: true
---

> [!TIP]
>
> 文档编写时间：2021-11-17

# AWS EKS中部署Cluster Autoscaler

当 Pod 失败或被重新安排到其他节点时，Kubernetes [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) 会自动调整集群中的节点数。Cluster Autoscaler 通常作为[部署](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws/examples)安装在集群中。它使用[领导选举](https://en.wikipedia.org/wiki/Leader_election)来确保高可用性，但一次只能由一个副本完成扩展。

## 一、Prerequisites

在部署集群 Cluster Autoscaler之前，您必须满足以下先决条件：

- 拥有现有 Kubernetes 集群 – 如果您没有集群，请参阅 [创建 Amazon EKS 集群](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/create-cluster.html)。

- 集群的现有 IAM OIDC 提供商。要确定是否具有 IAM OIDC 提供商，还是需要创建一个，请参阅 [为集群创建 IAM OIDC 提供商](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)。

- 具有 Auto Scaling 组标签的节点组 – Cluster Autoscaler 要求 Auto Scaling 组带有以下标签，以便能够自动发现它们。

  - 如果您使用 `eksctl` 创建节点组，则节点组会自动应用这些标签。
  - 如果未使用 `eksctl`，则必须使用以下标签手动标记您的 Auto Scaling 组。有关更多信息，请参阅适用于 Linux 实例的 Amazon EC2 用户指南 中的[标记 Amazon EC2 资源](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html)。

  | 密钥                                       | 值      |
  | :----------------------------------------- | :------ |
  | `k8s.io/cluster-autoscaler/<cluster-name>` | `owned` |
  | `k8s.io/cluster-autoscaler/enabled`        | `TRUE`  |

## 二、创建 IAM 策略和角色

创建授予 Cluster Autoscaler 使用 IAM 角色所需权限的 IAM 策略。将整个过程中的所有 `<example-values>`（包括 `<>`）替换为您自己的值。

### 2.1 创建一个 IAM 策略。

1. 将以下内容保存到名为 `cluster-autoscaler-policy.json` 的文件中。如果现有节点组是使用 `eksctl` 创建的并且您使用了 `--asg-access` 选项，则此策略已存在，您可以跳至第 2 步。

   ```
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Action": [
                   "autoscaling:DescribeAutoScalingGroups",
                   "autoscaling:DescribeAutoScalingInstances",
                   "autoscaling:DescribeLaunchConfigurations",
                   "autoscaling:DescribeTags",
                   "autoscaling:SetDesiredCapacity",
                   "autoscaling:TerminateInstanceInAutoScalingGroup",
                   "ec2:DescribeLaunchTemplateVersions"
               ],
               "Resource": "*",
               "Effect": "Allow"
           }
       ]
   }
   ```

2. 使用以下命令创建策略。您可以更改 `policy-name` 的值。

   ```
   aws iam create-policy \
       --policy-name AmazonEKSClusterAutoscalerPolicy \
       --policy-document file://cluster-autoscaler-policy.json
   ```

   记下输出中返回的 Amazon Resource Name (ARN)。您需要在后面的步骤中用到它。

### 2.2 创建IAM角色

创建一个 IAM 角色并使用 `eksctl` 或 AWS Management Console 向其附加 IAM 策略。为以下说明选择所需的选项卡。

使用 `--asg-access` 选项创建集群，

<AmazonEKSClusterAutoscalerPolicy>` 替换为 `eksctl` 为您创建的 IAM 策略的名称` 

 策略名称类似于 `eksctl-<cluster-name>-nodegroup-ng-<xxxxxxxx>-PolicyAutoScaling`

```shell
eksctl create iamserviceaccount \
  --cluster=<my-cluster> \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/<AmazonEKSClusterAutoscalerPolicy> \
  --override-existing-serviceaccounts \
  --approve
```

## 三、部署 Cluster Autoscaler

要部署 Cluster Autoscaler，请完成以下步骤。建议您查看 [部署注意事项](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/cluster-autoscaler.html#ca-deployment-considerations) 并优化 Cluster Autoscaler 部署，然后再将其部署到生产集群。

### 3.1 部署 Cluster Autoscaler

1. 部署 Cluster Autoscaler。

   ```
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
   ```

2. 使用您以前创建的 IAM 角色的 ARN 对 `cluster-autoscaler` 服务账户添加注释。将`<示例值>`替换为您自己的值。

   ```
   kubectl annotate serviceaccount cluster-autoscaler \
     -n kube-system \
     eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/<AmazonEKSClusterAutoscalerRole>
   ```

3. 使用以下命令修补部署以向 Cluster Autoscaler Pod 添加 `cluster-autoscaler.kubernetes.io/safe-to-evict` 注释。

   ```
   kubectl patch deployment cluster-autoscaler \
     -n kube-system \
     -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'
   ```

4. 使用以下命令编辑 Cluster Autoscaler 部署。

   ```
   kubectl -n kube-system edit deployment.apps/cluster-autoscaler
   ```

   编辑 `cluster-autoscaler` 容器命令，将 `<YOUR CLUSTER NAME>`（包括 `<>`）替换为您的集群名称，然后添加以下选项。

   - `--balance-similar-node-groups`
   - `--skip-nodes-with-system-pods=false`

   ```
       spec:
         containers:
         - command:
           - ./cluster-autoscaler
           - --v=4
           - --stderrthreshold=info
           - --cloud-provider=aws
           - --skip-nodes-with-local-storage=false
           - --expander=least-waste
           - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
           - --balance-similar-node-groups
           - --skip-nodes-with-system-pods=false
   ```

   保存并关闭该文件以应用更改。

5. 在 Web 浏览器中从 GitHub 打开 Cluster Autoscaler [版本](https://github.com/kubernetes/autoscaler/releases)页面，找到与您集群的 Kubernetes 主版本和次要版本相匹配的 Cluster Autoscaler 最新版本。例如，如果您集群的 Kubernetes 版本是 1.21，则查找以 1.21 开头的最新 Cluster Autoscaler 版本。记录该版本的语义版本号 (`1.21.*n*`) 以在下一步中使用。

6. 使用以下命令，将 Cluster Autoscaler 映像标签设置为您在上一步中记录的版本。将 `1.21.*n*` 替换为您自己的值。

   ```
   kubectl set image deployment cluster-autoscaler \
     -n kube-system \
     cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v<1.21.n>
   ```

### 3.2 查看 Cluster Autoscaler 日志

部署 Cluster Autoscaler 之后，您可以查看日志并验证它在监控您的集群负载。

使用以下命令查看您的 Cluster Autoscaler 日志。

```
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```

您可以在一个 (扩展) 代码行中执行所有这些操作：

```
I0926 23:15:55.165842       1 static_autoscaler.go:138] Starting main loop
I0926 23:15:55.166279       1 utils.go:595] No pod using affinity / antiaffinity found in cluster, disabling affinity predicate for this loop
I0926 23:15:55.166293       1 static_autoscaler.go:294] Filtering out schedulables
I0926 23:15:55.166330       1 static_autoscaler.go:311] No schedulable pods
I0926 23:15:55.166338       1 static_autoscaler.go:319] No unschedulable pods
I0926 23:15:55.166345       1 static_autoscaler.go:366] Calculating unneeded nodes
I0926 23:15:55.166357       1 utils.go:552] Skipping ip-192-168-3-111.<region-code>.compute.internal - node group min size reached
I0926 23:15:55.166365       1 utils.go:552] Skipping ip-192-168-71-83.<region-code>.compute.internal - node group min size reached
I0926 23:15:55.166373       1 utils.go:552] Skipping ip-192-168-60-191.<region-code>.compute.internal - node group min size reached
I0926 23:15:55.166435       1 static_autoscaler.go:393] Scale down status: unneededOnly=false lastScaleUpTime=2019-09-26 21:42:40.908059094 ...
I0926 23:15:55.166458       1 static_autoscaler.go:403] Starting scale down
I0926 23:15:55.166488       1 scale_down.go:706] No candidates for scale down
```
