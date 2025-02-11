## 部署Prometheus

```shell
kubectl create namespace prometheus

helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"
```

启动端口转发

```shell
kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090
```

访问Node节点的8080，即可打开Prometheus界面

## 部署Grafana

编写配置文件，引入前面安装的Prometheus作为数据源

```shell
mkdir ~/environment/grafana

cat << EoF > ~/environment/grafana/grafana.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.prometheus.svc.cluster.local
      access: proxy
      isDefault: true
EoF
```

安装

```shell
kubectl create namespace grafana

helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='EKS!sAWSome' \
    --values ${HOME}/environment/grafana/grafana.yaml \
    --set service.type=LoadBalancer
```

在这里部署Grafana为LoadBalance模式，查看ELB URL

```shell
export ELB=$(kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "http://$ELB"
```

查看默认的密码

```shell
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```