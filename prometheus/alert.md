# 邮箱告警

## 开启邮箱 SMTP

开启 SMTP 服务，并记录授权码。

## 编写告警规则

`prometheus/rules/host_monitor.yml`:

```yaml
groups:
- name: node-up
  rules:
  - alert: node-up
    expr: up == 0
    for: 10s
    labels:
      severity: warning
      team: node
    annotations:
      summary: "{{ $labels.instance }} 服务已停止运行超过 10s！"

====================================================================================
# alert：告警规则的名称。 
# expr：基于 PromQL 表达式告警触发条件，用于计算是否有时间序列满足该条件。
# for：评估等待时间，可选参数。用于表示只有当触发条件持续一段时间后才发送告警。在等待期间新产生告警的状态为 pending。 
# labels：自定义标签，允许用户指定要附加到告警上的一组附加标签。 
# annotations：用于指定一组附加信息，比如用于描述告警详细信息的文字等，annotations 的内容在告警产生时会一同作为参数发送到 Alertmanager。 
# summary 描述告警的概要信息，description 用于描述告警的详细信息。 
# 同时 Alertmanager 的 UI 也会根据这两个标签值，显示告警信息。
```

> 告警状态有三种状态：Inactive、Pending、Firing。
>
> 1、Inactive：非活动状态，表示正在监控，但是还未有任何警报触发。
>
> 2、Pending：表示这个警报必须被触发。由于警报可以被分组、压抑/抑制或静默/静音，所 以等待验证，一旦所有的验证都通过，则将转到 Firing 状态。
>
> 3、Firing：将警报发送到 AlertManager，它将按照配置将警报的发送给所有接收者。一旦警 报解除，则将状态转到 Inactive，如此循环。

## 编写邮件模板

email.tmpl：

```tmpl
{{ define "email.to.html" }}
{{- range $index, $alert := .Alerts -}}
======== 异常告警 ========
告警名称：{{ $alert.Labels.alertname }}
告警级别：{{ $alert.Labels.severity }}
告警机器：{{ $alert.Labels.instance }} {{ $alert.Labels.device }}
告警详情：{{ $alert.Annotations.summary }}
告警时间：{{ $alert.StartsAt.Format "2006-01-02 15:04:05" }}
========== END ==========
{{- end }}
{{- end }}
{{- if gt (len .Alerts.Resolved) 0 -}}
{{- range $index, $alert := .Alerts -}}
======== 告警恢复 ========
告警名称：{{ $alert.Labels.alertname }}
告警级别：{{ $alert.Labels.severity }}
告警机器：{{ $alert.Labels.instance }}
告警详情：{{ $alert.Annotations.summary }}
告警时间：{{ $alert.StartsAt.Format "2006-01-02 15:04:05" }}
恢复时间：{{ $alert.EndsAt.Format "2006-01-02 15:04:05" }}
========== END ==========
{{- end }}
{{- end }}
```

以上为基本模板，此处做一些修改：

```tmpl
{{ define "email.from" }}<EMAIL>{{ end }}
{{ define "email.to" }}<EMAIL>{{ end }}
{{ define "email.to.html" }}
{{ if gt (len .Alerts.Firing) 0 }}{{ range .Alerts }}
@告警: <br>
告警程序: prometheus_alert <br>
告警级别: {{ .Labels.severity }} 级 <br>
告警类型: {{ .Labels.alertname }} <br>
故障主机: {{ .Labels.instance }} <br>
告警主题: {{ .Annotations.summary }} <br>
告警详情: {{ .Annotations.description }} <br>
触发时间: {{ .StartsAt.Add 28800e9 }} <br>
{{ end }}
{{ end }}
{{ if gt (len .Alerts.Resolved) 0 }}{{ range .Alerts }}
@恢复: <br>
告警主机：{{ .Labels.instance }} <br>
告警主题：{{ .Annotations.summary }} <br>
恢复时间: {{ .EndsAt.Add 28800e9 }} <br>
{{ end }}
{{ end }}
{{ end }}
```

`<EMAIL>` 替换为实际邮箱。 如果希望动态传参发件人收件人，可以参考链接[1]。

注意报警时间，显示的是 utc 时间，比北京时间慢了8个小时，因此增加了8小时。

## 配置 AlertManager 告警

alertmanager.yml 配置文件详解：

```yaml
# 全局配置项
global: 
  resolve_timeout: 5m #处理超时时间，默认为5min
  smtp_from: 'xxx@163.com' # 发送邮箱名称
  smtp_smarthost: 'smtp.163.com:25' # 邮箱 smtp 服务器代理
  smtp_auth_username: 'xxx@163.com' # 邮箱名称
  smtp_auth_password: 'xxx' # 邮箱授权码

templates:
  - '/usr/local/alertmanager/email.tmpl'
  
# 定义路由树信息
route:
  group_by: ['alertname'] # 报警分组依据
  group_wait: 10s # 最初即第一次等待多久时间发送一组警报的通知
  group_interval: 10s # 在发送新警报前的等待时间
  repeat_interval: 1m # 发送重复警报的周期 对于 email 配置中，此项不可以设置过低，否则将会由于邮件发送太多频繁，被 smtp 服务器拒绝
  receiver: 'email' # 发送警报的接收者的名称，以下 receivers name 的名称

# 定义警报接收者信息
receivers:
  - name: 'email' # 警报
    email_configs: # 邮箱配置
    - to: '{{ template "email.to" . }}'  # 接收警报的email配置
      html: '{{ template "email.to.html" . }}'
      send_resolved: true
# 一个inhibition规则是在与另一组匹配器匹配的警报存在的条件下，使匹配一组匹配器的警报失效的规则。两个警报必须具有一组相同的标签.
inhibit_rules: #抑制规则
  - source_match: #源标签
     severity: 'critical'
    target_match:
     severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

# 企业微信告警

## 创建企业微信应用

1. 手机下载企业微信，创建企业；
2. 百度“企业微信”；
3. 扫码登录->应用管理-->创建应用，记录下 agentID 和 secret；

> corp_id: 企业微信账号唯一 ID， 可以在我的企业中查看。
>
> to_party: 需要发送的组(部门)。 可以在通讯录中查看
>
> agent_id: 第三方企业应用的 ID 
>
> api_secret: 第三方企业应用的密钥

## 添加微信模板

wechat.tmpl：

```tmpl
{{ define "wechat.tmpl" }}
{{- if gt (len .Alerts.Firing) 0 -}}{{ range .Alerts }}
@警报 <br>
实例: {{ .Labels.instance }} <br>
信息: {{ .Annotations.summary }} <br>
详情: {{ .Annotations.description }} <br>
时间: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }} <br>
{{ end }}{{ end -}}
{{- if gt (len .Alerts.Resolved) 0 -}}{{ range .Alerts }}
@恢复 <br>
实例: {{ .Labels.instance }} <br>
信息: {{ .Annotations.summary }} <br>
时间: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }} <br>
恢复: {{ (.EndsAt.Add 28800e9).Format "2006-01-02 15:04:05" }} <br>
{{ end }}{{ end -}}
{{- end }}
```

alertmanager.yml：

```tmpl
========================================================
templates:
  - '/usr/local/alertmanager/email.tmpl'
  - '/usr/local/alertmanager/wechat.tmpl' #加入定义好的微信模板
=========================================================
  recevier:'wechat'
=========================================================
  wechat_configs:
  - corp_id: 'wwd5348195e1cdd999'
    to_party: '1'
    agent_id: '1000018'
    api_secret: 'zO7QTGNTS3ASQWnqNWl0d5s-8A0TFEnVkiU3J9W-abc'
    send_resolved: true
    message: '{{ template "wechat.tmpl" . }}  #应用上面已有的模板
```

注：如果配置完成仍然无法接收到报价可以参考链接[2]。

# 钉钉报警

## 创建报警机器人

创建群聊->群设置->智能群助手->添加机器人->自定义

```shell
# 将 Webhook 和 Secret 记录
Webhook: https://oapi.dingtalk.com/robot/send？access_token=d3f1f891b45215fe63fdc9caddebf236746fd1d3642cfff2e79318e5339*****

Secret: SECde52a64789df2f0389d4f1089377a4b39f6908a80159ec36f86bfe77eff*****
```

## 安装插件

Alertmanager 需要额外安装钉钉报警的插件，[插件地址](https://github.com/timonwong/prometheus-webhook-dingtalk)。

```shell
# /usr/local/alertmanager 目录下
wget https://github.com/timonwong/prometheus-webhook-dingtalk/releases/download/v2.0.0/prometheus-webhook-dingtalk-2.0.0.linux-amd64.tar.gz
tar -xvzf prometheus-webhook-dingtalk-2.0.0.linux-amd64.tar.gz
cd prometheus-webhook-dingtalk-2.0.0.linux-amd64
cp config.example.yml config.yml

#config.yml 中添加刚才记录的 Webhook 地址和 Secret

# 启动插件
nohup ./prometheus-webhook-dingtalk --config.file=config.yml
```

## 自定义钉钉模板

ding.tmpl：

```tmpl
{{ define "ding.link.content" }}
{{ if gt (len .Alerts.Firing) 0 -}}
告警列表:
-----------
{{ template "__text_alert_list" .Alerts.Firing }}
{{- end }}
{{ if gt (len .Alerts.Resolved) 0 -}}
恢复列表:
{{ template "__text_resolve_list" .Alerts.Resolved }}
{{- end }}
{{- end }}
```

alertmanager.yml:

```yaml
templates:
  - <PATH>/ding.tmpl
```

config.yml:

```yaml
targets:
  webhook1:
    message:
      text: '{{ template "ding.link.content"  . }}'
```

## 配置告警

alertmanager.yml:

```yaml
  receiver: "dingding"
=========================================================
receivers:
- name: 'dingding'
  webhook_configs:
  - send_resolved: true
        url: http://<IP>:<PORT>/dingtalk/webhook1/send # 钉钉插件地址 
```

# 告警标签、路由、分组

在定义报警规则的时候，每个监控，定义好一个可识别的标签，可以作为报警级别，比如：`severity: error`、`severity: warning`、`severity: info`。不同的报警级别，发送到不同的接收者；

这里就以 `severity: warning` 为例，在应用场景中，也就是，不同的报警级别，发送到不同的接收者。

alertmanager.yml：

```yaml
route:
  group_by:['severity']
  ...
  receiver: 'dingding'
  routes:
  - match:
      severity: warning # 匹配到此标签，发送到 wechat
    receiver: 'wechat'
    continue: true
  - match_re: # 正则
      severity: ^(warning|critical)
      receiver: 'email'
      continue: true
```

- group by 指定以什么标签进行分组，这里指定的是 severity ;

- 默认接收者是 dingding ，如果报警信息中没有 severity 类的标签，匹配不到，会默认发送给 dingding 接收者；

- 常规匹配到 `severity: warning`，发送到 wechat 接收者；

- 正则匹配到 `severity: warning` 或者 `severity: critical`，发送到 email 接收者；

- 接收者要定义好，按照如上配置，email，wechat 接收者，会接收到报警消息；

# 参考链接

[1]: https://www.cnblogs.com/east4ming/p/16917370.html	"根据 label 发送 alert 到邮箱"

[2]: https://www.yuque.com/youngfit/qok2pe/gliiowczn15ktgiq#ZfjOU	"Prometheus 企业微信报警实战"

