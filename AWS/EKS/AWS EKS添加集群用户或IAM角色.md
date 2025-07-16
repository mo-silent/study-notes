---
title: AWS EKS 添加集群用户或 IAM 角色
slug: eks-add-iam-user-or-role
categories:
  - EKS
tags:
  - AWS
halo:
  site: https://blog.silentmo.cn
  name: 290aef1e-c2c9-4d97-be96-dc5b7b7c408b
  publish: true
---
> [!TIP]
>
> 文档编写时间：2021-11-15

# 前言

Amazon EKS 使用 IAM 为Kubernetes 集群提供身份验证（通过 AWS CLI 的 1.16.156 版或更高版本中可用的 `aws eks get-token` 命令或者[适用于 Kubernetes 的 AWS IAM 身份验证器](https://github.com/kubernetes-sigs/aws-iam-authenticator)），但它仍依赖于 Kubernetes [基于角色的访问控制](https://kubernetes.io/docs/admin/authorization/rbac/) (RBAC) 来进行授权。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/55f344aae21d5a8ef812d1a22953db07.png#pic_center)

## 一、将 aws-auth ConfigMap 应用到集群

1. 检查您是否已经应用了 `aws-auth` ConfigMap。

   ```
   kubectl describe configmap -n kube-system aws-auth
   ```

   如果您收到错误指示“`Error from server (NotFound): configmaps "aws-auth" not found`”，则继续以下步骤以应用库存 ConfigMap。

2. 下载、编辑和应用 AWS 身份验证器配置映射。

   1. 下载配置映射。

      ```
      curl -o aws-auth-cm.yaml https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/aws-auth-cm.yaml
      ```

   2. 使用您常用的文本编辑器打开文件。将 `<ARN of instance role (not instance profile)>` 替换为与节点关联的 IAM 角色的 Amazon Resource Name (ARN)，然后保存相应文件。请勿修改此文件中的任何其他行。

      **重要**

      角色 ARN 不能包含路径。角色 ARN 的格式必须为 `arn:aws:iam::<123456789012>:role/<role-name>`。有关更多信息，请参阅[aws-auth ConfigMap 不授予对集群的访问权限](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/troubleshooting_iam.html#security-iam-troubleshoot-ConfigMap)。

      ```
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: aws-auth
        namespace: kube-system
      data:
        mapRoles: |
          - rolearn: <ARN of instance role (not instance profile)>
            username: system:node:{{EC2PrivateDNSName}}
            groups:
              - system:bootstrappers
              - system:nodes
      ```

      您可以检查工作线程节点组的 AWS CloudFormation 堆栈输出，并查找以下值：

      - **InstanceRoleARN**（针对已使用 `eksctl` 创建的节点组）
      - **NodeInstanceRole**（针对已在 AWS Management Console 中使用 Amazon EKS 提供的 AWS CloudFormation 模板创建的节点组）

   3. 应用配置。此命令可能需要几分钟才能完成。

      ```
      kubectl apply -f aws-auth-cm.yaml
      ```

      **注意**

      如果您收到任何授权或资源类型错误，请参阅故障排除部分中的[未经授权或访问被拒绝 (kubectl)](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/troubleshooting.html#unauthorized)。

3. 查看节点的状态并等待它们达到 `Ready` 状态。

   ```
   kubectl get nodes --watch
   ```

## 二、将 IAM 用户或角色添加到 Amazon EKS 集群

1. 确保 AWS 要使用的 `kubectl` 凭证已为集群授权。预设情况下，创建该集群的 IAM 用户具有这些权限。

2. 打开 `aws-auth` ConfigMap。

   ```
   kubectl edit -n kube-system configmap/aws-auth
   ```

   **注意**

   如果您收到错误指示“`Error from server (NotFound): configmaps "aws-auth" not found`”，则使用上述过程应用库存 ConfigMap。

   示例 ConfigMap：

   ```
   apiVersion: v1
   data:
     mapRoles: |
       - groups:
         - system:bootstrappers
         - system:nodes
         rolearn: arn:aws:iam::111122223333:role/eksctl-my-cluster-nodegroup-standard-wo-NodeInstanceRole-1WP3NUE3O6UCF
         username: system:node:{{EC2PrivateDNSName}}
   kind: ConfigMap
   metadata:
     creationTimestamp: "2020-09-30T21:09:18Z"
     name: aws-auth
     namespace: kube-system
     resourceVersion: "1021"
     selfLink: /api/v1/namespaces/kube-system/configmaps/aws-auth
     uid: dcc31de5-3838-11e8-af26-02e00430057c
   ```

3. 将 IAM 用户、角色或AWS账户添加到 configMap。您无法将 IAM 组添加到 configMap 中。

   - **添加 IAM 角色（例如，对于[联合身份用户](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers.html)）：**将角色详细信息添加到 `data` 下 ConfigMap 的 `mapRoles` 部分。如果此部分在文件中尚不存在，请添加它。每个条目支持以下参数：
     - **rolearn**：要添加的 IAM 角色的 ARN。
     - **username**：Kubernetes 内要映射到 IAM 角色的用户名。
     - **groups**：Kubernetes 内角色要映射到的组的列表。有关更多信息，请参阅 Kubernetes 文档中的[默认角色和角色绑定](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings)。
   - **添加 IAM 用户：**将用户详细信息添加到 `data` 下 ConfigMap 的 `mapUsers` 部分。如果此部分在文件中尚不存在，请添加它。每个条目支持以下参数：
     - **userarn**：要添加的 IAM 用户的 ARN。
     - **username**：Kubernetes 内要映射到 IAM 用户的用户名。
     - **groups**：Kubernetes 内用户要映射到的组的列表。有关更多信息，请参阅 Kubernetes 文档中的[默认角色和角色绑定](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings)。

   例如，下面的块包含：

   - `mapRoles` 部分，此部分添加节点实例角色，以便节点可以自行注册到集群。
   - `mapUsers` 部分，其中包含来自默认AWS账户的AWS用户 `admin` 和来自其他AWS账户的 `ops-user`。两个用户均添加到 `system:masters` 组。

   将所有 `<example-values>`（包含 `<>`）替换为您自己的值。

   ```
   # Please edit the object below. Lines beginning with a '#' will be ignored,
   # and an empty file will abort the edit. If an error occurs while saving this file will be
   # reopened with the relevant failures.
   #
   apiVersion: v1
   data:
     mapRoles: |
       - rolearn: <arn:aws:iam::111122223333:role/eksctl-my-cluster-nodegroup-standard-wo-NodeInstanceRole-1WP3NUE3O6UCF>
         username: <system:node:{{EC2PrivateDNSName}}>
         groups:
           - <system:bootstrappers>
           - <system:nodes>
     mapUsers: |
       - userarn: <arn:aws:iam::111122223333:user/admin>
         username: <admin>
         groups:
           - <system:masters>
       - userarn: <arn:aws:iam::111122223333:user/ops-user>
         username: <ops-user>
         groups:
           - <system:masters>
   ```

4. 保存文件并退出您的文本编辑器。

5. 确保将 Kubernetes 用户或组（被映射了 IAM 用户或角色）绑定到具有 `RoleBinding` 或 `ClusterRoleBinding` 的 Kubernetes 角色。有关更多信息，请参阅 Kubernetes 文档中的[使用 RBAC 授权](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)。您可以下载以下示例清单，这些清单创建 `clusterrole` 和 `clusterrolebinding` 或 `role` 和 `rolebinding`：

   - **查看所有命名空间中的 Kubernetes 资源** – 文件中的组名为 `eks-console-dashboard-full-access-group`，您的 IAM 用户或角色需要在 `aws-auth` configmap 中映射到此组。如果需要，您可以在将组应用到集群之前更改该组的名称，然后在 configmap 中将您的 IAM 用户或角色映射到该组。从下列位置下载文件：

     ```
     https://s3.us-west-2.amazonaws.com/amazon-eks/docs/eks-console-full-access.yaml
     ```

   - **查看特定命名空间中的 Kubernetes 资源** – 此文件中的命名空间是 `default`，因此，如果要指定不同的命名空间，请编辑该文件，然后将其应用到集群。文件中的组名称为 `eks-console-dashboard-restricted-access-group`，您的 IAM 用户或角色需要在 `aws-auth` configmap 中映射到此组。如果需要，您可以在将组应用到集群之前更改该组的名称，然后在 configmap 中将您的 IAM 用户或角色映射到该组。从下列位置下载文件：

     ```
     https://s3.us-west-2.amazonaws.com/amazon-eks/docs/eks-console-restricted-access.yaml
     ```

6. （可选）如果您希望已添加到 configmap 中的用户能够在 AWS Management Console 中 [查看节点](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/view-nodes.html) 或 [查看工作负载](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/view-workloads.html)，则用户或角色必须具有以下两种类型的权限：

   - 在 Kubernetes 中查看资源的 Kubernetes 权限
   - 在 AWS Management Console 中查看资源的 IAM 权限。有关更多信息，请参阅[在 AWS Management Console 中查看所有集群的节点和工作负载](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/security_iam_id-based-policy-examples.html#policy_example3)。
