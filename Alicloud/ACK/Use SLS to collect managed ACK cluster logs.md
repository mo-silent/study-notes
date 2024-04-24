## 前言

### 什么是 ACK

ACK 是阿里云容器服务 Kubernetes的简称，阿里云用户可以创建ACK 专有集群和ACK 托管集群。ACK 专有集群与ACK托管集群最大的区别在于用户是否需要管理Kubernetes 集群的控制层面。在企业中，会普遍使用ACK托管集群来减少运维控制层面的成本。

**ACK专有集群和ACK托管集群对比**

![comparison](D:\study-notes\Alicloud\ACK\image\托管集群和专有集群对比.png)

### 为什么使用 SLS

一般云企业都会使用ACK托管集群，意味着控制层面的组件会被阿里云托管，用户不可见。托管的组件包括Kube-apiserver、kube-controller-manager、ack-scheduler和etcd。

虽然企业选择了ACK托管集群，但是在ACK 运维的过程中，这些组件的日志也是至关重要。运维人员能够通过托管组件日志，分析定位出具体问题所在。

**问题来了，组件是托管的，如何去采集日志？分析日志？**

在自建的Kubernetes 集群中，运维人员一般借助Loki，Logstash，fluntd，fluntbit 等开源组件来采集日志。在阿里云托管ACK 集群中，这些开源组件只能采集数据层面的数据，也就是在节点上运行的deployment, statefulset, pod 等等运行日志。

如果想要采集控制层面日志，则需要借助阿里云SLS，即日志服务。但如果控制层面使用SLS，而数据层面使用其他组件采集日志，日志采集就会存在两套方案。所以在阿里云ACK 中，集成了SLS采集数据层面日志的组件Logtail，该组件能够采集数据层面日志并推送到SLS日志中。这样日志采集系统就只有一套，运维成本会降低。

**如果多云环境下，SLS只能用于阿里云，那么IDC(自建)或其他云厂商的集群日志怎么办？**

很明显SLS的日志方案与阿里云高度耦合，不适用于IDC或其他云厂商。这个问题的日志方案会在另一个博客介绍。

## ACK 日志采集方案

### 架构





### 收集ACK 托管集群控制层组件日志

#### 开启收集

1.  创建时开启
    -   通过控制台创建集群时，组件配置页面中控制平面组件日志区域勾选开启
2.  已有集群开启
    -   登录[容器服务管理控制台](https://cs.console.aliyun.com/)，在左侧导航栏选择**集群**。
    -   在**集群列表**页面，单击目标集群名称，然后在左侧导航栏，选择**运维管理 > 日志中心**。
    -   在**日志中心**页面，单击**控制平面组件日志**页签，然后单击**开启组件日志**。

#### 查看日志

1.  Kubernetes 控制台查看
    -   通过**集群信息**入口查看控制平面组件。
        -   在集群信息管理页面单击**集群资源**页签，在列表中单击控制平面组件日志对应的Project链接。
        -   在**日志存储**页面左侧的**日志库**列表选择目标控制平面组件的日志库（Logstore）。
    -   通过**运维管理**入口查看四种控制平面组件。
        -   在集群管理左侧导航栏中，选择**运维管理 > 日志中心**。
        -   单击**控制平面组件日志**页签，然后选择目标组件查看相应的组件日志信息。
2.   SLS 控制台查看
    -   登录[日志服务控制台](https://sls.console.aliyun.com/)。
    -   在**Project列表**区域，单击目标集群对应的日志服务Project名称。
    -   在**日志存储 > 日志库**页签中，单击目标日志库（Logstore）。

#### Logstore 组件说明

| **组件**                 | **Logstore** | **说明**                                                     |
| ------------------------ | ------------ | ------------------------------------------------------------ |
| kube-apiserver           | apiserver    | kube-apiserver组件是暴露Kubernetes API接口的控制层面的组件。更多信息，请参见[kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)。 |
| kube-controller-manager  | kcm          | kube-controller-manager组件是Kubernetes集群内部的管理控制中心，内嵌了Kubernetes发布版本中核心的控制链路。更多信息，请参见[kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)。 |
| kube-scheduler           | scheduler    | kube-scheduler组件是Kubernetes集群的默认调度器。更多信息，请参见[kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)。 |
| Cloud Controller Manager | ccm          | Cloud Controller Manager提供Kubernetes与阿里云基础产品的对接能力，例如CLB（原SLB）、VPC等，功能包括管理负载均衡、跨节点通信等。更多信息，请参见[Cloud Controller Manager](https://www.alibabacloud.com/help/zh/ack/product-overview/cloud-controller-manager#concept-wk1-grd-qfb)。 |

### 通过 DaemonSet 方式采集 Kubernetes 容器文本日志

#### 工作原理

![work-principle](D:\study-notes\Alicloud\ACK\image\work-principle.svg)

**DaemonSet模式**

-   在DaemonSet模式中，Kubernetes集群确保每个节点（Node）只运行一个Logtail容器，用于采集当前节点内所有容器（Containers）的日志。
-   当新节点加入集群时，Kubernetes集群会自动在新节点上创建Logtail容器；当节点退出集群时，Kubernetes集群会自动销毁当前节点上的Logtail容器。通过DaemonSet的自动扩缩容机制以及标识型机器组，用户无需手动管理Logtail实例。

**容器发现**

-   Logtail容器采集其他容器的日志，必须发现和确定哪些容器正在运行，这个过程称为容器发现。
-   在**容器发现阶段**，Logtail容器不与Kubernetes集群的kube-apiserver进行通信，而是直接和节点上的**容器运行时守护进程**（Container Runtime Daemon）进行通信，从而获取当前节点上的所有容器信息，避免容器发现对集群kube-apiserver产生压力。kube-apiserver是集群的中央管理组件，负责提供Kubernetes API的服务，更多信息请参见[kube-apiserver](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-apiserver/)。
-   Logtail支持通过**Namespace名称**、**Pod名称**、**Pod标签、容器****环境变量**等条件指定或排除采集相应容器的日志。

**容器文件路径映射**

在Kubernetes集群中，因为Pod之间资源隔离，所以Logtail容器无法直接访问其他Pod中的容器的文件。但是，容器内的文件系统都是由宿主机的文件系统挂载形成，通过将宿主机根目录所在的文件系统挂载到Logtail容器，就可以访问宿主机上的任意文件，从而间接采集业务容器文件系统的文件。容器内文件路径与宿主机文件路径之间的关系被称为文件路径映射。

某个文件在当前容器内的路径是`/log/app.log`，假设映射后的宿主机路径是`/var/lib/docker/containers//log/app.log`。Logtail默认将宿主机根目录所在的文件系统挂载到自身的`/logtail_host`目录下，因此Logtail实际采集的文件路径为`/logtail_host/var/lib/docker/containers//log/app.log`。

#### 启用 Logtail

1.  创建时启用 Logtail

    -   在**组件配置**配置项页中，选中**使用日志服务**，表示在新建的Kubernetes集群中安装日志插件。

        当选中**使用日志服务**后，会出现创建项目（Project）的提示。关于日志服务管理日志的组织结构，请参见[项目（Project）](https://www.alibabacloud.com/help/zh/sls/product-overview/project#concept-t3x-hqn-vdb)。有以下两种创建Project的方式。

2.  已有集群启用

    -   登录[容器服务管理控制台](https://cs.console.aliyun.com/)。
    -   在控制台左侧导航栏，单击**集群**。
    -   在**集群列表**页面，单击目标集群名称或者目标集群右侧**操作**列下的**详情**。
    -   在集群管理页左侧导航栏中，选择**运维管理**>**组件管理**，并在**日志与监控**区域找到**logtail-ds**。
    -   在**logtail-ds**组件右侧，单击**安装**，并在**安装组件**对话框中单击**确定**。

#### 通过 CRD 管理 Logtail 采集配置

**CRD 工作原理**

![Logtail-crd-work-principle](D:\study-notes\Alicloud\ACK\image\Logtail-crd-work-principle.svg)

1.  日志服务通过`Deployment`部署了一个控制器`alibaba-log-controller`，该控制器负责监听 `AliyunLogConfig` 资源的变化。
2.  当用户通过kubectl或其他Kubernetes管理工具创建、更新或删除`AliyunLogConfig`资源时，`alibaba-log-controller`会监测到这些变化，然后根据资源配置文件中的内容和服务端Logtail采集配置的状态，自动向日志服务提交各种请求，如采集配置变更等。

**配置过程**

1.  编写 yaml 配置

    ```shell
    vi log-cube.yaml
    # 写以下内容
    
    apiVersion: log.alibabacloud.com/v1alpha1
    kind: AliyunLogConfig
    metadata:
      # 设置资源名，在当前Kubernetes集群内唯一。
      name: k8s-stdout-example
      namespace: kube-system
    spec:
      # [可选]设置Project名称。默认为安装Logtail组件时设置的Project。
      # project: ack-b2b-project
      # 设置Logstore名称。如果您所指定的Logstore不存在，日志服务会自动创建。
      logstore: k8s-stdout-example
      # [可选]设置机器组.默认为安装Logtail组件时，日志服务自动创建名为k8s-group-${your_k8s_cluster_id}的机器组。
      # machineGroups: k8s-group
      # 设置Logtail配置。
      logtailConfig:
        # 设置采集的数据源类型。采集标准输出时，需设置为plugin。
        inputType: plugin
        # 设置Logtail配置的名称，必须与资源名（metadata.name）相同。
        configName: k8s-stdout-example
        inputDetail:
          plugin:
            inputs:
              -
                # input type
                type: service_docker_stdout
                detail:
                  # 指定采集stdout和stderr。
                  Stdout: true
                  Stderr: true
            global:
              AlwaysOnline: true
              type: k8sStdout
    ```

2.  apply 配置文件

    ```
    kubectl apply -f log-cube.yaml
    ```

3.  控制台查看，找到Logtail 默认生成的 Project，找到 k8s-stdout-example logstore
    ![sls-log](D:\study-notes\Alicloud\ACK\image\sls-log.png)

4.  点击查询日志
    ![search-log](D:\study-notes\Alicloud\ACK\image\search-log.png)

