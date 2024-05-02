



内存中存两个小时数据，最多丢2小时数据

每隔两小时后做一次存储，存储到磁盘的块中 一个块中包含 chunks文件，index meta.json,tombstone 

2G 存一次  一个shunks 512m  可能会存多个chunks





promethues Server配置文件

```bash
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network.target
[Service]
Restart=on-failure
WorkingDirectory=/apps/prometheus/
ExecStart=/apps/prometheus/prometheus --config.file=/apps/prometheus/prometheus.yml --web.enable-lifecycle
[Install]
WantedBy=multi-user.target

systemctl daemon-reload &&  systemctl restart prometheus && systemctl enable prometheus.service

```





node_exporter 启动文件

```
vim /etc/systemd/system/node-exporter.service
[Unit]
 Description=Prometheus Node Exporter
 After=network.target
 [Service]
 ExecStart=/apps/node_exporter/node_exporter
 [Install]
 WantedBy=multi-user.target

systemctl daemon-reload &&  systemctl  restart prometheus && systemctl enable prometheus.service
```





```
node_fileysetem_free_in
```













Kubernetes中部署 cadvionr

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cadvisor
  namespace: prometheus
spec:
  selector:
    matchLabels:
      app: cAdvisor
  template:
    metadata:
      labels:
        app: cAdvisor
    spec:
      tolerations:    #污点容忍,忽略master的NoSchedule
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
      hostNetwork: true
      restartPolicy: Always   
      containers:
      - name: cadvisor
        image: registry.cn-hangzhou.aliyuncs.com/zhangshijie/cadvisor-amd64:v0.45.0
        imagePullPolicy: IfNotPresent  
        securityContext:
          privileged: true
        ports:
        - containerPort: 8080
        volumeMounts:
          - name: root
            mountPath: /rootfs
            readOnly: true
          - name: run
            mountPath: /var/run
           # readOnly: true
          - name: sys
            mountPath: /sys
            readOnly: true
          - name: docker
            mountPath: /var/lib/containerd
             readOnly: true
          - name: devdisk
            mountPath: /dev/disk
            readOnly: true
      volumes:
      - name: root
        hostPath:
          path: /
      - name: run
        hostPath:
          path: /var/run
      - name: sys
        hostPath:
          path: /sys
      - name: docker
        hostPath:
          path: /var/lib/containerd
      - name: devdisk
        hostPath:
          path: /dev/disk
```



由于adviers只收集容器数据 我还要部署node收集器

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: prometheus
  labels:
    k8s-app: node-exporter
spec:
  selector:
    matchLabels:
      k8s-app: node-exporter
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
      containers:
      - image: registry.cn-hangzhou.aliyun.com/zhangshijie/node-exporter:v1.6.0
        imagePullPolicy: IfNotPresent
        name: prometheus-node-exporter
        ports:
        - containerPort: 9100
          hostPort: 9100
          protocol: TCP
          name: metrics
        volumeMounts:
        - mountPath: /host/proc
          name: proc
        - mountPath: /host/sys
          name: sys
        - mountPath: /host
          name: rootfs
         args:
        - "--path.procfs=/host/proc"
        - "--path.sysfs=/host/sys"
        - "--path.rootfs=/host"
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
      hostNetwork: true
      hostPID: true
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: "true"
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: monitoring 
spec:
  type: NodePort
  ports:
  - name: http
    port: 9100
    nodePort: 39100
    protocol: TCP
  selector:
    k8s-app: node-exporter
registry.cn-hangzhou.aliyun.com/zhangshijie/node-exporter:v1.6.0
```









```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
      component: server
    #matchExpressions:
    #- {key: app, operator: In, values: [prometheus]}
    #- {key: component, operator: In, values: [server]}
  template:
    metadata:
      labels:
        app: prometheus
        component: server
      annotations:
        prometheus.io/scrape: 'false'
    spec:
      #nodeName: 172.31.7.113
      serviceAccountName: monitor
      containers:
      - name: prometheus
        image: prom/prometheus:v2.31.2
        imagePullPolicy: IfNotPresent
        command:
          - prometheus
          - --config.file=/etc/prometheus/prometheus.yml
          - --storage.tsdb.path=/prometheus
          - --storage.tsdb.retention=720h
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/prometheus/prometheus.yml
          name: prometheus-config
          subPath: prometheus.yml
        - mountPath: /prometheus/
          name: prometheus-storage-volume
      volumes:
        - name: prometheus-config
          configMap:
            name: prometheus-config
            items:
              - key: prometheus.yml
                path: prometheus.yml
                mode: 0644
        - name: prometheus-storage-volume
          hostPath:
           path: /data/prometheusdata
           type: Directory

```

1. **请求到达 NodePort：** 当你在集群外部的任意节点上访问 NodePort 指定的端口（比如 `30090`），请求会到达该节点的这个端口。
2. **iptables 规则转发：** Kubernetes 使用 iptables 规则，将这个端口的流量正确转发到 Service 上。
3. **Service 选择合适的 Pod：** Service 根据其配置和选择器（selectors）来选择应该接收请求的 Pod。选择器是在 Service 中指定的，它确定了哪些 Pod 是 Service 的后端。
4. **Service 转发请求到 Pod：** Service 将请求转发到选择的 Pod。这个 Pod 可能在当前节点上，也可能在其他节点上。
5. **Pod 响应请求：** 最终，目标 Pod 在其容器中响应请求，并将响应返回给发起请求的客户端。





## 基本概念

TSDB 时序数据库

监控系统：通过主动、被动、黑河、白盒、

不管什么情况下，我们搭建的服务都是建立在物理主机之上的，我们得监控物理主机的性能使用情况，才能判断我们的服务稳定性，比如 CPU，内存, Disk，，系统是我们需要监控的第一要点，系统之间药通讯监控网络

监控基础设施，DB  Lvs  keeplived

应用服务，web服务，java服务，

他们的工作情况我们都要知道 



指标，Metrc，测量维度CPU 各个核心的使用率，三缓

周期性采集数据，

一个主机上要监控很多Metric，一个Metric有很多个测量维度

为了以后分析或者排错，为了能够让用户过了采集点还能查看到监控指标，监控服务器

每一个时间点的数据 样本

按照时间顺序存储 TSDB  mysql不行 性能太差

直接看数据库数据太反人类，图形化展示接口， 

即使有了图形化我们得一只看着窗口，比如我们休息的时候呢没人看怎么看到数据异常，

出现异常指标，可能是因为网络抖动的软状态

Alet 告警



采集：采集器

被监控端：代理 ，应用程序自带暴露接口信息

存储：分析 SQL、NoSQL（k\v document tsdb）

展示：grafana

Alert：告警系统

​	媒介：Email、短信，Wechat、dd、 



prometheus:

​	Exporters

​	instrumentaition

​	pushgateway

​	tsDB

Altmanager 

gragana 



**数据模型**

![image-20240220144458451](../images/image-20240220144458451.png)



一个指标下可能有多个时序， Cpu_usage是一个指标名称，在这个指标名称下，可能有多个不同维度的时序

cpu_usage指标名     cpu_usage{core=1 }   cpu_usage{core=2 }

同一Metric可能会适配到多个目标或设备，因而它使用“标签”作为无数据，从而为Metric添加，更多的信息描述纬度;
比如，node1 上的cpu_uusage  或 node2上的cpu_usage, 标签还可以作为过滤器进行指标过滤及聚合运算;

一个时序数据，由样本标识 标签和样本值 组成  Key Metricname + Lable =  Value

同一时间点，多个不同时序的某个时间的值，即时向量，

所谓即时向量就是，我们可能通过标签过滤出同一个时间点，多个不同时序，也可以用offset 过去某个时间点的时序值，可能是多个

范围向量

一个时序，某个时间范围内的数据，可以能是多个时序同一个时间范围内





一个job中是多个taget的集合(instance)



样本在存储的时候，要说明自己属于那个指标

指标名   标签     时间戳   =   Value

**数据类型**

![image-20240319100344220](../images/image-20240319100344220.png)





**告警**

监控系统只是触发告警，监控系统挂了怎么触发告警，触发 告警，告警系统挂了怎么告警，因此他们都要做冗余

![image-20240318131300646](../images/image-20240318131300646.png)









![image-20240220155953795](../images/image-20240220155953795.png)







![image-20240220160014671](../images/image-20240220160014671.png)





## 服务发现

Prometheus Server的数据抓取工作于pull模型，因而，它必需要事先知道各Target位置，然后才能从相应的Exporter或Instrumentation中抓取数据口对于小型的系统环境来说，通过statc_confgs指定各Target便能解决问题，这也是最简单的配置方法;

对于中大型的系统环境或具有较强动态性的云计算环境来说，静态配置显然难以适用:因此，Prometheus为此专门设计了一组服务发现机制，以便于能够基于服务注册中心(服务总线)自动发现、检测、分类可被监控的各Target，以及更新发生了变动的Target;



### 指标抓取的生命周期

第一个重新标记指的是，对targe本身重新打标签，比如这个targe来自哪里，他的指标路径，他所在的job

第二个重新标记指的是，对targe之上抓取的Metric，也就是时间序列（Metric + Lable + time +  value）重新打标。例如，usage_cup 

![image-20240319110717517](../images/image-20240319110717517.png)









### 服务发现的类别



#### 基于文件的服务发现

![image-20240220160453818](../images/image-20240220160453818.png)

prometheus配置

![image-20240220161426112](../images/image-20240220161426112.png)



### 基于KubernetesAPl的服务发现

基于Kubernetes API的服务发现机制，支持将APIServer中Node、Service、EndpointPod和Imgress等资源类型下相应的各资源对象视作target，并持续监视相关资源的变动
Node、Servite、Endpoint、Pod和Ingress资源分别由各自的发现机制进行定义负责发现每种类型资源对象的组件，在Prometheus中称为一个“role”同时支持在集群上

基于DaemonSet控制器部署node-exporter后发现各节点

![image-20240319115222286](../images/image-20240319115222286.png)



#### Node 资源发现

![image-20240220163304426](../images/image-20240220163304426.png)

#### pod 资源发现

![image-20240220163617065](../images/image-20240220163617065.png)

#### Server 资源发现

![image-20240220163735571](../images/image-20240220163735571.png)

#### Endpoint 资源发现

![image-20240220163819395](../images/image-20240220163819395.png)

#### **ingress 资源发现**

![image-20240220164037327](../images/image-20240220164037327.png)





#### 重新打标

![image-20240220165122219](../images/image-20240220165122219.png)



### rlablel_config

```yaml
# The source labels select values from existing labels. Their content is concatenated
# using the configured separator and matched against the configured regular expression
# for the replace, keep, and drop actions.
[ source_labels: '[' <labelname> [, ...] ']' ]

# Separator placed between concatenated source label values.
[ separator: <string> | default = ; ]

# Label to which the resulting value is written in a replace action.
# It is mandatory for replace actions. Regex capture groups are available.
[ target_label: <labelname> ]

# Regular expression against which the extracted value is matched.
[ regex: <regex> | default = (.*) ]

# Modulus to take of the hash of the source label values.
[ modulus: <int> ]

# Replacement value against which a regex replace is performed if the
# regular expression matches. Regex capture groups are available.
[ replacement: <string> | default = $1 ]

# Action to perform based on regex matching.
[ action: <relabel_action> | default = replace ]
```



source_label, Target_label.的值取出来定义acton 替换，赋值给一个新的标签。

source_label,可以写多个，并且可以把多个源标签的值取出来串联起来赋值给新的标签

比如标签1=123，标签2=456， 新标签=123456

不仅是这样，我们还可以通过正则匹配对这一串数值做模式匹配，匹配完以后我只取匹配到的值作为新标签的值

比如标签1=123，标签2=abc，123abc   ([0-9]+)a-z      新标签= $1_new

通过modulus，对源标签做hash计算把结果给新标签，

![image-20240220170000469](../images/image-20240220170000469.png)


下面示例，将三个源标签的值接顺序串联后，由指定的正则表达式进行模式匹面而后由replacement引用模式匹配的结果，并加以改造后，将其赋值给endpoint标

![image-20240319144610415](../images/image-20240319144610415.png)



#### Labelmap

不用指souce_labels，对targe上的所有标签进行匹配的标签名，而新标签的值则与其原标签的值相同;下面的示例，将regex指定的模式对target上的所有标签进行匹配判定，对于匹配到的标签名，它将以该标签名中匹配的部分为前缀，指定的“name”为后缀生成新的标签名，注意：新标签的值和源标签的值是一样的，只是修改名字 

#一般用于生成新标签，将regex对source1abe1s中指定的标签名称进行匹配，而后将匹配到的标签的值赋值给replacement字段指定的标签，通常用于取出匹配的标签名的一部分生成新标签,旧的标签仍会存在

匹配标签，把匹配的标签名取出来，创建新的标签，并且把匹配到标签值赋值给新的标签。



动作是replace   replacement替换的是值

动作是labelmap replacement替换标签

![image-20240319145406694](../images/image-20240319145406694.png)









报警流程







企业微信报警





## 二进制部署

官方下载安装包

```bash
#解包
tar xvf prometheus-2.50.1.linux-amd64.tar.gz -C /usr/local/
ln -s prometheus-2.50.1.linux-amd64/  prometheus
mkdir  bin conf data
mv prometheus.yml conf/
mv prometheus  promtool bin/


useradd -r -s /sbin/nologin prometheus
chown -R prometheus.prometheus  /usr/local/prometheus/

#修改环境变量
vim /etc/profile
export PROMETHEUS_HOME=/usr/local/prometheus
export PATH=${PROMETHEUS_HOME}/bin:$PATH

source /etc/profile
```



#### 创建Service文件

```bash
vim /lib/systemd/system/prometheus.service

[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network.target
[Service]
Restart=on-failure
User=prometheus
Group=prometheus
WorkingDirectory=/usr/local/prometheus/
ExecStart=/usr/local/prometheus/bin/prometheus --config.file=/usr/local/prometheus/conf/prometheus.yml --web.enable-lifecycle
ExecReload=/bin/kill -HUP \$MAINPID LimitNOFILE=65535
[Install]
WantedBy=multi-user.target

systemctl daemon-reload && systemctl enable --now prometheus.service

curl  http://10.0.0.200:9090/-/ready
```



#### 在200和201安装 node exproter

```bash
tar xfv node_exporter-1.7.0.linux-amd64.tar.gz  -C /usr/local/
ln -s /usr/local/node_exporter-1.7.0.linux-amd64/ /usr/local/node_exporter
cd /usr/local/node_exporter
mkdir bin && mv node_exporter bin/

```

#### 准备 service 文件

```bash
useradd -r -s /sbin/nologin prometheus

vim /lib/systemd/system/node_exporter.service

[Unit]
Description=Prometheus Node Exporter
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/node_exporter/bin/node_exporter
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
User=prometheus
Group=prometheus
[Install]
WantedBy=multi-user.target

systemctl daemon-reload && systemctl enable --now node_exporter
```



#### 安装granfa

```bash
apt install ./grafana_10.4.0_amd64.deb
systemctl enable --now  grafana-server.service
```



登录 Grafana 官网查找模板



#### 配置prometheus 抓取 Grafana 的指标 然后通过Grafana 展示

Grafana 内置了支持Prometheus监控接口

```bash
vim /usr/local/prometheus/conf/prometheus.yml
  - job_name: "Grfana"
    static_configs:
      - targets: 
        - '10.0.0.200:3000'
```





## Pushgateway



```bash
#安装
tar xf pushgateway-1.7.0.linux-amd64.tar.gz  -C /usr/local/
ln -s /usr/local/pushgateway-1.7.0.linux-amd64/ /usr/local/pushgateway
ls /usr/local/pushgateway
mkdir /usr/local/pushgateway/bin
mv /usr/local/pushgateway/pushgateway /usr/local/pushgateway/bin

#准备service文件
vim /lib/systemd/system/pushgateway.service
[Unit]
Description=Prometheus Pushgateway
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/pushgateway/bin/pushgateway
#ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
User=prometheus
Group=prometheus
[Install]
WantedBy=multi-user.target

systemctl daemon-reload && systemctl enable --now pushgateway.service
```



#### 配置 Prometheus 收集 Pushgateway 数据

```bash
vim /usr/local/prometheus/conf/prometheus.yml 
  - job_name: "Pushgetway"
    honor_labels: true   #保持原本的标签
    static_configs:
      - targets:
        - '10.0.0.200:9091'
```



```
 echo "some_metric 3.14" | curl --data-binary @- 
http://10.0.0.200:9091/metrics/job/some_job


cat <<EOF | curl --data-binary @- http://10.0.0.200:9091/metrics/job/some_job/instance/10.0.0.201
# TYPE another_metric gauge
# HELP another_metric Just an example.
another_metric 2398.283
EOF
```







### 标签重写

```bash
  - job_name: "node_exproet"
    static_configs:
      - targets:
        - '10.0.0.200:9100'
        - '10.0.0.201:9100'
        labels: {zu: "bai"}
    relabel_configs:
    - source_labels: [__scheme__,__address__,__metrics_path__]
      regex: "(https?)(.*)"
      separator: ""
      replacement: "${1}://${2}"
      action: replace
      target_label: "URL"

    - regex: "(zu)"
      separator: ""
      replacement: "${1}_VTYPE"
      action: labelmap

    - regex: "(zu)"
      action: labeldrop
#    metric_relabel_configs:
#    - source_labels:
#      - __name__
#      regex: "go_info"
#      action: drop

```







### 记录规则

```bash
#创建文件夹
mkdir /usr/local/prometheus/rules

#在主配置文件包含此目录下的文件
[root@ubuntu2204 ~]#vim /usr/local/prometheus/conf/prometheus.yml 
rule_files:
    - "../rules/*.yml"
  # - "first_rules.yml"
  # - "second_rules.yml"
```

#编写规则文件

```yml
---
groups:
- name: myrules
  rules:
  - record: "request_process_per_time"
    expr: request_processing_seconds_sum{job="flask"} /  request_processing_seconds_count{job="flask"}
    labels:
      app: "flask"
      role: "web"
  - record: "request_count_per_minute"
    expr: increase(request_count_total{job="flask"}[1m])
    labels:
      app: "flask"
      role: "web"
      
promtool check rules /usr/local/prometheus/rules/jilu.yml
```





### 安装 Alertmanager

```bash
#官方下包，导入服务器
#解压
tar xf alertmanager-0.27.0.linux-amd64.tar.gz -C /usr/local/
ln -s /usr/local/alertmanager-0.27.0.linux-amd64/  /usr/local/alertmanager

#准备启动
cd /usr/local/alertmanager
mkdir {bin,conf,data}
mv alertmanager amtool bin/ 
cp alertmanager.yml conf/

useradd -r -s /sbin/nologin prometheus
chown -R prometheus.prometheus /usr/local/alertmanager/
```

服务文件

```bash
vim /usr/lib/systemd/system/alertmanager.service
[Unit]
Description=alertmanager project
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/alertmanager/bin/alertmanager --config.file=/usr/local/alertmanager/conf/alertmanager.yml --storage.path=/usr/local/alertmanager/data --web.listen-address=0.0.0.0:9093
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
User=prometheus
Group=prometheus
[Install]
WantedBy=multi-user.targe

[root@ubuntu2204 alertmanager]#systemctl daemon-reload
[root@ubuntu2204 alertmanager]#systemctl enable --now alertmanager.service 
[root@ubuntu2204 alertmanager]#systemctl status  alertmanager.service 

自带指标暴露
#查看AlertManager也会暴露指标
# curl http://127.0.0.1:9093/metrics
```



修改prometheus配置 alet地址，告警规则文件 包含，编写告警规则文件，修改aletmanger配置文件

```yaml
#在prometheus上定制告警规则（触发器）
vim /usr/local/prometheus/rules/alert.yml
groups:
- name: flask_web
  rules:
  - alert: InstanceDown
    expr: up{job="job_name="flask"} == 0
    #expr: up == 0     #所有targets
    for: 1m
    labels:
      severity: warning
    annotations:
      #title: Instance Down    
      summary: "Instance {{ $labels.instance  }} 停止工作"
      description: "{{ $labels.instance  }} job {{ $labels.job_name  }} 已经停止1m以上"
      


#指定alertmanage地址
alerting:
  alertmanagers:
    - static_configs:
        - targets:
           - 10.0.0.200:9093

#告警规则文件包含
vim /usr/local/prometheus/conf/prometheus.yml 
rule_files:
    - "../rules/*.yml"
  # - "first_rules.yml"
  # - "second_rules.yml"
  
  
  
#修改aletmanger配置文件
global:
  resolve_timeout: 5m                        
  smtp_smarthost: 'smtp.qq.com:465' 
  smtp_from: '29308620@qq.com'
  smtp_auth_username: '29308620@qq.com'
  smtp_auth_password: 'jnnxnypbyzuxbghe'
  smtp_hello: 'qq.com'
  smtp_require_tls: false        #启用tls安全,默认true
# 路由配置
route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 20s  #此值不要过低，否则短期内会收到大量告警通知
  receiver: 'email'     #指定接收者名称
# 收信人员
receivers:
- name: 'email'
  email_configs:
  - to: '505697096@qq.com'
    send_resolved: true           #问题解决后也会发送恢复通知  
```







#### 模板

邮件

```bash
mkdir /usr/local/alertmanager/tmpl

{{ define "test.html" }}
<table border="1">
        <tr>
                <th>报警项</th>
                <th>实例</th>
                <th>报警阀值</th>
                <th>开始时间</th>
        </tr>
        {{ range $i, $alert := .Alerts }}
                <tr>
                        <td>{{ index $alert.Labels "alertname" }}</td>
                        <td>{{ index $alert.Labels "instance" }}</td>
                        <td>{{ index $alert.Annotations "value" }}</td>
                        <td>{{ $alert.StartsAt }}</td>
                </tr>
        {{ end }}
</table>
{{ end }}


                                
```

邮件

```tmpl
{{ define "email.html" }}
{{- if gt (len .Alerts.Firing) 0 -}}
{{ range .Alerts }}
=========start==========<br>
告警程序: prometheus_alert <br>
告警级别: {{ .Labels.severity }} <br>
告警类型: {{ .Labels.alertname }} <br>
告警主机: {{ .Labels.instance }} <br>
告警主题: {{ .Annotations.summary }}  <br>
告警详情: {{ .Annotations.description }} <br>
触发时间: {{ .StartsAt.Format "2006-01-02 15:04:05" }} <br>
=========end==========<br>
{{ end }}{{ end -}}
 
{{- if gt (len .Alerts.Resolved) 0 -}}
{{ range .Alerts }}
=========start==========<br>
告警程序: prometheus_alert <br>
告警级别: {{ .Labels.severity }} <br>
告警类型: {{ .Labels.alertname }} <br>
告警主机: {{ .Labels.instance }} <br>
告警主题: {{ .Annotations.summary }} <br>
告警详情: {{ .Annotations.description }} <br>
触发时间: {{ .StartsAt.Format "2006-01-02 15:04:05" }} <br>
恢复时间: {{ .EndsAt.Format "2006-01-02 15:04:05" }} <br>
=========end==========<br>
{{ end }}{{ end -}}
{{- end }}
```



微信

```
{{ define "wechat.default.message" }}
{{- if gt (len .Alerts.Firing) 0 -}}
{{- range $index, $alert := .Alerts -}}
{{- if eq $index 0 }}
========= 监控报警 =========
告警状态：{{   .Status }}
告警级别：{{ .Labels.severity }}
告警类型：{{ $alert.Labels.alertname }}
故障主机: {{ $alert.Labels.instance }}
告警主题: {{ $alert.Annotations.summary }}
告警详情: {{ $alert.Annotations.message }}{{ $alert.Annotations.description}};
触发阀值：{{ .Annotations.value }}
故障时间: {{ ($alert.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}  #注意:此为golang的时间Format,表示1234567
========= = end =  =========
{{- end }}
{{- end }}
{{- end }}
{{- if gt (len .Alerts.Resolved) 0 -}}
{{- range $index, $alert := .Alerts -}}
{{- if eq $index 0 }}
========= 异常恢复 =========
告警类型：{{ .Labels.alertname }}
告警状态：{{   .Status }}
告警主题: {{ $alert.Annotations.summary }}
告警详情: {{ $alert.Annotations.message }}{{ $alert.Annotations.description}};
故障时间: {{ ($alert.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
恢复时间: {{ ($alert.EndsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
{{- if gt (len $alert.Labels.instance) 0 }}
实例信息: {{ $alert.Labels.instance }}
{{- end }}
========= = end =  =========
{{- end }}
{{- end }}
{{- end }}
{{- end }}
```



#在altermager中调用此模板文件

```bash
templates:
  - '../tmpl/*.tmpl'
# 路由配置
route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 20s  #此值不要过低，否则短期内会收到大量告警通知
  receiver: 'email'     #指定接收者名称
# 收信人员
receivers:
- name: 'email'
  email_configs:
  - to: '505697096@qq.com'
    send_resolved: true           #问题解决后也会发送恢复通知  
    headers: { Subject: "[WARN] 报警邮件"}
    html: '{{ template "email.html" . }}'   
```



#### 告警路由

定义告警规则，然后配置路由，可以根据标签的值转发给不同的守信人

matchers 字段支持等值匹配 ，也支持正则匹配

```
route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  routes:
  - receiver: 'email'
    matchers:
    - severity = "critical"
  - receiver: 'email1'
    matchers:
    - severity =~ "^(warning)$"  repeat_interval: 20s  #此值不要过低，否则短期内会收到大量告警通知



# 收信人员
receivers:
- name: 'email'
  email_configs:
  - to: '505697096@qq.com'
    send_resolved: true           #问题解决后也会发送恢复通知  
    headers: { Subject: "[WARN] 报警邮件"}
    html: '{{ template "email.html" . }}'
receivers:
- name: 'email1'
  email_configs:
  - to: '2429365664@qq.com'
    send_resolved: true           #问题解决后也会发送恢复通知  
    headers: { Subject: "[WARN] 报警
```

配置告警规则

```yml
groups:
- name: flask_web
  rules:
  - alert: InstanceDown
    expr: up{job="my_metric"} == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Instance {{ $labels.instance  }} 停止工作"
      description: "{{ $labels.instance  }} job {{ $labels.job  }} 已经停止1分钟以上"
      value: "{{$value}}"

- name: flask_QPS
  rules:
  - alert: InstanceQPSIsHight
    expr: increase(request_count_total{job="my_metric"}[1m]) > 500
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Instance {{ $labels.instance  }} QPS 持续过高"
      description: "{{ $labels.instance  }} job {{ $labels.job  }} QPS 持续过高"
      value: "{{$value}}"
```

语法检查

```
/usr/local/alertmanager/bin/amtool   check-config /usr/local/alertmanager/conf/alertmanager.yml 
```

我在prometheus上定义了一组告警规则，QPS 触发就会发送 warngin，DOWN触发就会给alter发送 critical

然后我在alter配置路由，如果是warngin就发送 给A，如果是critical 就发送给B





#### 告警抑制

告警规则

```yaml
groups:
- name: flask_web
  rules:
  - alert: InstanceDown
    expr: up{job="my_metric"} == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "PromAlert Instance {{ $labels.instance  }} 停止工作"
      description: "PromAlert {{ $labels.instance  }} job {{ $labels.job  }} 已经停止1分钟以上"
      value: "{{$value}}"
- name: flask_QPS
  rules:
  - alert: InstanceQPSIsHight
    expr: increase(request_count_total{job="my_metric"}[1m]) > 500
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Instance {{ $labels.instance  }} QPS 持续过高"
      description: "{{ $labels.instance  }} job {{ $labels.job  }} QPS 持续过高"
      value: "{{$value}}"
  - alert: InstanceQPSIsLow                  #判断是否QPS访问为0
    expr: up{job="my_metric"} == 0
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "PromAlert Instance {{ $labels.instance  }} QPS 异常为零"
      description: "PromAlert {{ $labels.instance  }} job {{ $labels.job  }} QPS 异常为0"
      value: "{{$value}}"

```



配置抑制策略 根据标签判断

```bash
inhibit_rules:
- source_match:
    severity: critical
  target_match:
    severity: warning
  equal:
    - instancei  #如果在同一IP才算，如果不在一个targes就不算
```







### 微信告警

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_from: '29308620@qq.com'
  smtp_auth_username: '29308620@qq.com'
  smtp_auth_password: 'jnnxnypbyzuxbghe'
  smtp_hello: 'qq.com'
  smtp_require_tls: false        #启用tls安全,默认true
 
  wechat_api_corp_id: 'ww644a0d95807e476b' #企业ID
  #此处的微信信息可省略,下面wechat_configs 也提供了相关信息
  wechat_api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'  #API URL
  wechat_api_secret: 'qYgLlipdHtZidsd8qAZaTKKkGkzIyWxuQSeQOk9Si0M #密钥




- name: 'wechat'
  wechat_configs:
  - to_party: '2'  #部门
    to_user: '@all'
    agent_id: '1000004' #应用
    #api_secret: 'qYgLlipdHtZidsd8qAZaTKKkGkzIyWxuQSeQOk9Si0M'
    send_resolved: true
    message: '{{ template "wechat.default.message" . }}'
```



DD告警





安装第三方插件

```bash
tar xf prometheus-webhook-dingtalk-2.1.0.linux-amd64.tar.gz  -C /usr/local/
ln -s /usr/local/prometheus-webhook-dingtalk-2.1.0.linux-amd64/  /usr/local/dingtalk

#整理文件
cd /usr/local/dingtalk
mkdir bin conf
mv prometheus-webhook-dingtalk bin/
cp config.example.yml conf/config.yml

#修改配置
targets:
  webhook1:
    url: https://oapi.dingtalk.com/robot/send?access_token=ea692ba6096f7139d81ab64e5dfad45d2bcb299419c672219739ad7b1f2e8b30
    # secret for signature
    secret: SECe6bce9be79db1b5822a9e8834ef1d8e47c51045824842d213d3107f8c0cdd179



#启动服务
/usr/local/dingtalk/bin/prometheus-webhook-dingtalk --config.file=/usr/local/dingtalk/conf/config.yml --web.listen-address=0.0.0.0:8060
```



 Alertmanager 配置调用 插件

```yml
#此配置方法不支持关键字，如果有关键字请在告警规则中写关键字，或者调用模板
route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 10s
  receiver: 'dingtalk'

# 收信人员
receivers:
- name: 'dingtalk'
  webhook_configs:
  - url: 'http://10.0.0.200:8060/dingtalk/webhook1/send'
    send_resolved: true
```





### 服务发现

#### File

```yaml
#编写服务发现文件
mkdir /usr/local/prometheus/targets/
vim /usr/local/prometheus/targets/file.yaml

- targets:
  - 10.0.0.201:9100
  labels:
    app: file_ds

#修改prometheus配置文件
[root@ubuntu2204 ~]#vim /usr/local/prometheus/conf/prometheus.yml 
  - job_name: 'file_sd'
    scrape_interval: 10s
    file_sd_configs:
      - files:
          - /usr/local/prometheus/targets/*.yaml
        refresh_interval: 2m
```









#### DNS

```
$TTL 1D
@   IN SOA  master admin (
                    1   ; serial
                    1D  ; refresh
                    1H  ; retry
                    1W  ; expire
                    3H )    ; minimum
        NS       master
master  A        10.0.0.200
node    A    10.0.0.201

_prometheus._tcp.li.org. 1H IN SRV 10 10 9100 node.li.org.

#A记录和SRV都可以做服务发现  SRV服务名要对应Prometheus中的名字
```



Prometheus配置

```
  - job_name: 'dns_sd_AAAAA'
    dns_sd_configs:
    - names: ['node.li.org']
      type: A
      port: 9100
      refresh_interval: 10s

  - job_name: 'dns_sd_SRV'
    dns_sd_configs:
    - names: ['_prometheus._tcp.li.org.']  #对应DNS中的服务名，好处是在DNS中指定一堆，在Prometheus中 指定一个就可以 批量
      refresh_interval: 10s



rndc reload && netplan apply
```







#### Consul

```bash
#手动注册
curl -X PUT -d '{"id": "node","name": "node","address": "10.0.0.201","port": 9100,"tags": ["service"],"checks": [{"http": "http://10.0.0.201:9100/","interval": "5s"}]}' http://10.0.0.202:8500/v1/agent/service/registe
```

Prometheus集成

```yml
  - job_name: 'consul'
    honor_labels: true
    consul_sd_configs:
      - server: '10.0.0.202:8500'
        services: ['node','node1']
    relabel_configs:
     - source_labels: ['__meta_consul_service']
       target_label: 'consul_service'
     - source_labels: ['__meta_consul_dc']
       target_label: 'datacenter'
     - source_labels: ['__meta_consul_tags']
       target_label: 'app'
     - source_labels: ['__meta_consul_service']
       regex: "consul"
       action: drop
```





### 各种 Exporter

#### Node Exporter 监控服务

```
--collector.systemd --collector.systemd.unit-include=".*(ssh|mysql|node_exporter|nginx).*"
```

![image-20240323093536490](../images/image-20240323093536490.png)

msyql

```
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
```

```
[Unit]
Description=mysqld exporter project
After=network.target
[Install]
WantedBy=multi-user.target
[Service]
Type=simple
ExecStart=/usr/local/mysqld_exporter/bin/mysqld_exporter --config.my-cnf="/usr/local/mysqld_exporter/.my.cnf"
Restart=on-failure

[Install]
WantedBy=multi-user.target
```



prometheus

```
  - job_name: "mysql"

    metrics_path: '/metrics'
    scheme: 'http'
    static_configs:
      - targets: ["10.0.0.202:9104"]
```



haproxy-exproter

```bash
vim /lib/systemd/system/haproxy_exporter.service

[Unit]
Description=haproxy exporter project
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/haproxy_exporter/bin/haproxy_exporter --haproxy.scrape-uri="http://admin:123456@10.0.0.202:9999/haproxy-status;csv"
#也支持通过socket文件实现
#ExecStart=/usr/local/haproxy_exporter/bin/haproxy_exporter --haproxy.scrape-uri=unix:/run/haproxy/admin.sock
Restart=on-failure
[Install]
WantedBy=multi-user.target
```





java  exproter

```
JAVA_OPTS="-javaagent:/usr/share/tomcat9/jmx_prometheus_javaagent-0.20.0.jar=8888:/usr/share/tomcat9/tomcat.yml"
```







consul exproter

```
[Unit]
Description=consul_exporter
Documentation=https://prometheus.io/docs/introduction/overview/
After=network.target
[Service]
Type=simple
User=consul
EnvironmentFile=-/etc/default/consul_exporter
ExecStart=/usr/local/consul_exporter/consul_exporter \--consul.server="http://localhost:8500" \--web.listen-address=":9107" \--web.telemetry-path="/metrics" \--log.level=info \
 $ARGS
ExecReload=/bin/kill -HUP $MAINPID
TimeoutStopSec=20s
Restart=always
[Install]
WantedBy=multi-user.target

```



黑盒



![image-20240323114953233](../images/image-20240323114953233.png)

```
- job_name: 'ping_status_blackbox_exporter'
    metrics_path: /probe
    params:
      module: [icmp]                        #探测方式
    static_configs:
      - targets: ['www.wangxiaochun.com','www.google.com']   #探测的目标主机地址
        labels:
          instance: 'ping_status'
          group: 'icmp'
    relabel_configs:
      - source_labels: [__address__] #修改目标URL地址的标签[__address__]为 __param_target,用于发送给blackbox使用
        target_label: __param_target #此为必须项
      - target_label: __address__    #添加新标签.用于指定black_exporter服务器地址,此为必须项
        replacement: black-exporter.wang.org:9115 #指定black_exporter服务器地址,注意名称解析
      - source_labels: [__param_target]  #Grafana 使用此标签进行显示
        target_label: ipaddr

```





```yml
 - job_name: 'port_exporter'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets: ['www.wangxiaochun.com:8888']
        labels:
          instance: 'port_status'
          group: 'port'
    relabel_configs:
      - source_labels: [__address__] #修改目标URL地址的标签[__address__]为 __param_target,用于发送给blackbox使用
        target_label: __param_target #此为必须项
      - target_label: __address__    #添加新标签.用于指定black_exporter服务器地址,此为必须项
        replacement: 10.0.0.201:9115 #指定black_exporter服务器地址,注意名称解析
      - source_labels: [__param_target]  #Grafana 使用此标签进行显示
        target_label: ipaddr_port

```





```
  - job_name: 'http_status_blackbox_exporter'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:  #如果有https网站,还会自动额外显示TLS版本和证书的有效期,但有些网站可能不会显示证书信息,比如taobao
      - targets: ['https://www.baidu.com','http://www.wangxiaochun.com:8888'] #baidu必须加https才能显示,xiaomi不需要
        #- www.jd.com
        #- www.taobao.com
        labels:                       #添加标签
          instance: http_status
          group: web
    relabel_configs:
      - source_labels: [__address__]  #修改目标URL地址的标签[__address__]为 __param_target,用于发送给blackbox使用
        target_label: __param_target  #此为必须项
      - target_label: __address__     #添加新标签,用于指定black_exporter服务器地址,此为必须项
        replacement: black-exporter.wang.org:9115  #指定black_exporter服务器地址,注意名称解析
      - source_labels: [__param_target]  #Grafana 使用此url标签,否则有些模板可能无法显示
        target_label: url
      #- target_label: region        #测试添加标签,可选
      #  replacement: "remote"
```







```
docker pull gcr.dockerproxy.com/cadvisor/cadvisor:v0.49.1

docker run --volume=/:/rootfs:ro --volume=/var/run:/var/run:ro  --volume=/sys:/sys:ro  --volume=/var/lib/docker/:/var/lib/docker:ro --volume=/dev/disk/:/dev/disk:ro --publish=8848:8080 --detach=true --name=cadvisor --privileged --device=/dev/kmsg  gcr.dockerproxy.com/cadvisor/cadvisor:v0.49.1
```

