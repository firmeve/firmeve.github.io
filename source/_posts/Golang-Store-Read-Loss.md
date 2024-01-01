---
title: "# 记一次Golang 无法正确读取 map 值的坑"
date: 2023-12-31 20:50:18
tags:
  - store
  - net
  - golang
category: developer
---

## Problem

在异步的网络通信中，为了保存处理时的上下文，通常在发送时会记录到 map, 收到 ACK 后会读取上下文进行处理，最后会删除。示例代码：

```go
var store sync.Map
go func(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
		return
		// Mock ack get response ack
		case key := <-respAck:
		// Do something...
			v, ok := store.LoadAndDelete(key)
			if !ok {
				panic("not found value")
			}
			fmt.Printf("%#v", v)
		}
	}
}
func Send(key string, value any) {
	// send to other nodes, net request
	// if send ok, record to map
	result := client.send(key, value)
	if result {
		store.Store(key, value)
	}
}

```

上述示例有一个坑，没有考虑到 GC 问题，在压测过程中会出现部分 panic，特别是当数据量非常大的时候，会更频繁的出现 panic。
在内网中可能会出现通信时间比 GC 时间更短的问题，假如 code 走到 `client.send` 方法时，因为网络请求已经发送，如果此时遇到 GC，GC 后 goroutine 可能会立刻执行并返回ack结果，而此时store因为gc并没有写入result, 将导致未找到指定 key 的 result，进而出现panic

## Fixed

所有的网络请求，在发送前必须优先存储，如果请求失败进行删除，这样就可以避免此类问题。

```go
var store sync.Map
go func(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
		return
		// Mock ack get response ack
		case key := <-respAck:
		// Do something...
			v, ok := store.LoadAndDelete(key)
			if !ok {
				panic("not found value")
			}
			fmt.Printf("%#v", v)
		}
	}
}

func Send(key string, value any) {
	// send to other nodes, net request
	// if send ok, record to map
	store.Store(key, value)
	result := client.send(key, value)
	if ！result {
		store.Delete(key, value)
	}
}
```

