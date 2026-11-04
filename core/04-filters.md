# 04 フィルタ

`REQ` と `COUNT` で使う、取得条件のJSONオブジェクト。**配列内は OR、項目間は AND。** 複数のフィルタを渡したときは全体が OR。

```json
{
  "ids":     ["<id>", …],
  "authors": ["<pubkey>", …],
  "kinds":   [1, 2],
  "#e":      ["<id>", …],
  "#p":      ["<pubkey>", …],
  "#t":      ["hello"],
  "has":     ["e"],
  "without": ["e"],
  "since":   1730000000,
  "until":   1730700000,
  "limit":   20
}
```

| 項目 | 意味 |
|---|---|
| `ids` `authors` | 完全一致(前方一致はしない) |
| `kinds` | kind の一致 |
| `#<1文字>` | その名前のタグの(先頭の)値がどれかに一致 |
| `has` | 指定した名前のタグを**すべて持つ** |
| `without` | 指定した名前のタグを**ひとつも持たない** |
| `since` `until` | `created_at` の範囲。両端を含む |
| `limit` | 保存済みイベントの返却上限(新しい順に数える) |

`has` / `without` は、現行の `parent_id IS NULL`(返信ではないトップレベル投稿)を `kinds:[1], without:["e"]` と表すために使う。

## 並び順と件数

- 保存済みイベントは `created_at` 降順、同値は `id` 降順。
- `limit` 未指定はリレーの既定値(`limits.default_limit`)。上限は `limits.max_limit`。
- 保存済み分を返し終えたら `EOSE`、以降は新着をそのまま流す([05-messages.md](05-messages.md))。

## ページ送り

最後に受け取ったイベントの `created_at` を `until` にして再度 `REQ`。同じ秒のイベントが重複するので**クライアントが `id` で重複排除**する。
(現行の `before` + `before_id` カーソルより単純。)

## 拡張フィルタ項目

リレーが対応していれば使える。非対応のリレーは**無視する**ので結果が広がる点に注意。クライアントは `GET /` の `features` で確認する。

| 項目 | 機能 | 仕様 |
|---|---|---|
| `search` | 全文検索 | [../features/search.md](../features/search.md) |
| `feed` | リレー側の並べ替え(おすすめ等) | [../features/recommendation-feed.md](../features/recommendation-feed.md) |
| `group` | `COUNT` の集計単位 | [../features/counts.md](../features/counts.md) |

## 制限(ポリシー)

`max_filters`(1REQあたり)、`max_authors`、`max_limit` はリレーが決める。超えたら `CLOSED` で `invalid: too many authors` のように返す。
