# 搭建私有日志收集系统

> 日志收集采用 [Fluentd](https://docs.fluentd.org/)
> 日志分析采用 [ZincSearch](https://zincsearch-docs.zinc.dev/)

## 如何使用

### 首先安装 ZincSearch

官方[安装](https://zincsearch-docs.zinc.dev/installation/)文档。

新建 **zincsearch** 目录，并增加权限，存放一些配置文件、数据或者日志。

```bash
# 增加权限
chmod a+rwx ./zincsearch

# docker install
docker run -d -v ./zincsearch:/data -e ZINC_DATA_PATH="/data" -p 7100:4080 \
    -e GIN_MODE=release \
    -e ZINC_FIRST_ADMIN_USER=root -e ZINC_FIRST_ADMIN_PASSWORD=password \
    --name zincsearch public.ecr.aws/zinclabs/zincsearch:latest
```

### 安装 fluentd-zincsearch

新建 **fluentd** 目录，增加 **fluent.conf** 文件，内容如下：

```
<source>
  @type http
  port 7090
  bind 0.0.0.0
</source>

<match **>
  @type copy
  <store>
    @type elasticsearch
    host 192.168.100.1
    port 7100
    path /es
    user root
    password "password"
    logstash_format true
    logstash_prefix ${tag}
    logstash_dateformat %Y%m%d
    include_tag_key true
    type_name access_log
    tag_key @log_name
  </store>
  <store>
    @type stdout
  </store>
</match>
```

* source 节点 @type 使用 [http](https://docs.fluentd.org/input/http) 协议收集
* 其中 match 节点中的 host 和 port 为部署 ZincSearch 服务地址
* 其中 match 节点中的 user/password 为部署 ZincSearch 登陆用户和密码

执行下面命令：

```bash
docker run -d -p 7090:7090 --restart unless-stopped -v /opt/x/web/fluentd:/fluentd/etc --name fluentd-zincsearch xbf321/fluentd-zincsearch:latest -c /fluentd/etc/fluentd.conf
```

## 测试

发送以下数据，然后去 [ZincSearch](http://192.168.100.1:7100/ui/search) 系统中查看数据是否收到。

```
curl -X POST -d 'json={"foo":"bar"}' http://localhost:7090/app.log
```

## 开发

```bash
docker build --no-cache -t xbf321/fluentd-zincsearch:latest ./
docker push xbf321/fluentd-zincsearch:latest
```