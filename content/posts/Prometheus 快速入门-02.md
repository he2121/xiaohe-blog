---
title: "Prometheus 快速入门-02"
date: 2021-12-08T14:30:00+08:00
tags: ["prometheus", "cloud native","监控"]
---

## 数据模型

Prometheus 底层将所有数据存储为时间序列:  即以相同 metric 和 label 维度聚合的带时间戳的流。

### 时序格式

下面格式表示一个 metric 和 label 的时序数据集合

```bash
<metric name>{<label name>=<label value>, ...}
```

例如，一个时间序列的度量名称 api_http_requests_total 和 label 方法="POST" 和handler="/messages"可以写成这样:

```bash
api_http_requests_total{method="POST", handler="/messages"}
```

### 采样

Prometheus 通过 pull 收到各个 metric 的实际数据，样本形成实际的时间序列数据，每个样本包括:

- float64值

- 一个毫秒精度时间戳

### 小结

- Prometheus 以 metric 和 label 的维度聚合时序数据
- Prometheus 以 pull 方式，收集各个监控目标上 metric 的值与时间戳

## Metric 类型

Prometheus Metric 有四种类型：Counter，Gauge，Histogram，Summary

### Counter

- **特性**：只增不减

- **适用**：服务请求数、已完成任务数、错误出现次数等
- **示例**：http 请求数

```bash
# TYPE prometheus_http_requests_total counter
prometheus_http_requests_total{code="200",handler="/-/ready"} 6
prometheus_http_requests_total{code="200",handler="/api/v1/label/:name/values"} 43
prometheus_http_requests_total{code="200",handler="/api/v1/labels"} 35
```

- **Go SDK**:

```GO
// counter + 1
Inc()
// counter + 任意值，这个值小于 0， 则 panic
Add(float64)
```

### Gauge

- **特性**：数据可以任意变化，可增可减

- **适用**：温度、内存/磁盘使用率等
- **示例**：go 当前协程数

```bash
# TYPE go_goroutines gauge
go_goroutines 35
```

- **Go SDK**:

```go
// 设置任意值
Set(float64)
// 加一
Inc()
// 减1
Dec()
// 加任意值
Add(float64)
// 减任意值
Sub(float64)
// 设置成当前时间的 unix 秒值
SetToCurrentTime()
```

### Histogram（直方图）

- **特性**：一段时间范围内对数据进行采样，对其指定区间以及总数进行统计
- **适用**：页面的响应时间分布、resp 的 body 大小分布等...
- **示例**：http 请求延时

```bash
# HELP prometheus_http_request_duration_seconds Histogram of latencies for HTTP requests.
# TYPE prometheus_http_request_duration_seconds histogram
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.1"} 6	// 延时小于 0.1 s 的请求数 6 个
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.2"} 6 // 延时小于 0.2 s 的请求数 6 个
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.4"} 6
prometheus_http_request_duration_seconds_bucket{handler="/",le="1"} 6
prometheus_http_request_duration_seconds_bucket{handler="/",le="3"} 6
prometheus_http_request_duration_seconds_bucket{handler="/",le="8"} 6
prometheus_http_request_duration_seconds_bucket{handler="/",le="20"} 6
prometheus_http_request_duration_seconds_bucket{handler="/",le="60"} 6
prometheus_http_request_duration_seconds_bucket{handler="/",le="120"} 6
prometheus_http_request_duration_seconds_bucket{handler="/",le="+Inf"} 6	// 这个实际上等于总请求数
prometheus_http_request_duration_seconds_sum{handler="/"} 0.00016767900000000003	// 上述样本求和 sum
prometheus_http_request_duration_seconds_count{handler="/"} 6							// 这次采样总请求数
```

- **Go SDK** ：

```go
// 观测这个值落在了哪个 buckte 中
Observe(float64)
```

### Summary

- **特性**：与 Histogram 类似
- **适用**：与 Histogram 类似
- **示例**：垃圾回收停顿时间

```bash
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 4.2354e-05
go_gc_duration_seconds{quantile="0.25"} 9.222e-05
go_gc_duration_seconds{quantile="0.5"} 0.000133648
go_gc_duration_seconds{quantile="0.75"} 0.000193116
go_gc_duration_seconds{quantile="1"} 0.00906632
go_gc_duration_seconds_sum 0.064656275
go_gc_duration_seconds_count 295
```

- **Go SDK**：

```go
// 观测值
Observe(float64)
```

##### Summary VS Histogram

**相同**

- 都用来表示一段时间内数据采样结果
- 适用于查看值分布的场景

**不同点**

- Histogram：客户端在收集数据时，不存储具体的值，还是多个 bucket counter，性能相对 Counter，Gauge 无差别。但可以在服务端计算出分位数据
- Summary： 客户端把一段时间内（默认是10分钟）的数据存储下来，再去计算分位数，性能相对低一些

**如何选择**

- 需要聚合，选 Histogram。Summary 无法聚合，因为一般要监控多台机器，而 Summary 在客户端计算分位数据
- 分位数据要精确，选 Summary

## PormQL 使用

### PormQL 查询返回数据类型

1.  **瞬时向量（Instant vector）**: 包含一组时序数据，默认返回**最新时间点的值**

查询语句，格式与上述提到的[时序格式](#时序格式)一致，例如

```bash
// 例子1 查询 http 接口请求数
http_requests_total
// 例子2 查询 http 接口请求数，限定 hanlder
http_requests_total{handler="/api/comments"}
```

- 返回结果

![image-20211208170850461](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20211208170850461-2021-12-08.png)

2. **范围向量（Range vector）**: 一组时序数据，每个时序有多个值（限定在一段时间内，不同时间点的值）

- 查询语句，

```bash
// 例子1 查询 http 5 分钟内 上报数据
http_requests_total[5m]
// 例子2 查询 http 指定 handler 5 分钟内 上报的数据
http_requests_total{handler="/api/comments"}[5m]
```

- 返回结果

  ![image-20211208171741338](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20211208171741338-2021-12-08.png)

3. **标量**

单纯的数字：2，2.0

### 查询语句

1. `metric_name{label_name=lable_value}`查询出瞬时向量，例如

```bash
http_requests_total{code="200"} // 表示查询 metric 为 http_requests_total，label code 为 "200" 的数据
```

2. 瞬时查询语句后加上`[time]`，例如`metric_name{label_name=lable_value}[5m]`表示限定 5 分钟内的数据，返回的是范围向量

3. 查询条件支持不等于`!=`，正则 `=~` `!~`, 例如

```bash
http_requests_total{code!="200"}  // 表示查询 code 不为 "200" 的数据
http_requests_total{code=～"2.."} // 表示查询 code 为 "2xx" 的数据
http_requests_total{code!～"2.."} // 表示查询 code 不为 "2xx" 的数据
```

- 瞬时向量结果支持[运算](https://prometheus.io/docs/prometheus/latest/querying/operators/)，`+，-，*，/，%，^，==，!=，>，<，>=，<=，and，or，unless，sum，min，max，avg，stddev，stdvar，count，count_values，bottomk，topk，quantile`

```bash
http_requests_total{code="200"} * 2 // 把里面的值 * 2
http_requests_total{code="200"} >= 2 // 只要值 > 2 的数据
http_requests_total{code="200"} >= 2 or http_requests_total{code="200"} == 0 // 只要值 > 2 的数据和值为 0 的
sum(http_requests_total{code="200"}) // 各个时序数据里的值求和
topk(5, http_requests_total{code="200"}) // 只要值在前 5 的时序数据
```

- [内置函数 ](https://prometheus.io/docs/prometheus/latest/querying/functions/) 如 `abs,floor, rate`, 不同函数有着不同的操作对象，例如 `rate` 针对与范围向量，瞬时向量没有增长率

```bash
floor(avg(http_requests_total{code="200"}))	// 向下取整
ceil(avg(http_requests_total{code="200"}))	// 向上取整
rate(http_requests_total[5m])								// 求 5m 增长率
```

## Go Client 实现自定义 exporter

在上一篇文章中，提到了 Prometheus 提供了许多插件式的 [Exporter](https://prometheus.io/docs/instrumenting/exporters/), 可以通过安装这些 exporter 来监控各种指标，如机器、容器、中间件、MySQL ... 

但是如果我们想监控应用程序呢，如 http，rpc 接口的 QPS、接口时延、错误率。这时候我们可以利用 Prometheus Client SDK 自定义监控

### 示例

- 如下面代码所示，有一个 web 的 demo，分别有两个接口 `/api1` 与 `api2`

```go
package main

import (
  "fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/api1", api1)
	http.HandleFunc("/api2", api2)

	err := http.ListenAndServe(":1234", nil)
	if err != nil {
		panic(err)
	}
}

func api1(w http.ResponseWriter, r *http.Request) {
	fmt.Println("hello api1")
}

func api2(w http.ResponseWriter, r *http.Request) {
	fmt.Println("hello api2")
}

```

- 如何利用 prometheus 监控这两个接口请求量与接口时延呢

```bash
go get github.com/prometheus/client_golang/prometheus
```

1. 向外暴露 exporter 地址，下述例子使用默认的 exporter 提供了一些 go 程序的基本监控，如 go 协程数，垃圾回收时间..

```go
http.Handle("/metrics", promhttp.Handler())
go func() {
	_ = http.ListenAndServe(":1235", nil)
}()
```
exporter 上报数据示例如下：
```bash
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 9
# HELP go_info Information about the Go environment.
# TYPE go_info gauge
go_info{version="go1.17.2"} 1
...
```

2. 注册自定义接口 Counter

   - `NewCounter`:一个 Counter

   - `NewCounterVec`: 一组 Counter，按里面的 label 值分组

 使用示例如下（完整例子在这节最后）

```go
// 声明一个 Counter
var apiCounterVec = prometheus.NewCounterVec(prometheus.CounterOpts{
	Name: "http_api_counter",
	Help: "web http requests number",
}, []string{"handler"})

// 声明 一组 Counter
var apiTotalCounter = prometheus.NewCounter(prometheus.CounterOpts{
	Name: "http_api_total_counter",
	Help: "web http requests number",
})
// 注册到 prometheus exporter 中
prometheus.MustRegister(apiTotalCounter)
prometheus.MustRegister(apiCounterVec)

// 计数
apiTotalCounter.Inc()
apiCounterVec.WithLabelValues("/api").Inc()
```

3. 注册自定义接口 Histogram, 示例如下

```go
// 声明一组 Histogram
var apiHandleMS = prometheus.NewHistogramVec(prometheus.HistogramOpts{
	Name: "http_api_handler_ms",
	Help: "Microseconds of HTTP interface processing",
	Buckets: prometheus.LinearBuckets(1,19,10),
}, []string{"handler"})
// 注册到 prometheus exporter 种 
prometheus.MustRegister(apiHandleMS)
// 计数
apiHandleMS.WithLabelValues("/api").Observe(xx)
```

- [完整示例仓库](https://github.com/he2121/demos/tree/master/prometheus), 大致流程如下

```go

func main() {
	// 初始化 prometheus exporter
	initPrometheus()

	// web 示例
	http.HandleFunc("/api1", prometheusMetric(http.HandlerFunc(api1)))
	http.HandleFunc("/api2", prometheusMetric(http.HandlerFunc(api2)))
	err := http.ListenAndServe(":1234", nil)
	if err != nil {
		panic(err)
	}
}

func initPrometheus() {
	// 注册自定义的监控指标
	prometheus.MustRegister(apiTotalCounter)
	prometheus.MustRegister(apiCounterVec)
	prometheus.MustRegister(apiHandleMS)
	// 暴露 exporter 地址，prometheus server 通过 pull 这个地址，拉取指标数据
	http.Handle("/metrics", promhttp.Handler())
	go func() {
		_ = http.ListenAndServe(":1235", nil)
	}()
}

// metric 中间件
func prometheusMetric(handler http.Handler) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// 总接口访问量计数
		apiTotalCounter.Inc()
		// 单个接口访问量计数
		apiCounterVec.WithLabelValues(r.URL.String()).Inc()
		start := time.Now()
		handler.ServeHTTP(w, r)
		ms := time.Now().UnixMicro() - start.UnixMicro()
		// 接口时延计数
		apiHandleMS.WithLabelValues(r.URL.String()).Observe(float64(ms))
	}
}
```

自定义指标结果：

![image-20211209111424017](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20211209111424017-2021-12-09.png)

## 总结

- 介绍了 Prometheus 四种 Metric 类型
- 介绍了 PormQL 简单使用
- 以一个实际例子介绍了如何用 Prometheus 提供的 client SDK 自定义监控
