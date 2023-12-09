---
layout:     post
title:      "云上存储的分布式缓存"
subtitle:   "架构，性能瓶颈分析和优化手段"
date:       2023-12-09 19:00:00
author:     "Hexi"
header-img: "img/bg/2023-12-09-cache-group.jpg"
tags:
    - object-storage
    - cache-group
---

## 云上存储系统的特点

- 规模大：~100T（热存）- ~10P（冷存）
- 吞吐高，延迟高
- 以文件为写入单位（对象存储）


### 磁盘 IO 的特点

- IOPS 主要由硬件本身决定，增加负载线程 IOPS 会降低（HDD）
- 顺序读写性能远好于随机读写
- 适用工作模式：两个单独线程分别负责读写；写入最好只append；读出需要做合并和预读

### 网络 IO 的特点

- 网卡带宽一般很高，IOPS 受应用层协议和CPU资源影响
- 可以充分利用多核来提高 IOPS
- 适用工作模式：TCP 连接数跟系统线程一致，每个连接由两个线程分别负责读写并解包/封包，再把包交给其它线程处理

### zero-copy 的特点

- 数据拷贝全由 DMA 完成，节省 CPU，增加网络 IOPS
- 需要数据盘和消费端无瓶颈
- 适用场景：数据全内存缓存，消费端使用用户态 SDK（非 FUSE）

### Rust 实现 zero-copy

- IO runtime 没有抢占式调度，sendfile/splice 不能在 runtime 线程池内运行
- 对于已经主存中的数据，spwan 到 IO 线程里 splice 反而是负优化
- Page cache 全缓存或内存盘情况下可用，或是纯内存缓存使用 vmsplice
