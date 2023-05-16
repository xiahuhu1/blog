### 1 部署mysqld_exporter

#### 1.1 安装

```bash
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.12.1/mysqld_exporter-0.12.1.linux-amd64.tar.gz
tar xf mysqld_exporter-0.12.1.linux-amd64.tar.gz -C /opt/
cd /opt &&mv mysqld_exporter-0.12.1.linux-amd64 mysqld_exporter
```

#### 1.2 配置

**`mysql`添加授权用户**
在需要监控的数据库管理系统中创建监控用户，mysqld_exporter通过该用户获取MySQL监控指标数据

```sql
db01 [(none)]>GRANT SELECT, PROCESS, SUPER, REPLICATION CLIENT, RELOAD ON *.* TO
'exporter'@'localhost' IDENTIFIED BY '123456';
Query OK, 0 rows affected, 1 warning (0.00 sec)
db01 [(none)]>flush privileges;
```

**添加`mysqld_exporter`配置文件**

`mysqld_exporter`通过该配置获取监控用户信息，从而连接`MySQL`获取监控指标

```bash
cat <<EOF >/opt/mysqld_exporter/.my.cnf
[client]
host=localhost
user=exporter
password=123456
socket=/tmp/mysql3306.sock
EOF
```

#### 1.3 启动

`systemd`方式启动

```bash
cat <<EOF >/usr/lib/systemd/system/mysqld_exporter.service
[Unit]
Description=mysql Monitoring 
SystemDocumentation=mysql Monitoring System
[Service]
ExecStart=/opt/mysqld_exporter/mysqld_exporter \
--collect.info_schema.processlist \
--collect.info_schema.innodb_tablespaces \
--collect.info_schema.innodb_metrics \
--collect.perf_schema.tableiowaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tablelocks \
--collect.engine_innodb_status \
--collect.perf_schema.file_events \
--collect.binlog_size \
--collect.info_schema.clientstats \
--collect.perf_schema.eventswaits \
--config.my-cnf=/opt/mysqld_exporter/.my.cnf
[Install]
WantedBy=multi-user.target
EOF
systemctl start mysqld_exporter&&systemctl enable mysqld_exporter
```

`mysqld_exporter`默认监听本地`9104`端口
`metrics endpoint： http://ip:9104/metrics`
`mysql_up` 1 ##该`metrics`代表 `mysql` 被监控并且已经启动

### 2 配置prometheus连接mysqld_exporter

`prometheus.yml`的`scrape_configs`段添加以下配置

```yaml
- job_name: 'mysql_monitor'
static_configs:
- targets: ['172.16.0.8:9104']
```

重启`prometheus`
在`prometheus`的`web UI`的`targets`界面可以看到名为`mysql_monitor`的`targets`，状态为`up`
现在`MySQL`监控已经初步完成了，`MySQL`监控指标数据采集到`prometheus`后端存储，但需要做进一步的数据可视化。

#### 3 Grafana dashboard

导入官方`mysql`监控模板，`dashboard Id`：`7362`
`MySQL`主从监控`dashboard Id`：`7371`

#### 4 添加MySQL告警规则

```yaml
groups:
- name: MySQL-rules
rules:
- alert: MySQL Status
  expr: up == 0
  for: 5s
  labels:
    severity: warning
    annotations:
      summary: "{{$labels.instance}}: MySQL has stop "
      description: "MySQL 数据库挂了,请检查"
- alert: MySQL Slave IO Thread Status
  expr: mysql_slave_status_slave_io_running == 0
  for: 5s
  labels:
  severity: warning
    annotations:
      summary: "{{$labels.instance}}: MySQL Slave IO Thread has stop "
      description: "检测 MySQL 主从 IO 线程运行状态"
- alert: MySQL Slave SQL Thread Status
  expr: mysql_slave_status_slave_sql_running == 0
  for: 5s
  labels:
    severity: warning
    annotations:
      summary: "{{$labels.instance}}: MySQL Slave SQL Thread has stop "
      description: "检测 MySQL 主从 SQL 线程运行状态
```

重启`prometheus`
后续`grafana dashboard`做自定义监控项`graph`可视化和完善告警规则，需自行研究`mysqld_exporter`监控获取的相关`metrics`含义