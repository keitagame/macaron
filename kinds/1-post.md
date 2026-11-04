# kind 1: 投稿 (post)

- 保存: regular
- 現行API: `post`(返信・引用・メディア・センシティブ指定を含む)
- 関連機能: [thread-and-reply](../features/thread-and-reply.md) / [quote](../features/quote.md) / [mention-and-hashtag](../features/mention-and-hashtag.md) / [media](../features/media.md) / [sensitive-content](../features/sensitive-content.md) / [custom-emoji](../features/custom-emoji.md)

## 目的

テキスト(+メディア)の投稿。返信・引用も同じ kind で、**付いているタグで区別する**。

## 構造

```json
{
  "kind": 1,
  "content": "こんにちは macaron:pub:3f9a… #hello :party:",
  "tags": [
    ["t", "hello"],
    ["p", "<メンション先pubkey>"],
    ["emoji", "party", "https://cdn.example.com/e/party.webp"],
    ["imeta", "url https://cdn.example.com/m/ab.jpg", "m image/jpeg", "dim 1200x800", "x <sha256>"],
    ["cw"]
  ]
}
```

| 種類 | 付けるタグ |
|---|---|
| 通常投稿(トップレベル) | `e` も `q` も無し |
| 返信 | `["e", <親id>]` + `["p", <親の作者>]` |
| 引用 | `["q", <引用元id>]` + `["p", <引用元の作者>]`(**`e` は付けない**) |
| ハッシュタグ | 本文の `#語` ごとに `["t", "語(小文字)"]` |
| メンション | 本文の参照ごとに `["p", pubkey]` |
| メディア | `imeta` 1つにつき1ファイル |
| センシティブ | `["cw"]` または `["cw", "理由"]` |
| カスタム絵文字 | 本文の `:name:` ごとに `["emoji", name, url]` |

## content

- プレーンテキスト。HTML・Markdown は使わない(クライアントが `#…` `@…` `:…:` `https://…` を描画時にリンク化する)。
- 本文なしでメディアだけの投稿は `content` が `""`(現行と同じ)。
- 参照の書き方は [mention-and-hashtag](../features/mention-and-hashtag.md)。

## リレーの検証

- `e` と `q` の両方を持つイベントは `invalid: reply and quote are exclusive`(現行の「返信と引用は同時にできません」)。
- `content` が `max_content_length`(情報ドキュメント。現行 500)以下。文字数は Unicode コードポイント数。
- `imeta` が `max_media`(現行 4。動画は1本のみ)以下。
- 本文が空かつ `imeta` なしは `invalid:`。
- 制御文字(改行・タブ以外)を含まない。

## クライアントの扱い

- 返信先が見つからない/削除済みでも、投稿自体は表示する(「返信元を表示」は取得できたときだけ)。
- 受信したら `sig` を検証し、`e` / `q` の参照は必要になったときに `ids` フィルタで取得する。
- 引用元は **1段だけ**展開する(現行の `quote` と同じ)。

## 拡張ポイント

- 投票・位置情報・言語などは**タグの追加**で表現する([09-extending](../core/09-extending.md))。
- 長文記事・スレッドまとめは別 kind を作る(kind 1 の `max_content_length` を伸ばさない)。
