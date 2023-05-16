### 1、influxdb部署

```bash
#配置yum源
cat <<EOF | sudo tee /etc/yum.repos.d/influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL \$releasever
baseurl =
https://repos.influxdata.com/rhel/\$releasever/\$basearch/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key
EOF
#yum安装
yum install influxdb -y
#启动
systemctl start influxdb
```

### 2、创建prometheus数据库

```bash
登录influxdb
$influx
>create database prometheus;
>show databases;
```

### 3、修改prometheus指定存储路径

```bash
–storage.tsdb.path=/data/prometheus
```

重启prometheus

### 4、配置 prometheus 添加远程读写

```bash
remote_write:
- url: "http://172.16.0.8:8086/api/v1/prom/write?db=prometheus"
remote_read:
- url: "http://172.16.0.8:8086/api/v1/prom/read?db=prometheus"
```

### 5、验证 influxdb 是否有数据写入

```bash
> use prometheus
> show measurements
> select * from prometheus_http_requests_total limit 5;
```

### 6、验证数据可靠性

停止 Prometheus 服务，同时删除 Prometheus 的 data 目录,重启 Prometheus。打开 Prometheus UI ，如果配置正常， Prometheus 可以正常查询到本地存储以删除的历史数据记录。

相关参考文档： https://docs.influxdata.com/influxdb/v1.8/supported_protocols/prometheus/