---
tags: [moc, 软件设计, Kafka]
created: 2026-05-15
publish: true
---

# Kafka 专题

> 掌握高吞吐消息队列的核心原理与生产实践

## 核心知识点

### 基础架构
- Producer / Consumer / Broker / ZooKeeper
- Topic / Partition / Offset 模型

### 消息可靠性
- ACK 机制（0 / 1 / all）
- 幂等性与事务消息
- 消费者 offset 管理

### 性能优化
- 顺序写磁盘 + Page Cache
- 零拷贝（sendfile）
- 批量压缩（Snappy / LZ4）

### 高可用
- 副本机制（ISR / OSR）
- Leader 选举
- 分区再均衡

---

## 练习题（待补充）

内容正在整理中，敬请期待。
