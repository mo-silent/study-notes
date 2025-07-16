---
title: AWS EKS上使用 OpenSearch 部署 EFK 日志分析收集系统
slug: eks-use-efk-log-system
categories:
  - EKS
tags:
  - AWS
halo:
  site: https://blog.silentmo.cn
  name: 184c07b6-e3fa-459b-9aa1-24952d1ea36c
  publish: true
---

> [!TIP]
>
> 文档编写时间：2021-12-03

# 一、为`Fluent Bit`配置`IRSA`

1. 创建OIDC身份提供商

   ```shell
   eksctl utils associate-iam-oidc-provider \
       --cluster {clustername} \
       --approve
   ```

2. 创建IAM角色和策略

   ```shell
   mkdir ~/environment/logging/
   
   export ES_DOMAIN_NAME="eks-logging"
   
   cat <<EoF > ~/environment/logging/fluent-bit-policy.json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Action": [
                   "es:ESHttp*"
               ],
               "Resource": "arn:aws:es:${AWS_REGION}:${ACCOUNT_ID}:domain/${ES_DOMAIN_NAME}",
               "Effect": "Allow"
           }
       ]
   }
   EoF
   
   aws iam create-policy   \
     --policy-name fluent-bit-policy \
     --policy-document file://~/environment/logging/fluent-bit-policy.json
   ```

   创建角色

   ```shell
   kubectl create namespace logging
   
   eksctl create iamserviceaccount \
       --name fluent-bit \
       --namespace logging \
       --cluster {clustername} \
       --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/fluent-bit-policy" \
       --approve \
       --override-existing-serviceaccounts
   ```

## 二、配置`Amazon OpenSearch`访问

`${ES_DOMAIN_NAME}`为`OpenSearch`的名称

```shell
# We need to retrieve the Fluent Bit Role ARN
export FLUENTBIT_ROLE=$(eksctl get iamserviceaccount --cluster {clustername} --namespace logging -o json | jq '.[].status.roleARN' -r)

# Get the Amazon OpenSearch Endpoint
export ES_ENDPOINT=$(aws opensearch describe-domain --domain-name ${ES_DOMAIN_NAME} --output text --query "DomainStatus.Endpoint")

# Update the Elasticsearch internal database
curl -sS -u "${ES_DOMAIN_USER}:${ES_DOMAIN_PASSWORD}" \
    -X PATCH \
    https://${ES_ENDPOINT}/_opendistro/_security/api/rolesmapping/all_access?pretty \
    -H 'Content-Type: application/json' \
    -d'
[
  {
    "op": "add", "path": "/backend_roles", "value": ["'${FLUENTBIT_ROLE}'"]
  }
]
'
```

## 三、部署`Fluent Bit`

```shell
cd ~/environment/logging

# get the Amazon OpenSearch Endpoint
export ES_ENDPOINT=$(aws es describe-elasticsearch-domain --domain-name ${ES_DOMAIN_NAME} --output text --query "DomainStatus.Endpoint")

curl -Ss https://www.eksworkshop.com/intermediate/230_logging/deploy.files/fluentbit.yaml \
    | envsubst > ~/environment/logging/fluentbit.yaml
    
# deploy
kubectl apply -f ~/environment/logging/fluentbit.yaml

# watch
kubectl --namespace=logging get pods
```

## 四、在`OpenSearch`查看仪表盘

创建`Dashboards`

![dashboards](https://gallery-lsky.silentmo.cn/i_blog/2025/07/eks-opensearch-dashbord.png)


添加数据

![add-data](https://gallery-lsky.silentmo.cn/i_blog/2025/07/eks-opensearch-dashbord-add-data.png)


添加索引

![add-index](https://gallery-lsky.silentmo.cn/i_blog/2025/07/eks-opensearch-dashbord-add-index.png)


添加`*fluent-bit*`作为索引模式，然后单击**下一步**

选择**@timestamp**作为时间过滤器字段名称并通过单击**创建索引模式**关闭配置窗口

![add-index-filter](https://gallery-lsky.silentmo.cn/i_blog/2025/07/eks-opensearch-dashbord-add-index-filter.png)
