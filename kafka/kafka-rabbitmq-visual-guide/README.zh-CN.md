# Kafka 讲人话：从消息发送到 Kafka 与 RabbitMQ 选型

> 消息队列就像公司群聊：发消息容易，难的是保证该看的人看见、别看两遍、顺序别乱，服务器挂了还能接着聊。

[English version](README.md)

![Kafka, Made Simple](imgs/cover.png)

这篇文章不背配置表。我们顺着一条消息走一遍：它怎么进入 Kafka、为什么会进入某个分区、消费者如何接手、什么时候会重复，以及 Kafka 和 RabbitMQ 到底怎么选。

---

## 1. Kafka 到底是什么？

Kafka 经常被叫作消息队列，但把它只想成“发完就取走的箱子”会漏掉一半本领。

它更像一本多人共享、只往后追加的事件账本：

- Producer 往账本末尾追加事件；
- Kafka 按保留策略保存事件；
- Consumer Group 用自己的书签记录读到哪里；
- 不同 Group 可以独立读取同一批事件；
- 只要数据仍在保留范围内，就可以把书签往回移并重放。

![Kafka 心智模型](imgs/01-framework-kafka-mental-model.png)

所以 Kafka 特别适合：

- 业务系统解耦；
- 流量削峰；
- 事件驱动架构；
- 日志和变更数据采集；
- 需要重放历史事件的流处理。

### 闲狗言

Kafka 不会自动让系统可靠。它只是把“谁在什么时候处理了什么”这件事变得更可控。Producer、Broker、Consumer 和业务数据库，每一段仍然都要设计失败处理。

---

## 2. Topic、Partition、Replica 和 Offset

![Topic、Partition 与副本](imgs/02-framework-topic-partitions.png)

先把几个词翻译成人话：

- **Topic**：一类事件的逻辑名字，例如 `order-created`。
- **Partition**：Topic 的并行车道，每条消息只进入其中一个分区。
- **Offset**：消息在某个分区里的位置编号。
- **Leader replica**：负责这个分区读写的副本。
- **Follower replica**：复制 Leader 数据，用来容灾。
- **Broker**：运行 Kafka、保存分区副本的服务器进程。

一个 Topic 可以有多个 Partition，一个 Partition 可以有多个副本，但同一条消息不是“业务上被存进多个分区”，而是它所在分区的日志被复制到多个 Broker。

### Kafka 4.x 还需要 ZooKeeper 吗？

不需要。Apache Kafka 4.0 起全面运行在 KRaft 模式，不再依赖 ZooKeeper；Controller 使用 Kafka 自己的元数据仲裁机制。旧教程里的 ZooKeeper 安装步骤已经不适用于 Kafka 4.x。[Apache Kafka 4.0 发布说明](https://kafka.apache.org/blog/2025/03/18/apache-kafka-4.0.0-release-announcement/)。

---

## 3. Producer 发一条消息，内部忙了什么？

![Producer 发送链路](imgs/03-flowchart-producer-pipeline.png)

简化过程：

1. 创建 `ProducerRecord`；
2. Interceptor 做审计或补充信息；
3. Serializer 把 Key 和 Value 变成字节；
4. Partitioner 决定目标分区；
5. 相同分区的消息进入 Batch；
6. Sender 线程批量发送给 Leader；
7. Broker 返回成功或错误；
8. Future 或 Callback 通知业务代码。

`batch.size` 和 `linger.ms` 不是“两个条件都满足才发送”。Batch 满了可以发，等待时间到了也可以发；吞吐和延迟在这里做交换。

```java
producer.send(
    new ProducerRecord<>("order-created", orderId, event),
    (metadata, error) -> {
        if (error != null) {
            // 记录失败并按业务策略处理
            return;
        }
        // metadata 包含 topic、partition、offset
    }
);
```

不要只调用 `send()` 然后假装世界永远风平浪静。异步发送仍然需要检查最终结果。

---

## 4. `acks`、ISR 和幂等生产者

![Kafka 写入可靠性](imgs/04-infographic-acks-idempotence.png)

### `acks=0`

Producer 发完就走，不等 Broker 回答。快，但连 Broker 是否收到都不知道。

### `acks=1`

Leader 写入本地日志后确认。如果 Leader 刚确认就故障，而 Follower 还没复制，消息可能丢失。

### `acks=all`

Leader 等待当前 ISR 中满足要求的副本确认。通常还要配合：

```properties
acks=all
min.insync.replicas=2
```

这样当可同步副本太少时，Kafka 会拒绝写入，而不是假装安全。

### 旧文章里最需要更新的一点

现代 Kafka Producer 在没有冲突配置时，`enable.idempotence` 默认开启，`retries` 默认也不再是 0。幂等生产者通过 Producer ID 和序列号，避免一次 Producer 会话内因自动重试而写入重复记录。

开启幂等后，满足 `acks=all`、`retries>0`、`max.in.flight.requests.per.connection<=5` 时，允许多个未确认请求仍能保持分区内顺序。不要再机械地说“为了顺序必须把 max.in.flight 设成 1”。[Kafka 4.2 Producer 配置](https://kafka.apache.org/42/generated/producer_config.html)。

不过，应用代码自己重复调用两次 `send()`，Kafka 不知道这两次业务请求其实是同一件事。业务幂等仍然不能省。

---

## 5. Kafka 能保证顺序吗？

能，但要先把范围说完整：**Kafka 保证的是分区内顺序，不保证跨分区的全局顺序。**

![Key 与分区内顺序](imgs/05-infographic-key-ordering.png)

```java
new ProducerRecord<>("user-follow-events", userId, event)
```

使用稳定的 `userId` 作为 Key，同一个用户的关注、取关事件会被哈希到同一个分区，于是可以按写入顺序处理。

这比把 Topic 设成一个分区更实用：

- 同一用户保持顺序；
- 不同用户可以并行处理；
- 不必为了全局顺序牺牲整个 Topic 的吞吐。

### 一个隐藏的坑

增加分区数后，`hash(key) % partitionCount` 的结果可能变化。同一个 Key 的新事件可能进入新分区，与旧分区里的历史事件形成并行处理。

所以扩分区不是“点一下就无风险提速”。顺序敏感的 Topic 要提前规划分区，或设计迁移窗口和新的路由方案。

---

## 6. Consumer Group：不是消费者越多越快

![Consumer Group](imgs/06-framework-consumer-groups.png)

规则很简单：

- 同一 Group 内，一个 Partition 同时只分配给一个 Consumer；
- 一个 Consumer 可以负责多个 Partition；
- Consumer 数量超过 Partition 数量，多出来的 Consumer 会闲置；
- 不同 Group 可以分别消费同一个 Topic，各自维护 Offset。

假设 Topic 有 4 个分区：

```text
2 个 Consumer：每人约 2 个分区
4 个 Consumer：每人约 1 个分区
8 个 Consumer：约 4 个在工作，4 个发呆
```

因此“积压了，先扩 20 台消费者”不一定有效。先看 Partition 数，再看真正的瓶颈是不是数据库、远程接口、CPU 或锁竞争。

---

## 7. Offset：书签指向下一条，不是上一条

![Offset 书签](imgs/07-infographic-offset-bookmark.png)

假设已经处理完 Offset 12，应该提交的通常是：

```text
13
```

因为提交位移表示“应用下一次从哪里继续”，而不是“最后处理了哪一条”。Kafka 4.2 的 Consumer API 也明确说明：Committed Offset 应该指向下一条要处理的记录。[KafkaConsumer 官方文档](https://kafka.apache.org/42/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html)。

### 先提交，再处理

进程在两者之间崩溃，重启会从后面继续，业务处理可能丢失。

### 先处理，再提交

进程处理成功但提交前崩溃，重启会再次收到记录，产生重复。

工程中常用第二种，也就是 At-least-once，再用幂等处理重复：

```sql
INSERT INTO consumed_event(event_id, consumed_at)
VALUES (?, NOW());
```

给 `event_id` 加唯一索引，并把“记录已消费”和业务更新放进同一个本地事务。不要先查再写，否则并发下仍可能一起通过检查。

---

## 8. Rebalance 没那么神秘，也不能乱甩锅

![Rebalance：Classic 与 Incremental](imgs/08-comparison-rebalance.png)

Consumer Group 需要重新分配 Partition 时会发生 Rebalance，常见原因包括：

- Consumer 加入或离开；
- Topic 分区数变化；
- 正则订阅匹配到新的 Topic；
- Consumer 太久没有 Poll；
- Classic 协议下心跳超时。

但是 Broker 故障、Partition Leader 切换，本身不等于 Consumer Group 一定重新分配 Partition。Leader 变化主要由 Kafka 元数据和副本机制处理，旧文章把两类事件混在了一起。

### Kafka 4.x 的变化

Kafka 4.0 起，新 Consumer Rebalance Protocol 已正式可用。它采用完全增量的设计，不再依赖全组同步屏障，能降低大规模消费组的 Rebalance 时间。[官方说明](https://kafka.apache.org/42/operations/consumer-rebalance-protocol/)。

可通过客户端配置逐步采用：

```properties
group.protocol=consumer
```

它不是让 Rebalance 消失，而是减少“所有人一起停下来重新排座位”的范围。

### 减少无意义 Rebalance

- 不要在 Poll 线程执行不可控的超长任务；
- 合理配置 `max.poll.interval.ms`；
- 控制 `max.poll.records`，让一批消息能按时处理完；
- 监控 Full GC、网络停顿和处理耗时；
- 使用静态成员关系时，认真规划 `group.instance.id`；
- 不要照抄固定的 6 秒、2 秒参数，先看网络与故障检测目标。

---

## 9. 不丢、不重、Exactly-once 到底怎么理解？

![消息交付语义](imgs/09-framework-delivery-semantics.png)

### At-most-once

先提交 Offset，再处理。最多一次，但可能没处理。

### At-least-once

先处理，再提交。至少一次，但可能重复。多数业务系统选择它，再用业务幂等兜底。

### Kafka Transaction

Kafka 可以把消费输入、生产输出以及 Offset 提交放进一个事务，让 Kafka 内部的 read-process-write 链路具备 Exactly-once 语义。

但如果处理中还要写 MySQL：

```text
Kafka Transaction ≠ 自动包含 MySQL Transaction
```

跨系统仍需 Transactional Outbox、CDC、幂等键、状态机或补偿对账。不要看到 “Exactly-once” 就以为任何外部副作用都只会发生一次。

---

## 10. 消息积压：先找瓶颈，别急着堆机器

![消息积压事故时间线](imgs/10-timeline-backlog-incident.png)

原文中的促销案例很有代表性：

```text
20:00  活动开始
20:05  Consumer Lag 快速增长
20:11  分区从 4 扩到 10
20:15  MySQL CPU 升到 90%
20:20  积压恢复
```

问题是：Kafka 消费变快后，压力转移给了 MySQL。它不是消失了，只是换了一个报警群。

### 正确的排查顺序

1. Producer 速率是否异常？
2. Consumer 是否报错、GC 或频繁 Rebalance？
3. 单条消息耗时变慢在哪里？
4. 分区是否限制并行度？
5. 数据库、缓存、第三方接口能否承受扩容后的流量？
6. 消息保留时间是否足够撑到恢复？
7. Lag 是持续增长，还是正在下降？

### 扩容消费者前

先设置下游保护：

- 数据库连接池上限；
- 并发信号量；
- 批量写入；
- 限流与退避；
- DLQ 或 Parking Lot Queue；
- 降级非关键处理。

恢复后还要对账，确认“Lag 归零”不等于“业务结果完整”。

---

## 11. RabbitMQ 的思路为什么不一样？

RabbitMQ 更像一个聪明的快递分拣中心：

1. Publisher 把消息交给 Exchange；
2. Exchange 根据类型、Routing Key 和 Binding 路由；
3. 消息进入一个或多个 Queue；
4. Broker 主动向 Consumer 投递；
5. Consumer 通过 Ack/Nack 表达处理结果；
6. 过期或失败消息可以进入 DLX/DLQ。

![RabbitMQ 路由模型](imgs/11-framework-rabbitmq-routing.png)

RabbitMQ 的优势经常体现在“消息该去哪里”：

- Direct Exchange：精确路由；
- Topic Exchange：通配符路由；
- Fanout Exchange：广播到所有绑定队列；
- Headers Exchange：按消息头路由；
- TTL：消息或队列过期；
- DLX：失败、拒绝或过期后的转移；
- Prefetch：限制每个 Consumer 尚未 Ack 的消息数；
- Priority：特定 Queue 类型支持优先级。

需要高可用复制时，RabbitMQ 官方建议优先考虑 **Quorum Queue**。Publisher Confirm 在消息被法定多数副本接受后返回；Consumer 应使用手动 Ack。[RabbitMQ Quorum Queue 文档](https://www.rabbitmq.com/docs/quorum-queues)。

---

## 12. Kafka 和 RabbitMQ 怎么选？

![Kafka 与 RabbitMQ 选型](imgs/12-comparison-kafka-rabbitmq.png)

| 维度 | Kafka | RabbitMQ |
|---|---|---|
| 核心模型 | 分区化、可保留的事件日志 | Exchange 路由到 Queue |
| 消费方式 | Consumer 主动 Pull | Broker 向 Consumer Push，Prefetch 控流 |
| 消费后消息 | 按保留策略继续存在，可重放 | Ack 后通常从 Queue 删除 |
| 扩展单位 | Partition | Queue、Consumer，也有 Stream/Super Stream |
| 顺序 | 分区内有序 | 单 Queue 可有序，但多 Consumer、重投和优先级会影响观察顺序 |
| 路由 | Topic + Key + Partition | Exchange、Binding、Routing Key 很灵活 |
| 历史重放 | 原生强项 | 普通 Queue 不擅长；RabbitMQ Streams 支持重放 |
| 消费进度 | Group Offset | Ack 状态；Streams 使用 Offset |
| 失败处理 | 通常由应用设计重试 Topic、DLQ | Nack、Requeue、TTL、DLX 更直接 |
| 常见场景 | 事件总线、CDC、日志、流处理、大规模重放 | 任务队列、命令消息、复杂路由、请求工作流 |

### 选 Kafka

- 事件需要保留并反复重放；
- 多个系统要按各自进度读取同一事件流；
- 吞吐高、积压可能很长；
- 需要 Kafka Streams、Flink 等流处理；
- 可以接受 Partition 带来的顺序和并行模型。

### 选 RabbitMQ

- 消息更像“请执行一次任务”；
- 需要灵活路由、TTL、DLX、优先级；
- 希望按 Queue 管理消费和失败处理；
- 业务拓扑比历史重放更重要；
- 需要成熟的 AMQP 工作队列模型。

### 别忽略 RabbitMQ Streams

RabbitMQ Streams 也是持久、复制、可重放的追加日志，并支持 Super Stream 分区，因此与 Kafka 的能力存在重叠。[RabbitMQ Streams 文档](https://www.rabbitmq.com/docs/streams)。

选型不能只看产品名，要看使用的具体数据结构：

```text
Kafka Partition
RabbitMQ Classic Queue
RabbitMQ Quorum Queue
RabbitMQ Stream / Super Stream
```

它们解决的并不是同一个问题。

---

## 13. 最后背一张小抄

```text
Topic：事件分类
Partition：并行与顺序边界
Offset：分区里的位置
Replica：分区日志副本
Consumer Group：共享一组分区

可靠写入：acks=all + 合理的 min ISR + 幂等生产者
可靠消费：处理成功后提交 + 业务幂等
严格顺序：相同 Key 进入相同 Partition
积压处理：先找瓶颈，再扩容，并保护下游

Kafka：保留、重放、事件流
RabbitMQ：路由、任务、Ack、TTL、DLX
```

如果只记一句：

> **Kafka 不是一个大号 List，RabbitMQ 也不是一个低配 Kafka；先确定消息是“历史事件”还是“待办任务”，选型会简单很多。**

---

## 参考资料

- [Apache Kafka 4.2 Producer 配置](https://kafka.apache.org/42/generated/producer_config.html)
- [Apache Kafka 4.2 KafkaConsumer](https://kafka.apache.org/42/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html)
- [Apache Kafka Consumer Rebalance Protocol](https://kafka.apache.org/42/operations/consumer-rebalance-protocol/)
- [Apache Kafka KRaft](https://kafka.apache.org/42/operations/kraft/)
- [RabbitMQ Exchanges](https://www.rabbitmq.com/docs/exchanges)
- [RabbitMQ Reliability Guide](https://www.rabbitmq.com/docs/reliability)
- [RabbitMQ Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues)
- [RabbitMQ Streams and Super Streams](https://www.rabbitmq.com/docs/streams)
- [原始文章：Kafka 详解](https://xiangou.blog.csdn.net/article/details/128408727)

