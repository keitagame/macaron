# 機能: カスタム絵文字

- 使う kind: [30000](../kinds/30000-emoji.md) / [1](../kinds/1-post.md) / [3](../kinds/3-reaction.md) / [10000](../kinds/10000-profile.md)
- 現行API: `emoji_add` `emoji_delete` `emoji_list`

## 作る

1. 画像を `POST /media?use=emoji` にアップロード(128px以内に再エンコード、アニメGIFは256px・512KB以内)→ URL。
2. [kind 30000](../kinds/30000-emoji.md) を署名して送る(`d` = 名前、`url`)。

名前は `^[a-z0-9_]{2,32}$`。1人50個まで(リレーのポリシー)。

## 使う

投稿・リアクション・プロフィールに `:name:` を書き、**必ず同じ名前の `emoji` タグを付ける**:

```json
{ "kind": 1, "content": "やった :party:", "tags": [["emoji","party","https://cdn.example.com/e/party.webp"]] }
```

- 受け手はタグだけで描画できる(作者の絵文字リストを探しに行かない)。
- 同じ名前が別人のものとぶつかっても、**そのイベントのタグ**が優先。
- 現行の `emoji_map()` による後付け解決は不要になる。

## ピッカー

自分と、フォロー中の人の [kind 30000](../kinds/30000-emoji.md) を集めて表示。同名は「名前 (作者名)」で区別する。

## リアクションに使う

[kind 3](../kinds/3-reaction.md) の `content` を `:name:` にして `emoji` タグを付ける。

## 削除

[kind 5](../kinds/5-deletion.md)(`a` タグ)で定義を消す。**過去の投稿のタグは残る**ので表示はそのまま(画像が消えたら `:name:` と文字表示)。

## バッジ・名前での利用

[profile-decoration-and-badges](profile-decoration-and-badges.md)。

## 拡張ポイント

- セット(まとめて共有)・ライセンス・タグ付けは [kind 30000](../kinds/30000-emoji.md) にタグを足すだけ。
