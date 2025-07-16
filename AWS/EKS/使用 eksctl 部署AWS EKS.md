---
title: 使用 eksctl 部署 AWS EKS
slug: eks-use-eksctl-deploy
categories:
  - EKS
tags:
  - AWS
halo:
  site: https://blog.silentmo.cn
  name: dd43562a-c8cd-44a7-93e3-03485ea1908c
  publish: true
---
# 一、工具安装

### 1.1 安装kubectl

**在 Linux 上安装 `kubectl`**

1. 从 Amazon S3 为集群的 Kubernetes 版本下载 Amazon EKS 提供的 `kubectl` 二进制文件。要下载 Arm 版本，请先将 `amd64` 更改为 `arm64`，然后再运行相应命令。

   - **Kubernetes 1.21：**

     ```shell
     curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
     ```

   - **Kubernetes 1.20：**

     ```shell
     curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.20.4/2021-04-12/bin/linux/amd64/kubectl
     ```

   - **Kubernetes 1.19：**

     ```shell
     curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
     ```

   - **Kubernetes 1.18：**

     ```shell
     curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
     ```

2. 将执行权限应用于二进制文件。

   ```shell
   chmod +x ./kubectl
   ```

3. 将二进制文件复制到您的 `PATH` 中的文件夹。如果您已经安装了某个版本的 `kubectl`，建议您创建一个 `$HOME/bin/kubectl` 并确保 `$HOME/bin` 先出现在您的 `$PATH` 中。

   ```shell
   mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
   ```

4. (可选) 将 `$HOME/bin` 路径添加到 shell 初始化文件，以便在打开 shell 时配置此路径。

   **注意**

   这一步假设您使用 Bash Shell；如果使用其他 Shell，请将命令更改为使用您的特定 Shell 的初始化文件。

   ```shell
   echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
   ```

5. 安装 `kubectl` 后，可以使用以下命令验证其版本：

   ```shell
   kubectl version --short --client
   ```

### 1.2 安装eksctl

**使用 `eksctl` 在 Linux 上安装或升级 `curl`**

1. 使用以下命令下载并提取最新版本的 `eksctl`。

   ```shell
   curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
   ```

2. 将提取的二进制文件移至 `/usr/local/bin`。

   ```shell
   sudo mv /tmp/eksctl /usr/local/bin
   ```

3. 使用以下命令测试您的安装是否成功。

   ```shell
   eksctl version
   ```

### 1.3 安装Helm(可选)

- 如果您将 macOS 与 [Homebrew](https://brew.sh/) 配合使用，请使用以下命令安装二进制文件。

  ```shell
  brew install helm
  ```

- 如果您将 Windows 与 [Chocolatey](https://chocolatey.org/) 配合使用，请使用以下命令安装二进制文件。

  ```shell
  choco install kubernetes-helm
  ```

- 如果您正在使用 Linux，请使用以下命令来安装二进制文件。

  ```shell
  $ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
  $ chmod 700 get_helm.sh
  $ ./get_helm.sh
  ```

# 二、命令式创建

### 2.1 创建Amazon EC2 Linux托管节点集群

```shell
eksctl create cluster \
--name my-cluster \
--region us-west-2 \
--with-oidc \
--ssh-access \
--ssh-public-key <your-key> \
--managed
```

### 2.2 创建Fargate Linux节点

```shell
eksctl create cluster \
--name my-cluster \
--region us-west-2 \
--fargate
```

### 2.3 查看资源

```shell
kubectl get nodes -o wide
```

### 2.4 删除集群

```shell
eksctl delete cluster --name my-cluster --region us-west-2
```

# 三、配置文件(yaml)创建

### 3.1 不选择VPC

`nodeGroups`是非托管的节点组，`managedNodeGroups`为EKS托管节点组，托管节点组能够在控制台显示。

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: basic-cluster
  region: eu-north-1

nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 10
    volumeSize: 80
    ssh:
      allow: true # will use ~/.ssh/id_rsa.pub as the default ssh key
  - name: ng-2
    instanceType: m5.xlarge
    desiredCapacity: 2
    volumeSize: 100
    ssh:
      publicKeyPath: ~/.ssh/ec2_id_rsa.pub
managedNodeGroups:
  - name: ng-1-workers
    labels: { role: workers }
    instanceType: m5.xlarge
    desiredCapacity: 10
    volumeSize: 80
    privateNetworking: true
  - name: ng-2-builders
    labels: { role: builders }
    instanceType: m5.2xlarge
    desiredCapacity: 2
    volumeSize: 100
    privateNetworking: true
```

### 3.2 选择现有VPC

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: cluster-in-existing-vpc
  region: eu-north-1

vpc:
  subnets:
    private:
      eu-north-1a: { id: subnet-0ff156e0c4a6d300c }
      eu-north-1b: { id: subnet-0549cdab573695c03 }
      eu-north-1c: { id: subnet-0426fb4a607393184 }

nodeGroups:
  - name: ng-1-workers
    labels: { role: workers }
    instanceType: m5.xlarge
    desiredCapacity: 10
    privateNetworking: true
  - name: ng-2-builders
    labels: { role: builders }
    instanceType: m5.2xlarge
    desiredCapacity: 2
    privateNetworking: true
    iam:
      withAddonPolicies:
        imageBuilder: true
```

### 3.3 运行创建命令

```shell
eksctl create cluster -f cluster.yaml
```

### 3.4 删除集群

```shell
eksctl delete cluster -f cluster.yaml
```

# 参考

**eksctl**：https://eksctl.io/introduction/

**AWS EKS文档**: https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/what-is-eks.html
