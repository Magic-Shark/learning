![prometheus_gif](../resource/img_13.gif)
# Prometheus 架构概述
Prometheus主要由以下部分组成：
* Prometheus Server
* Service Discovery
* Time-Series Database (TSDB)
* Targets
* Exporters
* Push Gateway
* Alert Manager
* Client Libraries
* PromQL

# Prometheus Server
Prometheus Server是Prometheus组件中的核心部分，负责实现对监控数据的`获取`，`存储`以及`查询`。
* 获取数据(Retrival)。Prometheus Server可以通过静态配置管理监控目标，也可以配合使用Service Discovery的方式动态管理监控目标，并从这些监控目标中获取数据。
* 存储数据(Storage)。其次Prometheus Server需要对采集到的监控数据进行存储，Prometheus Server本身就是一个时序数据库，将采集到的监控数据按照时间序列的方式存储在本地磁盘当中。
* 查询数据。最后Prometheus Server对外提供了自定义的PromQL语言，实现对数据的查询以及分析。  

![prometheus_server](../resource/img_14.png)
配置示例:
```
global:
  scrape_interval: 15s 
  evaluation_interval: 15s 
  scrape_timeout: 10s 

rule_files:
  - "rules/*.rules"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090'] 
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100'] 

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

# Time-Series Database (TSDB)
prometheus 接收到的指标数据随着时间的推移而变化（CPU、内存、网络 IO 等）。它被称为时间序列数据。因此 Prometheus 使用时间序列数据库（TSDB）来存储其所有数据。默认情况下，Prometheus 以高效的格式（块）将其所有数据存储在本地磁盘中。随着时间的推移，它会压缩所有旧数据以节省空间。它还具有删除旧数据的保留策略。TSDB 具有内置的机制来管理长期保存的数据。您可以选择以下任意数据保留策略。
* 基于时间的保留：数据将保留指定的天数。默认保留期为 15 天。
* 基于大小的保留：您可以指定 TSDB 可以容纳的最大数据量。一旦达到这个限制，普罗米修斯将释放空间来容纳新数据。

<br>Prometheus 还提供远程存储选项。这主要是存储可扩展性、长期存储、备份和灾难恢复等所需要的。

#  Exporters & Targets
Exporter 就像在目标上运行的代理。它将指标从特定系统转换为普罗米修斯可以理解的格式。它可以是系统指标，如 CPU、内存等，也可以是 Java JMX 指标、MySQL 指标等。  
![prometheus_exporter](../resource/img_15.png)  
<br>默认情况下，这些转换后的指标由 Exporter 在目标的 /metrics 路径（HTTP 端点）上公开。例如，如果要监控服务器的 CPU 和内存，则需要在该服务器上安装 Node Exporter，并且 Node Exporter 以 prometheus 指标格式在 /metrics 上公开 CPU 和内存指标。一旦 Prometheus 提取指标，它将结合指标名称、标签、值和时间戳生成结构化数据。
<br>社区有很多 Exporters 可用，但只有其中一些获得 Prometheus 官方认可。如果您需要更多自定义采集，则需要创建自己的导出器。Prometheus 将 Exporter 分为各个部分，例如数据库、硬件、问题跟踪器和持续集成、消息系统、存储、公开 Prometheus 指标的软件、其他第三方实用程序等。您可以从官方文档中查看每个类别的 Exporter 列表。
<br>在 Prometheus 配置文件中，所有 Exporter 的详细信息将在 scrape_configs 下给出
```
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter1:9100', 'node-exporter2:9100']

  - job_name: 'blackbox-exporter'
    static_configs:
      - targets: ['blackbox-exporter1:9115', 'blackbox-exporter2:9115']
    metrics_path: /probe

  - job_name: 'snmp-exporter'
    static_configs:
      - targets: ['snmp-exporter1:9116', 'snmp-exporter2:9116']
    metrics_path: /snmp
```
而 Target 是 Prometheus 抓取指标的来源。目标可以是服务器、服务、Kubernetes Pod、应用程序端点等。默认情况下，prometheus 会在目标的 /metrics 路径下查找指标。可以在目标配置中更改默认路径。这意味着，如果您不指定自定义指标路径，Prometheus 会在 /metrics 下查找指标。
<br>Prometheus 需要来自目标端点的特定文本格式的数据。每个指标都必须换行。通常，这些指标使用 Prometheus exporters 来暴露。`Prometheus exporter 通常和 target 伴生在一起`。

# Push Gateway & Client Libraries
由于Prometheus数据采集基于`Pull`模型进行设计，因此在网络环境的配置上必须要让Prometheus Server能够直接与Exporter进行通信。 当这种网络需求无法直接满足时，就可以利用PushGateway来进行中转。可以通过PushGateway将内部网络的监控数据主动Push到Gateway当中。而Prometheus Server则可以采用同样Pull的方式从PushGateway中获取到监控数据。  
![prometheus_exporter](../resource/img_16.png)  
Pushgateway 将指标数据临时存储在内存中。它更像是一个临时缓存。Pushgateway 配置也将在 Prometheus 配置中的 scrape_configs 部分下进行配置。
```
scrape_configs:
  - job_name: "pushgateway"
    honor_labels: true
    static_configs:
    - targets: [pushgateway.monitoring.svc:9091]
```
要将指标发送到 Pushgateway，您需要使用 prometheus 客户端库对应用程序插桩，或使用脚本暴露指标。Prometheus 客户端库是可用于检测应用程序代码的软件库，以 Prometheus 理解的方式公开指标。如果您需要自行埋点插桩或想要创建自己的Exporter，则可以使用客户端库。
<br>一个非常好的用例是需要将指标推送到 Pushgateway 的批处理作业。批处理作业使用客户端库来埋点，以 prometheus 格式暴露指标。下面是一个 Python Client Library 的示例，它公开了名为 batch_job_records_processed_total 的自定义指标。
```
from prometheus_client import start_http_server, Counter
import time
import random

RECORDS_PROCESSED = Counter('batch_job_records_processed_total', 'Total number of records processed by the batch job')

def process_record():
    time.sleep(random.uniform(0.01, 0.1))
    RECORDS_PROCESSED.inc()

def batch_job():
   
    for _ in range(100):
        process_record()

if __name__ == '__main__':
 
    start_http_server(8000)
    print("Metrics server started on port 8000")

    batch_job()
    print("Batch job completed")

    while True:
        time.sleep(1)
```
在使用客户端库时，prometheus_client 会在 /metrics 端点上公开指标。


# Service Discovery
Prometheus 使用两种方法从目标中获取指标。
* 静态配置：当目标具有静态 IP 或 DNS 端点时，我们可以使用这些端点作为目标。
* 服务发现：在大多数自动伸缩系统和 Kubernetes 等分布式系统中，目标不会有静态端点。在这种情况下，使用 prometheus 服务发现来发现目标端点，并且目标会自动添加到 prometheus 配置中。

![prometheus_exporter](../resource/img_17.png)

<br>在云上所有资源都是弹性的，这些资源可以随着需求规模的变化而变化。例如在AWS的AutoScaling，可以根据用户定义的规则动态地创建或者销毁EC2实例，从而使用户部署在AWS上的应用可以自动的适应访问规模的变化；在Kubernetes中，pod也会随时在创建和被销毁。
<br>这种按需的资源使用方式对于监控系统而言就意味着没有了一个固定的监控目标，所有的监控对象(基础设施、应用、服务)都在动态的变化。对于Prometheus这一类基于Pull模式的监控系统，显然也无法继续使用的static_configs的方式静态的定义监控目标。而对于Prometheus而言其解决方案就是引入一个中间的代理人（服务注册中心），这个代理人掌握着当前所有监控目标的访问信息，Prometheus只需要向这个代理人询问有哪些监控目标控即可， 这种模式被称为服务发现。
<br>Prometheus通过使用平台提供的API就可以找到所有需要监控的云主机。在Kubernetes这类容器管理平台中，Kubernetes掌握并管理着所有的容器以及服务信息，那此时Prometheus只需要与Kubernetes打交道就可以找到所有需要监控的容器以及服务对象。Prometheus还可以直接与一些开源的服务发现工具进行集成，例如在微服务架构的应用程序中，经常会使用到例如Consul这样的服务发现注册软件，Promethues也可以与其集成从而动态的发现需要监控的应用服务实例。
示例如下：
```
scrape_configs:
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
    - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
    - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
      action: keep
      regex: default;kubernetes;https
```

<br>Kubernetes 中还有基于文件的服务发现 file_sd_configs 。它适用于静态目标，但经典静态配置 static_configs 和 file_sd_configs 之间的主要区别在于，在这种情况下，我们创建单独的 JSON 或 YAML 文件并将目标信息保存在文件中。Prometheus 将读取文件来识别目标。
<br>不仅这两种，还可以使用各种服务发现方法，例如 consul_sd_configs（prometheus 从 consul 获取目标详细信息）、ec2_sd_configs 等
<br>**_与Push Gateway的区别_**
A. 服务发现 (Service Discovery)  
服务发现解决的是 “去哪儿找目标” 的问题。   
在 K8s 环境中，它不是构建一个 Gateway 来推送数据，而是：
1. Prometheus 配置了 kubernetes_sd_configs。
2. Prometheus 主动 向 Kubernetes API Server 查询：“现在有哪些 Pod 或 Service 带有我想要的 Label？”
3. K8s API Server 返回一个 目标列表（例如：10.42.0.1:8080, 10.42.0.2:8080）。
4. Prometheus 拿着这个列表，直接 去拉取每个目标的 /metrics 接口。  

所以，服务发现解决的是目标地址的动态更新问题，数据流向始终是 Prometheus -> 目标 Exporter (Pull)。  
B. Pushgateway  
Pushgateway 解决的是 “目标活不长” 的问题。  
Prometheus 依赖目标长期存活，但像定时脚本或数据分析批处理任务，可能运行几秒钟就退出了。如果 Prometheus 还没来得及拉取，指标就消失了。  
Pushgateway 的作用就是：
1. 短生命周期任务执行完毕后，将指标 主动 推送 (Push) 到 Pushgateway。
2. Pushgateway 存储这些指标，并暴露 /metrics 接口。
3. Prometheus 仍然是主动的，它定时去 拉取 (Pull) Pushgateway 上的数据。
  
核心区别： 服务发现是目标地址的发现机制；Pushgateway 是数据采集模型的特例，它是将短暂的 Push 行为转变成 Prometheus 能够处理的 Pull 目标。

# Prometheus Alert Manager
Alertmanager 是 Prometheus 监控系统的关键部分。它的主要工作是根据 Prometheus 警报配置中设置的指标阈值发送警报。警报由 Prometheus 触发（注意，是由 Prometheus 进程触发原始告警）并发送到 Alertmanager。Alertmanager 对告警去重、抑制、静默、分组，最后使用各类通知媒介（电子邮件、slack 等）发出告警事件。其具体功能：
* Alert Deduplicating：消除重复警报
* Grouping：将相关警报分组在一起
* Silencing：静默维护
* Routing：路由，根据严重性将警报路由到适当的接收者
* Inhibition：抑制，当存在中高严重性警报时停止低严重性警报的过程

![prometheus_exporter](../resource/img_18.png)

以下是警报规则的配置示例：
```
groups:
- name: microservices_alerts
  rules:
  - record: http_latency:average_latency_seconds
    expr: sum(http_request_duration_seconds_sum) / sum(http_request_duration_seconds_count)
  - alert: HighLatencyAlert
    expr: http_latency:average_latency_seconds > 0.5
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High latency detected in microservices"
      description: "The average HTTP latency is high ({{ $value }} seconds) in the microservices cluster."
```
这是 Alertmanager 配置文件的路由配置示例：
```
routes:
- match:
    severity: 'critical'
  receiver: 'pagerduty-notifications'

- match:
    severity: 'warning'
  receiver: 'slack-notifications'
```
<br>**_与Grafana告警系统的区别_**

| 告警系统 |谁来计算？|谁来通知？|优势/特点                                                         |
|-----------------------------|-------------------|------------------------------------|--------------------------------------------------------------|
|Prometheus + Alertmanager	|Prometheus	| Alertmanager | 	专业降噪、分组、抑制、路由复杂，是生产环境的标准配置。 |
|Grafana 内置告警	|Grafana	|Grafana| 	简单方便，直接基于图表阈值，缺乏高级告警管理功能。 |