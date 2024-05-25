---
title: Some think about distributed data sync
date: 2024-05-25 15:50:18
tags:
  - etcd
  - distributed
  - data sync
category: developer
---

## Background

1. Keep observes idempotent
2. Use etcd election leader
3. Use etcd sync data to every node

## Crucial point

1. Every node is concurrent
2. Only the leader can write data, and the write operation must support transactions.
3. Other nodes receive data and trigger app memory
4. The initial leader and memory initial version is zero, clearing all stored data
5. If old leader released, next new leader must wait old release completed
6. Follower starting need to bring version to ask leader if syncing data is necessary.
   If syncing is not needed, the follower can start using local storage
   Otherwise it need to wait for the leader to sync all data before starting.