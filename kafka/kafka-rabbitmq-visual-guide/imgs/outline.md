---
type: mixed
density: rich
style: sketch-notes
palette: warm-paper-navy-teal-coral
image_count: 13
---

# Illustration outline

1. Cover — messages flowing through a partitioned event highway.
2. Kafka mental model — durable shared event log, not a disappearing mailbox.
3. Topic, partitions, replicas, offsets.
4. Producer pipeline — serialize, partition, batch, send, acknowledge.
5. `acks`, ISR and idempotent producer reliability.
6. Key-based ordering and the scope of ordering guarantees.
7. Consumer groups and partition ownership.
8. Offset commit as a bookmark pointing to the next record.
9. Consumer rebalance: classic stop-the-world versus incremental protocol.
10. End-to-end delivery semantics and business idempotency.
11. Backlog incident response timeline and bottleneck checks.
12. RabbitMQ routing model: exchange, binding, queue, ack, DLX.
13. Kafka versus RabbitMQ decision map.

