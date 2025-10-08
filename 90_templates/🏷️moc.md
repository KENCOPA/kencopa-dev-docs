---
moc:
---
## データビュー
```dataview
table file.tags as "タグ", file.mtime as "更新日"
from "03_projects" or "04_standards" or "05_research"
where contains(moc, [[🏷️${title}]])
sort file.mtime desc
```




<%*
// ファイル名を window から入力
let title = await tp.system.prompt("[必須] ファイル名を入力（拡張子不要）:");
// ファイル名の移動
await tp.file.move(`02_mocs/🏷️${title}`);
%> 