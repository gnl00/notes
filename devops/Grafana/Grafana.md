---
description: Grafana 监控搭建
sticky: true
tag: 
  - Grafana
  - DevOps
---

# Grafana

## 服务监控

### Prometheus

1、安装 Prometheus

1.1、创建配置 `/etc/prometheus.yaml`

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["192.168.2.203:9990"]
```

1.2、[Docker 安装](https://prometheus.io/docs/prometheus/latest/installation/)

```shell
docker run \
    -p 9990:9090 \
    -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus
```

可以通过 `http://localhost:9990` 来检查服务状态。

Prometheus 只是一个时序数据库，作为数据源用来保存监控信息。需要配合各种 exporter 来获取到对应的平台/软件/监控信息。

### Node Exporter

2、安装启动 [node_exporter](https://github.com/prometheus/node_exporter) 进行 Linux 服务器监控

```shell
docker run --name=node-exporter -p 9100:9100 -d \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host
```

访问 `<ip>:9100` 检查 node_exporter 是否启动成功。

…

3、使用 Prometheus 收集 node_exporter 数据，修改配置

```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "linux_monitor"
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ["192.168.2.203:9100", "192.168.2.230:9100"]
```

### Grafana

4、安装 [Grafana](https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/)，作为监控面板。启动并访问，初始用户名密码为 `admin/admin`

5、使用 Grafana 添加 Prometheus 数据源

![image-20231024171116053](./assets/image-20231024171116053.png)

![image-20231024171139054](./assets/image-20231024171139054.png)

编辑完连接信息点击保存。

6、导入 Dashboard 模板

![image-20231024171344216](./assets/image-20231024171344216.png)

Linux 服务器可以选择：8919 或者 12633，可以[选择自己喜欢的模板](https://grafana.com/grafana/dashboards/)

### PostgreSQL Exporter

7、添加 [postgresql_exporter](https://github.com/prometheus-community/postgres_exporter)

```shell
docker run --name=pg-exporter -d --net=host -p 9187:9187 -e DATA_SOURCE_NAME="postgresql://root:pg.ldzl@192.168.2.200:5432/ldzl_rt?sslmode=disable" quay.io/prometheuscommunity/postgres-exporter
```

访问 `localhost:9187` 无误，添加到 `prometheus.yaml`。

### Patroni

8、添加 Patroni 监控。

因为 Patroni 本身提供了 REST API 来监控状态，因此不需要额外安装 exporter。只需要将 REST API 路径配置在 Prometheus 配置文件中即可。

patroni.yml

```yaml
restapi:
  listen: 0.0.0.0:8890
  connect_address: 192.168.2.230:8890
```

prometheus.yml

```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "patroni_monitor"
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ["192.168.2.203:8890", "192.168.2.230:8890"]
```

在 Grafana 中添加 Patroni 模板。





## 参考

* https://www.cnblogs.com/hong-fithing/p/14695803.html
* https://www.modb.pro/db/154937