> [!TIP]
>
> 文档编写时间：2021-11-17

## 一、介绍

容器化的应用程序经常需要访问集群内运行的其他服务以及外部AWS服务，例如数据库服务（Amazon RDS）

在AWS上，控制服务之间的网络访问通常是通过[安全组](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html)

由于Node组内的所有节点共享安全组，通过将访问RDS实例的安全组附加到Node组，这些节点上运行的所有Pod都可以范围数据库。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/eb1a3fd1ed6add24dffc54d8e8e068fc.png#pic_center)


如上图，实际生产只需要绿色的Pod可以访问RDS数据库，但只是节点安全组是做不到的。

[Pod 的安全组](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html)将 Amazon EC2 安全组与 Kubernetes pod 集成。可以使用 Amazon EC2 安全组来定义规则，这些规则允许传入和传出部署到在许多 Amazon EC2 实例类型上运行的节点的 pod 的入站和出站网络流量。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/76548184038fcae0b0188f404e6214b4.png#pic_center)


<font color=red size=5>前置条件:对于Pod安全组，目前不支持t3类型的实例，只支持m5`, `c5`, `r5`, `p3`, `m6g`, `c6g`, and `r6g实例类型</font>

## 二、创建安全组（可选）

在这里创建两个安全组，一个用于RDS数据库的安全组，一个用于Pod安全组。可通过控制台创建，添加相应规则即可

### 2.1 首先创建RDS安全组（可选，数据库存在可跳过）

`<clusterName>`为EKS集群名称

```shell
export VPC_ID=$(aws eks describe-cluster \
    --name <clusterName> \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)

# create RDS security group
aws ec2 create-security-group \
    --description 'RDS SG' \
    --group-name 'RDS_SG' \
    --vpc-id ${VPC_ID}

# save the security group ID for future use
export RDS_SG=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=RDS_SG Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" --output text)

echo "RDS security group ID: ${RDS_SG}"
```

### 2.2 创建Pod安全组

```shell
# create the POD security group
aws ec2 create-security-group \
    --description 'POD SG' \
    --group-name 'POD_SG' \
    --vpc-id ${VPC_ID}

# save the security group ID for future use
export POD_SG=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=POD_SG Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" --output text)

echo "POD security group ID: ${POD_SG}"
```

Pod 需要与其节点通信以进行 DNS 解析，因需要更新节点组安全组规则

`--filters`中的`Name`和`Values`根据实际EKS节点组安全组的标签填写

```shell
export NODE_GROUP_SG=$(aws ec2 describe-security-groups \
    --filters Name=tag:Name,Values=eks-cluster-sg-workshop-eksctl-* Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" \
    --output text)
echo "Node Group security group ID: ${NODE_GROUP_SG}"

# allow POD_SG to connect to NODE_GROUP_SG using TCP 53
aws ec2 authorize-security-group-ingress \
    --group-id ${NODE_GROUP_SG} \
    --protocol tcp \
    --port 53 \
    --source-group ${POD_SG}

# allow POD_SG to connect to NODE_GROUP_SG using UDP 53
aws ec2 authorize-security-group-ingress \
    --group-id ${NODE_GROUP_SG} \
    --protocol udp \
    --port 53 \
    --source-group ${POD_SG}
```

### 2.3 为RDS安全组添加入站规则，允许Pod安全组连接到数据库

如果RDS数据库已经存在，将`${RDS_SG}`修改为RDS数据库的安全组ID

```shell
# Allow POD_SG to connect to the RDS
aws ec2 authorize-security-group-ingress \
    --group-id ${RDS_SG} \
    --protocol tcp \
    --port 5432 \
    --source-group ${POD_SG}
```

## 三、获取RDS Endpoint

### 3.1 创建RDS数据库（可选，数据库存在可跳过）

`PUBLIC_SUBNETS_ID`获取子网ID，这里获取EKS集群子网ID来创建RDS测试数据库子网组

```shell
export PUBLIC_SUBNETS_ID=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=eksctl-workshop-eksctl-cluster/SubnetPublic*" \
    --query 'Subnets[*].SubnetId' \
    --output json | jq -c .)

# create a db subnet group
aws rds create-db-subnet-group \
    --db-subnet-group-name rds-workshop \
    --db-subnet-group-description rds-workshop \
    --subnet-ids ${PUBLIC_SUBNETS_ID}
```

创建数据库

```shell
# get RDS SG ID
export RDS_SG=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=RDS_SG Name=vpc-id,Values=${VPC_ID} \
    --query "SecurityGroups[0].GroupId" --output text)

# generate a password for RDS
export RDS_PASSWORD="$(date | md5sum  |cut -f1 -d' ')"
echo ${RDS_PASSWORD}  > ~/environment/sg-per-pod/rds_password


# create RDS Postgresql instance
aws rds create-db-instance \
    --db-instance-identifier rds-workshop \
    --db-name workshop \
    --db-instance-class db.t3.micro \
    --engine postgres \
    --db-subnet-group-name rds-workshop \
    --vpc-security-group-ids $RDS_SG \
    --master-username workshop \
    --publicly-accessible \
    --master-user-password ${RDS_PASSWORD} \
    --backup-retention-period 0 \
    --allocated-storage 20
```

### 3.2 获取数据库的Endpoint

这里通过CLI来获取，也可以通过控制台直接查看

```shell
# get RDS endpoint
export RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier rds-workshop \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text)

echo "RDS endpoint: ${RDS_ENDPOINT}"
```

### 3.3 添加测试数据（可选）

```shell
sudo amazon-linux-extras install -y postgresql12

cd sg-per-pod

cat << EoF > ~/environment/sg-per-pod/pgsql.sql
CREATE TABLE welcome (column1 TEXT);
insert into welcome values ('--------------------------');
insert into welcome values ('test');
insert into welcome values ('--------------------------');
EoF

export RDS_PASSWORD=$(cat ~/environment/sg-per-pod/rds_password)

psql postgresql://workshop:${RDS_PASSWORD}@${RDS_ENDPOINT}:5432/workshop \
    -f ~/environment/sg-per-pod/pgsql.sql
```

## 四、EKS CNI网络配置

使用Pod安全组，需要启用CNI配置，EKS集群在Kubernetes控制平面上需要运行以下两个组件：

- **mutating webhook**:负责向需要安全组的 pod 添加限制和请求。
- **resource controller**:负责管理[网络接口](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html) 与这些Pod相关联。

此功能，每个工作节点将与单个主干网络接口和多个分支网络接口相关联。中继接口充当连接到实例的标准网络接口。

然后VPC 资源控制器将分支接口关联到中继接口。这增加了每个实例可以附加的网络接口的数量。

由于安全组是通过网络接口指定的，因此可以将需要特定安全组的 pod 调度到分配给工作节点的这些额外网络接口上。

### 4.1 为节点组角色授权

首先，需要为节点组角色附加一个新的 IAM 策略，以允许 EC2 实例管理网络接口、它们的私有 IP 地址以及它们与实例的连接和分离。

`${ROLE_NAME}`为节点组角色名称

```shell
aws iam attach-role-policy \
    --policy-arn arn:aws:iam::aws:policy/AmazonEKSVPCResourceController \
    --role-name ${ROLE_NAME}
```

### 4.2 启用CNI插件

通过`ENABLE_POD_ENI`在 aws-node中将变量设置为 true来启用 CNI 插件，来管理 pod 的网络接口`DaemonSet`。

```shell
kubectl -n kube-system set env daemonset aws-node ENABLE_POD_ENI=true

# let's way for the rolling update of the daemonset
kubectl -n kube-system rollout status ds aws-node
```

一旦此设置设置为 true，对于集群中的每个节点，插件都会`vpc.amazonaws.com/has-trunk-attached=true`向兼容实例添加带有值的标签。VPC 资源控制器创建并附加一个称为中继网络接口的特殊网络接口，其描述为 aws-k8s-trunk-eni。

```shell
 kubectl get nodes \
  --selector  eks.amazonaws.com/nodegroup=nodegroup-sec-group \
  --show-labels
```

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3d48e3bc69e96552390c3d36fee1e0e6.png#pic_center)


## 五、安全组策略

在创建集群时还自动添加了一个新的自定义资源定义 (CRD)。集群管理员可以通过`SecurityGroupPolicy`CRD指定将哪些安全组分配给 pod 。在命名空间中，可以根据 Pod 标签或根据与 Pod 关联的服务帐户的标签来选择 Pod。对于任何匹配的 pod，还可以定义要应用的安全组 ID。

Webhook 会监视`SecurityGroupPolicy`自定义资源的任何更改，并自动将匹配的 Pod 与将 Pod 调度到具有可用分支网络接口容量的节点所需的扩展资源请求一起注入。一旦 pod 被调度，资源控制器将创建一个分支接口并将其附加到主干接口。成功连接后，控制器会向带有分支接口详细信息的 pod 对象添加注释。

创建带有标签`app:green-pod`的pod会自动附加一个安全组的策略

```shell
cat << EoF > ~/environment/sg-per-pod/sg-policy.yaml
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: allow-rds-access
spec:
  podSelector:
    matchLabels:
      app: green-pod
  securityGroups:
    groupIds:
      - ${POD_SG}
EoF
```

将策略部署到特定的`namespace`中

```shell
kubectl create namespace sg-per-pod

kubectl -n sg-per-pod apply -f ~/environment/sg-per-pod/sg-policy.yaml

kubectl -n sg-per-pod describe securitygrouppolicy
```

Output：

```yaml
Name:         allow-rds-access
Namespace:    sg-per-pod
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"vpcresources.k8s.aws/v1beta1","kind":"SecurityGroupPolicy","metadata":{"annotations":{},"name":"allow-rds-access","namespac...
API Version:  vpcresources.k8s.aws/v1beta1
Kind:         SecurityGroupPolicy
Metadata:
  Creation Timestamp:  2020-12-03T04:35:57Z
  Generation:          1
  Resource Version:    9142629
  Self Link:           /apis/vpcresources.k8s.aws/v1beta1/namespaces/sg-per-pod/securitygrouppolicies/allow-rds-access
  UID:                 bf1e329d-816e-4ab0-abe8-934cadabfdd3
Spec:
  Pod Selector:
    Match Labels:
      App:  green-pod
  Security Groups:
    Group Ids:
      sg-0ff967bc903e9639e
Events:  <none>
```

## 六、创建Secret

Pod需要访问数据库，因此需要安全凭证。

在Kubernetes中，可通过secret为Pod提供安全凭证

`RDS_PASSWORD`:为数据库密码

`RDS_ENDPOINT`:为数据库端点

```shell
export RDS_PASSWORD=$(cat ~/environment/sg-per-pod/rds_password)

export RDS_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier rds-workshop \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text)

kubectl create secret generic rds\
    --namespace=sg-per-pod \
    --from-literal="password=${RDS_PASSWORD}" \
    --from-literal="host=${RDS_ENDPOINT}"

kubectl -n sg-per-pod describe  secret rds
```

Output：

```yaml
Name:         rds
Namespace:    sg-per-pod
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
host:      56 bytes
password:  32 bytes
```

## 7、测试(可选)

部署两个测试Pod，一个带有标签`app:green-pod`，一个标签`app:red-pod`

```shell
cd ~/environment/sg-per-pod

curl -s -O https://www.eksworkshop.com/beginner/115_sg-per-pod/deployments.files/green-pod.yaml
curl -s -O https://www.eksworkshop.com/beginner/115_sg-per-pod/deployments.files/red-pod.yaml
```

### 7.1 Green Pod

```shel
kubectl -n sg-per-pod apply -f ~/environment/sg-per-pod/green-pod.yaml

kubectl -n sg-per-pod rollout status deployment green-pod
```

验证日志：

```shell
export GREEN_POD_NAME=$(kubectl -n sg-per-pod get pods -l app=green-pod -o jsonpath='{.items[].metadata.name}')

kubectl -n sg-per-pod  logs -f ${GREEN_POD_NAME}
```

Output:

```shell
[('-------------------------',), ('test',), ('-- ---------------------',)] 
[('---------------------- ----',), ('test',), ('----------------------------------------',)]
```

查看ENI

```shell
kubectl -n sg-per-pod  describe pod $GREEN_POD_NAME | head -11
```

Output：

```yaml
Name:         green-pod-5c786d8dff-4kmvc
Namespace:    sg-per-pod
Priority:     0
Node:         ip-192-168-33-222.us-east-2.compute.internal/192.168.33.222
Start Time:   Thu, 03 Dec 2020 05:25:54 +0000
Labels:       app=green-pod
              pod-template-hash=5c786d8dff
Annotations:  kubernetes.io/psp: eks.privileged
              vpc.amazonaws.com/pod-eni:
                [{"eniId":"eni-0d8a3a3a7f2eb57ab","ifAddress":"06:20:0d:3c:5f:bc","privateIp":"192.168.47.64","vlanId":1,"subnetCidr":"192.168.32.0/19"}]
Status:       Running
```



通过上述命令拿到ENI，登陆AWS控制台，查看前面创建的Pod安全组，（Network interface ID）是否附加了上述命令的ENI

### 7.2 Red Pod

```shell
kubectl -n sg-per-pod apply -f ~/environment/sg-per-pod/red-pod.yaml

kubectl -n sg-per-pod rollout status deployment red-pod
```

验证日志：

```shell
export RED_POD_NAME=$(kubectl -n sg-per-pod get pods -l app=red-pod -o jsonpath='{.items[].metadata.name}')

kubectl -n sg-per-pod  logs -f ${RED_POD_NAME}
```

会得到错误信息`Database connection failed due to timeout expired`

查看ENI，Pod没有*eniId* `annotation`

```shell
kubectl -n sg-per-pod  describe pod ${RED_POD_NAME} | head -11
```

## 参考

[1] 适用于Pod的安全组：https://docs.amazonaws.cn/eks/latest/userguide/security-groups-for-pods.html