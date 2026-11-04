# kind 10000: プロフィール (profile)

- 保存: replaceable(`pubkey` ごとに最新1件)
- 現行API: `profile`, `upload_image`(avatar/banner), `upload_deco`, `remove_image`, `remove_deco`
- 関連機能: [handle](../features/handle.md) / [profile-decoration-and-badges](../features/profile-decoration-and-badges.md) / [custom-emoji](../features/custom-emoji.md)

## 目的

表示名・自己紹介・アイコンなど、そのアカウントの「見た目」。

## 構造

```json
{
  "kind": 10000,
  "content": "{\"name\":\"ゼッド\",\"bio\":\"こんにちは\",\"avatar\":\"https://cdn.example.com/a/1.jpg\",\"banner\":\"https://cdn.example.com/b/1.jpg\"}",
  "tags": [
    ["handle", "macaron@example.com"],
    ["deco", "https://cdn.example.com/d/1.png"],
    ["badge", "🔥"],
    ["badge", ":party:", "https://cdn.example.com/e/party.webp"],
    ["emoji", "party", "https://cdn.example.com/e/party.webp"]
  ]
}
```

### content(JSON文字列)

| キー | 現行 | 説明 |
|---|---|---|
| `name` | `users.name` | 表示名(現行 30文字まで) |
| `bio` | `users.bio` | 自己紹介(現行 160文字まで) |
| `avatar` | `users.avatar` | アイコンURL(正方形、現行 400px) |
| `banner` | `users.banner` | ヘッダー画像URL(3:1、現行 1500×500) |

- **知らないキーは捨てずに保持する**(更新時に消さない)。

### tags

| タグ | 現行 | 説明 |
|---|---|---|
| `handle` | `users.handle` | `名前@ドメイン`([handle](../features/handle.md))。確認できたものだけ表示 |
| `deco` | `users.deco` | アイコンに重ねる透過画像(PNG/WebP/GIF、現行 256px・3MB) |
| `badge` | `users.badges` | 表示名の横に出す絵文字。最大 `max_badges`(現行 5)。カスタム絵文字は URL も書く |
| `emoji` | — | 表示名・自己紹介内の `:name:` を解決する定義 |

## リレーの検証

- `content` がJSONオブジェクト、`name` が `max_name_length` 以下。
- `badge` の数が上限以内。標準絵文字は grapheme 1個(現行 `valid_emoji`)。
- 画像URLは `https:` のみ。サイズ・形式の検査は**メディアサーバー側**で行う([media](../features/media.md))。

## クライアントの扱い

- 画像のアップロード後、そのURLで新しい kind 10000 を署名して送るだけ(`upload_image` と `profile` が1つになる)。
- 古いプロフィールが別リレーに残る場合があるので、`created_at` が最新のものを採用する。
- プロフィール未取得のユーザーは、公開鍵の先頭8文字などで仮表示する。

## 拡張ポイント

- 固定投稿(`["pin", id]`)、リンク集(`["link", name, url]`)、誕生日などは**タグ追加**で足す。
