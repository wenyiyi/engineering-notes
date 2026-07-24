---
type: framework
style: sketch-notes
aspect: "16:9"
---

# Delivery semantics

Create a 16:9 hand-drawn framework with three lanes. At-most-once: Commit then Process, possible loss. At-least-once: Process then Commit, possible duplicate, protected by idempotency key. Kafka transaction lane: Read, Process, Write offsets and output atomically inside Kafka, with a boundary warning that external databases still need coordination or an outbox. Labels: At most once, At least once, Transaction, Loss, Duplicate, Idempotent, External DB boundary. Warm cream paper, navy, teal, coral.

