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

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/dba3de8d236a4d73d7ba1b997718f2e2.png#pic_center)


添加数据

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a786cfc7359f9215ebe4b8aa9c58ecaf.png#pic_center)


添加索引

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b8907017f9bd829b0baa677d263f3fe3.png#pic_center)


添加`*fluent-bit*`作为索引模式，然后单击**下一步**

选择**@timestamp**作为时间过滤器字段名称并通过单击**创建索引模式**关闭配置窗口

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a4af46339b45d7fea71efdc63f5ee40d.png#pic_center)