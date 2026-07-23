# Near-real-time write path

Create a 16:9 hand-drawn engineering diagram for near-real-time search. Documents enter an in-memory buffer, a refresh opens a small searchable Lucene segment, later background merges combine several small segments into one larger segment. Show a clock around one second to communicate near-real-time, not instant consistency. A separate durability trail may suggest translog without overwhelming the main story. Minimal labels: Index, Buffer, Refresh, Segment, Merge, Searchable. Warm off-white paper, navy, teal, coral, yellow.

