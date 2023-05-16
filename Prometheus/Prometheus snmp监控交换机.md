### 一、交换机开启SNMP协议

```bash
snmp-agent sys-info version v2 #v2为snmp协议版本
snmp-agent community read ci test1234 #团体名
snmp-agent trap enable
snmp-agent target-host trap address udp-domain 192.168.1.9 params securityname Public123456 #允许向snmp服务器发送trap报文
```

### 二、部署snmp_exporter

github官方地址：

https://github.com/prometheus/snmp_exporter

#### 1、下载snmp_exporter

```bash
wget https://github.com/prometheus/snmp_exporter/releases/download/v0.19.0/snmp_exporter-0.19.0.linux-amd64.tar.gz
tar xf snmp_exporter-0.19.0.linux-amd64.tar.gz -C /opt/
```

#### 2、编译安装generator，生成snmp_exporter配置文件snmp.yml

`generator`：使用`netsnmp`解析`mibs`，通过它为`snmp_exporter`生成配置文件信息

**2.1 安装Go环境**
`golang`官方下载地址：` https://golang.org/dl/`

```bash
wget https://dl.google.com/go/go1.11.5.linux-amd64.tar.gz
tar xf go1.11.5.linux-amd64.tar.gz -C /usr/local/
vim /etc/profile
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
EOF
source /etc/profile
go version #显示版本号即安装成功
```

**2.2 编译安装generator**

```bash
yum install gcc gcc-g++ make net-snmp net-snmp-utils net-snmp-libs net-snmp-devel -y
go get github.com/prometheus/snmp_exporter/generator
cd ${GOPATH-$HOME/go}/src/github.com/prometheus/snmp_exporter/generator
go build
make mibs #如果已下载相关设备厂家提供的mib文件则无需再次生成mib库，跳过此步
```

**2.3 获取需要监控的硬件设备mib库**
去相关交换机厂商官方地址下载相关mib文件

思科：` ftp://ftp.cisco.com/pub/mibs/v2/v2.tar.gz`

华为：` https://support.huawei.com/mibtoolweb/enterpriseMibInfo/zh`

我这里监控的是华为`S5700`交换机

将获取的`mib`文件上传至`${GOPATH-$HOME/go}/src/github.com/prometheus/snmp_exporter/generator`

**2.4 编写generator配置文件**
`generator.yml`提供模块列表。最简单的模块只是一个`name`和一组`walk`的`oid`

```bash
modules：
   module_name：   ＃模块名称。您可以根据需要拥有任意数量的模块。
    walk：        ＃要走路的OID列表。也可以是SNMP对象名称或特定实例。
      - 1.3.6.1.2.1.2               ＃一样的“接口” 
      -的sysUpTime                   ＃同为“1.3.6.1.2.1.1.3” 
      - 1.3.6.1.2.1.31.1.1.1.6.40   ＃的“的ifHCInOctets”实例索引“40 “ 

    version：2   ＃要使用的SNMP版本。默认值为2。
                ＃ 1将使用GETNEXT，2和3将使用GETBULK。
    max_repetitions：25   ＃使用GET / GETBULK请求多少个对象，默认为25。
                         ＃对于有
    故障的设备，可能需要减少。retries：3    ＃重试失败请求的次数，默认为
    3。timeout：5s   ＃每个SNMP请求的超时，默认为5s。

    auth：
       ＃社区字符串与SNMP v1和v2一起使用。默认为“ public”。
      社区：public 

      ＃ v3具有不同且更复杂的设置。
      ＃所需的内容取决于security_level。
      ＃
      还列出了NetSNMP命令上的等效选项，例如snmpbulkwalk ＃和snmpget。请参见snmpcmd（1）。
      username：用户  ＃必需，没有默认值。NetSNMP的-u选项。
      security_level：noAuthNoPriv   ＃默认为noAuthNoPriv。NetSNMP的-l选项。
                                    ＃可以是noAuthNoPriv，authNoPriv或authPriv。
      密码：pass   ＃没有默认值。也称为authKey，NetSNMP的-A选项。
                      ＃如果security_level为authNoPriv或authPriv，则为必需。
      auth_protocol：MD5   ＃MD5，SHA，SHA224，SHA256，SHA384或SHA512。默认为MD5。-NetSNMP的选项。
                          ＃在security_level为authNoPriv或authPriv时使用。
      priv_protocol：DES   ＃ DES，AES，AES192或AES256。默认为DES。NetSNMP的-x选项。
                          ＃在security_level为authPriv时使用。
      priv_password：otherPass ＃没有默认值。也称为privKey，NetSNMP的-X选项。
                               ＃如果security_level为authPriv，则为必需。
      context_name：context ＃没有默认值。NetSNMP的-n选项。
                            ＃如果在设备上配置了上下文，则为必需。

    查找：  ＃要执行的查找的可选列表。
              ＃为`keep_source_indexes`默认为false。索引必须唯一才能使用此选项。

      ＃如果一个表的索引是bsnDot11EssIndex，通常是会是标签
      ＃上从该表得到的指标。而是使用索引
      ＃查找bsnDot11EssSsid表条目，并
      使用该值创建一个bsnDot11EssSsid标签＃。
      - source_indexes： [bsnDot11EssIndex]
        查找： bsnDot11EssSsid 
        drop_source_indexes：假  ＃如果为true，删除源的索引标识这个查询。
                                    ＃当新索引唯一时，这可以避免标签混乱。

     覆盖：＃允许按模块覆盖MIB的位
       metricName：
          ignore：true ＃从输出中删除度量。
         regex_extracts：
            Temp：＃将创建一个新指标，并将其附加到metricName上，成为metricNameTemp。
             -正则表达式：“（。*）”  ＃正则表达式从返回的SNMP散步的价值提取的值。
               值：' $ 1 '  ＃结果将解析为float64，默认为$ 1。
           状态：
             -正则表达式：“ *示例”
               值：“ 1 ”  ＃中的第一项，其正则表达式匹配，其价值解析获胜。
             -正则表达式：“ * ”
               值：“ 0 ”
         类型：DisplayString ＃覆盖度量标准类型，可能的类型有：
                             ＃   计：用型轨距的整数。
                             ＃   柜台：用脉冲型计数器的整数。
                             ＃   OctetString：一个位字符串，呈现为0xff34。
                             ＃    DateAndTime：一个RFC 2579 DateAndTime字节序列。如果设备没有时区数据，则使用UTC。
                             ＃    DisplayString：ASCII或UTF-8字符串。
                             ＃    PhysAddress48：一个48位MAC地址，表示为00：01：02：03：04：ff。
                             ＃   浮子：与类型规A 32位浮点值。
                             ＃   双：与类型规A 64位浮点值。
                             ＃    InetAddressIPv4：IPv4地址，呈现为1.2.3.4。
                             ＃    InetAddressIPv6：一个IPv6地址，表示为0102：0304：0506：0708：090A：0B0C：0D0E：0F10。
                             ＃   的InetAddress：根据RFC 4001一个InetAddress必须由InetAddressType之后进行。
                             ＃   InetAddressMissingSize：一个InetAddress，它通过
                             ＃       没有索引的大小
                             来违反RFC 4001的4.1节。必须以InetAddressType开头。＃    EnumAsInfo：为其创建单个时间序列的枚举。适用于恒定值。
                             ＃    EnumAsStateSet：每个状态的时间序列的枚举。适用于可变的低基数枚举。
                             ＃   位：RFC 2578 BITS构造，它生成每个位具有时间序列的StateSet。
```

这里generator.yml需要注意两点：

1、walk下面写的是你需要查询的信息对应的`oid`，这个像华为交换机都是可以在原厂文档上查的到的；

要监控交换机的端口流量、状态，`CPU`使用率，内存状态，温度等，关键是找出与之相对应的oid

oid信息可参考如下地址： http://oid-info.com/

#### 获取到的监控信息相关的oid可通过snmpwalk过滤查询

`snmpwalk`命令格式如下：

```bash
snmpwalk -v "snmp版本" -c "coummunity" IP oid(可选)
```

```bash
#通过snmpwalk查询过滤出有值的数据，获取子节点值
snmpwalk -v 2c -c test1234 192.168.1.100 1.3.6.1.4.1.2011.5.25.31.1.1.1.1.11|grep -v 0$
```

（2）还有一点就是community这个要写自己设置的团体名，如果不知道可以自己手动设置一个，我这里设置的是test1234。
其他的地方基本不需要修改，模块名也可以改，后面使用snmp查的时候、写prometheus配置文件的时候对应也写自己刚刚修改的mib名称就好。

###EnumAsInfo和EnumAsStateSet

SNMP包含整数索引枚举（枚举）的概念。在Prometheus中有两种表示这些字符串的方法。它们可以是“信息”指标，也可以是“状态集”。SNMP没有指定应使用的内容，这取决于数据的使用情况。一些用户可能更喜欢原始整数值，而不是字符串。

为了将枚举整数设置为字符串映射，必须使用两个替代之一。

EnumAsInfo应该用于提供类似库存数据的属性。例如设备类型，颜色名称等。此值必须恒定。

EnumAsStateSet应该用于表示状态或您可能需要警惕的事物。例如，链接状态是打开还是关闭，处于错误状态，面板是打开还是关闭等。请注意不要将其用于高基数值，因为它将为每个可能的值生成1个时间序列。

**2.5 运行generator生成snmp.yml**

```bash
export MIBDIRS=mibs #指定mib库路径
./generator generate
cp snmp.yml /usr/local/snmp_exporter/ #将生成的snmp配置移到snmp_exporter目录下
```

### 3、运行snmp_exporter

```bash
./snmp_exporter --config.file=/usr/local/snmp_exporter/snmp.yml
```

比较大的counter类型数据的值如何处理

为了为Counter64较大的值提供准确的计数器，snmp_exporter将每2 ^ 53自动包装该值，以避免64位浮点舍入，

要禁用此功能，可使用命令行参数-no-snmp.wrap-large-counters

### 4、测试

访问192.168.1.9:9116

traget:192.168.1.222 #snmp设备

module：huawei_mib #模块名，与generator.yml中设置的模块名保持一致

点击提交，可获取snmp数据

![image-20230314105202931](/Users/xiahuhu/Library/Application Support/typora-user-images/image-20230314105202931.png)

### 三、配置prometheus

**1、修改prometheus.yml**

在`scrape_configs`数据抓取配置中加入以下配置信息

```yml
scrape_configs:
  - job_name: 'snmp'
    static_configs:
      - targets:
        - 192.168.1.222  # SNMP device.
    metrics_path: /snmp
    params:
      module: [if_mib]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 192.168.1.9:9116  # The SNMP exporter's real hostname:port.
```

**2、重启prometheus**

```bash
curl -X POST http://localhost:9090/-/reload #如果prometheus启动参数有--web.enable-lifecycle，可通过此命令平滑重启
```

### 四、grafana数据可视化

根据`snmp_exporter`查询到的`HELP`后查到的参数信息，进行图形化显示。大家可以根据自己的需要去查相应的参数信息，如果想查更多的信息，基本上的流程就是：

1、将需要查询的交换机信息oid加入到generator.yml文件

2、重新编译generator.yml文件生成snmp.yml文件

3、替换原来的snmp.yml文件

4、重启snmp_exporter

![image-20230314105349134](/Users/xiahuhu/Library/Application Support/typora-user-images/image-20230314105349134.png)