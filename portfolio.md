---
schemaVersion: 1
color: "#b0785a"
initials: "cc"
cat:
  ja: "ノウハウ / Claude Code"
  en: "Know-how / Claude Code"
short:
  ja: "Claude Code の常駐 context を実測し、削る価値を判定してから減らす手順集と簡易スキル。効くキーを実測で見極めた実践知。"
  en: "A guide and small skill for measuring Claude Code's resident context, deciding whether trimming is worth it, and cutting only what measurably works."
tech: ["Claude Code", "Documentation"]
store: null
live: null
guide: null
featured: false
---
## ja

Claude Code の常駐 context を実測し、削る価値があるかを判定してから減らすための手順集と簡易スキルです。1M 窓が既定の環境では settings キーで削れるのは Total の 1% 未満で、目的は指示の遵守率へ移ります。permissions.deny では context は減らない、disableBundledSkills は単独では増える、といった実測で分かる要点を、効くキーに絞って整理しています。

## en

A guide and small skill for measuring Claude Code's resident context and deciding whether trimming is worth it before cutting. With the 1M window now the default, settings keys recover under 1% of the total, so the goal shifts to instruction adherence. It records what measurement actually shows — permissions.deny does not cut context, disableBundledSkills alone makes it larger — instead of folklore.
