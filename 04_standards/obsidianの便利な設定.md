---
type: note
moc: "[[🏷️standard]]"
---
# 🔌 プラグイン
KENCOPAおすすめのプラグインはGitからCloneした時点で使えるようになっている。
> [!tip] おすすめのプラグインがあればぜひ追加してください！
## 2Hop Link Plus
- ページ下部にLinkとBack Linkが表示されるようになる。
![[Pasted image 20250419002021.png]]
## Key-Value List
- 「key{delimeter}value」の形式でリストを書くといい感じに表示してくれる。

↓こんな感じ

- key1: value
- key2: #hoge #piyo
- key3: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
## Auto Card Link
- Linkを貼ると勝手にいい感じに整形してくれる。
![[Pasted image 20250419002238.png]]
## Dataview
- DataviewJSを使ってノートを柔軟に集計しテーブル表示することができる。
## Emoji Toolbar
- 絵文字を簡単に入力できるようになる。
## Excalidraw
- obsidianでExcalidrawが使えるようになる。
- obsidianを使う理由の８割はこれ。
## Floating TOC
- ノートの左部にTable Of Contentsが表示されるようになる。（obsidianのデフォルトTOCは不便...）
## Git
- obsidianからGitを使ったノートの管理ができるようになる。
## Iconize
- ファイルに可愛いアイコンをつけられるようになる。
## Templater
- JSを使った柔軟なTemplateファイルを作成できるようになる。
## Copilot
- obsidian上で呼び出せるLLM。
## SPX Link Copier
- 自作plugin。
- SPXを使って共有可能なObsidianリンクをコピーできる。
- https://github.com/KENCOPA/obsidian-spx-link-copier-plugin
# ⌨️ ショートカット
- cmd + eで編集モードと閲覧モードを切り替え
- 設定（cmd + ,）> Hotkeysからショートカットを変更できる。おすすめの設定は下記。
	- Templater: Create new note from template ⇨ 「cmd + n」
		- デフォルトの「cmd + n」は「Create new note（白紙からノートを作る）」でほとんど使わないので置き換えた方が便利。
	- Emoji Toolbr: Open emoji picker ⇨ 「なんでも（筆者はcmd + j）」
		- Emojiはパッと使えた方が便利なので開きやすいショートカットに登録推奨。
	- SPX Link Copier: Copy SPX-wrapped Obsidian URL ⇨ 「なんでも（筆者はcmd + shift + c）」
		- 簡単にページのURLを取得できた方が便利。

# 🎨 デザイン
- obsidianは`.obsidian/snippets`以下にcssファイルを置くことでデザインを変更することができる。

```cardlink
url: https://help.obsidian.md/snippets
title: "CSS snippets - Obsidian Help"
description: "CSS snippets - Obsidian Help"
host: help.obsidian.md
favicon: https://publish-01.obsidian.md/access/f786db9fac45774fa4f0d8112e232d67/favicon.ico
```

> [!info] 
> KENCOPAおすすめのデザインを`.obsidian/snippets`に配置しているのでよかったら使ってください！設定（cmd + ,）⇨ Appearance ⇨ CSS snippets から有効にしたいcssをオンにしてください！（よりよいデザインの提案お待ちしております！）
> 
> ![[Pasted image 20250418141724.png]]
>  