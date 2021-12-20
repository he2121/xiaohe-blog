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



## 参考

1. https://github.com/jaegertracing/jaeger
2. https://www.jaegertracing.io/docs/1.29/
3. https://opentracing.io/
4. https://github.com/yurishkuro/opentracing-tutorial/tree/master/go
