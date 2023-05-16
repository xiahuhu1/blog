> process_exporter负责采集进程相关数据
>
> github： https://github.com/ncabatoff/process-exporter

#### 1、被监控端安装process_exporter

```bash
wget https://github.com/ncabatoff/process-exporter/releases/download/v0.7.5/process-exporter-0.7.5.linux-amd64.tar.gz
tar xf process-exporter-0.7.5.linux-amd64.tar.gz -C /opt/
mv /opt/process-exporter-0.7.5.linux-amd64 /opt/process-exporter
```

#### 2、创建配置文件

可用的模板变量：
`{{.Comm}}` 包含原始可执行文件的`basename`，`/proc//stat `中的换句话说，`2nd` 字段
`{{.ExeBase}} `包含可执行文件的`basename`
`{{.ExeFull}} `包含可执行文件的完全限定路径
`{{.Matches}} `映射包含应用命令行`tlb`所产生的所有匹配项

**定义进程名监控**
`Process-exporter` 可以进程名字匹配进程，获取进程信息。匹配规则由`name`对应的模板变量决定，以下表示监控进程名字为`nginx 与 zombie` 的进程状态

```yaml
vim process-name.yaml
process_names:
  - name: "{{.Matches}}"
    cmdline:
    - 'nginx'

  - name: "{{.Matches}}"
    cmdline:
    - 'zombie'
```

**定义全部进程监控**

```yaml
vim conf.yaml
process_names:
  - name: "{{.Comm}}"
    cmdline:
    - '.+'
```

#### 3、启动process_exporter

`systemd`启动脚本

```bash
cat <<EOF >/usr/lib/systemd/system/process_exporter.service
[Unit]
Description=process_exporter
Documentation=https://prometheus.io/
After=network.target
[Service]
Type=simple
ExecStart=/opt/process-exporter/process-exporter -config.path=/opt/process-exporter/process-name.yaml
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
systemctl start process_exporter
```

查看监控采集指标,默认监听端口`9256`

```bash
curl http://localhost:9256/metrics
```

**常见进程监控项：**
总进程数
`sum(namedprocess_namegroup_states)`

总僵尸进程数
`sum(namedprocess_namegroup_states{state=“Zombie”})`

#### 4、配置prometheus

```bash
- job_name: 'process'
    static_configs:
    - targets: ['172.16.0.10:9256']
```

重启prometheus

#### 5、添加Grafana dashboard

导入`process`监控模板，`dashboard ID`：`249`

![image-20230314103833078](/Users/xiahuhu/Library/Application Support/typora-user-images/image-20230314103833078.png)

#### 6、添加告警规则

```yml
roups:
- name: process
rules:
 - alert: nginxDown
    expr: absent(namedprocess_namegroup_states{groupname="map[:nginx]"})
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: nginx Down (instance {{ $labels.instance }})
      description: "nginx process is down\n LABELS = {{ $labels }}"
```

