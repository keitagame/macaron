# 09 拡張のしかた

## 互換性ルール

1. **既存の kind・tag・フィルタの意味を変えない。** 変えたくなったら新しい名前を作る。
2. **知らないものは無視する。** 知らない kind は保存・転送だけして解釈しない。知らない tag・JSON キーは捨てずに保持する(置き換え型で上書きするとき、知らないキーを消さない)。
3. **必須にしない。** 新機能が無くても旧クライアントの基本機能(投稿・閲覧)が壊れないようにする。
4. **機能はリレーが宣言する。** 新しい機能は `GET /` の `features` に名前を足す。

## 新しい kind を作る手順

1. 保存方法を決めて [02-kind-ranges.md](02-kind-ranges.md) の範囲から未使用の番号を選ぶ。
   - 履歴が要る → regular / 最新だけ → replaceable / キー別に最新だけ → addressable / 流すだけ → ephemeral
2. `kinds/<番号>-<名前>.md` を書く(テンプレは下)。
3. READMEの表と 02 の割り当て表に1行足す。
4. 実験段階は 40000番台(private use)で始め、安定したら正式番号を取る。

### kind md のテンプレート

```
# kind N: 名前
- 保存 / 現行API / 関連機能
## 目的
## 構造(JSON例)
## tags / content
## リレーの検証
## クライアントの扱い
## 拡張ポイント
```

## 新しい機能を作る手順

1. 既存の kind + tag の組み合わせで表現できないか考える(例: 通知は新 kind を作らず `#p` 検索で実現している)。
2. できなければ kind を足す。リレー側の計算が要るなら、拡張フィルタ項目(`feed` のように)か、リレー署名イベント([kinds/30002-trends.md](../kinds/30002-trends.md))か、HTTPエンドポイント(`/x/…`)のどれか1つで提供する。
3. `features/<名前>.md` に書く。

## 登録表(1文字タグ・フィルタ項目)

衝突を避けるための一覧。新規追加時はここに書く。

| 名前 | 用途 | 定義 |
|---|---|---|
| `e` `p` `q` `t` `d` `a` `k` `r` | 標準タグ | [03-tags.md](03-tags.md) |
| `u` | HTTP認証の対象URL | [../kinds/20001-http-auth.md](../kinds/20001-http-auth.md) |
| `x` | ファイルの sha256 | [../kinds/20001-http-auth.md](../kinds/20001-http-auth.md) / [../features/media.md](../features/media.md) |
| `ids` `authors` `kinds` `since` `until` `limit` `has` `without` | 標準フィルタ | [04-filters.md](04-filters.md) |
| `search` `feed` `group` | 拡張フィルタ | features/ |

## 例: 投票機能を足す

- 投票 = kind 1 に `["poll", "選択肢A", "選択肢B"]` と `["expiration", …]` を付ける。タグを足すだけで、旧クライアントは普通の投稿として表示する。
- 回答 = 新 kind(regular)`["e", pollId]` + `["response", "0"]`。1人1票は「同じ `pubkey`+`e`」の重複をリレーが `duplicate:` で拒否して実現(kind 3 と同じ方式)。
- 集計 = `COUNT` に `group: ["content"]`。リレーに新しいコードは不要。

## アルゴリズムの追加(署名方式など)

pubkey を 64 hex 以外の形式にする場合は、`<alg>:<hex>` 形式(例 `p256:…`)にし、`:` を含まない従来の64hexは Ed25519 とする。ID の計算は共通のまま。
