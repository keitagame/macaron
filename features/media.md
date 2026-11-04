# 機能: 画像・動画

- 使う kind: [1](../kinds/1-post.md) の `imeta` / [20001](../kinds/20001-http-auth.md)
- 現行API: `upload_media` `delete_media`(`media` 表、GDで再エンコード)

## 考え方

イベントには**バイナリを入れない**。ファイルは HTTP のメディアサーバーに置き、イベントは URL とハッシュだけを持つ。メディアサーバーはリレーと同居でも別でもよい。

## アップロード

```
POST /media
Authorization: Macaron <kind 20001, x=<ファイルのsha256>>
Content-Type: image/jpeg
<バイナリ>
→ 200 { "url": "https://cdn.example.com/m/<sha256>.jpg", "sha256": "…", "type": "image/jpeg", "w": 1200, "h": 800, "size": 183000 }
```

- 署名者の pubkey がファイルの「所有者」。
- **サーバーが検査・再エンコードして**から保存する(現行と同じ): 形式の確認(JPEG/PNG/WebP/GIF/MP4/WebM/MOV)、最大辺 2048px、アニメGIFはそのまま、EXIF位置情報の除去。ハッシュは**保存後のファイル**のもの(返ってきた値を使う)。
- 上限: 画像 10MB・動画 50MB・未投稿の保留 8件など(現行の定数)はサーバーの `limits` に出す。
- 同じ内容は同じ sha256 → **重複排除**・キャッシュが簡単。

## 投稿に付ける

```json
["imeta", "url https://cdn.example.com/m/ab12.jpg", "m image/jpeg", "dim 1200x800", "x ab12…", "alt 夕焼けの写真"]
```

- `imeta` の各要素は `キー 値` の文字列。`url` と `m`(MIMEタイプ)は必須。
- 画像は最大 `max_media`(現行 4)。動画は1本のみで画像と併用不可(現行)。
- 表示順 = タグの並び順(現行の `pos`)。
- `x`(sha256)があれば、クライアントは取得後に**ハッシュを確認**できる(CDN差し替え対策)。

## 削除

`DELETE /media/<sha256>`(+ 認証)。投稿の削除([kind 5](../kinds/5-deletion.md))ではファイルは消えない(他の投稿が使っているかもしれない)。メディアサーバーは「所有者が削除した」「どの投稿からも参照されず一定期間たった」ものを消す。

## アイコン・ヘッダー・デコ・絵文字

同じ `POST /media` を使い、用途ごとに**サーバー側の検査を変える**: `?use=avatar|banner|deco|emoji`(アスペクト・最大サイズ・透過必須かなど。現行 `upload_image` `upload_deco` `emoji_add` の各制限に相当)。

## 拡張ポイント

- 動画のサムネイル・変換は `["thumb url …"]` を `imeta` に足す。
- 外部のメディアサーバーを自由に選べる(URLが `https:` なら何でもよい)。
