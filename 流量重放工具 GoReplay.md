---
title: 流量重放工具 GoReplay
date: 2018-02-24 10:34:30
categories: 运维
tags:
  - HTTP
  - GoReplay
---
# GoReplay 简介

GoReplay 是一个简单且安全通过真实流量测试未上线产品的工具

GoReplay 工作方式：listener server 捕获流量，并将其发送至 replay server 或者保存至文件。replay server 会将流量转移至配置的地址

![GoReplay Workflow](https://web-blog-1252155400.cos.ap-beijing.myqcloud.com/blog_photo/2018/02/goreplay.png)


# GoReplay 下载

GoReplay 安装，支持  Windows（新版本可能不支持）, Linux x64 and Mac OS

下载地址：请参照 [Latest release](https://github.com/buger/goreplay/releases/latest)

-----------------

# GoReplay 安装

可直接下载编译好的版本，也可自行编译，或者参照 [Getting Started](https://github.com/buger/goreplay/wiki/Getting-Started#installing-gor)

`本文使用 mac v0.16.1 版本`

编译方法：[Compilation](https://github.com/buger/goreplay/wiki/Compilation)

使用 `./goreplay` 测试，输出如下：

```
Version: 0.16.1
2018/02/24 10:47:37 Required at least 1 input and 1 output
```
<!-- more -->

-----------------

# GoReplay 输入输出参数

## 1. 输入参数

* `--input-raw` 捕获 HTTP 流量，需指定端口号，[详细说明](https://github.com/buger/goreplay/wiki/Capturing-and-replaying-traffic)
* `--input-file` 使用 `--output-file` 记录的文件作为输入，[详细说明](https://github.com/buger/goreplay/wiki/Saving-and-Replaying-from-file)
* `--input-tcp` 将多个 Gor 实例获取的流量聚集到一个 Gor 实例，[详细说明](https://github.com/buger/goreplay/blob/master/docs/Distributed-configuration.md)

## 2. 输出参数

* `--output-http` 相应流量的终端，接收 URL 地址
* `--output-file` 将获取的流量记录如文件，[详细说明](https://github.com/buger/goreplay/wiki/Saving-and-Replaying-from-file)
* `--output-tcp` 将获取的流量转移至另外的 Gor 实例，与 `--input-tcp` 组合使用
* `--output-stdout` debug 工具，将流量信息直接输出

-----------------

# GoReplay 使用

此部分介绍 GoReplay 部分使用方法，其它高级进阶请参照 [wiki](https://github.com/buger/goreplay/wiki)

## 1. 将 8888 端口流量输出到终端

命令

```
sudo ./goreplay --input-raw :8888 --output-stdout
```
输出

```
1 a979cfad878d60d399203b771f8d297dbbce9ffc 1519442439270338000
GET / HTTP/1.1
Host: t:8888
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.167 Safari/537.36
Accept: application/x.bi_data.v1+json
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: XDEBUG_SESSION=PHPSTORM


1 6a623d48e9fa43d604b3c6d8c3bfef6e4a5b7895 1519442441093641000
GET /favicon.ico HTTP/1.1
Host: t:8888
Connection: keep-alive
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.167 Safari/537.36
Accept: application/x.bi_data.v1+json
Referer: http://t:8888/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: XDEBUG_SESSION=PHPSTORM
```
## 2. 将 8888 端口流量镜像至 http://t1:8888

命令

```
sudo ./goreplay --input-raw :8888 --output-http "http://t1:8888"
```

## 3. 将 8888 端口流量写入文件

命令

```
sudo ./goreplay --input-raw :8888 --output-file requests.log
```
## 4. 流量限制

使用线上流量测试时，如线上流量过大，如果将全部流量导入演示（staging）或者 测试（dev）服务器，后果可想而知

GoReplay 流量限制策略为

* 随机丢弃一部分请求
* 根据 Header 或者 URL 参数丢弃一部分请求

### 随机丢弃一部分请求

* **Absolute** 如果当前秒请求达到限制，忽略剩余请求，直至下一秒计数器恢复
* **Percentage** 对于输入文件设定该参数将加速或者减缓请求执行速度。根据设定百分比使用随机数生成器决定请求是否被接受

命令 **Absolute**

```
# 每秒最大 10 个请求数
sudo ./goreplay --input-raw :8888 --output-http "http://t1:8888|10"
```

命令 **Percentage**

```
# 获取少于 10% 的流量
sudo ./goreplay --input-raw :8888 --output-http "http://t1:8888|10%"
```
### 根据 Header 或者 URL 参数丢弃一部分请求

命令

```
# Limit based on header value
sudo ./goreplay --input-raw :8888 --output-tcp "replay.local:28020|10%" --http-header-limiter "X-API-KEY: 10%"
```

## 5. 请求过滤

* `--http-allow-url` 允许正则 URL，不包换主句名部分
* `--http-disallow-url` 不允许正则 URL
* `--http-allow-header` 允许的 Header 头
* `--http-disallow-header` 不允许的 Header 头
* `--http-allow-method` 允许的请求方法

命令

```
# only forward requests being sent to the /api endpoint
sudo ./goreplay --input-raw :8888 --output-http staging.com --http-allow-url /api

# only forward requests NOT being sent to the /api... endpoint
sudo ./goreplay --input-raw :8888 --output-http staging.com --http-disallow-url /api

# only forward requests with an api version of 1.0x
sudo ./goreplay --input-raw :8888 --output-http staging.com --http-allow-header api-version:^1\.0\d

# only forward requests NOT containing User-Agent header value "Replayed by Gor"
sudo ./goreplay --input-raw :8888 --output-http staging.com --http-disallow-header "User-Agent: Replayed by Gor"

sudo ./goreplay --input-raw :8888 --output-http "http://staging.server" \
    --http-allow-method GET \
    --http-allow-method OPTIONS
```
-----------------

# 注意事项

## 1. 本机自访问

上文监听 8888 端口，又将流量指向本机的 t1:8888，会导致流量一直循环，解决办法是使用 `--http-allow-header` 过滤掉无效的请求

命令

```
sudo ./goreplay --input-raw :8888 --output-http "http://t1:8888" --verbose --debug --http-allow-header "Host:t1:8888"
```
`--verbose` `--debug` 用于开启 debug，两个命令必须**同时使用**。

输出
```
[DEBUG][PID 60323][1519449007464424000][1519449007464.423828ms] Listening for traffic on: :8888
Version: 0.16.1
[DEBUG][PID 60323][1519449011473978000][4009.554000ms] [EMITTER] input: 1 5a29e8440342c987846af2160fd84f83aec7d71e 1519449009833813000
GET /api/expires/no_return?date=2017-12-07&project=total&business=total&section=total&level=A1 HTTP/1.1
Host: t:8888
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.167 Safari/537.36
Accept: application/x.bi_data.v1+json
Accept-Encoding: gzip, deflate
Accept-Language 553 from: Intercepting traffic from: :8888
[DEBUG][PID 60323][1519449013475018000][2001.040000ms] [EMITTER] input: 1 7aa1b183ee5139299c30371e6666442442b10b92 1519449012694215000
GET /favicon.ico HTTP/1.1
Host: t:8888
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.167 Safari/537.36
Accept: application/x.bi_data.v1+json
Referer: http://t:8888/api/expires/no_return?date=2017-12-07&project=total&business=total&section=total&level=A1
Accept-Encoding: gzip, deflate
Accept-La 559 from: Intercepting traffic from: :8888
```

## 2. --input-raw

`-input-raw` 使用 RAW sockets 并需要 sudo 权限，本文全文使用 sudo，实际使用请按照情况选择。

-----------------

# goreplay 参数

```
Gor is a simple http traffic replication tool written in Go. Its main goal is to replay traffic from production servers to staging and dev environments.
Project page: https://github.com/buger/gor
Author: <Leonid Bugaev> leonsbox@gmail.com
Current Version: 0.16.1

  -cpuprofile string
    	write cpu profile to file
  -debug verbose
    	Turn on debug output, shows all intercepted traffic. Works only when with verbose flag
  -exit-after duration
    	exit after specified duration
  -http-allow-header value
    	A regexp to match a specific header against. Requests with non-matching headers will be dropped:
	 gor --input-raw :8080 --output-http staging.com --http-allow-header api-version:^v1
  -http-allow-method value
    	Whitelist of HTTP methods to replay. Anything else will be dropped:
	gor --input-raw :8080 --output-http staging.com --http-allow-method GET --http-allow-method OPTIONS
  -http-allow-url value
    	A regexp to match requests against. Filter get matched against full url with domain. Anything else will be dropped:
	 gor --input-raw :8080 --output-http staging.com --http-allow-url ^www.
  -http-disallow-header value
    	A regexp to match a specific header against. Requests with matching headers will be dropped:
	 gor --input-raw :8080 --output-http staging.com --http-disallow-header "User-Agent: Replayed by Gor"
  -http-disallow-url value
    	A regexp to match requests against. Filter get matched against full url with domain. Anything else will be forwarded:
	 gor --input-raw :8080 --output-http staging.com --http-disallow-url ^www.
  -http-header-limiter value
    	Takes a fraction of requests, consistently taking or rejecting a request based on the FNV32-1A hash of a specific header:
	 gor --input-raw :8080 --output-http staging.com --http-header-imiter user-id:25%
  -http-original-host
    	Normally gor replaces the Host http header with the host supplied with --output-http.  This option disables that behavior, preserving the original Host header.
  -http-param-limiter value
    	Takes a fraction of requests, consistently taking or rejecting a request based on the FNV32-1A hash of a specific GET param:
	 gor --input-raw :8080 --output-http staging.com --http-param-limiter user_id:25%
  -http-rewrite-header value
    	Rewrite the request header based on a mapping:
	gor --input-raw :8080 --output-http staging.com --http-rewrite-header Host: (.*).example.com,$1.beta.example.com
  -http-rewrite-url value
    	Rewrite the request url based on a mapping:
	gor --input-raw :8080 --output-http staging.com --http-rewrite-url /v1/user/([^\/]+)/ping:/v2/user/$1/ping
  -http-set-header value
    	Inject additional headers to http reqest:
	gor --input-raw :8080 --output-http staging.com --http-set-header 'User-Agent: Gor'
  -http-set-param value
    	Set request url param, if param already exists it will be overwritten:
	gor --input-raw :8080 --output-http staging.com --http-set-param api_key=1
  -input-dummy value
    	Used for testing outputs. Emits 'Get /' request every 1s
  -input-file value
    	Read requests from file:
	gor --input-file ./requests.gor --output-http staging.com
  -input-file-loop
    	Loop input files, useful for performance testing.
  -input-kafka-host string
    	Send request and response stats to Kafka:
	gor --output-stdout --input-kafka-host '192.168.0.1:9092,192.168.0.2:9092'
  -input-kafka-json-format
    	If turned on, it will assume that messages coming in JSON format rather than  GoReplay text format.
  -input-kafka-topic string
    	Send request and response stats to Kafka:
	gor --output-stdout --input-kafka-topic 'kafka-log'
  -input-raw value
    	Capture traffic from given port (use RAW sockets and require *sudo* access):
	# Capture traffic from 8080 port
	gor --input-raw :8080 --output-http staging.com
  -input-raw-engine libpcap
    	Intercept traffic using libpcap (default), and `raw_socket` (default "libpcap")
  -input-raw-expire duration
    	How much it should wait for the last TCP packet, till consider that TCP message complete. (default 2s)
  -input-raw-realip-header string
    	If not blank, injects header with given name and real IP value to the request payload. Usually this header should be named: X-Real-IP
  -input-raw-track-response
    	If turned on Gor will track responses in addition to requests, and they will be available to middleware and file output.
  -input-tcp value
    	Used for internal communication between Gor instances. Example:
	# Receive requests from other Gor instances on 28020 port, and redirect output to staging
	gor --input-tcp :28020 --output-http staging.com
  -input-tcp-certificate string
    	Path to PEM encoded certificate file. Used when TLS turned on.
  -input-tcp-certificate-key string
    	Path to PEM encoded certificate key file. Used when TLS turned on.
  -input-tcp-secure
    	Turn on TLS security. Do not forget to specify certificate and key files.
  -memprofile string
    	write memory profile to this file
  -middleware string
    	Used for modifying traffic using external command
  -output-dummy value
    	DEPRECATED: use --output-stdout instead
  -output-file value
    	Write incoming requests to file:
	gor --input-raw :80 --output-file ./requests.gor
  -output-file-append
    	The flushed chunk is appended to existence file or not.
  -output-file-flush-interval duration
    	Interval for forcing buffer flush to the file, default: 1s. (default 1s)
  -output-file-queue-limit int
    	The length of the chunk queue. Default: 256 (default 256)
  -output-file-size-limit value
    	Size of each chunk. Default: 32mb (default 33554432)
  -output-http value
    	Forwards incoming requests to given http address.
	# Redirect all incoming requests to staging.com address
	gor --input-raw :80 --output-http http://staging.com
  -output-http-debug
    	Enables http debug output.
  -output-http-elasticsearch string
    	Send request and response stats to ElasticSearch:
	gor --input-raw :8080 --output-http staging.com --output-http-elasticsearch 'es_host:api_port/index_name'
  -output-http-header --output-http-header
    	WARNING: --output-http-header DEPRECATED, use `--http-set-header` instead
  -output-http-header-filter --output-http-header-filter
    	WARNING: --output-http-header-filter DEPRECATED, use `--http-allow-header` instead
  -output-http-header-hash-filter output-http-header-hash-filter
    	WARNING: output-http-header-hash-filter DEPRECATED, use `--http-header-hash-limiter` instead
  -output-http-method --output-http-method
    	WARNING: --output-http-method DEPRECATED, use `--http-allow-method` instead
  -output-http-redirects int
    	Enable how often redirects should be followed.
  -output-http-response-buffer int
    	HTTP response buffer size, all data after this size will be discarded.
  -output-http-rewrite-url --output-http-rewrite-url
    	WARNING: --output-http-rewrite-url DEPRECATED, use `--http-rewrite-url` instead
  -output-http-stats
    	Report http output queue stats to console every 5 seconds.
  -output-http-timeout duration
    	Specify HTTP request/response timeout. By default 5s. Example: --output-http-timeout 30s (default 5s)
  -output-http-track-response
    	If turned on, HTTP output responses will be set to all outputs like stdout, file and etc.
  -output-http-url-regexp --output-http-url-regexp
    	WARNING: --output-http-url-regexp DEPRECATED, use `--http-allow-url` instead
  -output-http-workers int
    	Gor uses dynamic worker scaling by default.  Enter a number to run a set number of workers.
  -output-kafka-host string
    	Read request and response stats from Kafka:
	gor --input-raw :8080 --output-kafka-host '192.168.0.1:9092,192.168.0.2:9092'
  -output-kafka-json-format
    	If turned on, it will serialize messages from GoReplay text format to JSON.
  -output-kafka-topic string
    	Read request and response stats from Kafka:
	gor --input-raw :8080 --output-kafka-topic 'kafka-log'
  -output-null
    	Used for testing inputs. Drops all requests.
  -output-stdout
    	Used for testing inputs. Just prints to console data coming from inputs.
  -output-tcp value
    	Used for internal communication between Gor instances. Example:
	# Listen for requests on 80 port and forward them to other Gor instance on 28020 port
	gor --input-raw :80 --output-tcp replay.local:28020
  -output-tcp-secure
    	Use TLS secure connection. --input-file on another end should have TLS turned on as well.
  -output-tcp-stats
    	Report TCP output queue stats to console every 5 seconds.
  -prettify-http
    	If enabled, will automatically decode requests and responses with: Content-Encodning: gzip and Transfer-Encoding: chunked. Useful for debugging, in conjuction with --output-stdout
  -split-output true
    	By default each output gets same traffic. If set to true it splits traffic equally among all outputs.
  -stats
    	Turn on queue stats output
  -verbose
    	Turn on more verbose output
```

-----------------

**END**