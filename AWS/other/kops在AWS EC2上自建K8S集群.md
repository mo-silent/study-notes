---
title: kops 在 AWS EC2 上自建 K8S 集群
slug: kops-deploy-custom-k8s
categories:
  - EKS
tags:
  - AWS
halo:
  site: https://blog.silentmo.cn
  name: 
  publish: true
---
> [!TIP]
>
> 文档编写时间：2021-11-25

## 一、下载kops

```
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
```
## 二、安装aws-iam-authenticator

1. 从 Amazon S3 中下载 Amazon EKS 提供的 `aws-iam-authenticator` 二进制文件。

   ```
   curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/aws-iam-authenticator
   ```

2. （可选）使用同一存储桶前缀中提供的 SHA-256 总和验证下载的二进制文件。

   1. 下载系统的 SHA-256 总和。要下载 Arm 版本，请先将 `<amd64>` 更改为 `arm64`，然后再运行相应命令。

      ```
      curl -o aws-iam-authenticator.sha256 https://amazon-eks.s3-us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/aws-iam-authenticator.sha256
      ```

   2. 检查下载的二进制文件的 SHA-256 总和。

      ```
      openssl sha1 -sha256 aws-iam-authenticator
      ```

   3. 将命令输出中生成的 SHA-256 总和与下载的 `aws-iam-authenticator.sha256` 文件进行比较。这两者应该匹配。

3. 将执行权限应用于二进制文件。

   ```
   chmod +x ./aws-iam-authenticator
   ```

4. 将二进制文件复制到您的 `$PATH` 中的文件夹。我们建议创建一个 `$HOME/bin/aws-iam-authenticator` 并确保 `$HOME/bin` 首先显示在您的 `$PATH` 中。

   ```
   mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$PATH:$HOME/bin
   ```

5. 将 `$HOME/bin` 添加到 `PATH` 环境变量。

   ```
   echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
   ```

6. 测试 `aws-iam-authenticator` 二进制文件的工作原理。

   ```
   aws-iam-authenticator help
   ```

## 三、创建密钥对
需要在机器中自签密钥对，用于连接创建后的集群。
```
ssh-keygen
kops create secret --name ${KOPS_CLUSTER_NAME} sshpublickey admin -i ~/.ssh/id_rsa.pub
kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
ssh -i ~/.ssh/id_rsa ec2-user@ec2-13-59-4-99.us-east-2.compute.amazonaws.com
```
## 四、部署集群

export KOPS_STATE_STORE=s3://mogd-k8s
```shell
kops create cluster --zones=ap-southeast-1a,ap-southeast-1b,ap-southeast-1c \
--name=mogd.k8s.local \
--master-zones ap-southeast-1a \
--vpc=vpc-0c350dbafa165cefb \
--ssh-public-key=~/.ssh/id_rsa.pub \
--networking amazonvpc \
--node-count 3 \
--node-size t3.medium  \
--master-count 1 \
--api-loadbalancer-type=internal \
--dns private \
--master-size t3.medium \
--topology private \
# --image ami-07191cf2912e097a6
```

修改以自动部署 `aws-iam-authenticator`。为此，您需要运行 `kops edit cluster`：

```shell
kops edit cluster --name ${NAME}
```

在 `spec,` `authorization.rbac` 和 `authentication.aws` 下添加两个密钥。应用后，这将配置控制平面以自动配置 Kubernetes RBAC 并部署 AWS IAM Authenticator。

```yaml
# ...
spec:
  # ...
  authentication:
    aws: {}
  authorization:
    rbac: {}
```

现在，保存并关闭此文件。保存后，您需要使用 `kops update cluster` 创建 `kops` 集群：

```shell
kops update cluster ${NAME} --yes --admin
```

使用 `kubectl describe pod` 进行检查：

```shell
kubectl describe po -n kube-system -l k8s-app=aws-iam-authenticator
```

这将显示您有一个集群运行，但无法启动 `aws-iam-authenticator` pod：该 pod 正在等待创建 `ConfigMap` 以便启动。现在，我们将创建 AWS IAM 策略、角色和 `ConfigMap`。

## 五、创建策略

在向任何人授予对集群的访问权限之前，您首先需要为其他 `admin` 用户创建 AWS IAM 角色和信任策略。您可以通过 AWS 控制台或使用 AWS CLI 执行此操作：

```shell
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query 'Account')
export POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')
aws iam create-role \
  --role-name KubernetesAdmin \
  --description "Kubernetes administrator role (for AWS IAM Authenticator for AWS)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'
```

现在，您可以创建一个 `ConfigMap`，用于定义拥有集群访问权限的 AWS IAM 角色：

```shell
cat >aws-auth.yaml <<EOF
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: kube-system
  name: aws-iam-authenticator
  labels:
    k8s-app: aws-iam-authenticator
data:
  config.yaml: |    
    clusterID: ${NAME}
    server:
      mapRoles:
      - roleARN: arn:aws:iam::${ACCOUNT_ID}:role/KubernetesAdmin
        username: kubernetes-admin
        groups:
        - system:masters
EOF
```

创建此文件后，您现在可以 `apply` 此配置：

```shell
kubectl apply -f aws-auth.yaml
```

部署完成后，您需要在 `kubeconfig` 中创建一个新用户。为此，您可以使用自己常用的编辑器打开 `~/. kube/config`。创建一个用户，将 `${NAME}` 替换为您的集群名称，并将 `${ACCOUNT_ID}` 替换为您的账户 ID。

```yaml
# ...
users:
- name: ${NAME}.exec
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      command: aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "${NAME}"
        - "-r"
        - "arn:aws:iam::${ACCOUNT_ID}:role/KubernetesAdmin"
```

然后，您需要修改 `context` 以引用此新用户：

```shell
kubectl config set-context $NAME --user=$NAME.exec
```