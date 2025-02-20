> [!TIP]
>
> 文档编写时间：2021-11-17
>
> 计划更新时间：2025-02-22

# 一、RBAC是什么？

基于角色（Role）的访问控制（RBAC）是一种基于组织中用户的角色来调节控制对 计算机或网络资源的访问的方法。

RBAC 鉴权机制使用 `rbac.authorization.k8s.io` [API 组](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning) 来驱动鉴权决定，允许你通过 Kubernetes API 动态配置策略。

RBAC 的核心逻辑组件是：

**实体**
组、用户或服务帐户（代表想要执行某些操作（动作）并需要权限的应用程序的身份）。

**资源**
实体想要使用特定操作访问的 Pod、服务或机密。

**角色(Role)**
用于定义实体可以对各种资源采取的操作的规则。

**角色绑定(Role binding)**
这将**角色**附加（绑定）到实体，说明规则集定义了附加实体在指定资源上允许的操作。

有两种类型的角色（Role、ClusterRole）和各自的绑定（RoleBinding、ClusterRoleBinding）。这些区分命名空间或集群范围内的授权。

**命名空间(Namespace)**

命名空间是创建安全边界的绝佳方式，正如“命名空间”名称所暗示的那样，它们还为对象名称提供了唯一的范围。它们旨在用于多租户环境，以在同一物理集群上创建虚拟 kubernetes 集群。

## 二、将IAM实体映射到K8S集群

<font color=red size=4>前置条件：用于访问K8S集群的IAM实体</font>

获取现有的ConfigMap并保存到名为`aws-auth.yaml`的文件中

```shell
kubectl get configmap -n kube-system aws-auth -o yaml | grep -v "creationTimestamp\|resourceVersion\|selfLink\|uid" | sed '/^  annotations:/,+2 d' > aws-auth.yaml
```

<font color=red size=5>如果不存在configmap，请参考上一篇文章[AWS EKS添加集群用户或IAM角色](https://blog.csdn.net/weixin_41335923/article/details/121335441) </font>

将IAM实体映射到现有configmap

`mapUsers`使用的是IAM用户的方式，  还可以为IAM Role方式

`userarn`为IAM实体的ARN

`username`用于K8S的`RoleBinding`，Kubernetes 内要映射到 IAM 用户的用户名

```shell
cat << EoF >> aws-auth.yaml
data:
  mapUsers: |
    - userarn: arn:aws:iam::${ACCOUNT_ID}:user/rbac-user
      username: rbac-user
EoF
```

检查添加的内容是否正确

```shell
cat aws-auth.yaml
```

输出例子：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/rbac-user
      username: rbac-user
```

`apply`文件生效到集群中

```shell
kubectl apply -f aws-auth.yaml
```

## 三、创建Role和RoleBinding

创建一个名为`pod-reader`的Role

```shell
cat << EoF > rbacuser-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: rbac-test
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["list","get","watch"]
- apiGroups: ["extensions","apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EoF
```

创建一个RoleBinding将Role与User绑定

```shell
cat << EoF > rbacuser-role-binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: rbac-test
subjects:
- kind: User
  name: rbac-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EoF
```

应用Role和RoleBinding

```shell
kubectl apply -f rbacuser-role.yaml
kubectl apply -f rbacuser-role-binding.yaml
```

## 四、测试

查看`rbac-test`命名空间，可以获得结果

```shell
kubectl get pods -n rbac-test
```

如果查看其他命名空间，会出现下列错误

```shell
kubectl get pods -n kube-system

---------------------------------

No resources found.
Error from server (Forbidden): pods is forbidden: User "rbac-user" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

## 五、总结

在这里使用的是IAM User方式，还可以通过IAM Role的模式，只需要修改configmap配置文件即可。

另外本文使用的是Role+RoleBinding方式授权用户访问

Role的方式是命名空间级别，只能授予某一个`namespace`中资源的访问权限

但是在K8S中，可以使用ClusterRole的方式来授权集群范围的访问

RoleBinding是将同一个`namespace`中的subject(用户)绑定到某个特定权限的Role下

ClusterRoleBinding在整个集群级别和所有namespaces将特定的subject与ClusterRole绑定，授予权限

有关Role与ClusterRole，RoleBinding与ClusterRoleBinding的说明查看官方文档[1]

## 参考

[1] 使用 RBAC 鉴权: https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/