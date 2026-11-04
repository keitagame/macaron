# kind 10001: リレーリスト (relay list)

- 保存: replaceable
- 現行API: なし(新規。分散化のために必要)
- 関連機能: [multi-relay](../features/multi-relay.md)

## 目的

「このユーザーの投稿はここに書かれている / ここで読める」を公表する。他の人のクライアントが、そのユーザーの投稿を探しに行く先になる。

## 構造

```json
{
  "kind": 10001,
  "content": "",
  "tags": [
    ["relay", "wss://relay.example.com", "write"],
    ["relay", "wss://other.example.net",  "read"],
    ["relay", "https://sse.example.org"]
  ]
}
```

| 位置 | 意味 |
|---|---|
| 2番目 | リレーの URL。`wss://`(WebSocket)または `https://`(HTTP+SSE) |
| 3番目 | `write`=自分が投稿する先 / `read`=自分が読む先(自分宛てを受け取る先)。省略は両方 |

## リレーの検証

- `relay` タグが `max_relays` 以下(目安 10)。URL 形式が正しい。

## クライアントの扱い

- 他人の投稿を探す = その人の `write` リレーへ `REQ`(アウトボックス方式)。
- 自分宛て(メンション・DM・通知)を探す = 自分の `read` リレーへ。
- 自分が書くとき = 自分の `write` リレー全部へ `EVENT`。
- URLは取得できなければ [handle](../features/handle.md) の `.well-known/macaron.json` の `relays` を使う。

## 拡張ポイント

- `["relay", url, "dm"]` のように用途を足せる(未知の用途は無視)。
