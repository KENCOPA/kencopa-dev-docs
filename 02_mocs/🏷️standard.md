---
moc:
---
## データビュー
```dataview
table file.tags as "タグ", file.mtime as "更新日"
from "03_projects" or "04_standards" or "05_research"
where contains(moc, [[🏷️ADR]])
sort file.mtime desc
```


