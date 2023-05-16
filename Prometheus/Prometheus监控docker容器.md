### Prometheus监控docker容器

`cAdvisor `将容器统计信息公开为` Prometheus` 指标。默认情况下， 这些指标在`/metrics HTTP` 端点下提供。可以通过设置-`prometheus_endpoint` 命令行标志来自定义此端点。
要使用 `Prometheus` 监控 `cAdvisor`， 只需在 `Prometheus` 中配置一个或多个作业， 这些作业会
在该指标端点处刮取相关的 `cAdvisor` 流程.

#### 1 启动cAdvisor容器

```bash
docker run \
--volume=/:/rootfs:ro \
--volume=/var/run:/var/run:ro \--volume=/sys:/sys:ro \
--volume=/var/lib/docker/:/var/lib/docker:ro \
--volume=/dev/disk/:/dev/disk:ro \
--publish=8080:8080 \
--detach=true \
--name=cadvisor \
google/cadvisor:latest
```

验证采集的指标

```bash
curl http://172.16.0.8:8080/metrics
```

常用的一些容器监控项（根据cadvisor获取的指标promQL计算）：
容器 CPU 使用率:
sum(irate(container_cpu_usage_seconds_total{image!=“”}[1m])) without (cpu)
查询容器内存使用量（单位： 字节） :
container_memory_usage_bytes{image!=“”}
查询容器网络接收量速率（单位： 字节/秒） ：sum(rate(container_network_receive_bytes_total{image!=“”}[1m])) without (interface)
查询容器网络传输量速率（单位： 字节/秒） ：
sum(rate(container_network_transmit_bytes_total{image!=“”}[1m])) without (interface)
查询容器文件系统读取速率（单位： 字节/秒） ：
sum(rate(container_fs_reads_bytes_total{image!=“”}[1m])) without (device)
查询容器文件系统写入速率（单位： 字节/秒） ：
sum(rate(container_fs_writes_bytes_total{image!=“”}[1m])) without (device)

#### 2 prometheus增加docker监控

```bash
- job_name: 'docker'
static_configs:
- targets: ['172.16.0.8:8080']
```

重启prometheus服务

#### 3 Grafana dashboard

导入`docker`监控模板，`dashboard Id`：`193`

#### 4 添加告警规则

```yaml
  - alert: ContainerKilled
    expr: time() - container_last_seen > 60
    for: 0m
    labels:
      severity: warning
    annotations:
      summary: Container killed (instance {{ $labels.instance }})
      description: "A container has disappeared\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
  # cAdvisor can sometimes consume a lot of CPU, so this alert will fire constantly.
  # If you want to exclude it from this alert, exclude the serie having an empty name: container_cpu_usage_seconds_total{name!=""}
  - alert: ContainerCpuUsage
    expr: (sum(rate(container_cpu_usage_seconds_total[3m])) BY (instance, name) * 100) > 80
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: Container CPU usage (instance {{ $labels.instance }})
      description: "Container CPU usage is above 80%\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
  # See https://medium.com/faun/how-much-is-too-much-the-linux-oomkiller-and-used-memory-d32186f29c9d
  - alert: ContainerMemoryUsage
    expr: (sum(container_memory_working_set_bytes) BY (instance, name) / sum(container_spec_memory_limit_bytes > 0) BY (instance, name) * 100) > 80
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: Container Memory usage (instance {{ $labels.instance }})
      description: "Container Memory usage is above 80%\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
  - alert: ContainerVolumeUsage
    expr: (1 - (sum(container_fs_inodes_free) BY (instance) / sum(container_fs_inodes_total) BY (instance))) * 100 > 80
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: Container Volume usage (instance {{ $labels.instance }})
      description: "Container Volume usage is above 80%\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
  - alert: ContainerVolumeIoUsage
    expr: (sum(container_fs_io_current) BY (instance, name) * 100) > 80
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: Container Volume IO usage (instance {{ $labels.instance }})
      description: "Container Volume IO usage is above 80%\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
  - alert: ContainerHighThrottleRate
    expr: rate(container_cpu_cfs_throttled_seconds_total[3m]) > 1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: Container high throttle rate (instance {{ $labels.instance }})
      description: "Container is being throttled\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```

