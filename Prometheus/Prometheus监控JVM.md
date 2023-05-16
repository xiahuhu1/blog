>jmx_exporter:
>一个可配置抓取和以HTTP方式暴露JVM指标的收集器，以java agent程序形式运行

### 1 部署jmx_exporter

```bash
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.13.0/jmx_prometheus_javaagent-0.13.0.jar
```

写一个`tomcat.yaml`文件，内容如下

```yaml
lowercaseOutputLabelNames: true
lowercaseOutputName: true
rules:
- pattern: 'Catalina<type=GlobalRequestProcessor, name=\"(\w+-\w+)-(\d+)\"><>(\w+):'
  name: tomcat_$3_total
  labels:
    port: "$2"
    protocol: "$1"
  help: Tomcat global $3
  type: COUNTER
- pattern: 'Catalina<j2eeType=Servlet, WebModule=//([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), name=([-a-zA-Z0-9+/$%~_-|!.]*), J2EEApplication=none, J2EEServer=none><>(requestCount|maxTime|processingTime|errorCount):'
  name: tomcat_servlet_$3_total
  labels:
    module: "$1"
    servlet: "$2"
  help: Tomcat servlet $3 total
  type: COUNTER
- pattern: 'Catalina<type=ThreadPool, name="(\w+-\w+)-(\d+)"><>(currentThreadCount|currentThreadsBusy|keepAliveCount|pollerThreadCount|connectionCount):'
  name: tomcat_threadpool_$3
  labels:
    port: "$2"
    protocol: "$1"
  help: Tomcat threadpool $3
  type: GAUGE
- pattern: 'Catalina<type=Manager, host=([-a-zA-Z0-9+&@#/%?=~_|!:.,;]*[-a-zA-Z0-9+&@#/%=~_|]), context=([-a-zA-Z0-9+/$%~_-|!.]*)><>(processingTime|sessionCounter|rejectedSessions|expiredSessions):'
  name: tomcat_session_$3_total
  labels:
    context: "$2"
    host: "$1"
  help: Tomcat session $3 total
  type: COUNTER
- pattern: ".*"
```

把下载的jar包和yaml配置文件移至tomcat的bin目录下

### 2 配置

修改`tomcat`的`catalina.sh`配置文件，加入下面一行

```bash
JAVA_OPTS="-javaagent:/usr/local/tomcat/bin/jmx_prometheus_javaagent-0.13.0.jar=30018:/usr/local/tomcat/bin/tomcat.yaml"
```

### 3 启动

重启`tomcat`，`jmx_exporter`就运行了，与`tomcat`状态一致

### 4 配置prometheus连接jmx_exporter

```yaml
vim /application/prometheus/prometheus.yml 
- job_name: 'tomcat'
    static_configs:
    - targets: ['localhost:30018']   #此端口是上面JAVA_OPTS配置项配置的
```

