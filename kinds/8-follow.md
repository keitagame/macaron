# kind 8: フォロー (follow)

- 保存: regular
- 現行API: `follow`, `follows`
- 関連機能: [timeline](../features/timeline.md) / [notifications](../features/notifications.md) / [counts](../features/counts.md)

## 目的

あるユーザーをフォローする。**1フォロー = 1イベント**(リスト丸ごとではない)。

## 構造

```json
{ "kind": 8, "content": "", "tags": [["p", "<フォロー先pubkey>"]] }
```

- `p` はちょうど1つ。自分自身は `blocked:`(現行「フォローできません」)。
- 解除 = [kind 5](5-deletion.md)。

## リレーの検証

- 同じ `(pubkey, p)` が既にある → `duplicate:`。

## クライアントの扱い

| 欲しいもの | 取得 |
|---|---|
| 自分がフォロー中の人 | `{"kinds":[8],"authors":["<me>"]}` |
| 自分のフォロワー | `{"kinds":[8],"#p":["<me>"]}` |
| フォロー数 / フォロワー数 | `COUNT` に同じフィルタ |
| フォローしているか | `{"kinds":[8],"authors":["<me>"],"#p":["<x>"]}` |

- フォロー中の人のタイムライン = `authors` に上記の `p` 一覧を入れる([timeline](../features/timeline.md))。数百〜千人を超えるときはリレーの `feed:following` 拡張を使う。
- 新しいフォロワーの通知は、`{"kinds":[8],"#p":["<me>"],"since":…}` で届く([notifications](../features/notifications.md))。

## リストを1つにまとめない理由

[02-kind-ranges](../core/02-kind-ranges.md) の設計メモ参照(複数端末での取りこぼし防止)。

## 拡張ポイント

- 呼び名・グループ分け: `["l","友達"]` のようなタグを足す。
- ミュート・ブロック: 同じ作りで別 kind を足す(`kind 8` に混ぜない)。
