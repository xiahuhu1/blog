#### 一、创建`postgres_exporter`账户

```sql
CREATE USER postgres_exporter PASSWORD 'postgres_exporter@2O23';
ALTER USER postgres_exporter SET SEARCH_PATH TO postgres_exporter,pg_catalog;

CREATE SCHEMA postgres_exporter AUTHORIZATION postgres_exporter;

CREATE FUNCTION postgres_exporter.f_select_pg_stat_activity()
RETURNS setof pg_catalog.pg_stat_activity
LANGUAGE sql
SECURITY DEFINER
AS $$
  SELECT * from pg_catalog.pg_stat_activity;
$$;

CREATE FUNCTION postgres_exporter.f_select_pg_stat_replication()
RETURNS setof pg_catalog.pg_stat_replication
LANGUAGE sql
SECURITY DEFINER
AS $$
  SELECT * from pg_catalog.pg_stat_replication;
$$;

CREATE VIEW postgres_exporter.pg_stat_replication
AS
  SELECT * FROM postgres_exporter.f_select_pg_stat_replication();

CREATE VIEW postgres_exporter.pg_stat_activity
AS
  SELECT * FROM postgres_exporter.f_select_pg_stat_activity();

GRANT SELECT ON postgres_exporter.pg_stat_replication TO postgres_exporter;
GRANT SELECT ON postgres_exporter.pg_stat_activity TO postgres_exporter;
```

创建`postgres_exporter`，密码为`postgres_exporter@2O23`

#### 二、部暑`postgres_exporter`

```bash
[root@docker-48 postgres-exporter]# cat start.sh 
#!/bin/bash
docker stop postgres-exporter
docker rm -f postgres-exporter

docker run -d \
--restart=always \
--name postgres-exporter \
-p 9187:9187 \
-e DATA_SOURCE_NAME="postgresql://postgres_exporter:postgres_exporter@2O23@10.0.0.48:5433/postgres?sslmode=disable" \
-v /etc/localtime:/etc/localtime:ro \
prometheuscommunity/postgres-exporter:v0.12.0
```

运行脚本`./start.sh`

```bash
[root@docker-48 postgres-exporter]# docker ps |grep postgres-ex
7d8937bbbf57   prometheuscommunity/postgres-exporter:v0.12.0                 "/bin/postgres_expor…"   2 hours ago      Up 2 hours                0.0.0.0:9187->9187/tcp                                                                                                              postgres-exporter
```

#### 三、配置`prometheus`

```yml
# vim prometheus.yml
- job_name: 'postgres'
    file_sd_configs:
      - files:
        - /etc/prometheus/sd_config/postgres.yaml
        refresh_interval: 5s
    relabel_configs:
      - source_labels: [__address__]
        regex: (.*)
        target_label: instance
        replacement: $1
      - source_labels: [__address__]
        regex: (.*):(.*)
        target_label: __address__
        replacement: $1:9187
```

配置服务器的自动发现

```yaml
[root@yunyi-test03 prometheus]# cat sd_config/postgres.yaml 
# postgres
- labels:
    project: postgres
  targets:
  - 10.0.0.48:5433
  - 172.16.146.77:5432
```

![image-20230330163634162](/Users/xiahuhu/Library/Application Support/typora-user-images/image-20230330163634162.png)

#### 四、Grafana导入模版

`grafanaid: 455`

![image-20230330163452092](/Users/xiahuhu/Library/Application Support/typora-user-images/image-20230330163452092.png)

`grafanaid: 9628`

![image-20230330163521096](/Users/xiahuhu/Library/Application Support/typora-user-images/image-20230330163521096.png)

#### 五、警报规则

```yaml
vi /data/prometheus/conf/rules/postgres.rules
groups:
- name: postgresql-监控告警    
  rules:
  - alert: 警报！Postgresql宕机
    expr: pg_up == 0
    for: 0m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql down"
      description: "Postgresql instance is down\n  当前值={{ $value }}"

  - alert: 警报！Postgresql被重启
    expr: time() - pg_postmaster_start_time_seconds < 60
    for: 0m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql restarted"
      description: "Postgresql restarted\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlExporterError
    expr: pg_exporter_last_scrape_error > 0
    for: 0m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql exporter error"
      description: "Postgresql exporter is showing errors. A query may be buggy in query.yaml\n  当前值={{ $value }}"

  - alert: 警报！Postgresql主从复制不同步
    expr: pg_replication_lag > 30 and ON(instance) pg_replication_is_replica == 1
    for: 0m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql replication lag"
      description: "PostgreSQL replication lag is going up (> 30s)\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlTableNotVaccumed
    expr: time() - pg_stat_user_tables_last_autovacuum > 60 * 60 * 24
    for: 0m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql table not vaccumed"
      description: "Table has not been vaccum for 24 hours\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlTableNotAnalyzed
    expr: time() - pg_stat_user_tables_last_autoanalyze > 60 * 60 * 24
    for: 0m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql table not analyzed"
      description: "Table has not been analyzed for 24 hours\n  当前值={{ $value }}"

  - alert: 警报！Postgresql连接数太多
    expr: sum by (datname) (pg_stat_activity_count{datname!~"template.*|postgres"}) > pg_settings_max_connections * 0.8
    for: 2m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql too many connections"
      description: "PostgreSQL instance has too many connections (> 80%).\n  当前值={{ $value }}"

  - alert: 警报！Postgresql连接数太少
    expr: sum by (datname) (pg_stat_activity_count{datname!~"template.*|postgres"}) < 5
    for: 2m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql not enough connections"
      description: "PostgreSQL instance should have more connections (> 5)\n  当前值={{ $value }}"

  - alert: 警报！Postgresql死锁
    expr: increase(pg_stat_database_deadlocks{datname!~"template.*|postgres"}[1m]) > 5
    for: 0m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql dead locks"
      description: "PostgreSQL has dead-locks\n  当前值={{ $value }}"

  - alert: 警报！Postgresql慢查询
    expr: pg_slow_queries > 0
    for: 2m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql slow queries"
      description: "PostgreSQL executes slow queries\n  当前值={{ $value }}"

  - alert: 警报！Postgresql回滚率高
    expr: rate(pg_stat_database_xact_rollback{datname!~"template.*"}[3m]) / rate(pg_stat_database_xact_commit{datname!~"template.*"}[3m]) > 0.02
    for: 0m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql high rollback rate"
      description: "Ratio of transactions being aborted compared to committed is > 2 %\n  当前值={{ $value }}"

  - alert: 警报！Postgresql提交率低
    expr: rate(pg_stat_database_xact_commit[1m]) < 10
    for: 2m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql commit rate low"
      description: "Postgres seems to be processing very few transactions\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlLowXidConsumption
    expr: rate(pg_txid_current[1m]) < 5
    for: 2m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql low XID consumption"
      description: "Postgresql seems to be consuming transaction IDs very slowly\n  当前值={{ $value }}"

  - alert: 警报！PostgresqllowXlogConsumption
    expr: rate(pg_xlog_position_bytes[1m]) < 100
    for: 2m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresqllow XLOG consumption"
      description: "Postgres seems to be consuming XLOG very slowly\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlWaleReplicationStopped
    expr: rate(pg_xlog_position_bytes[1m]) == 0
    for: 0m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql WALE replication stopped"
      description: "WAL-E replication seems to be stopped\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlHighRateStatementTimeout
    expr: rate(postgresql_errors_total{type="statement_timeout"}[1m]) > 3
    for: 0m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql high rate statement timeout"
      description: "Postgres transactions showing high rate of statement timeouts\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlHighRateDeadlock
    expr: increase(postgresql_errors_total{type="deadlock_detected"}[1m]) > 1
    for: 0m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql high rate deadlock"
      description: "Postgres detected deadlocks\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlReplicationLagBytes
    expr: (pg_xlog_position_bytes and pg_replication_is_replica == 0) - on(environment) group_right(instance) (pg_xlog_position_bytes and pg_replication_is_replica == 1) > 1e+09
    for: 0m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql replication lag bytes"
      description: "Postgres Replication lag (in bytes) is high\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlUnusedReplicationSlot
    expr: pg_replication_slots_active == 0
    for: 1m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql unused replication slot"
      description: "Unused Replication Slots\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlTooManyDeadTuples
    expr: ((pg_stat_user_tables_n_dead_tup > 10000) / (pg_stat_user_tables_n_live_tup + pg_stat_user_tables_n_dead_tup)) >= 0.1 unless ON(instance) (pg_replication_is_replica == 1)
    for: 2m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql too many dead tuples"
      description: "PostgreSQL dead tuples is too large\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlSplitBrain
    expr: count(pg_replication_is_replica == 0) != 1
    for: 0m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql split brain"
      description: "Split Brain, too many primary Postgresql databases in read-write mode\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlPromotedNode
    expr: pg_replication_is_replica and changes(pg_replication_is_replica[1m]) > 0
    for: 0m
    labels:
      severity: 一般告警
    annotations:
      summary: "{{$labels.instance}} Postgresql promoted node"
      description: "Postgresql standby server has been promoted as primary node\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlSslCompressionActive
    expr: sum(pg_stat_ssl_compression) > 0
    for: 0m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql SSL compression active"
      description: "Database connections with SSL compression enabled. This may add significant jitter in replication delay. Replicas should turn off SSL compression via `sslcompression=0` in `recovery.conf`.\n  当前值={{ $value }}"

  - alert: 警报！PostgresqlTooManyLocksAcquired
    expr: ((sum (pg_locks_count)) / (pg_settings_max_locks_per_transaction * pg_settings_max_connections)) > 0.20
    for: 2m
    labels:
      severity: 严重告警
    annotations:
      summary: "{{$labels.instance}} Postgresql too many locks acquired"
      description: "Too many locks acquired on the database. If this alert happens frequently, we may need to increase the postgres setting max_locks_per_transaction.\n  当前值={{ $value }}"
```

![image-20230330163702924](/Users/xiahuhu/Library/Application Support/typora-user-images/image-20230330163702924.png)