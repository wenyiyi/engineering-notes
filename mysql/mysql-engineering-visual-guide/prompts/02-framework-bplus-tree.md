---
illustration_id: 02
type: framework
style: sketch-notes
palette: cool-macaron
---

Why InnoDB Uses a B+ Tree — technical framework diagram.
STRUCTURE: top root page, middle internal pages, bottom linked leaf pages.
NODES: root keys; high fan-out internal pages; 16 KiB page callout; leaf records; left-to-right leaf links.
RELATIONSHIPS: arrows show one root-to-leaf lookup and a separate leaf-to-leaf range scan.
LABELS: “Root”, “Internal”, “Leaf”, “16 KiB Page”, “Point Lookup”, “Range Scan”.
COLORS: cream paper, black ink, pastel blue internal pages, mint leaves, lavender arrows, coral only for disk I/O.
STYLE: hand-drawn engineering notebook, compact accurate B+ tree, not a binary tree, no invented row-count capacity.
Clean composition with generous white space. No logo, watermark, or paragraph text.
ASPECT: 16:9.
