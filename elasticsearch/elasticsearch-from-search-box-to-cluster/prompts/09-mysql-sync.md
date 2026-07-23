# Reliable MySQL synchronization

Create a 16:9 hand-drawn data pipeline. MySQL transaction writes business data and an outbox event together. CDC/binlog carries ordered events through a durable queue to an idempotent Elasticsearch indexer. Add retry, dead-letter queue and a reconciliation worker comparing MySQL with Elasticsearch. Show dual-write without an outbox as a cracked bridge marked “Risk”. Minimal labels: MySQL, Outbox, CDC, Queue, Idempotent, Retry, DLQ, Reconcile. Warm paper, navy, teal, coral, yellow.

