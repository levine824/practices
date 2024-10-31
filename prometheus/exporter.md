# Mysql Exporter

## 先决条件

1. 已搭建 Kubernetes；
2. 已部署 Prometheus ;
3. 已部署 Mysql。

## 创建 Mysql 用户

```sql
# 在被监控 mysql 中执行如下 sql，创建 mysql-exporter 对应用户
CREATE USER 'mysqlexporter'@"%"  IDENTIFIED BY 'mysqlexporter';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysqlexporter'@'%'  IDENTIFIED BY 'mysqlexporter' WITH MAX_USER_CONNECTIONS 30; 
GRANT select on performance_schema.* to "mysqlexporter"@"%" IDENTIFIED BY 'mysqlexporter';
flush privileges;
```

## 安装 Mysql Exporter

下载 mysql-exporter 的 chart：

```she
helm search repo mysql-exporter
helm pull stable/prometheus-mysql-exporter
tar -zxvf prometheus-mysql-exporter-0.7.1.tgz
```

修改 values.yaml 中的配置：

```shell
mysql:
  db: ""
  host: "<IP>"
  param: ""
  pass: "mysqlexporter"
  port: 3306
  protocol: ""
  user: "mysqlexporter"
  existingSecret: false
```

安装 mysql-exporter：

```shell
helm install mysql-exporter -f values.yaml ./prometheus-mysql-exporter -n sdp-test
```

## 创建 ServiceMonitor

```shell
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    release: kube-prometheus-stack # prometheus 默认通过 prometheus: kube-prometheus 发现 ServiceMonitor，只要写上这个标签 prometheus 服务就能发现这个 ServiceMonitor
  name: prometheus-exporter-mysql
  namespace: monitoring
spec:
  jobLabel: mysql # jobLabel 指定的标签的值将会作为 prometheus 配置文件中 scrape_config下 job_name 的值，也就是target，如果不写，默认为 service 的 name
  selector:
    matchLabels:
      app: prometheus-mysql-exporter # 由于前面查看 mysql-exporter 的 service 信息中标签包含了 app: prometheus-mysql-exporter 这个标签，写上就能匹配到
      release: mysql-exporter
  namespaceSelector:
    any: true #表示从所有 namespace 中去匹配，如果只想选择某一命名空间中的 service，可以使用 matchNames: []的方式
    # mathNames： []
  endpoints:
  - port: mysql-exporter #前面查看 mysql-exporter 的 service 信息中，提供 mysql 监控信息的端口是 Port: mysql-exporter  9104/TCP，所以这里填 mysql-exporter
    interval: 30s #每30s获取一次信息
  # path: /metrics HTTP path to scrape for metrics，默认值为/metrics
    honorLabels: true
```

```shell
kubectl create -f servicemonitor.yaml
```

成功以后在 Prometheus 界面上可以看到如下 target：

![img](../assets/exporter_1.png)

## 配置告警规则

创建 PrometheusRule，mysql-rules.yaml：

```yaml
apiVersion: monitoring.coreos.com/v1 # 这和 ServiceMonitor 一样
kind: PrometheusRule  # 该资源类型是 PrometheusRule，这也是一种自定义资源（CRD）
metadata:
  labels:
    app: kube-prometheus-stack
    release: kube-prometheus-stack  # 同 ServiceMonitor，ruleSelector也会默认选择标签为 prometheus: kube-prometheus 的 PrometheusRule 资源
  name: prometheus-rule-mysql
  namespace: monitoring
spec:
  groups: # 编写告警规则，和 prometheus 的告警规则语法相同
  - name: mysql.rules
    rules:
    - alert: TooManyErrorFromMysql
      expr: sum(irate(mysql_global_status_connection_errors_total[1m])) > 10
      labels:
        severity: critical
      annotations:
        description: mysql产生了太多的错误.
        summary: TooManyErrorFromMysql
    - alert: TooManySlowQueriesFromMysql
      expr: increase(mysql_global_status_slow_queries[1m]) > 10
      labels:
        severity: critical
      annotations:
        description: mysql一分钟内产生了{{ $value }}条慢查询日志.
        summary: TooManySlowQueriesFromMysql
```

```shell
kubectl apply -f mysql-rules.yaml
```

![img](../assets/exporter_2.png)

更新 Alertmanager 配置：

```shell
kubectl describe sts alertmanager-kube-prometheus-stack-alertmanager -n monitoring
```

![img](../assets/exporter_3.png)

![img](../assets/exporter_4.png)

修改 alertmanager-kube-prometheus-stack-alertmanager，alertmanager.yaml：

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.163.com:25'
  smtp_from: 'prometheusAlert0@163.com'
  smtp_auth_username: 'prometheusAlert0'
  smtp_auth_password: 'IPWIRJQJPFMBURFN'
  smtp_require_tls: false
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'email'
receivers:
- name: 'email'
  email_configs:
  - to: '13917665872@163.com'
templates:
- /etc/alertmanager/config/*.tmpl
```

```shell
base64 alertmanager.yaml > secrets.txt
kubectl edit secrets alertmanager-kube-prometheus-stack-alertmanager -n monitoring
```

将 Alertmanager.yaml 后面的内容替换为 secrets.txt 中的内容，并删除 alertmanager-kube-prometheus-stack-alertmanager-generated secret。

# 参考链接

[1]: https://www.yuque.com/youngfit/qok2pe/nypstd#b8abd38a	"组件监控实例"

