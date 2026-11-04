# 03 タグ

`tags` は `["名前", 値1, 値2, …]` の配列の配列。**名前が1文字のタグだけが索引され、フィルタで検索できる**(`#e` など)。それ以外は付加情報で、知らなければ無視する。

## 標準タグ

| タグ | 形式 | 索引 | 意味 |
|---|---|---|---|
| `e` | `["e", <event-id>, <relay-hint>?, <marker>?]` | ✓ | イベントへの参照。返信では「返信先」、リアクション・リポストでは「対象」 |
| `p` | `["p", <pubkey>, <relay-hint>?]` | ✓ | ユーザーへの参照(メンション・宛先・対象の作者) |
| `q` | `["q", <event-id>, <relay-hint>?]` | ✓ | 引用元 |
| `t` | `["t", <hashtag>]` | ✓ | ハッシュタグ。小文字に正規化 |
| `d` | `["d", <identifier>]` | ✓ | addressable イベントの識別子 |
| `a` | `["a", "<kind>:<pubkey>:<d>", <relay-hint>?]` | ✓ | addressable イベントへの参照 |
| `k` | `["k", "<kind>"]` | ✓ | 対象の kind(削除などで使う) |
| `r` | `["r", <url>]` | ✓ | 参照URL |
| `emoji` | `["emoji", <name>, <url>]` | | カスタム絵文字([../features/custom-emoji.md](../features/custom-emoji.md)) |
| `imeta` | `["imeta", "url …", "m …", …]` | | メディア情報([../features/media.md](../features/media.md)) |
| `cw` | `["cw", <reason>?]` | | センシティブ指定([../features/sensitive-content.md](../features/sensitive-content.md)) |
| `expiration` | `["expiration", <unix>]` | | この時刻以降は削除してよい(リレーは MAY 対応) |
| `handle` `deco` `badge` | プロフィール用 | | [../kinds/10000-profile.md](../kinds/10000-profile.md) |
| `relay` `challenge` | 認証用 | | [../kinds/20000-relay-auth.md](../kinds/20000-relay-auth.md) |
| `u` `x` `method` | HTTP認証用(`u` `x` は1文字なので索引対象) | ✓(`u` `x`) | [../kinds/20001-http-auth.md](../kinds/20001-http-auth.md) |

## 規則

- `relay-hint` は不要なら `""`(位置を詰めるため)または省略。
- 同じ名前のタグは複数回書ける(`p` を複数、`t` を複数)。
- `e` の `marker` は未指定なら「返信先(親)」。将来 `"root"` などを足せる。
- 索引されるのは各タグの**先頭の値**だけ(`["e", id, hint]` なら `id`)。
- 1イベントあたりのタグ数・長さの上限はリレーのポリシー([08-relay.md](08-relay.md))。

## 新しいタグを作る

- 検索したいなら**1文字**の名前を使う(衝突回避のため [09-extending.md](09-extending.md) の登録表に追記)。
- 検索不要なら任意の長さの名前でよい。
