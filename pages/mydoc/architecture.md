---
title: 海量分布式系统设计基本思想
sidebar: mydoc_sidebar
permalink: architecture.html
folder: mydoc
---

虽然抛开业务和成本谈具体的架构设计是无意义的。但是架构设计的一些关键通用方法沉淀，能够在架构设计时提醒自己，以增加考虑全面的概率

- 容灾
    - 冗余存储
        - 实时同步磁盘：数据无丢失、吞吐量低
        - 非实时同步磁盘：数据可能丢失、吞吐量高
- 链路高吞吐、低延迟
    - 少同步
        - 业务数据分层：区分同步数据和非同步数据
    - 缓存
        - 单写系统：查询可提前（多写系统：容忍数据时效问题时，查询可提前）
        - 数据聚合，降低下游压力
    - 异步
        - 可丢失数据，异步提高速度
    - 负载均衡
        - 以此保障每台机器的资源利用率
- 运维办法
    - 限流（入口流量调整100%-0%）：减少总体输入
    - 扩容：减少单机输入
    - 降低计算精度：减少单机计算量