# kind 2: リポスト (repost)

- 保存: regular
- 現行API: `repost`
- 関連機能: [timeline](../features/timeline.md) / [notifications](../features/notifications.md)

## 目的

他人の投稿を自分のフォロワーのタイムラインに再掲する(コメントなし)。コメント付きは[引用](../features/quote.md)。

## 構造

```json
{
  "kind": 2,
  "content": "",
  "tags": [["e", "<元投稿id>", "wss://relay.example.com"], ["p", "<元の作者>"]]
}
```

- `e` は**ちょうど1つ**。対象は kind 1 のみ(返信でもよい)。
- `content` は空、または元イベント(kind 1)の**JSON全文**。リレーが元を持っていない場合のフォールバックで、クライアントは必ず `sig` を検証してから使う。

## リレーの検証

- `e` が1つでない → `invalid:`
- 同じ `(pubkey, e)` が既にある → `duplicate:`(現行の `PRIMARY KEY(user_id, post_id)`)
- 対象が存在しない・削除済み → 受理してよい(あとで届く場合がある)が、`content` が空で対象不明なら `invalid:` にしてもよい(ポリシー)

## クライアントの扱い

- 取り消し = [kind 5](5-deletion.md) で自分のリポストイベントを削除。
- タイムラインでは、元投稿と、フォロー中の人のリポストを**同じ投稿IDについて1回だけ**表示する。時刻はリポストの `created_at` を使い、「〇〇さんがリポスト」を付ける(現行の `rtBy`)。
- 数 = `COUNT {kinds:[2], "#e":[id]}`。

## 拡張ポイント

- 「リポストの対象に kind 1 以外を許す」場合は `k` タグ(`["k","1"]`)で対象の kind を明示する。
