# 機能: ハンドル(@ユーザー名)

- 使う kind: [10000 プロフィール](../kinds/10000-profile.md) の `handle` タグ / [20001](../kinds/20001-http-auth.md)
- 現行: `users.handle`(`^[A-Za-z0-9_]{3,15}$`、大文字小文字を区別せず一意、予約語あり)

## 考え方

公開鍵は読めないので、**人間が読める名前 `名前@ドメイン`** を別に持つ。名前の一意性は**ドメインが保証**し、プロトコルは保証しない。ドメイン = そのリレーの運営者。

## 確認(クライアント)

プロフィールの `["handle", "macaron@example.com"]` を見たら:

```
GET https://example.com/.well-known/macaron.json?name=macaron
→ { "names": { "macaron": "<pubkey>" }, "relays": { "<pubkey>": ["wss://relay.example.com"] } }
```

- 返った pubkey がプロフィールの pubkey と**一致したときだけ**ハンドルを表示。
- 表示は `@macaron`(リレーと同じドメインなら `@ドメイン` を省略)、他ドメインは `@macaron@example.com`。
- 結果は短時間キャッシュ。失敗しても投稿の表示は止めない。

## 取得(リレー運営者側)

```
POST /x/handle
Authorization: Macaron <kind 20001>
{ "name": "macaron" }
→ 200 { "ok": true } / 409 {"error":"taken"} / 403 {"error":"reserved"}
```

- 検査は現行どおり: 正規表現 `^[A-Za-z0-9_]{3,15}$`、予約語(`admin` `root` `api` …)、重複なし。
- pubkey ↔ 名前の対応を DB に持ち、`/.well-known/macaron.json` で返す。
- 1 pubkey 1ハンドル(変更は可、旧名は一定期間予約)。**登録制・有料化・招待制はここで決める**(プロトコルの外)。

## メンション・検索への影響

- メンションは `@名前` ではなく**pubkeyの参照**で書く([mention-and-hashtag](mention-and-hashtag.md))。入力補完でハンドルを pubkey に変換する。
- ユーザー検索 `@macaron` は、リレーの `search`([search](search.md))が `handle` タグも対象にする。

## 拡張ポイント

- DNS ではなく別方式(公開鍵に別の署名で「この名前は私」を証明)にしたければ、プロフィールにタグを足せる。
