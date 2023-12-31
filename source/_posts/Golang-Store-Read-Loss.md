---
title: "# 记一次Galang 无法正确读取 map 值的坑"
date: 2023-12-31 20:50:18
tags:
  - store
  - net
  - golang
category: developer
---

## Problem

在异步的网络通信中，为了收到结果，通常在发送时会记录到 map, 收到 ACK 后会读取发送内容并且删除。示例代码：

```go
var store sync.Map

func Send(key string, value any) {
	// send to other nodes, net request
	// if send ok, record to map
	result := client.send("key", "value")
	if result {
		store.Store(key, value)
	}
}

// Mock ack
func Ack() {
	// receive ack
	v, ok := store.LoadAndDelete(key)
	if !ok {
		panic("not found value")
	}
	fmt.Printf("%#v", v)
}
```

上述示例有一个坑，没有考虑到 GC 问题，在压测过程中会出现部分 panic，因为当量非常大的时候遇到GC。
在内网中可能会出现通信时间比GC时间更短的问题，假如Code走到client.send方法时，因为网络请求已经发送，如果此时遇到 GC，GC后执行Ack方法，将导致未找到指定key的result，进而出现panic

## Fixed
```go
var store sync.Map

func Send(key string, value any) {
	// send to other nodes, net request
	// if send ok, record to map
	store.Store(key, value)
	result := client.send("key", "value")
	if !result {
		store.Delete(key)
	}
}

// Mock ack
func Ack() {
	// receive ack
	v, ok := store.LoadAndDelete(key)
	if !ok {
		panic("not found value")
	}
	fmt.Printf("%#v", v)
}
```

所有的网络请求，在发送前必须优先存储，如果请求失败进行删除，这样就可以避免此类问题。