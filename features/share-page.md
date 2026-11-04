# 機能: シェア用ページ

- 使う: HTTP `GET /s/<id>`(イベントは使わない)
- 現行API: `share`(`share_page()`、`/p/<id>`)

## 考え方

SNS や チャットに URL を貼ったとき、プレビューカードが出るようにする。クローラーは JavaScript を実行しないので、**リレーがメタタグだけの HTML を返し、人間はアプリへ飛ばす**。

## URL

```
GET https://relay.example.com/s/<投稿のイベントid>
```

## レスポンス

- 投稿が見つかる: `og:title`(「名前 (@ハンドル) さんのポスト」)、`og:description`(本文 140字、`:emoji:` を除去、空白を詰める。本文が空なら「画像のポスト」など)、`og:image`、`og:url`、`twitter:card`、`article:published_time` を含む HTML。末尾で JS により `https://<アプリ>/#/post/<id>` へ `location.replace`。`<noscript>` にはリンク。
- 見つからない/削除済み: 404 + トップへ飛ばす HTML(`noindex`)。
- `Cache-Control: public, max-age=300`、`Set-Cookie` は返さない。

## 画像の選び方(現行と同じ)

1. センシティブ判定([sensitive-content](sensitive-content.md))がなく、画像付き → 最初の画像、`summary_large_image`。
2. それ以外で作者のアバターあり → アバター、`summary`。
3. どちらも無し → サイト既定画像。
**センシティブな画像はカードに出さない。**

## 注意

- HTML に出す値は**必ずエスケープ**(`htmlspecialchars`)。`json_encode` に `JSON_HEX_TAG | JSON_HEX_AMP`。
- リレーはイベントの署名を検証済みなので、`content` を信頼して描いてよい(ただしリレーに保存されているものだけ)。
- どのリレーのどの投稿かを示すため、`e` タグの `relay-hint` にこのリレーのURLを入れて共有する運用を推奨。

## クライアントの共有ボタン

`https://<リレー>/s/<id>` をコピー。アプリ内ルーティングは [identity-and-keys](identity-and-keys.md) に書いた `#/post/<id>` `#/profile/<pubkey または handle>`。
