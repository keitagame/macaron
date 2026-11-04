# kind 6: フラグ (flag)

- 保存: regular
- 現行API: `sens_vote`(他人の報告)、`set_sensitive`(自分の指定)
- 関連機能: [sensitive-content](../features/sensitive-content.md)

## 目的

投稿に「この投稿は〇〇です」という申告を付ける。まずは `sensitive`(センシティブ・モザイク対象)だけを定義する。スパム等は後から `content` の値を足す。

## 構造

```json
{
  "kind": 6,
  "content": "sensitive",
  "tags": [["e", "<投稿id>"], ["p", "<投稿の作者>"]]
}
```

- `e` はちょうど1つ。`content` は申告の種類。現在の定義: `sensitive`。
- 取り消し = [kind 5](5-deletion.md)。

## リレーの検証

- 同じ `(pubkey, e, content)` が既にある → `duplicate:`。
- **他人の投稿への `sensitive`** は、`pubkey` の初出時刻(リレーが最初にそのpubkeyのイベントを見た時刻)から `flag_min_account_age`(現行 86400秒)以上たっていること。満たさなければ `restricted: account too new`(現行の `SENS_MIN_AGE`)。
- 対象にメディア(`imeta`)が無い → `invalid:`(現行「画像・動画のないポストには報告できません」)。
- 時間あたりの上限(現行 60回/時)はレート制限([rate-limit-and-abuse](../features/rate-limit-and-abuse.md))。

## クライアントの扱い

認定ルール(現行と同じ):

```
センシティブ表示 = 投稿に cw タグがある
                 || 作者本人の flag(sensitive) がある
                 || 作者以外の異なる pubkey からの flag(sensitive) が sensitive_threshold 以上(現行 3)
```

- 作者以外の件数: `COUNT {kinds:[6], "#e":[id], group:["content"]}` から作者分を引く(または `authors` で除外した別フィルタ)。
- 作者本人は、他人の認定を外せない(自分の flag を消しても、他人の分は残る)。

## 拡張ポイント

- `content` に値を足す: `spam` `harassment` など。それぞれ別のしきい値・別の表示をクライアントが決める。
- 信頼できる pubkey(モデレーター)の flag だけを数える「信頼リスト」方式にも拡張できる。
