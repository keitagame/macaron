# 02 kind の範囲と保存ルール

kind の数値範囲で、リレーの保存方法が決まる。**kind ごとの特別扱いは作らない。**

| 範囲 | 名前 | リレーの動作 |
|---|---|---|
| 0〜9999 | regular | すべて保存する |
| 10000〜19999 | replaceable | `(pubkey, kind)` ごとに最新の1件だけ保存 |
| 20000〜29999 | ephemeral | 保存しない。購読中の接続にだけ配る |
| 30000〜39999 | addressable | `(pubkey, kind, dタグ)` ごとに最新の1件だけ保存 |
| 40000〜65535 | private use | 実験・私用。regular として扱う。仕様化されない |

## 「最新」の決め方

1. `created_at` が大きい方
2. 同じなら `id` が辞書順で小さい方

古い方を後から受け取っても、保存済みの新しい方を残して `OK true`(`duplicate:` 付き)で返す。

## addressable の参照

`a` タグに `"<kind>:<pubkey>:<d値>"` を書く。

## 現在の割り当て

| kind | 名前 |
|---|---|
| 1 | [post](../kinds/1-post.md) |
| 2 | [repost](../kinds/2-repost.md) |
| 3 | [reaction](../kinds/3-reaction.md) |
| 4 | [direct-message](../kinds/4-direct-message.md) |
| 5 | [deletion](../kinds/5-deletion.md) |
| 6 | [flag](../kinds/6-flag.md) |
| 7 | [view](../kinds/7-view.md) |
| 8 | [follow](../kinds/8-follow.md) |
| 9 | [bookmark](../kinds/9-bookmark.md) |
| 10000 | [profile](../kinds/10000-profile.md) |
| 10001 | [relay-list](../kinds/10001-relay-list.md) |
| 20000 | [relay-auth](../kinds/20000-relay-auth.md) |
| 20001 | [http-auth](../kinds/20001-http-auth.md) |
| 30000 | [emoji](../kinds/30000-emoji.md) |
| 30001 | [setting](../kinds/30001-setting.md) |
| 30002 | [trends](../kinds/30002-trends.md) |

## 設計メモ

- フォロー・ブックマークを「リスト1個を置き換える」方式にしなかった理由: 複数端末の同時操作で**最後の書き込みが他方を消す**(lost update)ため。1件=1イベントなら競合しない。通知(新しくフォローされた)や件数(`COUNT`)もそのまま取れる。
- 取り消しは削除イベント(kind 5)で行う。トグルUI(いいね / いいね解除)は「kind 3 を送る / kind 5 を送る」に対応する。
