# 機能: タイムライン

- 使う kind: [1](../kinds/1-post.md) [2](../kinds/2-repost.md) [8](../kinds/8-follow.md)
- 現行API: `timeline`(`scope` = 全体 / `following` / `rec`)

## 全体

```
REQ "tl" [
  {"kinds":[1], "without":["e"], "limit":20},
  {"kinds":[2], "limit":20}
]
```

- 1つめ=返信ではないトップレベル投稿(現行 `parent_id IS NULL`)、2つめ=リポスト。
- 受け取ったら**同じ投稿idは1つにまとめる**。リポストがあれば `created_at` の新しい方(リポスト)を並び順に使い、「〇〇さんがリポスト」を付ける(現行 `rtBy`)。
- 古い順の取りこぼしを避けるため、現行どおり**多めに取って**(×3)重複を除いてから `limit` に切り詰める。

## フォロー中

```
1) REQ "f" [{"kinds":[8],"authors":["<me>"]}]   → フォロー先の一覧 F
2) REQ "tl" [
     {"kinds":[1],"without":["e"],"authors":F+[me],"limit":20},
     {"kinds":[2],"authors":F+[me],"limit":20}
   ]
```

- 自分も含める(現行と同じ)。
- `authors` が上限(`max_authors`)を超えるなら、拡張 `feed:following`:

```
REQ "tl" [{"feed":"following","of":"<me>","limit":20}]
```
リレーが kind 8 から展開して返す。`feed` 非対応のリレーでは `authors` を分割して複数 REQ。

## おすすめ

`feed: "rec"` — [recommendation-feed](recommendation-feed.md)。

## ページ送り

`until` を最後の `created_at` にして再 `REQ`(`id`で重複排除)。`feed` 系は `EOSE` の `meta.next` を次の `REQ` に渡す。

## 新着

`EOSE` 後の `EVENT` がそのまま新着。画面の一番上に溜めて「新しい投稿があります」バーを出し、クリックで反映するのが推奨(読んでいる位置が動かない)。ポーリング不要([live-updates](live-updates.md))。

## 未ログイン

`authors` を使わない「全体」だけ見られる(現行と同じ)。`feed:rec` は未ログインでも使えるが、個人化されない。

## 複数リレー

[multi-relay](multi-relay.md)。同じ投稿が複数リレーから届くので `id` で重複排除。
