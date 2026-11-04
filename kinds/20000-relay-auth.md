# kind 20000: リレー認証 (relay auth)

- 保存: ephemeral(保存しない)
- 現行API: `login`(の役割のうち、「誰か」を証明する部分)
- 関連機能: [identity-and-keys](../features/identity-and-keys.md) / [kind 4](4-direct-message.md) / [kind 7](7-view.md)

## 目的

WebSocket 接続上で「自分はこの公開鍵の持ち主」と証明する。パスワードやセッションの代わり。**認証が必要な REQ のためだけ**に使う。投稿(EVENT)は署名があるので認証不要。

## 構造

```json
{
  "kind": 20000,
  "content": "",
  "tags": [
    ["relay", "wss://relay.example.com"],
    ["challenge", "<リレーが送った文字列>"]
  ]
}
```

## 流れ

```
R: ["AUTH", "b3a1…"]                        ← 接続直後、または必要になったとき
C: ["AUTH", { kind:20000, tags:[["relay",…],["challenge","b3a1…"]], … }]
R: ["OK", "<id>", true, ""]                 ← この接続は <pubkey> として扱う
```

## リレーの検証

- `challenge` が、その接続に送った値と一致する。
- `relay` が自分のURLと一致する(別のリレーへの使い回し防止)。
- `created_at` が現在から ±`created_at_window`(目安 10分)以内。
- 署名が正しい。

## クライアントの扱い

- 認証が必要な `REQ` に `CLOSED auth-required:` が返ったら、`AUTH` を済ませて `REQ` を再送する。
- 1接続で複数の鍵を認証できる実装も可(リレーが許せば)。

## HTTP との違い

HTTP+SSE では接続が状態を持たないので、リクエストごとに [kind 20001](20001-http-auth.md) を使う。
