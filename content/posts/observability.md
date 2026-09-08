---
title: "可观测性"
description: "Logging / Metrics / Tracing，以及一个最小 Tracing 实现"
date: 2026-08-27
draft: false

categories: ["技术教程"]
tags: []
---

# 可观测性

可观测性是衡量软件部署到生产环境后能否被有效运维，以及能否快速发现问题的一个重要指标。

可观测性主要包含三大支柱：

- **Logging**：记录软件运行过程中重要事件的离散数据。
- **Metrics**：定时采集的时间序列数据，用于反映系统状态和变化趋势。
- **Tracing**：追踪一次请求所经过的完整流程。

## Tracing 的简单实现

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

### 输出示例

```text
trace_id=f13ab1e98bfe04d070ba0c2c6c21d082 span_id=ac2457ed96cda12a parent_span_id=6390a0de25c1605f name="redis.GET" duration=50.382959ms
trace_id=f13ab1e98bfe04d070ba0c2c6c21d082 span_id=6390a0de25c1605f parent_span_id=a0acf62459855909 name="queryDB" duration=150.616125ms
trace_id=f13ab1e98bfe04d070ba0c2c6c21d082 span_id=a0acf62459855909 parent_span_id=0000000000000000 name="main" duration=150.621167ms
```

## 生产环境中的 Tracing

上面只是一个简单的本地实现。生产环境中的 Tracing 通常还应包含：

- **Connector（连接器）**：用于连接远程收集器。
- **Exporter（导出器）**：定义导出协议（如 OTLP、gRPC 等），以及采用单条发送还是批量发送。
- **Sampler（采样器）**：定义采样频率。
