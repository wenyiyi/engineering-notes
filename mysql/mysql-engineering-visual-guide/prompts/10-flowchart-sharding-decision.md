---
illustration_id: 10
type: flowchart
style: sketch-notes
palette: cool-macaron
---

When to Shard MySQL — decision flowchart.
STEPS: Measure bottleneck → fix query/index/schema → scale resources/read replicas/partition or archive → confirm single-node limit → choose stable shard key → plan migration and operations.
DECISION: “Still exceeds one node?” No → stop; Yes → shard.
SIDE COSTS: cross-shard transactions, joins, pagination, global IDs, rebalancing, hot shards.
LABELS: “Measure”, “Optimize”, “Scale”, “Shard Key”, “Route”, “Rebalance”.
COLORS: blue measurement, mint simpler fixes, lavender shard routing, coral distributed-system costs.
STYLE: hand-drawn engineering decision tree with small database-cylinder icons and clear arrows.
Clean composition with generous white space. No logo, watermark, or fixed row-count threshold.
ASPECT: 16:9.
