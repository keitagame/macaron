# 08 リレー

リレーの責務は **検証 → 保存 → 配信** の3つだけ。アカウント管理・パスワード・セッションは持たない。

## イベント受理の手順

`EVENT` を受け取ったら、この順に処理する。途中で失敗したら `["OK", id, false, "接頭辞: 理由"]`。

| # | 検査 | 失敗時 |
|---|---|---|
| 1 | サイズが `max_event_bytes` 以下 | `invalid:` |
| 2 | 形式・型 | `invalid:` |
| 3 | `id` 再計算 | `invalid:` |
| 4 | `sig` 検証 | `invalid:` |
| 5 | `created_at` が現在時刻より `created_at_window` 秒(目安 15分)以上**未来**でない。過去は制限しない(移行・ミラー・オフライン投稿のため)。ephemeral(20000番台)だけは過去も ±`created_at_window` | `invalid:` |
| 6 | kind が許可されている(許可リスト or 禁止リスト) | `blocked:` |
| 7 | タグ数・長さの上限 | `invalid:` |
| 8 | **kind ごとの検証**(各 kinds/*.md の「リレーの検証」) | `invalid:` / `restricted:` |
| 9 | 認証が必要な kind か | `auth-required:` |
| 10 | レート制限([../features/rate-limit-and-abuse.md](../features/rate-limit-and-abuse.md)) | `rate-limited:` |
| 11 | 既に保存済みか(id一致、削除済みidも含む) | `duplicate:`(`OK true`) |
| 12 | 保存([02-kind-ranges.md](02-kind-ranges.md))、削除イベントなら対象を削除 | `error:` |
| 13 | マッチする購読へ配信 | — |

## 最小ストレージ(SQLite)

```sql
CREATE TABLE events(
  seq        INTEGER PRIMARY KEY AUTOINCREMENT,   -- SSEのidに使う
  id         TEXT NOT NULL UNIQUE,
  pubkey     TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  kind       INTEGER NOT NULL,
  d          TEXT,                                 -- addressable用
  json       TEXT NOT NULL                         -- イベント丸ごと
);
CREATE INDEX ev_time ON events(created_at DESC, id DESC);
CREATE INDEX ev_author ON events(pubkey, kind, created_at DESC);
CREATE TABLE tags(                                 -- 1文字タグだけ索引
  seq INTEGER NOT NULL, name TEXT NOT NULL, value TEXT NOT NULL, created_at INTEGER NOT NULL,
  PRIMARY KEY(seq, name, value)
);
CREATE INDEX tg_q ON tags(name, value, created_at DESC);
CREATE TABLE tombstones(id TEXT PRIMARY KEY, pubkey TEXT NOT NULL);   -- 削除済みid
```

現行の `posts` `likes` `reactions` `reposts` `bookmarks` `follows` `dms` `notifs` `views` `sens_votes` `emojis` の11テーブルが、この3テーブルに集約される。
(必要ならカウンタ用のキャッシュテーブルを足してよい。仕様外の最適化。)

## 情報ドキュメント

```
GET /
Accept: application/macaron+json
```

```json
{
  "name": "Z Relay",
  "description": "…",
  "pubkey": "<リレーの公開鍵(トレンド署名用)>",
  "contact": "admin@example.com",
  "software": "macaron-relay-php",
  "version": "0.1.0",
  "macaron": {
    "transports": ["ws", "sse"],
    "kinds": [1,2,3,4,5,6,7,8,9,10000,10001,30000,30001,30002],
    "features": ["search", "feed:rec", "count", "count:group", "ogp", "media", "handle"],
    "endpoints": { "msg": "/msg", "sse": "/sse", "ws": "/ws", "media": "/media", "ogp": "/x/ogp", "handle": "/x/handle", "share": "/s" },
    "limits": {
      "max_event_bytes": 65536, "max_content_length": 500, "max_tags": 100,
      "max_filters": 10, "max_authors": 2000, "max_limit": 300, "default_limit": 20,
      "max_subscriptions": 20, "max_batch": 50, "created_at_window": 900
    },
    "policy": {
      "signup": "open",
      "auth_required_kinds": [4, 7],
      "flag_min_account_age": 86400,
      "sensitive_threshold": 3
    }
  }
}
```

- クライアントはこの値で入力欄の文字数制限・機能の表示を決める(ハードコードしない)。
- 知らない項目は無視する。リレーは自由に項目を足せる。
- 現行の `SIGNUP_OPEN` `MAX_POST` `SENS_THRESHOLD` などの定数は、ここに出す設定値に当たる。

## 運営者ができること(ポリシー)

禁止kind・禁止pubkey・禁止語・登録制(handle取得を要認証)・有料化など。**イベントの仕様は変えずに拒否するだけ**。拒否は必ず `blocked:` か `restricted:` で理由を返す(黙って捨てない)。
