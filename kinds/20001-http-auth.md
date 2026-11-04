# kind 20001: HTTP認証 (http auth)

- 保存: ephemeral(保存しない)
- 現行API: セッション Cookie + `X-CSRF-Token`
- 関連機能: [transport-sse](../core/07-transport-sse.md) / [media](../features/media.md) / [handle](../features/handle.md) / [identity-and-keys](../features/identity-and-keys.md)

## 目的

HTTP リクエスト1回ごとに「この鍵の持ち主が、この URL を、このメソッドで叩いている」と署名で証明する。**セッションもCookieもCSRFトークンも不要**(署名にURLとメソッドが入るので、他のサイトから勝手に叩かれても使えない)。

## 構造

```json
{
  "kind": 20001,
  "content": "",
  "tags": [
    ["u", "https://relay.example.com/media"],
    ["method", "POST"],
    ["x", "<アップロードするファイルの sha256>"]
  ]
}
```

ヘッダー: `Authorization: Macaron <base64(イベントJSON)>`

| タグ | 必須 | 意味 |
|---|---|---|
| `u` | ✓ | 呼び出す URL(クエリを除いた絶対URL) |
| `method` | ✓ | HTTPメソッド(大文字) |
| `x` |  | ボディの sha256(アップロード時。改ざん・使い回し防止) |

## サーバーの検証

1. 署名が正しい。
2. `u` が実際のリクエストURLと一致、`method` が一致。
3. `created_at` が現在から ±60秒以内(短くする)。
4. `x` があればボディのハッシュと一致。
5. 同じ `id` の再利用を拒否(60秒分の id をメモリ/DBに保持)。

## クライアントの扱い

- リクエストのたびに署名して付ける。署名は速い(Ed25519)ので問題ない。
- 公開情報の GET(タイムライン閲覧など)には付けない。
- `EventSource` はヘッダーを付けられない: 認証つきSSEは `fetch` ストリームを使う([transport-sse](../core/07-transport-sse.md))。

## 使う場面

| 場面 | エンドポイント |
|---|---|
| 認証つき購読 | `POST /sse` |
| 画像・動画のアップロード/削除 | `POST /media`, `DELETE /media/<sha256>` |
| ハンドル取得 | `POST /x/handle` |
