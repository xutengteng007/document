## Prometheus配置文件动态配置

> 动态配置流程
>
> - 通过使用Nacos的配置中心，实现配置的发布
> - confd实时监控Nacos配置文件中的某一个发布出来的配置，confd将配置读取到本地并写入本地的某个配置文件中
> - Prometheus添加配置文件路径，并开启热加载，当配置文件发生变化后，prometheus检测到配置文件发生变化后会改变自身的配置，比如targets

### 安装包准备

**prometheus**

从prometheus官网下载二进制安装包，这里选择[版本2.7.1](https://github.com/prometheus/prometheus/releases/download/v2.7.1/prometheus-2.7.1.linux-amd64.tar.gz)

![image](uploads/prometheus1.png)

**confd**

confd需要源码编译，这里选择[版本0.18.0](https://github.com/nacos-group/confd/archive/v0.18.0.tar.gz)

![image](uploads/prometheus2.png)

**go**

confd源码编译需要通过go语言，因此需要安装go环境，这里选择[版本1.13](https://dl.google.com/go/go1.13.linux-amd64.tar.gz)

![image](uploads/go1png.png)

**alertmanager**

下载alertmanager，这里选择[版本v0.20.0-rc](https://github.com/prometheus/alertmanager/releases/download/v0.20.0/alertmanager-0.20.0.linux-amd64.tar.gz)

![image](uploads/go2.png)


## 安装部署

安装目录设定为`/data`（可自由设置）

### 安装prometheus

解压压缩包，并修改目录名称
```
cd /data
tar -xzvf prometheus-2.7.1.linux-amd64.tar.gz
mv prometheus-2.7.1 prometheus
```

启动prometheus
```
/data/prometheus/prometheus --config.file=/data/prometheus/prometheus.yml --web.listen-address=:9090 --web.enable-lifecycle
```
+ `--config.file`：指定prometheus的配置文件
+ `--web.listen-address`：指定prometheus的监听端口
+ `--web.enable-lifecycle`：可以通过web请求实现热加载

> TODO 之后可以将prometheus通过systemd管理起来

### 编译confd

**搭建go环境**

解压go的压缩包
```
tar -xzvf go1.13.linux-amd64.tar.gz
```
配置环境变量，这里设置全局的环境变量。可以选择在`/etc/profile.d`中创建shell文件`loushang.sh`（名字随意），配置内容如下
```
# go
export GOROOT=/data/go
export PATH=$GOROOT/bin:$PATH
export GOPATH=/data/work/go
```
以上配置了go的根目录，二进制工具使用路径，以及工作空间路径。

激活环境变量
```
source /etc/profile
```
在go的工作空间`$GOPATH`中创建工程目录
```
mkdir -p $GOPATH/src/github.com/kelseyhightower
```
解压并编译confd
```
dcd $GOPATH/src/github.com/kelseyhightower
tar -xzvf v0.18.0.tar.gz
mv confd-0.18.0 confd
cd confd
make
```
编译成功后，在`/data/go/work/src/github.com/kelseyhightower/confd/bin`会生成二进制文件confd
将该文件拷贝到可执行路径`/usr/local/bin`

### 配置confd

创建confd的配置目录
```
mkdir -p /etc/confd/{conf.d,templates}
```
+ conf.d目录：存放confd的脚本
+ templates目录：存放confd的模板

**创建prometheus规则同步策略**

在conf.d目录创建文件`alarm-rule.toml`
```
[template]
src = "alarm-rule.yml.tmpl"
dest = "/data/prometheus/alarm-rule.yml"
keys = ["/alarm-rule"]
reload_cmd = "/usr/bin/curl --connect-timeout 5 --retry 5 --retry-max-time 40 -X POST http://localhost:9090/-/reload"
```
+ src：confd的模板文件所在位置，即`/etc/confd/templates`中的模板文件
+ dest：为需要同步的prometheus规则文件的位置，这里是`/data/prometheus`
+ keys：是在nacos中设定的配置名称
+ reload_cmd：热加载命令

**创建prometheus规则模板**

在templates目录创建模板文件`alarm-rule.yml.tmpl`
```
groups:
- name: alert-rules
  rules:
  {{$data := jsonArray (getv "/alarm-rule")}}
  {{range $data}}
  - alert: {{.name}}
    expr: {{.content}}
    for: {{.period}}
    labels:
      status: {{.level}}
    annotations:
      summary: "{{.summary}}"
      description: "{{.description}}"
  {{end}}
```
> 其中的占位符变量与在Loushang微服务管理平台告警规则提供的基本信息有关。


**重启prometheus**

由于prometheus的配置文件`prometheus.yml`没有将`alarm-rule.yml`管理起来，因此需要修改prometheus.yml`。
首先关闭prometheus（ctrl+c或者ctrl+z，或者关闭进程）
修改`prometheus.yml`文件，增加对告警规则文件的管理

```
rule_files:
  - "alarm-rule.yml"
```

重启prometheus

**启动confd**

```
confd -backend nacos -node http://127.0.0.1:8848/nacos -group loushang -watch
```
+ `-backend`：后端存储，这里是nacos
+ `node`: 指定nacos服务
+ `group`：指定分组
+ `-watch`：监控配置文件alarm-rule是否变化

然后在prometheus的安装目录会发现`alarm-rule.yml`文件。 查看prometheus ![](uploads/prometheus3.png)

###  监控服务配置

prometheus需要监控nacos中注册的服务，但它并不支持对nacos的服务发现。 而由于prometheus内置模块支持consul，Loushang微服务平台实现nacos与consul的对接。 此时，需要继续配置`prometheus.yml`

```
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['127.0.0.1:9090']

  - job_name: 'nacos'
    metrics_path: '/nacos/actuator/prometheus'
    static_configs:
    - targets: ['127.0.0.1:8848']

  - job_name: 'nacos-register-service'
    metrics_path: '/actuator/prometheus'
    consul_sd_configs:
    - server: '127.0.0.1:8082'
      services: []
    relabel_configs:
    - source_labels: [__meta_consul_service]
      separator: ;
      regex: (.+)
      target_label: __metrics_path__
      replacement: /${1}/actuator/prometheus
      action: replace
```

以上配置为本地环境的服务，本地开发可以配置在线环境的服务地址。

配置完毕之后，重启prometheus服务。 查看prometheus![](uploads/prometheus4.png)