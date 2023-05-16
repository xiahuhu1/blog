# Nacos 监控手册

Nacos 0.8.0版本完善了监控系统，支持通过暴露*metrics*数据接入第三方监控系统监控Nacos运行状态，目前支持prometheus、elastic search和influxdb，下面结合prometheus和grafana如何监控Nacos，官网[grafana监控页面](http://monitor.nacos.io/)。与elastic search和influxdb结合可自己查找相关资料

## 搭建Nacos集群暴露*metrics*数据

按照[部署文档](https://www.bookstack.cn/read/nacos-2.0-zh/$a7fe555b26a1a9bb.html)搭建好Nacos集群

配置application.properties文件，暴露*metrics*数据

```
management.endpoints.web.exposure.include=*
```

访问{ip}:8848/nacos/actuator/prometheus，看是否能访问到*metrics*数据

## 搭建prometheus采集Nacos *metrics*数据

下载你想安装的prometheus版本，地址为[download prometheus](https://prometheus.io/download/)

### linux & mac

解压prometheus压缩包

```
tar xvfz prometheus-*.tar.gzcd prometheus-*
```

修改配置文件prometheus.yml采集Nacos *metrics*数据

```
    metrics_path: '/nacos/actuator/prometheus'    static_configs:      - targets: ['{ip1}:8848','{ip2}:8848','{ip3}:8848']
```

启动prometheus服务

```
./prometheus --config.file="prometheus.yml"
```

### windows

下载对应的windows版本并解压

修改配置文件prometheus.yml采集Nacos *metrics*数据

```
    metrics_path: '/nacos/actuator/prometheus'    static_configs:      - targets: ['{ip1}:8848','{ip2}:8848','{ip3}:8848']
```

启动prometheus服务

```
prometheus.exe --config.file=prometheus.yml
```

通过访问[http://{ip}:9090/graph可以看到prometheus的采集数据，在搜索栏搜索nacos\_monitor可以搜索到Nacos数据说明采集数据成功](http://{ip}:9090/graph可以看到prometheus的采集数据，在搜索栏搜索nacos/_monitor可以搜索到Nacos数据说明采集数据成功) ![IMAGE](https://static.sitestack.cn/projects/nacos-2.0-zh/d13961cbed134bcdb1252558be6ca99a.png)

## 搭建grafana图形化展示*metrics*数据

和prometheus在同一台机器上安装grafana，使用 yum 安装grafana

### mac

```
brew install grafanabrew services start grafana
```

### linux

```
sudo yum install https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-5.2.4-1.x86_64.rpmsudo service grafana-server start
```

### windows

参考文档：http://docs.grafana.org/installation/windows/

访问grafana: [http://{ip}:3000](http://{ip}:3000/)

配置prometheus数据源 ![IMAGE](https://static.sitestack.cn/projects/nacos-2.0-zh/d094e6801c6d192849ddb22f8e8be5d4.png)

导入Nacos grafana监控[模版](https://github.com/nacos-group/nacos-template/blob/master/nacos-grafana.json) ![IMAGE](https://static.sitestack.cn/projects/nacos-2.0-zh/3aa73994fd8f2349d214105b23a2f7ee.png)

Nacos监控分为三个模块:

- nacos monitor展示核心监控项 ![IMAGE](https://static.sitestack.cn/projects/nacos-2.0-zh/126eb6519923ef50219296e49179734b.png)
- nacos detail展示指标的变化曲线 ![IMAGE](https://static.sitestack.cn/projects/nacos-2.0-zh/add846a16f1002c80b78ce3906646c6f.png)
- nacos alert为告警项 ![IMAGE](https://static.sitestack.cn/projects/nacos-2.0-zh/8272e691edd6b1c1513b365261fb6509.png)

## 配置grafana告警

当Nacos运行出现问题时，需要grafana告警通知相关负责人。grafana支持多种告警方式，常用的有邮件，钉钉和webhook方式

### 钉钉告警

钉钉可以通过配置钉钉机器人 ![IMAGE](https://static.sitestack.cn/projects/nacos-2.0-zh/4cbe51d96cac4657fa2b3517d2e29679.png)

配置钉钉通知url ![IMAGE](https://static.sitestack.cn/projects/nacos-2.0-zh/2c882cc7e7ba34c6a80f1c7060158a4e.png)

测试告警项 ![IMAGE](https://static.sitestack.cn/projects/nacos-2.0-zh/b43a5530c35e016beb9bb9fd55865e0e.png)

### 邮件告警

修改defaults.ini配置文件，增加邮件告警

```
#################################### SMTP / Emailing ##########################[smtp]enabled = truehost = smtp.126.com:25user = xxxxxxpassword = xxxxx;cert_file =;key_file =skip_verify = truefrom_address = xxxxxx@126.com[emails];welcome_email_on_sign_up = false
```

配置通知邮箱 ![IMAGE](https://static.sitestack.cn/projects/nacos-2.0-zh/26c57281c35e4c0b3d6b3add994c01cd.png)

## Nacos *metrics*含义

### jvm *metrics*

| 指标                       | 含义                         |
| :------------------------- | :--------------------------- |
| system_cpu_usage           | CPU使用率                    |
| system_load_average_1m     | load                         |
| jvm_memory_used_bytes      | 内存使用字节，包含各种内存区 |
| jvm_memory_max_bytes       | 内存最大字节，包含各种内存区 |
| jvm_gc_pause_seconds_count | gc次数，包含各种gc           |
| jvm_gc_pause_seconds_sum   | gc耗时，包含各种gc           |
| jvm_threads_daemon         | 线程数                       |

### Nacos 监控指标

| 指标                                   | 含义                                    |
| :------------------------------------- | :-------------------------------------- |
| http_server_requests_seconds_count     | http请求次数，包括多种(url,方法,code)   |
| http_server_requests_seconds_sum       | http请求总耗时，包括多种(url,方法,code) |
| nacos_timer_seconds_sum                | Nacos config水平通知耗时                |
| nacos_timer_seconds_count              | Nacos config水平通知次数                |
| nacos_monitor{name=’longPolling’}      | Nacos config长连接数                    |
| nacos_monitor{name=’configCount’}      | Nacos config配置个数                    |
| nacos_monitor{name=’dumpTask’}         | Nacos config配置落盘任务堆积数          |
| nacos_monitor{name=’notifyTask’}       | Nacos config配置水平通知任务堆积数      |
| nacos_monitor{name=’getConfig’}        | Nacos config读配置统计数                |
| nacos_monitor{name=’publish’}          | Nacos config写配置统计数                |
| nacos_monitor{name=’ipCount’}          | Nacos naming ip个数                     |
| nacos_monitor{name=’domCount’}         | Nacos naming域名个数                    |
| nacos_monitor{name=’failedPush’}       | Nacos naming推送失败数                  |
| nacos_monitor{name=’avgPushCost’}      | Nacos naming平均推送耗时                |
| nacos_monitor{name=’leaderStatus’}     | Nacos naming角色状态                    |
| nacos_monitor{name=’maxPushCost’}      | Nacos naming最大推送耗时                |
| nacos_monitor{name=’mysqlhealthCheck’} | Nacos naming mysql健康检查次数          |
| nacos_monitor{name=’httpHealthCheck’}  | Nacos naming http健康检查次数           |
| nacos_monitor{name=’tcpHealthCheck’}   | Nacos naming tcp健康检查次数            |

### nacos 异常指标

| 指标                                               | 含义                                                    |
| :------------------------------------------------- | :------------------------------------------------------ |
| nacos_exception_total{name=’db’}                   | 数据库异常                                              |
| nacos_exception_total{name=’configNotify’}         | Nacos config水平通知失败                                |
| nacos_exception_total{name=’unhealth’}             | Nacos config server之间健康检查异常                     |
| nacos_exception_total{name=’disk’}                 | Nacos naming写磁盘异常                                  |
| nacos_exception_total{name=’leaderSendBeatFailed’} | Nacos naming leader发送心跳异常                         |
| nacos_exception_total{name=’illegalArgument’}      | 请求参数不合法                                          |
| nacos_exception_total{name=’nacos’}                | Nacos请求响应内部错误异常（读写失败，没权限，参数错误） |

### client *metrics*

| 指标                                   | 含义                                  |
| :------------------------------------- | :------------------------------------ |
| nacos_monitor{name=’subServiceCount’}  | 订阅的服务数                          |
| nacos_monitor{name=’pubServiceCount’}  | 发布的服务数                          |
| nacos_monitor{name=’configListenSize’} | 监听的配置数                          |
| nacos_client_request_seconds_count     | 请求的次数，包括多种(url,方法,code)   |
| nacos_client_request_seconds_sum       | 请求的总耗时，包括多种(url,方法,code) |