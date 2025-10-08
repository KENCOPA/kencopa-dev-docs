---
moc: "[[🏷️research]]"
---

<%*
// ファイル名を window から入力
let title = await tp.system.prompt("[必須] ファイル名を入力（拡張子不要）:");
// ファイル名の移動
await tp.file.move(`05_research/🧠${title}`);
%> 