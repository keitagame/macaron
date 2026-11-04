# 現行 api.php ⇔ Macaron プロトコル 対応表

## アクション

| 現行 `?a=` | Macaron での実現 | 参照 |
|---|---|---|
| `signup` | 鍵を作る + [kind 10000](../kinds/10000-profile.md) + ハンドル取得 | [identity-and-keys](../features/identity-and-keys.md) [handle](../features/handle.md) |
| `login` / `logout` | 鍵の読み込み / 破棄(サーバー側のセッション無し) | [identity-and-keys](../features/identity-and-keys.md) |
| `me` | 自分のプロフィール `REQ` + 未読 `COUNT` + 通知の購読 | [notifications](../features/notifications.md) [live-updates](../features/live-updates.md) |
| `timeline`(全体 / following) | `REQ` kind 1 `without:["e"]` + kind 2 | [timeline](../features/timeline.md) |
| `timeline`(rec) | `REQ {"feed":"rec"}` | [recommendation-feed](../features/recommendation-feed.md) |
| `thread` | `REQ ids` + `#e` | [thread-and-reply](../features/thread-and-reply.md) |
| `post` | [kind 1](../kinds/1-post.md) を署名して `EVENT` | |
| `post` + `parent_id` | kind 1 + `e`/`p` タグ | [thread-and-reply](../features/thread-and-reply.md) |
| `post` + `quote_id` | kind 1 + `q`/`p` タグ | [quote](../features/quote.md) |
| `post` + `media` | kind 1 + `imeta` タグ | [media](../features/media.md) |
| `post` + `sensitive` | kind 1 + `cw` タグ | [sensitive-content](../features/sensitive-content.md) |
| `set_sensitive` | 自分の [kind 6](../kinds/6-flag.md) を作る / 削除 | [sensitive-content](../features/sensitive-content.md) |
| `sens_vote` | 他人の投稿への [kind 6](../kinds/6-flag.md) を作る / 削除 | [sensitive-content](../features/sensitive-content.md) |
| `delete_post` | [kind 5](../kinds/5-deletion.md) | |
| `like`(トグル) | [kind 3](../kinds/3-reaction.md) `+` / その削除(kind 5) | |
| `react`(トグル) | kind 3 絵文字 / その削除 | [custom-emoji](../features/custom-emoji.md) |
| `repost`(トグル) | [kind 2](../kinds/2-repost.md) / その削除 | |
| `bookmark`(トグル) / `bookmarks` | [kind 9](../kinds/9-bookmark.md) / その削除 / 取得して復号 | [encryption](../features/encryption.md) |
| `follow`(トグル) | [kind 8](../kinds/8-follow.md) / その削除 | |
| `follows` | `REQ kind 8`(`authors` or `#p`) | |
| `user` | kind 10000 `REQ` + `COUNT` 数本 + 投稿 `REQ` | [counts](../features/counts.md) |
| `profile` | 新しい [kind 10000](../kinds/10000-profile.md) | |
| `upload_image` / `remove_image` | `POST /media?use=avatar\|banner` → kind 10000 更新 | [media](../features/media.md) |
| `upload_deco` / `remove_deco` | `POST /media?use=deco` → kind 10000 の `deco` タグ | [profile-decoration-and-badges](../features/profile-decoration-and-badges.md) |
| `search` | `search` フィルタ / `#t` | [search](../features/search.md) |
| `trends` | `REQ kind 30002`(リレー署名) | [trends](../features/trends.md) |
| `notifs` | `REQ #p` + 種類の判定 | [notifications](../features/notifications.md) |
| `convos` / `dm` | `REQ kind 4`(自分発 + 自分宛て)+ 復号 | [kind 4](../kinds/4-direct-message.md) |
| `dm_send` | [kind 4](../kinds/4-direct-message.md) | |
| `upload_media` / `delete_media` | `POST /media` / `DELETE /media/<sha256>` | [media](../features/media.md) |
| `emoji_list` / `emoji_add` / `emoji_delete` | `REQ kind 30000` / 作る / kind 5(`a`) | [custom-emoji](../features/custom-emoji.md) |
| `view_seen` / `noview` | [kind 7](../kinds/7-view.md) / [kind 30001](../kinds/30001-setting.md) `no_view` + kind 5 `k:7` | [view-history](../features/view-history.md) |
| `ogp` | `GET /x/ogp` | [link-preview](../features/link-preview.md) |
| `share` | `GET /s/<id>` | [share-page](../features/share-page.md) |

## 認証・セキュリティ機構

| 現行 | Macaron |
|---|---|
| セッション Cookie(`zsid`) | 不要(署名) |
| CSRF トークン `X-CSRF-Token` | 不要(署名に URL・メソッドが入る = [kind 20001](../kinds/20001-http-auth.md)) |
| `password_hash` | 不要(リレーはパスワードを持たない) |
| `throttle` 表 | [rate-limit-and-abuse](../features/rate-limit-and-abuse.md) |
| `RESERVED` ハンドル | [handle](../features/handle.md) の運営ポリシー |

## 定数 → 情報ドキュメントの `limits` / `policy`

| 現行定数 | 値 | Macaron での置き場所 |
|---|---|---|
| `MAX_POST` | 500 | `limits.max_content_length` |
| `MAX_MEDIA` | 4(動画は1) | `limits.max_media` |
| `MAX_IMG_BYTES` / `MAX_VIDEO_BYTES` | 10MB / 50MB | メディアサーバーの `limits` |
| `MAX_EMOJI_PER_USER` | 50 | `limits.max_emoji_per_user` |
| `MAX_BADGES` | 5 | `limits.max_badges` |
| `SIGNUP_OPEN` | true | `policy.signup` |
| `SENS_THRESHOLD` | 3 | `policy.sensitive_threshold` |
| `SENS_MIN_AGE` | 86400 | `policy.flag_min_account_age` |
| `RX_SET` | 8種 | `policy.reaction_set`(省略=制限なし) |
| `REC_*` `TREND_*` | — | リレー実装の内部設定(仕様外) |

## テーブル → イベント

| 現行テーブル | Macaron |
|---|---|
| `users` | [kind 10000](../kinds/10000-profile.md) + ハンドル表(リレー運営) + [kind 30001](../kinds/30001-setting.md) |
| `posts`(+ `tags` `terms`) | [kind 1](../kinds/1-post.md)(`t` タグで索引、`terms` はリレー内部) |
| `media` | イベントの `imeta` + メディアサーバー |
| `likes` `reactions` | [kind 3](../kinds/3-reaction.md) |
| `reposts` | [kind 2](../kinds/2-repost.md) |
| `bookmarks` | [kind 9](../kinds/9-bookmark.md) |
| `follows` | [kind 8](../kinds/8-follow.md) |
| `dms` | [kind 4](../kinds/4-direct-message.md) |
| `notifs` | **なし**(購読で導出) |
| `views` | [kind 7](../kinds/7-view.md) |
| `sens_votes` / `community_sens` | [kind 6](../kinds/6-flag.md)(認定はクライアントが導出) |
| `emojis` | [kind 30000](../kinds/30000-emoji.md) |
| `trend_cache` | [kind 30002](../kinds/30002-trends.md) |
| `ogp_cache` `throttle` | リレー内部(仕様外) |

## 現行にあって、分散で**変わる/失われる**もの

| 現行 | 変化 |
|---|---|
| パスワード再設定 | 無い(鍵を失ったら復旧不可) |
| 「いいねタブは本人だけ」 | いいねは公開イベント。UIで隠すのみ |
| 絵文字名のサービス全体で一意 | 作者ごとの名前空間(タグが優先) |
| 削除で関連データが全部消える | リレー上のみ。他リレー・他人の保存分は残る |
| 新規登録のアカウント年齢 | 「pubkey の初出から」で判断(リレーごとに違う) |
| 件数がサービス全体で同じ | リレーごとに見える範囲が違う |
