---
type: note
tags:
- '#ai/ref'
---
> [!info] 必要なセクションを選択して使用してください！

# 📜 文脈・背景


```cardlink
url: https://www.notion.so/Dairy-Scrum-24734568ef508049a04ff700d14495fb?source=copy_link
title: "The AI workspace that works for you. | Notion"
description: "A tool that connects everyday work into one space. It gives you and your teams AI tools—search, writing, note-taking—inside an all-in-one, flexible workspace."
host: www.notion.so
image: https://www.notion.so/images/meta/default.png
```

GET以外でレスポンスに値を含める必要はあるのでしょうか？FEで何も活用されてないので、何のために返しているのかなと → ないですね…. → たまーに、onSuccessでinvalidateQueriesのkey指定するときにresponseにidが含まれていると便利なくらい（それもqueryから取得すれば解決するケースが多い。）

# 🎨 対応案



# 🚀 決定

GET：Any
PUT：null
POST：null
DELETE：null
- PUT / POST / DELETEは基本空のresponseでOK
- なんか返したい値があるときだけ追加する

# 🪞 結果・影響


# 🍜 今後の検討事項



