# kind 3: リアクション (reaction)

- 保存: regular
- 現行API: `like`, `react`
- 関連機能: [custom-emoji](../features/custom-emoji.md) / [counts](../features/counts.md) / [notifications](../features/notifications.md)

## 目的

投稿への「いいね」と絵文字リアクションを**同じ kind** で表す。`content` で区別する。

## 構造

```json
{ "kind": 3, "content": "+",       "tags": [["e", "<投稿id>"], ["p", "<作者>"]] }
{ "kind": 3, "content": "🔥",      "tags": [["e", "<投稿id>"], ["p", "<作者>"]] }
{ "kind": 3, "content": ":party:", "tags": [["e", "<投稿id>"], ["p", "<作者>"],
                                           ["emoji", "party", "https://cdn.example.com/e/party.webp"]] }
```

| content | 意味 |
|---|---|
| `+` | いいね |
| 絵文字1つ | 標準絵文字リアクション |
| `:name:` | カスタム絵文字リアクション(`emoji` タグ必須) |

## リレーの検証

- `e` がちょうど1つ。`content` が上のいずれかの形式。
- `content` が `:name:` のとき、同名・同URLの `emoji` タグが存在する。
- 同じ `(pubkey, e, content)` が既にある → `duplicate:`(現行の `PRIMARY KEY(user_id, post_id, emoji)`)。
- 絵文字1つ判定は現行 `valid_emoji()` と同じ(grapheme 1個)。**使える絵文字を限定したい**場合(現行 `RX_SET` の8種)はリレーのポリシーで `blocked:`。

## クライアントの扱い

- 取り消し = [kind 5](5-deletion.md) で自分のリアクションを削除。トグルUIは「自分のリアクション一覧を `authors:[me], kinds:[3], #e:[…]` で取得 → あれば削除、なければ作成」。
- 数 = `COUNT {kinds:[3], "#e":[ids…], group:["e","content"]}` 1回で、投稿ごと・種類ごとの数が取れる([counts](../features/counts.md))。
- **いいねは公開される**(現行のように「自分のいいねタブだけ非公開」はできない)。UI上で隠すのは自由だが、データは公開。非公開リアクションが要るなら別 kind を作り [encryption](../features/encryption.md) を使う。

## 拡張ポイント

- `-`(低評価)や反応の強さなどは `content` の新しい約束事で足せる。知らないクライアントは無視する。
