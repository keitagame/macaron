# 機能: センシティブ(自己申告 + コミュニティ認定)

- 使う kind: [1](../kinds/1-post.md) の `cw` / [6 フラグ](../kinds/6-flag.md) / [5](../kinds/5-deletion.md)
- 現行API: `post` の `sensitive`、`set_sensitive`、`sens_vote`(`SENS_THRESHOLD`=3、`SENS_MIN_AGE`=24時間)

## 3つの経路を1つの仕組みで

| 現行 | プロトコル |
|---|---|
| 投稿時に「センシティブ」を指定 | 投稿に `["cw"]` タグ |
| 後から本人が付け外し | 自分の [kind 6](../kinds/6-flag.md)(`content:"sensitive"`)を作る / [削除する](../kinds/5-deletion.md) |
| 他人が報告 → 3人でモザイク | 他人の [kind 6](../kinds/6-flag.md) が3件以上 |

## 表示ルール(クライアント)

```
モザイクにする =
     投稿に cw タグがある
  || 作者自身の flag(sensitive) がある
  || 作者以外の pubkey の flag(sensitive) が N 件以上   // N = policy.sensitive_threshold(現行 3)
```

- 画像・動画がある投稿だけが対象(メディアなしには付けられない)。
- 数え方: `COUNT {kinds:[6],"#e":[ids…],group:["e","content"]}`(作者分を含むので、作者の flag は別フィルタで見て差し引く)。
- 本人は、他人の認定を**外せない**。本人が外せるのは自分の指定だけ。
- 引用に入ったメディア・通知の抜粋にも同じ判定を使う。

## 悪用対策

- **新しいアカウントの報告は無効**: リレーが「pubkey の初出から24時間未満」の flag を `restricted:` で拒否する([kind 6](../kinds/6-flag.md))。
- 時間あたりの回数制限([rate-limit-and-abuse](rate-limit-and-abuse.md))。
- 分散環境では pubkey は無限に作れる。より強い対策として「フォロー関係・信頼リストにいる人の flag だけ数える」をクライアント設定にできる。
- リレー運営者が違反投稿を**配信しない**のは自由(`blocked:`)。

## 拡張ポイント

- `cw` の理由(`["cw","violence"]`)で「暴力」「ネタバレ」などの折りたたみ表示。
- 認定のしきい値・信頼者を設定画面で変えられるようにする(クライアントの設定=[kind 30001](../kinds/30001-setting.md))。
