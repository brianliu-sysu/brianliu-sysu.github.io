---
title: 可观测性
date: 2026-08-27
draft: false
description: Logging / Metrics / Tracing，以及一个最小 Tracing 实现
categories:
  - 技术教程
tags:
  - arch
---

<!--more-->

# 可观测性

可观测性衡量的是：服务上线之后，出了问题能不能被看见、被定位、被复盘。监控告诉你「机器还活着」，可观测性要回答的是「这次请求为什么慢、错在哪一段」。

三大支柱：

- **Logging**：运行过程中重要事件的离散记录，回答「发生了什么」。
- **Metrics**：定时采集的时间序列，回答「现在是什么趋势」。
- **Tracing**：一次请求穿过的完整调用链，回答「慢在哪一段」。

三柱要能对上同一件事：Metrics 发现异常，Tracing 把请求拆到具体 span，Logging 给出那个点上的上下文。对不齐的话，三套系统只是三份账单。

## Logging

日志适合记录不可聚合的事实：错误堆栈、关键状态跳转、一次请求的入参摘要。

- 用结构化字段（`key=value` 或 JSON），不要把全部信息塞进一句自然语言。
- 每条日志带上 `request_id` / `trace_id`，才能从一条错误跳到整条链路。
- 级别要分清：`error` 可告警，`info` 可复盘，`debug` 默认不要在生产全量开。
- 日志基数高、存储贵，不要用它替代 Metrics。`user_id`、完整请求体这类字段尤其要克制。

## Metrics

指标适合看趋势和做告警：低基数标签 + 可加总的数值。

线上最常用的几类：

- **流量**：QPS、并发
- **错误**：错误率、按错误码分类的计数
- **延迟**：P99 / P999，不要只看平均值
- **饱和度**：队列长度、连接池、goroutine、CPU / 内存

标签基数必须受控。`interface`、`code`、`instance` 可以；`user_id`、`url` 原文会把时序库打爆。

Metrics 能告诉你「支付接口 P99 从 80ms 变成 800ms」，但说不清是 DB、Redis 还是下游 RPC。这就是 Tracing 的位置。

## Tracing

一次请求会穿过多层函数、多个下游。Trace 把这次请求收成一棵 span 树：

- 同一个 `TraceID` 标识整次请求
- 每个 `Span` 有自己的 `SpanID`、耗时和名称
- `ParentSpanID` 把 span 连成树
- 当前 span 放进 `context`，往下游传，子调用才能挂到正确的父节点上

下面是一个只覆盖进程内传播的最小实现：不采样、不导出，`End()` 时直接打印。

### `trace.go`

```go
// trace.go
package trace

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"time"
)

type TraceID [16]byte
type SpanID [8]byte

type Span struct {
	TraceID      TraceID
	SpanID       SpanID
	ParentSpanID SpanID

	Name string

	StartTime time.Time
	EndTime   time.Time
}

type spanKey struct{}

func newTraceID() TraceID {
	var id TraceID
	_, _ = rand.Read(id[:])
	return id
}

func newSpanID() SpanID {
	var id SpanID
	_, _ = rand.Read(id[:])
	return id
}

func (id TraceID) String() string {
	return hex.EncodeToString(id[:])
}

func (id SpanID) String() string {
	return hex.EncodeToString(id[:])
}

func spanFromContext(ctx context.Context) (*Span, bool) {
	span, ok := ctx.Value(spanKey{}).(*Span)
	return span, ok
}

func Start(ctx context.Context, name string) (context.Context, *Span) {
	var traceID TraceID
	var spanID SpanID

	if span, ok := spanFromContext(ctx); ok {
		traceID = span.TraceID
		spanID = span.SpanID
	} else {
		traceID = newTraceID()
	}

	span := &Span{
		TraceID:      traceID,
		SpanID:       newSpanID(),
		ParentSpanID: spanID,

		Name: name,

		StartTime: time.Now(),
	}

	ctx = context.WithValue(ctx, spanKey{}, span)
	return ctx, span
}

func (s *Span) End() {
	s.EndTime = time.Now()

	fmt.Printf(
		"trace_id=%s span_id=%s parent_span_id=%s name=%q duration=%s\n",
		s.TraceID.String(),
		s.SpanID.String(),
		s.ParentSpanID.String(),
		s.Name,
		s.EndTime.Sub(s.StartTime),
	)
}
```

`Start` 的规则很简单：context 里已有 span，就继承它的 `TraceID`，并把它的 `SpanID` 当作父节点；没有则新开一条 trace。根 span 的 `ParentSpanID` 是零值。

### `main.go`

```go
// main.go
package main

import (
	"context"
	"time"
	"your-path/trace"
)

func main() {
	ctx := context.Background()

	ctx, span := trace.Start(ctx, "main")
	defer span.End()

	queryDB(ctx)
}

func queryDB(ctx context.Context) {
	ctx, span := trace.Start(ctx, "queryDB")
	defer span.End()

	time.Sleep(100 * time.Millisecond)

	callRedis(ctx)
}

func callRedis(ctx context.Context) {
	_, span := trace.Start(ctx, "redis.GET")
	defer span.End()

	time.Sleep(50 * time.Millisecond)
}
```

调用关系是 `main → queryDB → redis.GET`。`queryDB` 和 `callRedis` 都必须把传入的 `ctx` 交给 `Start`，否则子 span 会变成一条新的、互不相干的 trace。

### 输出示例

```text
trace_id=f13ab1e98bfe04d070ba0c2c6c21d082 span_id=ac2457ed96cda12a parent_span_id=6390a0de25c1605f name="redis.GET" duration=50.382959ms
trace_id=f13ab1e98bfe04d070ba0c2c6c21d082 span_id=6390a0de25c1605f parent_span_id=a0acf62459855909 name="queryDB" duration=150.616125ms
trace_id=f13ab1e98bfe04d070ba0c2c6c21d082 span_id=a0acf62459855909 parent_span_id=0000000000000000 name="main" duration=150.621167ms
```

几点可以直接从输出读出来：

- 三条记录同一个 `trace_id`，说明还在一次请求里。
- `parent_span_id` 串起来就是 `main → queryDB → redis.GET`。根 span 的 parent 是全 0。
- 打印顺序是子 span 先结束：`defer` 后进先出。
- `queryDB` 的 duration ≈ 150ms，等于自己的 100ms 加上 Redis 的 50ms。父 span 的耗时包含子调用。

## 生产环境中的 Tracing

上面的实现只在本进程 `printf`。生产环境至少还要补三块，也就是 SDK 里常见的三层：

- **Sampler（采样器）**：决定这条 trace 记不记。全量采集成本很高，通常按比例或按错误/慢请求强制留下。采样必须在根 span 上决定，并随 context 传下去，否则同一条请求会缺段。
- **Exporter（导出器）**：用什么协议发出去（OTLP / gRPC / HTTP），以及一条条发还是批量发。批量能省网络，进程退出前要能 flush。
- **Connector（连接器）**：接到哪一个收集器。本机 agent、集群 collector、还是厂商入口，决定的是导出的目的地，不是 span 模型本身。

还有一件最小实现故意没做的事：跨服务传播。出进程时要把 `trace_id` / `span_id` 放进 RPC / HTTP header（例如 W3C `traceparent`），对端再解出来继续 `Start`。否则链路会在服务边界断开，Metrics 看到的慢接口和 Tracing 对不上。
