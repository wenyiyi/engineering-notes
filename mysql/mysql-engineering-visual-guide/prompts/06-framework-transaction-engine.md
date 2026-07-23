---
illustration_id: 06
type: framework
style: sketch-notes
palette: cool-macaron
---

Inside an InnoDB Transaction — layered framework.
STRUCTURE: central transaction card protected by four surrounding mechanisms.
NODES: Atomicity → Undo; Durability → Redo; Isolation → MVCC + Locks; Consistency → constraints + application rules.
RELATIONSHIPS: Undo arrow points backward to an earlier row version; Redo is generated during changes, written to the log buffer, and flushed according to commit durability settings before acknowledging a durable commit, then replayed during crash recovery; MVCC arrow to snapshot read; Locks arrow to current write.
LABELS: “Undo”, “Redo”, “MVCC”, “Locks”, “COMMIT”, “ROLLBACK”.
COLORS: blue transaction, lavender undo, mint redo, peach MVCC, coral locks.
STYLE: hand-drawn engineering framework, accurate conceptual relationships, minimal words.
Clean composition with generous white space. No logo or watermark. CRITICAL: never say “after COMMIT, write Redo Log”; never imply redo starts only after commit. Do not claim that consistency is a single log feature.
ASPECT: 16:9.
