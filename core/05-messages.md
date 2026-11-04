# 05 メッセージ

すべての通信は **JSON配列** で、先頭がメッセージ名。トランスポート(WS / SSE)が違っても同一。

## クライアント → リレー

| メッセージ | 意味 |
|---|---|
| `["EVENT", <event>]` | イベントを投稿する |
| `["REQ", <sub-id>, <filter>, …]` | 購読を開始する(保存済み → EOSE → ライブ) |
| `["CLOSE", <sub-id>]` | 購読を終了する |
| `["COUNT", <sub-id>, <filter>, …]` | 件数だけを数える |
| `["AUTH", <event>]` | 認証イベント(kind 20000)を返す |

`sub-id` は 1〜64文字の任意文字列。接続ごとに独立。同じ `sub-id` で `REQ` を再送すると**置き換え**。

## リレー → クライアント

| メッセージ | 意味 |
|---|---|
| `["EVENT", <sub-id>, <event>]` | 購読にマッチしたイベント |
| `["OK", <event-id>, <true/false>, <message>]` | `EVENT` の受理結果 |
| `["EOSE", <sub-id>, <meta>?]` | 保存済み分の終わり。`meta` は任意のオブジェクト(次ページ情報など) |
| `["CLOSED", <sub-id>, <message>]` | リレー側から購読を終了(理由つき) |
| `["COUNT", <sub-id>, {"count": n, …}]` | 件数の結果 |
| `["AUTH", <challenge>]` | 認証要求 |
| `["NOTICE", <message>]` | 人が読むための通知 |

## メッセージ文字列の接頭辞

`OK` `CLOSED` の `message` は `接頭辞: 説明` 形式。クライアントは接頭辞で分岐する。

| 接頭辞 | 意味 | 現行APIの対応 |
|---|---|---|
| `invalid:` | 形式・署名・内容が不正 | HTTP 400 |
| `duplicate:` | 既に持っている | `INSERT OR IGNORE` |
| `rate-limited:` | 送信が多すぎる | `throttle()` の 429 |
| `blocked:` | ポリシーで拒否(禁止kind・禁止語など) | 403 |
| `restricted:` | 条件を満たさない(アカウントが新しい等) | `SENS_MIN_AGE` |
| `auth-required:` | 認証してから再試行 | 401 |
| `error:` | リレー内部エラー | 500 |

## 例

```
C: ["EVENT", {"id":"aa…","kind":1,…}]
R: ["OK", "aa…", true, ""]

C: ["REQ", "tl", {"kinds":[1],"without":["e"],"limit":20}]
R: ["EVENT", "tl", {…}]  (×20)
R: ["EOSE", "tl"]
R: ["EVENT", "tl", {…}]  ← 以降はライブ
C: ["CLOSE", "tl"]
```

## バッチ(HTTP用)

HTTPで1リクエストに複数メッセージを入れるときは**配列の配列**で送る([07-transport-sse.md](07-transport-sse.md))。
