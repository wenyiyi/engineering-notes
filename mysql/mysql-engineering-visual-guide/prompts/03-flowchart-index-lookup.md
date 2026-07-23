---
illustration_id: 03
type: flowchart
style: sketch-notes
palette: cool-macaron
---

InnoDB Index Lookup Paths — side-by-side flowchart.
Layout: two horizontal lanes.
TOP LANE “Secondary + Table Lookup”: secondary B+ tree → primary-key value → clustered B+ tree → full row.
BOTTOM LANE “Covering Index”: composite secondary B+ tree → requested columns → result.
LABELS: “Secondary Index”, “Primary Key”, “Clustered Index”, “Full Row”, “Covering Index”, “1 tree”, “2 trees”.
CONNECTIONS: bold hand-drawn arrows; circle the avoided second lookup in the bottom lane.
COLORS: blue index nodes, lavender primary-key tokens, mint fast path, coral extra lookup.
STYLE: technical sketchnote with very short labels, database-page icons, clear spatial separation.
Clean composition with generous white space. No logo, watermark, SQL paragraph, or realistic imagery.
ASPECT: 16:9.
