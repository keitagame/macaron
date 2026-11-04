# 機能: アイコンデコレーション・バッジ

- 使う kind: [10000 プロフィール](../kinds/10000-profile.md)
- 現行API: `upload_deco` `remove_deco`、`profile` の `badges`(最大 `MAX_BADGES` = 5)

## アイコンデコレーション

アバターの周りに重ねる透過画像。

1. 透過のある PNG / WebP / GIF を `POST /media?use=deco`(256px以内・3MB以内に再エンコード。透過がなければ拒否。アニメGIFは512px・1MB)。
2. 返ってきた URL を、プロフィールの `["deco", url]` に入れて新しい kind 10000 を署名。

外す = `deco` タグ無しで kind 10000 を作り直す(replaceable なので古い方は消える)。

## バッジ

表示名の横に出す絵文字(最大5)。

```json
["badge", "🔥"]
["badge", ":party:", "https://cdn.example.com/e/party.webp"]
```

- 標準絵文字は1つだけ(grapheme 1個・`valid_emoji` 相当)。
- カスタム絵文字はURLも書く([custom-emoji](custom-emoji.md))。消えたカスタム絵文字は表示しない(現行 `badges_out`)。
- 順序 = タグの並び順。

## 表示

- プロフィールが取れたら、ユーザー名の横にバッジ、アイコンの上にデコを重ねる。
- 取れない/不正ならバッジ・デコなしで表示(壊さない)。

## 拡張ポイント

- バッジを「他人から贈られる」形にしたい場合は、別 kind(贈る側が署名、受ける側がプロフィールで採用)。今は不要。
