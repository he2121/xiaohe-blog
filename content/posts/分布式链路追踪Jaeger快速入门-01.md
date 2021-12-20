---
title: "分布式链路追踪 Jaeger 快速入门-01"
date: 2021-12-09T14:30:00+08:00
tags: ["Open Tracing", "Jaeger", "cloud native"]
---
### Logging，Metrics 与 Tracing 关系

**同**：都是为了提高基础设施和应用程序的**可观测性**

**区别**：

| ----     | Logging                           | Metrics           | Tracing                                       |
| -------- | --------------------------------- | ----------------- | --------------------------------------------- |
| 特点     | 记录离散的事件                    | 记录可聚合的数据  | 记录请求范围内的信息                          |
| 典型指标 | 用户自行打印的调试信息...         | QPS, 接口时延分布 | 一个具体 RPC 调用中的过程：各个服务的耗时占比 |
| 典型应用 | ELK(收集，分析), log4j（记录）... | Prometheus...     | Dapper, OpenZipkin,Jaeger...                  |

它们之间有重叠，但各自关注的重点不同

![img](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/p5876-2021-12-20.png)



### OpenTracing 是什么

- 当今乃是**微服务**的天下，Tracing 是给跨进程、服务的追踪提供了一种解决方案
- OpenTracing API 提供了一个标准的、厂商中立的规范。其允许系统中存在多种分布式追踪方案 - 只要符合规范

![img](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/p5878-2021-12-20.png)

### OpenTracing 数据模型

![Mental Model](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/tracing1_0-2021-12-20.png)

**Trace(调用链)**：一个调用链代表一个事务或者流程在（分布式）系统中的执行过程。在OpenTracing标准中，调用链是多个Span组成的一个有向无环图（Directed Acyclic Graph，简称DAG）

**Span(跨度)**：一个名字、有时间的操作，代表工作流程的一个部分。span 上一般有以下信息：span 名字，span 起始时间、结束时间，tag，log，span context，span 之间的引用关系。

**SpanContext**：伴随着分布式的跟踪信息，它通过网络或通过消息总线透传上下文信息。span 上下文包含 tracing 标识符、span 标识符以及追踪系统需要传播给下游服务的任何其他数据。

span 之间的关系：

```bash
单个Trace中Span间的因果关系


        [Span A]  ←←←(The root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C是Span A的子节点，ChildOf)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         （Span G在Span F后被调用, FollowsFrom）
```

### Jaeger 是什么

Jaeger 受 Dapper 和 OpenZipkin 的启发，是 Uber的开源分布式追踪系统，已同和 k8s，Prometheus 等著名项目成为云原生计算基金会 CNCF 所支持的项目。它用于监控和排除基于微服务的分布式系统的故障，包括:

- 分布式上下文传播
- 分布式事务监控
- 根因分析
- 服务依赖分析
- 性能/延时优化

**特性：**

1. 兼容 OpenTracing
2. 高扩展性
3. 支持多个存储组件：Cassandra， Elasticsearch，内存...
4. 现代 web 界面
5. 云原生部署
6. 系统拓扑图

**整体架构**

![Evolving Distributed Tracing at Uber Engineering - Uber Engineering Blog](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/Distributed_Tracing_Header-2021-12-20.png)

### Geting Start

1. docker 启动 jaeger collector

```bash
docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.28
```

2. UI 界面：http://localhost:16686
3. 应用程序中使用 jaeger-client 直接上报数据，这里的 demo 例子没有使用 jaeger-agent 

如下有一个简单的 web 应用：具有两个接口 api1 和 api2 

```go
package main

import (
	"net/http"
)

func main() {
	// web 示例
	http.HandleFunc("/api1", http.HandlerFunc(api1))
	http.HandleFunc("/api2", http.HandlerFunc(api2))
	err := http.ListenAndServe(":1234", nil)
	if err != nil {
		panic(err)
	}
}
```

```go
package main

import (
	"fmt"
	"net/http"
)

func api1(w http.ResponseWriter, r *http.Request) {
	fmt.Println("hello api1")
}

func api2(w http.ResponseWriter, r *http.Request) {
	fmt.Println("hello api2")
}
```

使用 jaeger client 监控每一个接口，引用以下两个包

```bash
 go get github.com/uber/jaeger-client-go
go get github.com/opentracing/opentracing-go
```

jaeger client 的初始化，和 简单使用

```Go
package main

import (
	"net/http"

	"github.com/opentracing/opentracing-go"
	"github.com/uber/jaeger-client-go/config"
)

func main() {
  // 初始化 jaeger client，参数需要服务名，采样机制，上报数据地址
	tracing, closer, err := config.Configuration{
		ServiceName: "hello.service",
		Sampler:     &config.SamplerConfig{Type: "const", Param: 1},
		Reporter:    &config.ReporterConfig{CollectorEndpoint: "http://localhost:14268/api/traces"},
	}.NewTracer()
	if err != nil {
		panic(err)
	}
	defer closer.Close()
  // 设置成全局默认
	opentracing.SetGlobalTracer(tracing)
	// web 示例
	http.HandleFunc("/api1", jaegerTracing(http.HandlerFunc(api1)))
	http.HandleFunc("/api2", jaegerTracing(http.HandlerFunc(api2)))
	err = http.ListenAndServe(":1234", nil)
	if err != nil {
		panic(err)
	}
}

// 添加jaeger 分布式追踪中间件
func jaegerTracing(handler http.Handler) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
    // 记录 tracing 中的一个 span 上报
		span := opentracing.StartSpan(r.URL.String())
    // span 结束
		defer span.Finish()
		handler.ServeHTTP(w, r)
	}
}

```

jaeger 的作用是追踪服务间的调用关系，上述使用意义不大，以下是模拟多服务调用使用 jaeger 的作用

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"time"

	"github.com/opentracing/opentracing-go"
)

func api1(w http.ResponseWriter, r *http.Request) {
	// 根 ctx
	ctx := context.Background()
	// 记录 sever1.api 的调动耗时等信息
	span, c := opentracing.StartSpanFromContext(ctx, "server1.api1")
	defer span.Finish()
	fmt.Println("hello api1")
	time.Sleep(time.Second)
	// 模拟一个跨服务的调用，ctx 上下文信息传递
	service2XXX(c)
}

func service2XXX(ctx context.Context) {
	// 从 ctx 获取 span， 这样就能把在一条调用链的 span 聚合在一起
	span, c := opentracing.StartSpanFromContext(ctx, "server2.XXX")
	defer span.Finish()
	time.Sleep(time.Second * 2)
	fmt.Println("hello server2 xxx", c)
}

func api2(w http.ResponseWriter, r *http.Request) {
	fmt.Println("hello api2")
}

```

**效果如下**

![image-20211221004524655](http://ganghuan.oss-cn-shenzhen.aliyuncs.com/img/image-20211221004524655-2021-12-21.png)

[demo 仓库](https://github.com/he2121/demos/tree/master/jaeger)

## 参考

1. https://github.com/jaegertracing/jaeger
2. https://www.jaegertracing.io/docs/1.29/
3. https://opentracing.io/
4. https://github.com/yurishkuro/opentracing-tutorial/tree/master/go
