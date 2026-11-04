# 機能: 引用

- 使う kind: [1](../kinds/1-post.md)
- 現行API: `post` の `quote_id`

## 書く

[kind 1](../kinds/1-post.md) に `["q", <引用元id>]` と `["p", <引用元の作者>]` を付ける。本文に感想を書く。

```json
{ "kind": 1, "content": "これは重要 macaron:note:<引用元id>",
  "tags": [["q","<引用元id>","wss://relay.example.com"],["p","<引用元の作者>"]] }
```

- 返信とは**排他**(`e` と `q` を同時に付けない)。
- 本文中の `macaron:note:<id>` は任意。付けるとタグ非対応のクライアントでも参照が分かる。

## 表示

- `ids` で引用元を取得し、**1段だけ**入れ子表示(引用の引用は展開しない)。
- 引用元が削除・未取得なら「引用元は削除されたか取得できません」。
- 引用元の本文は先頭300文字まで(現行 `mb_substr(…, 0, 300)`)、メディアは先頭1つだけ表示。
- 引用元がセンシティブ([sensitive-content](sensitive-content.md))なら、引用側でもモザイク。

## 件数・通知

- 数 = `COUNT {kinds:[1],"#q":[ids…],group:["q"]}`。
- 通知 = `p` 経由([notifications](notifications.md))。種類の判別は「`q` タグがあれば引用」。

## 拡張ポイント

- `q` に `["q", id, hint, <pubkey>]` で作者を持たせると、`p` を見なくても引用元を探せる。
