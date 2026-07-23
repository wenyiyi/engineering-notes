---
illustration_id: 08
type: comparison
style: sketch-notes
palette: cool-macaron
---

Access Path Determines Lock Scope — three-column comparison.
LEFT “Unique Lookup”: one matching index record, small lock footprint.
CENTER “Range Scan”: several records plus gaps under next-key locking.
RIGHT “Full Scan”: many scanned records, broad contention risk.
LABELS: “Unique”, “Range”, “Full Scan”, “Record Lock”, “Gap”, “Next-Key”.
Show SELECT FOR UPDATE / UPDATE / DELETE as locking operations; ordinary snapshot SELECT stays outside the lock lane.
COLORS: mint narrow footprint, lavender range, coral broad footprint, blue index pages.
STYLE: hand-drawn sketchnote, rows as small tiles, locks as simple padlock icons.
Clean composition with generous white space. No logo, watermark, or statement that InnoDB always locks the whole table.
ASPECT: 16:9.
