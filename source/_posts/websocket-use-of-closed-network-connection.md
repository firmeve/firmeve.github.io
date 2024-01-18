---
title: websocket use of closed network connection
date: 2024-01-18 20:05:06
tags:
  - websocket
---
最近在 dev 环境遇到大量的 "use of closed network connection" 错误，最开始分析是异常连接断开，抓包分析，后来才发现是触发了异常 case 每分钟 close client。

问题解决后，重新查看 code，也有可以优化的地方。
在我们在写入时使用一个 channel 保证并发问题，在读取的时候也是，出现这个错误的原因是channel中包含值，但是没有处理完成优先 close 了 socket 导致的，应该先断开 channel,再 close socket,这样才是安全的操作。
一个伪代码如下：

```go

func socketHandle(socket Socket) {
	defer socket.Close()
	for {
		select {
			case <-ctx.Done():
			return
			case writeChan <- bytes:
				// TODO: something
			default:
				msg, err := socket.Read()
				// TODO: something
		}
	}
}

```

