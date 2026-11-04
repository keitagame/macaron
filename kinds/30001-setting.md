# kind 30001: 設定 (setting)

- 保存: addressable(`pubkey` + `d` = 設定キー)
- 現行API: `noview`, `notif_read`(users テーブルの列)
- 関連機能: [notifications](../features/notifications.md) / [view-history](../features/view-history.md) / [encryption](../features/encryption.md)

## 目的

複数端末で揃えたい、本人専用の設定。現行は `users` 表の列だったものを、**キー別のイベント**にする。

## 構造

```json
{ "kind": 30001, "content": "1730700000", "tags": [["d", "notif_read"]] }
{ "kind": 30001, "content": "1",          "tags": [["d", "no_view"]] }
{ "kind": 30001, "content": "m1:<base64>", "tags": [["d", "mute_words"]] }
```

- `content` が `m1:` で始まれば暗号化(自分宛て)。そうでなければ平文。**公開してよい設定だけ平文**にする。
- キーは小文字英数字と `_`。

## 定義済みキー

| d | 現行 | 値 | 平文可 |
|---|---|---|---|
| `notif_read` | `users.notif_read` | 通知を最後に見た時刻(UNIX秒) | 可 |
| `no_view` | `users.no_view` | `"1"`=閲覧を記録しない | 可 |
| (任意) | — | クライアント独自の設定。キーに接頭辞(`myapp_…`)を付ける | 要暗号化を推奨 |

## リレーの検証

- `d` が1つ、値が `max_setting_bytes`(目安 4KB)以下。
- 読み出しは `authors:[認証済み自分]` のときだけ(ポリシー推奨)。

## クライアントの扱い

- 起動時に `{"kinds":[30001],"authors":["<me>"]}` で全部取得し、ローカルにキャッシュ。
- 更新は上書き(addressable なので古い値は自動で消える)。
- 通知音のオン/オフ、テーマなど**端末固有の設定**はプロトコルに乗せずローカルに置く(現行の `localStorage` 相当)。

## 拡張ポイント

- キーを足すだけ。リレーの変更は不要。
