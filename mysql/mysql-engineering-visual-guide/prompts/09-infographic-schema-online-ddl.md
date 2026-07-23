---
illustration_id: 09
type: infographic
style: sketch-notes
palette: cool-macaron
---

Schema Design and Online DDL — split infographic.
LEFT ZONE “Design for the workload”: InnoDB, utf8mb4, explicit primary key, short stable key, right-sized types, only useful indexes.
RIGHT ZONE “Choose the DDL algorithm”: INSTANT → INPLACE → COPY, with increasing work and blocking risk.
BOTTOM: test on production-like data; monitor metadata locks, replication lag, disk headroom, rollback plan.
LABELS: “INSTANT”, “INPLACE”, “COPY”, “LOCK”, “Primary Key”, “Index”.
COLORS: mint safe metadata-only changes, blue in-place work, coral copy/rebuild risk, lavender checklist.
STYLE: professional hand-drawn operational checklist, short labels only.
Clean composition with generous white space. No logo, watermark, or absolute claim that every ALTER blocks all reads and writes.
ASPECT: 16:9.
