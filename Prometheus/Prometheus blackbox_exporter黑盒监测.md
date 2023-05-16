**blackbox_exporter：**
`Prometheus` 官方提供的` exporter` 之一，可以提供` http`、`dns`、`tcp`、`icmp` 的监控数据采集
应用场景：

`HTTP` 测试
定义 `Request Header` 信息
判断` Http status / Http Respones Header / Http Body` 内容

- `TCP` 测试
  业务组件端口状态监听
  应用层协议定义与监听

- `ICMP` 测试
  主机探活机制

- `POST` 测试
  接口联通性

- `SSL` 证书过期时间

#### 1、blackbox_exporter部署

```bash
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.16.0/blackbox_exporter-0.16.0.linux-amd64.tar.gz
tar -zxvf blackbox_exporter-0.16.0.linux-amd64.tar.gz -C /opt
mv /opt/blackbox_exporter-0.16.0.linux-amd64 /opt/blackbox_exporter
#启动
nohup ./blackbox_exporter &
#查看服务是否启动，默认端口9115
ss -tunlp|grep 9115
```

#### 2、配置prometheus

**（1）检测主机存活状态**

```yaml
- job_name: node_status
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets: ['10.165.94.31']
        labels:
          instance: node_status
          group: 'node'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 172.19.155.133:9115
```

`10.165.94.31`是被监控端`ip`，`172.19.155.133`是`Blackbox_exporter`

**（2）监控主机端口存活状态**

```YAML
- job_name: 'prometheus_port_status'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets: ['172.19.155.133:8765']
        labels:
          instance: 'port_status'
          group: 'tcp'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 172.19.155.133:9115
```

**（3）监控网站状态**

```YAML
- job_name: web_status
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets: ['http://www.baidu.com']
        labels:
          instance: user_status
          group: 'web'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 172.19.155.133:9115
```

重启`prometheus`

```BASH
#检查语法
./promtool check config prometheus.yml
#平滑重启
curl -X POST http://localhost:9090/-/reload
```

访问`prometheus targets`面查看新添的`Job`，确认`instance`的`UP`状态

#### 3、添加Grafana dashboard

导入黑盒监控模板，`dashboard ID`:`9965`

![image-20230314104233799](/Users/xiahuhu/Library/Application Support/typora-user-images/image-20230314104233799.png)