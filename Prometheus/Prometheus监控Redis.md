## 1 部署redis_exporter

> redis_exporter现支持 Redis 2.x, 3.x, 4.x, 5.x and 6.x版本

### 1.1 安装

```bash
wget https://github.com/oliver006/redis_exporter/releases/download/v1.13.1/redis_exporter-v1.13.1.linux-amd64.tar.gz
tar xf redis_exporter-v1.13.1.linux-amd64.tar.gz -C /opt/
cd /opt &&mv redis_exporter-v1.13.1.linux-amd64.tar.gz redis_exporter
```



### 1.2 启动

```bash
#查看redis_exporter启动参数
./redis_exporter -h
#直接启动
## 无密码
./redis_exporter redis//172.16.0.9:6379 &
## 有密码
./redis_exporter -redis.addr 172.16.0.9:6379 -redis.password 123456
#Systemd 方式启动
cat <<EOF > /usr/lib/systemd/system/redis_exporter.service
[Unit]
Description=redis_exporter
Documentation=https://github.com/oliver006/redis_exporter
After=network.target
[Service]
Type=simple
ExecStart=/opt/redis_exporter/redis_exporter -redis.addr 172.16.0.9:6379
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
systemctl start redis_exporter && systemctl enable redis_exporter
```

redis_exporter启动参数介绍：
-redis.addr： 指明一个或多个 Redis 节点的地址， 多个节点使用逗号分隔， 默认为
redis://localhost:6379
-redis.password： 验证 Redis 时使用的密码；
-redis.file： 包含一个或多个 redis 节点的文件路径， 每行一个节点， 此选项与 -redis.addr 互
斥。
-web.listen-address： 监听的地址和端口， 默认为 0.0.0.0:9121



### 2. 配置prometheus连接redis_exporter

```yaml
- job_name: 'redis_exporter'
    scrape_interval: 10s
    static_configs:
    - targets: ['172.16.0.9:9121']
```


重启prometheus

```bash
#检查proemtheus配置语法
./promtool check config /opt/prometheus/prometheus.yml
#重启prometheus
curl -X POST http://localhost:9090/-/reload #此方式需配置启动参数--web.enable-lifecycle
```


### 3. Grafana dashboard

导入`Redis`监控模板，`dashboard Id`：`763` `11835`

![image-20230314101034784](/Users/xiahuhu/Library/Application Support/typora-user-images/image-20230314101034784.png)

需要注意的是，`redis`默认没有配置可用内存最大值

```bash
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "0"
```


这种情况下采集到的`redis`内存值`redis_memory_max_bytes`为`0`，在`grafana`界面中无法展示内存使用率

### 4 添加Redis告警规则

```yaml
groups:
- name: redis_instance
rules:
#redis 实例宕机 危险等级: 5
- alert: RedisInstanceDown
expr: redis_up == 0
for: 10s
labels:
severity: warning
annotations:
summary: "Redis down (export {{ $labels.instance }})"
description: "Redis instance is down\n VALUE = {{ $value }}\n INSTANCE: {{ $labels.addr }}
{{ $labels.alias }}"
#redis 内存占用过多 危险等级: 4
- alert: RedisOutofMemory
expr: redis_memory_used_bytes / redis_total_system_memory_bytes * 100 > 60for: 3m
labels:
severity: warning
annotations:
summary: "Out of memory (export {{ $labels.instance }})"
description: "Redis is running out of memory > 80%\n VALUE= {{ $value }}\n INSTANCE:
{{ $labels.addr }} {{ $labels.alias }}"
# redis 连接数过多 危险等级: 3
- alert: RedisTooManyConnections
expr: redis_connected_clients > 2000
for: 3m
labels:
severity: warning
annotations:
summary: "Too many connections (export {{ $labels.instance}})"
description: "Redis instance has too many connections\n value = {{$value}}\n INSTANCE:
{{ $labels.addr }} {{ $labels.alias }}"
```

重启prometheus

### 5 Redis集群监控

步骤1、2为`Redis`单节点监控，redis如果是集群形式部署的话，启动一个`redis_exporter`连接到集群里的一个节点即可，通过`exporter`可以查询到集群中其他节点的指标
部署`redis_exporter`同样按照步骤1完成
不同的是`prometheus`抓取配置

```YAML
scrape_configs:
  ## config for the multiple Redis targets that the exporter will scrape
  - job_name: 'redis_exporter_targets'
    static_configs:
      - targets:
        - redis://first-redis-host:6379
        - redis://second-redis-host:6379
        - redis://second-redis-host:6380
        - redis://second-redis-host:6381
    metrics_path: /scrape
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: <<REDIS-EXPORTER-HOSTNAME>>:9121

  ## config for scraping the exporter itself
  - job_name: 'redis_exporter'
    static_configs:
      - targets:
        - <<REDIS-EXPORTER-HOSTNAME>>:9121
```

