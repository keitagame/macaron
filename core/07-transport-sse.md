# 07 トランスポート: HTTP + SSE

WebSocket が使えない環境(共有ホスティングのPHPなど)向け。**同じメッセージ**を HTTP で運ぶ。

| 用途 | エンドポイント |
|---|---|
| 送信・1回きりの取得・件数 | `POST /msg` |
| ライブ購読 | `GET /sse` または `POST /sse` |

## POST /msg — リクエスト/レスポンス

ボディに**クライアント→リレーのメッセージ**を1つ入れる。レスポンスは**そのメッセージで生じたリレー→クライアントのメッセージの配列**。

```
POST /msg
["EVENT", {…}]
→ 200  [["OK","aa…",true,""]]

POST /msg
["REQ", "tl", {"kinds":[1],"without":["e"],"limit":20}]
→ 200  [["EVENT","tl",{…}], …, ["EOSE","tl"]]       ← ライブは続かない(1回で完結)

POST /msg
["COUNT", "c", {"kinds":[8],"#p":["<pk>"]}]
→ 200  [["COUNT","c",{"count":42}]]
```

- ボディの最初の要素が配列なら**バッチ**(複数メッセージ): `[["EVENT",{…}],["EVENT",{…}]]` → 結果は連結した配列。1リクエスト最大 `limits.max_batch`(目安 50)。
- `Content-Type: application/json`。HTTPステータスは常に 200(エラー内容は `OK false` / `CLOSED` で返す)。例外は 413(大きすぎる)・429(IP単位の制限)・5xx。
- `POST /msg` だけで**ポーリング運用**もできる(`REQ` に `since` を付けて数十秒おきに叩く)。SSEは最適化にすぎない。

## GET /sse — ライブ購読

```
GET /sse?f=<base64url(JSON配列)>&sub=<sub-id>
Accept: text/event-stream
```

`f` はフィルタの配列(例 `[{"kinds":[1],"without":["e"]}]`)を base64url にしたもの。URLが約4KBを超えるなら `POST /sse` を使う。

レスポンス(`text/event-stream`):

```
retry: 1000

id: 18231
event: m
data: ["EVENT","tl",{…}]

event: m
data: ["EOSE","tl"]

: ping
```

- 1つの SSE イベント = 1つのリレー→クライアントメッセージ。`event` 名は `m` 固定、`data` は JSON配列1行。
- `EVENT` には `id:`(リレー内部の連番 `seq`)を付ける。それ以外には付けない。
- 15〜25秒ごとにコメント行 `: ping` を送る(プロキシ切断防止)。

### 再接続と取りこぼし防止

- ブラウザの `EventSource` は切断すると自動で再接続し `Last-Event-ID: <seq>` を送る。
- リレーは `seq > Last-Event-ID` でフィルタにマッチするイベントだけを**`limit` を無視して**再送する。`EOSE` は初回のみ。
- 共有ホスティング向け: PHPの1接続の保持時間を **30〜55秒** に制限して自分で閉じ、`retry: 1000` で即再接続させる。`Last-Event-ID` で連続性が保たれるので、実質ロングポーリング。
- PHP実装の注意: `ob_end_flush()` / `output_buffering=off`、`X-Accel-Buffering: no`(nginx)、`set_time_limit` を保持時間+余裕に、`session_write_close()` を先に呼ぶ(現行 api.php のセッションロックが他リクエストを塞ぐため)。

## POST /sse — 大きなフィルタ・認証つき購読

`EventSource` は **カスタムヘッダーを付けられず、GETしかできない**。そのため大きなフィルタや認証が必要なら、`fetch` でレスポンスをストリーム読みする:

```js
const res = await fetch('/sse', {
  method: 'POST',
  headers: { 'Authorization': 'Macaron ' + b64(authEvent) },   // kind 20001
  body: JSON.stringify(['REQ', 'dm', { kinds: [4], '#p': [me] }]),
});
const reader = res.body.pipeThrough(new TextDecoderStream()).getReader();
// 受け取った SSE テキストを "\n\n" で区切って data: 行をJSON.parse
```

ボディは `["REQ", sub-id, filter…]`。GET と同じ形式で返る。再接続は自前(最後の `id:` を `Last-Event-ID` ヘッダーに付けて再POST)。

## 認証

`Authorization: Macaron <base64(kind 20001 イベントJSON)>`([kinds/20001-http-auth.md](../kinds/20001-http-auth.md))。GET で必要な場合のみクエリ `?auth=<base64url>` も許可(ログに残るので `created_at` の許容幅を短く)。

## 購読の閉じ方

接続を切ること自体が `CLOSE`。接続中に購読を入れ替えたいときは、接続を張り直す。
