> [!TIP]
>
> 文档编写时间：2021-12-08
>
> 更新时间：2021-12-08

## 一、创建的ALB多目标组

该场景解决的是一个Ingress配置多个规则，每个规则下的`SVC`检查路径和端口不一致

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: "*.foo.com"
    http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: service2
            port:
              number: 8080
```

### 方法

通过在`service`中添加`AWS  Load Balancer Controller`的目标组注解`Annotations`，而不是在`ingress`中添加 

```yaml
alb.ingress.kubernetes.io/healthcheck-port
alb.ingress.kubernetes.io/healthcheck-path
```

## 二、多个Ingress一个ELB

该场景解决的是多个`namespace`下的`Ingress`只创建一个ELB

### 方法

在对应的`Ingress`中添加`Annotations`来创建一个`IngressGroup`

```
alb.ingress.kubernetes.io/group.name
```

通过`alb.ingress.kubernetes.io/group.order` 指定 IngressGroup 中所有 Ingress 的顺序

### 限制

`IngressGroup `功能只应在所有具有` RBAC` 权限的 `Kubernetes` 用户创建/修改`Ingress `资源都在信任边界内时使用。

## 三、灰度发布

在AWS官方产品中，可以通过`APP Mesh`来实现集群的灰度发布，但不建议使用。

建议使用开源的服务治理`istio`

## 四、日志

在AWS EKS中使用的日志分析方案，基本上使用开源成熟的日志分析系统。

- ELK
- EFK：`https://blog.csdn.net/weixin_41335923/article/details/121692400`

针对集群节点的日志，可以使用`Cloudwatch Log agent`来推送到`Cloudwatch Log`

```shell
#!/bin/bash

echo "Install cloudwatch agent"
yum -y install wget
cd /opt && wget https://s3.cn-north-1.amazonaws.com.cn/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
rpm -U ./amazon-cloudwatch-agent.rpm

cd /opt/aws/amazon-cloudwatch-agent/etc && wget https://goclouds-ops-public.s3.cn-north-1.amazonaws.com.cn/CloudWatchAgent/amazon-cloudwatch-agent.json
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s

agent_status=`/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status | head -n 2 | grep "status" | awk -F ":" '{print $2}'`
if [ $agent_status == "running" ]
then
    echo "Cloudwatch agent install successful and push metrics successful"
else
    echo "Cloudwatch agent Out of order ！"
fi
```

## 五、监控

在AWS EKS中的监控方案，一般使用开源的`Prometheus+Grafana`

还可以使用AWS官方推出的`Cloudwatch Container Insight`

但AWS官方的方案相比开源方案的监控指标少，因此目前都在使用开源方案