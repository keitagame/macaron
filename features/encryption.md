# 機能: 暗号化(DM・ブックマーク・設定)

- 使う kind: [4](../kinds/4-direct-message.md) [9](../kinds/9-bookmark.md) [30001](../kinds/30001-setting.md)

## 形式 `m1:`

```
content = "m1:" + base64( nonce(24バイト) || 暗号文 )
```

- 方式: NaCl `crypto_box`(X25519 + XSalsa20-Poly1305)。libsodium の `crypto_box_easy`、tweetnacl の `nacl.box` と同じ。
- 鍵: Ed25519 の鍵を **X25519 に変換**して使う。
  - 公開鍵: `crypto_sign_ed25519_pk_to_curve25519`
  - 秘密鍵: `crypto_sign_ed25519_sk_to_curve25519`
- nonce は毎回乱数24バイト。再利用しない。

### 宛先ごと

| 用途 | 送信者の秘密鍵 | 相手の公開鍵 |
|---|---|---|
| DM | 自分 | 宛先 |
| ブックマーク・設定(自分用) | 自分 | **自分** |

### 実装メモ

- JS: `@noble/curves` の `edwardsToMontgomeryPub` / `edwardsToMontgomeryPriv` + `tweetnacl` の `box`、または `libsodium-wrappers`。
- PHP: `sodium_crypto_sign_ed25519_pk_to_curve25519` / `…_sk_to_curve25519` + `sodium_crypto_box`。リレー側は**復号できない**(秘密鍵を持たない)ので、PHPでの復号は必要ない。クライアントがPHPのとき用。

## 平文の形式

- DM: UTF-8 テキスト。
- ブックマーク・設定: JSON。

## パスワードで守った鍵の書き出し([identity-and-keys](identity-and-keys.md))

`m1:` + base64( salt(16) || nonce(24) || `secretbox`(秘密鍵) )。鍵は `scrypt(パスワード, salt, N=2^15, r=8, p=1)` の32バイト。

## 既知の限界(v1)

- **前方秘匿性なし**: 秘密鍵が漏れたら過去のDMも読まれる。
- **メタデータ(誰が誰に・いつ)は見える**。
- パディングなし。
- `m2:` を別形式として追加して解決できる(例: 使い捨て鍵で包む方式)。旧クライアントは `m2:` を「未対応」と表示すればよい。

## 拡張ポイント

- 接頭辞(`m1:` `m2:` …)で形式を識別。未知の接頭辞は復号を試みず、そのまま捨てずに保持する。
