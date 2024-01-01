---
title: ETCD mvcc database space exceeded
date: 2023-01-01 20:50:18
tags:
  - etcd
  - database
category: developer
---

## Background

在一次的压测试中，打满了 ETCD 的磁盘，etcd client 收到写入错误： `mvcc database space exceeded` 开始恢复 etcd ，因为当前项目仅仅是是使用 etcd 作通信，数据并不重要，所以直接压缩所有的 revision

## 恢复步骤
1. 获取最近的 revision
```shell
/usr/local/etcd/etcdctl --user=$user --password=$password --endpoints=$host endpoint status
```
   
2. 压缩 etcd 的 revision
```shell
/usr/local/etcd/etcdctl --user=$user --password=$password --endpoints=$host compact $revision
```

3. 整理 etcd 碎片
```shell
/usr/local/etcd/etcdctl --user=$user --password=$password --endpoints=$host defrag
```
   
4. 解除 Alarm 告警
   > **这一步非常重要，没有这一步即使磁盘占用下降，client 仍然会无法工作，因为告警未解除**
```shell
/usr/local/etcd/etcdctl --user=$user --password=$password --endpoints=$host alarm disarm
```


## 恢复中的问题
在磁盘占用100%时，其实访问 etcd server api 经常会出现 context.timeout 的超时错误，选择一台节点，强制重启时，发现重启也超时，确实非常的慢。所以需要修改 linux service 的启动超时时间。
```
TimeoutSec=0
```

再次重启 etcd，重启过程非常耗时，但耐心等待就好了
```shell
service restart etcd
```
重启后只调用当前重启节点的 server 超时问题也会解决

## 优化 etcd 参数
etcd 配额的大小和 compaction 的速度需要按 QPS 来评估

- `auto-compaction-retention` 
 压缩时间设置，默认可以使用时间单位，如果未设置时间单位，默认为小时，0表示禁用压缩
- `quota-backend-bytes`
 存储的大小 

