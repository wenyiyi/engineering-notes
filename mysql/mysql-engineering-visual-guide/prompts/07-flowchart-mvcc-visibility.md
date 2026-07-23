---
illustration_id: 07
type: flowchart
style: sketch-notes
palette: cool-macaron
---

MVCC Visibility and Undo Chain — technical flowchart.
Layout: left transaction timeline, center Read View gate, right row version chain.
Version chain: V3 trx 130 → V2 trx 118 → V1 trx 101 via roll pointers.
Read View shows active transactions and a visibility decision.
Two small lanes: “READ COMMITTED: new snapshot per statement” and “REPEATABLE READ: snapshot reused in transaction”.
LABELS: “DB_TRX_ID”, “DB_ROLL_PTR”, “Read View”, “Visible?”, “Undo”.
COLORS: blue visible versions, gray unavailable versions, lavender snapshot, coral active/uncommitted.
STYLE: precise hand-drawn database internals diagram with large legible keywords and minimal prose.
Clean composition with generous white space. No logo, watermark, or claim that every snapshot is always created at START TRANSACTION.
ASPECT: 16:9.
