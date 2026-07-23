---
illustration_id: 05
type: flowchart
style: sketch-notes
palette: cool-macaron
---

Query Tuning with EXPLAIN ANALYZE — circular process flow.
STEPS: 1 Observe slow query; 2 EXPLAIN plan; 3 EXPLAIN ANALYZE actual time and rows; 4 Change SQL or index; 5 Verify again.
LABELS: “estimated rows”, “actual rows”, “actual time”, “loops”, “rows scanned”.
CONNECTIONS: clockwise arrows and a feedback arrow from Verify to Observe.
Add a small warning triangle: “ANALYZE executes the statement”.
COLORS: pastel blue observation, lavender plan, peach analysis, mint improvement, coral warning.
STYLE: clean hand-drawn operations playbook, short large labels, simple stopwatch and tree-node icons.
Clean composition with generous white space. No logo, watermark, fake metrics, or long output dump.
ASPECT: 16:9.
