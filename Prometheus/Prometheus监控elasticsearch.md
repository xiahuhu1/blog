### 1 部署es_exporter

#### 1.1 安装

```bash
wget https://github.com/justwatchcom/elasticsearch_exporter/releases/download/v1.1.0/elasticsearch_exporter-1.1.0.linux-amd64.tar.gz
tar -xvf elasticsearch_exporter-1.1.0.linux-amd64.tar.gz
mv elasticsearch_exporter-1.1.0.linux-amd64 /opt/elasticsearch_exporter-1.1.0
ln -s /opt/elasticsearch_exporter-1.1.0 /opt/elasticsearch_exporter
```

#### 1.2 启动

```bash
cd /opt/elasticsearch_exporter
nohup ./elasticsearch_exporter --es.uri http://172.16.0.7:9200 &
#--es.uri 默认 http://localhost:9200， 连接到的 Elasticsearch 节点的地址（主机和端口）
#只需指定一个es node地址，通过该节点能连接到其所在集群的cluster state
```

`systemd`方式启动

```bash
cat <<EOF >/etc/systemd/system/elasticsearch_exporter.service
[Unit]
Description=Elasticsearch stats exporter for Prometheus
Documentation=Prometheus exporter for various metrics
[Service]
ExecStart=/opt/elasticsearch_exporter/elasticsearch_exporter --es.uri http://ip:9200
[Install]
WantedBy=multi-user.target
EOF
systemctl start elasticsearch_exporter&&systemctl enable elasticsearch_exporter
```

`es_exporter`默认占用本地`9114`端口，通过`web`访问`http://ip:9114/metrics`可以查询采集到的信息

#### 2 配置prometheus连接es_exporter

```yaml
- job_name: 'elasticsearch_exporter'
    scrape_interval: 10s
    metrics_path: "/metrics"
    static_configs:
    - targets: ['172.16.0.7:9114']
```

重启prometheus

#### 3 grafana dashboard

导入`es`监控模板，`dashboard Id`：`2322` ` 266`

![image-20230314102615348](/Users/xiahuhu/Library/Application Support/typora-user-images/image-20230314102615348.png)

#### 4 添加es告警规则

```yaml
groups:
- name: es
rules:
- alert: esclusterwrong
expr: elasticsearch_cluster_health_status{color="green"} != 1
for: 10s
labels:
severity: critical
annotations:
description: "elasticsearch cluster {{$labels.server}} 异常"
- alert: esDown
expr: elasticsearch_cluster_health_number_of_nodes != 3
for: 10s
labels:
severity: critical
annotations:
description: "elasticsearch service {{$labels.instance}} down"
```

