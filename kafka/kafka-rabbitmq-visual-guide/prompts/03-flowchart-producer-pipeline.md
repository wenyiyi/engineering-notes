---
type: flowchart
style: sketch-notes
aspect: "16:9"
---

# Producer pipeline

Create a 16:9 hand-drawn horizontal flowchart. A Java producer record passes through Interceptor, Serializer and Partitioner, then joins a per-partition Batch in a buffer. A Sender ships the batch to the partition leader and receives an Ack or Error callback. Show batch.size and linger.ms as two small triggers where either can release a batch. Minimal labels: Record, Serialize, Partition, Batch, Sender, Broker, Callback. Warm cream paper, navy linework, teal and coral accents.

