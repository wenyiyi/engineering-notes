# Shards and routing

Create a 16:9 hand-drawn distributed-search diagram. A client sends a request to any node, which becomes a coordinating node. For writes, a hash-routing sign directs a document to one primary shard. For search, the coordinator fans out to several shard copies, receives small ranked result piles, then merges them into one final top-results list. Clearly separate write routing and search fan-out with two lanes. Minimal labels: Client, Coordinator, Route, Shards, Top Results. Warm paper, navy/teal/coral palette.

