Goosefs 可以通过配置将指标数据输出到不同的监控系统中，Prometheus 是其中之一。Prometheus 是一个开源的监控框架，目前腾讯云监控已集成了 Prometheus，下文将重点介绍 Goosefs 监控指标，以及将监控指标上报到自建的 Prometheus 和云上 Prometheus 的流程。

## 准备工作

通过 Prometheus 构建监控体系需要先做如下准备工作：

- 准备好部署了 GooseFS 的集群
- 下载 Prometheus 安装包
- 下载腾讯云 Prometheus 安装包
- 下载 [Grafana](https://grafana.com/docs/grafana/latest/installation/#install-grafana/)

## 启用 GooseFS 监控指标上报配置


- 编辑 GooseFs 配置 conf/goosefs-site.properties， 添加如下配置项，并使用 goosefs copyDir conf/ 拷贝到所有 worker节点，并重启集群 `./bin/goosefs-start.sh all`。

```plaintext
goosefs.user.metrics.collection.enabled=true
goosefs.user.short.circuit.preferred=true
```

- master 和 worker 的 Prometheus 的监控指标可用如下的命令查看：

```plaintext
curl <LEADING_MASTER_HOSTNAME>:<MASTER_WEB_PORT>/metrics/prometheus/
# HELP Master_CreateFileOps Generated from Dropwizard metric import (metric=Master.CreateFileOps, type=com.codahale.metrics.Counter)
...

curl <WORKER_IP>:<WOKER_PORT>/metrics/prometheus/
# HELP pools_Code_Cache_max Generated from Dropwizard metric import (metric=pools.Code-Cache.max, type=com.codahale.metrics.jvm.MemoryUsageGaugeSet$$Lambda$51/137460818)
...
```


## **上报监控指标到自建 Prometheus** 

- 下载 Promethus 安装包并解压，修改 promethus.yml：


```plaintext
# prometheus.yml
global:
  scrape_interval:     1s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 1s # Eval

- job_name: 'goosefs masters'
    metrics_path: /metrics/prometheus
    file_sd_configs:
    - refresh_interval: 1m
      files:
      - "targets/cluster/masters/*.yml"
- job_name: 'goosefs workers'
    metrics_path: /metrics/prometheus
    file_sd_configs:
    - refresh_interval: 1m
      files:
      - "targets/cluster/workers/*.yml"
```


- 创建 targets/cluster/masters/masters.yml，添加 master 的 ip 和 port：

```plaintext
- targets:
- "<TARGERTS_MASTER_IP>:<TARGERTS_MASTER_PORT>"
```


- 创建 targets/cluster/workers/workers.yml，添加 worker 的 ip 和 port：

```plaintext
- targets:
- "<TARGERTS_WORKER_IP>:<TARGERTS_WORKER_PORT>"
```


- 启动 Prometheus，其中 --web.listen-address 指定 Prometheus 监听地址，默认端口号 9090：

```plaintext
nohup ./prometheus --config.file=prometheus.yml --web.listen-address="<LISTEN_IP>:<LISTEN_PORT>" > prometheus.log 2>&1 &
```


- 查看可视化界面：

```plaintext
http://<PROMETHEUS_BI_IP>:<PROMETHEUS_BI_PORT>
```



- 查看机器实例：

```plaintext
http://134.xxx.xx.247:9101/targets
```


## **上报监控指标到腾讯云 Prometheus** 

- 按照安装指南中的指引，在 master 机器上安装 promethus agent：

```plaintext
wget https://rig-1258344699.cos.ap-guangzhou.myqcloud.com/prometheus-agent/agent_install && chmod +x agent_install && ./agent_install prom-12kqy0mw agent-grt164ii ap-guangzhou <secret_id> <secret_key>
```

- 配置 master 和 worker 的抓取任务：

```plaintext
job_name: goosefs-masters
honor_timestamps: true
metrics_path: /metrics/prometheus
scheme: http
file_sd_configs:
- files:
  - /usr/local/services/prometheus-2.25.0.linux-amd64/targets/cluster/masters/*.yml
  refresh_interval: 1mjob_name: goosefs-workers
honor_timestamps: true
metrics_path: /metrics/prometheus
scheme: http
file_sd_configs:
- files:
  - /usr/local/services/prometheus-2.25.0.linux-amd64/targets/cluster/workers/*.yml
```


>! job_name 中没有空格，而单机的 Prometheus 的 job_name 中可以包含空格。


## **使用 Grafana 查看监控指标**

- 启动 Grafana：

```plaintext
nohup ./bin/grafana-server web > grafana.log 2>&1 &
```

- 打开登陆页面 http://134.xxx.xx.247:3000 ，username 和 password 都是 admin，首次登陆后修改密码。

- 进入页面后，添加 Prometheus 的 Datasource：

```plaintext
localhost:9101
```


- 导入 Goosefs 的 Grafana 模板，选择 json 导入（[点此下载 json](https://iwiki.woa.com/download/attachments/585423153/goosefs-prometheus-grafana-monitor-cache-hits.json?version=3&modificationDate=1618327767333&api=v2)），并选择上面创建的 Datasource。
>!云上 Prometheus 购买的时需设置密码，云上 Grafana 的可视化监控界面配置和上面类似，注意 job_name 需要配置成一致。
- 修改 DashBoard 以后，可以将 DashBoard 导出来。