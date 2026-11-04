# kind 4: ダイレクトメッセージ (direct message)

- 保存: regular
- 現行API: `dm_send`, `dm`, `convos`
- 関連機能: [encryption](../features/encryption.md) / [notifications](../features/notifications.md)

## 目的

1対1の暗号化メッセージ。

## 構造

```json
{
  "kind": 4,
  "content": "m1:<base64(nonce24 || 暗号文)>",
  "tags": [["p", "<宛先pubkey>"]]
}
```

- `p` はちょうど1つ(宛先)。
- `content` は [encryption](../features/encryption.md) の `m1:` 形式。平文は 1〜`max_dm_length`(現行 1000)文字。

## リレーの検証

- `p` が1つ、`content` が `m1:` で始まる(平文は見えないので長さは**暗号文の長さ**で制限)。
- 自分宛て(`pubkey == p`)は `blocked:`(現行「送信できません」)。
- **読み出しに認証が必要**: `kinds:[4]` を含む `REQ` は、フィルタが「`authors:[認証済み自分]`」または「`#p:[認証済み自分]`」のときだけ許可する。それ以外は `CLOSED auth-required:` / `restricted:`。

## クライアントの扱い

- 会話一覧(`convos`): 次の2つを取得し、相手pubkeyごとにグループ化して最新1件を表示。
  ```
  {"kinds":[4],"authors":["<me>"],"limit":200}
  {"kinds":[4],"#p":["<me>"],"limit":200}
  ```
- 会話(`dm`): 相手 `X` とのやりとり = 上の2つに `#p:[X]` / `authors:[X]` を足す。
- 復号はクライアントで行う。リレーは平文を一切持たない。
- ライブ受信は `REQ {"kinds":[4],"#p":["<me>"],"since":<now>}` を張っておく。

## 既知の限界(v1)

- **メタデータは見える**: 誰と誰が・いつ・何回やりとりしたかは、リレーに分かる。
- 送信後に内容を取り消せない(削除イベントで消すのはリレー上のみ)。
- 長さがほぼそのまま見える(パディングなし)。
- これらを解決する「包み込み(wrap)」方式は、`m2:` として別 kind で足す([09-extending](../core/09-extending.md))。

## 拡張ポイント

- 既読・入力中表示: ephemeral kind(2xxxx)を足す。
- グループDM: `p` を複数にし、共通鍵を各宛先ごとに暗号化して配る新 kind を作る。
