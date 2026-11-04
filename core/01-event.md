# 01 イベント

プロトコル上のデータはすべて **イベント** 1種類。

## 構造

```json
{
  "id":         "<32バイトのsha256 hex 64文字>",
  "pubkey":     "<Ed25519公開鍵 hex 64文字>",
  "created_at": 1730700000,
  "kind":       1,
  "tags":       [["t", "hello"], ["p", "<pubkey>"]],
  "content":    "こんにちは #hello",
  "sig":        "<Ed25519署名 hex 128文字>"
}
```

| 項目 | 型 | 説明 |
|---|---|---|
| id | string | 下記の正規化文字列の SHA-256 |
| pubkey | string | 作者の公開鍵(= アカウントID) |
| created_at | int | 作成時刻(UNIX秒)。ソートの基準 |
| kind | int | 0〜65535。意味は kinds/ で定義 |
| tags | string[][] | 構造化データ。[03-tags.md](03-tags.md) |
| content | string | 本文または kind ごとの任意文字列 |
| sig | string | `id` のバイト列(32B)への Ed25519 署名 |

## ID の計算

```
id = hex( sha256( utf8( JSON配列 [0, pubkey, created_at, kind, tags, content] ) ) )
```

JSON のシリアライズは **空白なし** で、文字列のエスケープは次だけ:

| 文字 | 出力 |
|---|---|
| `"` | `\"` |
| `\` | `\\` |
| 改行 / CR / タブ / BS / FF | `\n` `\r` `\t` `\b` `\f` |
| 上記以外の U+0000〜U+001F | `\u00XX`(小文字hex) |
| それ以外(日本語・絵文字・`/`・U+2028/2029 を含む) | **エスケープせず UTF-8 のまま** |

### 実装メモ
- JavaScript: `JSON.stringify` は上記とほぼ一致。孤立サロゲートだけ `\udXXX` になるので、含むイベントは `invalid:` で拒否する。
- PHP: `json_encode($a, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_LINE_TERMINATORS)`。現行 api.php の `out()` は `LINE_TERMINATORS` を付けていないので、ID計算では必ず付ける(付けないと U+2028 を含む本文でIDがずれる)。
- 数値は整数のみ。`created_at` に小数・指数表記を使わない。

## 署名

- アルゴリズム: **Ed25519**(RFC 8032)。署名対象は `id` の生32バイト。
- 理由: WebCrypto(`Ed25519`)・libsodium(PHP標準同梱)・各言語の標準的なライブラリで使えるため。
- 将来別アルゴリズムを足す場合は pubkey の形式(長さ・接頭辞)で判別する([09-extending.md](09-extending.md))。

## 検証(リレー・クライアント共通)

1. 必須項目が揃い、型が正しい
2. `id` を再計算して一致する
3. `sig` を `pubkey` で検証できる

失敗したイベントは保存も転送もしない。**クライアントもリレーから受け取ったイベントを必ず自分で検証する**(リレーを信用しない)。

## 不変性

イベントは作成後に変更できない。「編集」「取り消し」は新しいイベントで表現する
(置き換え型 kind は [02-kind-ranges.md](02-kind-ranges.md)、取り消しは [../kinds/5-deletion.md](../kinds/5-deletion.md))。
