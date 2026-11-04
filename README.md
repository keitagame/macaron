# macaron
# Macaron Protocol

Nostr に着想を得た、**署名付きイベント + リレー** 型の分散SNSプロトコル。
`ℤ`(api.php / app.js / index.html)と同等のSNSを、PHP+SQLite 1台の最小構成から、複数リレーの分散構成まで同じ仕様で作れることを目標にしている。

## 設計原則

1. **すべてはイベント。** データ構造は1種類(署名付きJSON)だけ。投稿もいいねもフォローもDMも同じ形。
2. **信頼は署名、サーバーは運び屋。** リレーは「検証して保存して配る」だけ。アカウントの持ち主はリレーではなく秘密鍵。
3. **加算的にだけ拡張する。** 知らない kind・tag・フィルタ項目は無視する。既存の意味は変えない。バージョン番号を持たない。
4. **プロトコルとポリシーを分ける。** 「500文字まで」「画像4枚まで」はリレーのポリシー(`GET /` で公開)であり、仕様ではない。
5. **1機能 = 1つのmd、1 kind = 1つのmd。** 機能は kind と tag の組み合わせで表現する。
6. **トランスポート非依存。** WebSocket でも SSE+HTTP でも、流れるメッセージは同一。共有ホスティングのPHPでも動く。



## 最小構成(現行 ℤ からの移行イメージ)

- リレー: PHP + SQLite。`events` テーブル1つ + タグ索引テーブル1つ。`api.php` の約40アクションは `EVENT` / `REQ` / `COUNT` の3つに集約される。
- トランスポート: `POST /msg`(送信・取得)+ `GET /sse`(ライブ更新)。WebSocket は後から足せる。
- クライアント: 既存の `app.js` の描画部分はほぼそのまま。`api('timeline')` が `REQ` に、`api('like')` が「kind 3 の署名・送信」に変わる。

## 目次

### core/ — 土台
| ファイル | 内容 |
|---|---|
| [01-event.md](core/01-event.md) | イベントの構造・ID・署名・正規化 |
| [02-kind-ranges.md](core/02-kind-ranges.md) | kind番号の範囲と保存ルール |
| [03-tags.md](core/03-tags.md) | 標準タグ一覧と索引ルール |
| [04-filters.md](core/04-filters.md) | 取得条件(フィルタ) |
| [05-messages.md](core/05-messages.md) | クライアント⇔リレーのメッセージ |
| [06-transport-websocket.md](core/06-transport-websocket.md) | WebSocket |
| [07-transport-sse.md](core/07-transport-sse.md) | HTTP + SSE |
| [08-relay.md](core/08-relay.md) | リレーの検証手順・ポリシー・情報ドキュメント |
| [09-extending.md](core/09-extending.md) | 拡張のしかた |

### kinds/ — イベントの種類(1ファイル1種類)
| kind | 名前 | 保存 | 現行API |
|---|---|---|---|
| [1](kinds/1-post.md) | 投稿(返信・引用含む) | regular | `post` |
| [2](kinds/2-repost.md) | リポスト | regular | `repost` |
| [3](kinds/3-reaction.md) | リアクション(いいね含む) | regular | `like` `react` |
| [4](kinds/4-direct-message.md) | ダイレクトメッセージ | regular | `dm_send` |
| [5](kinds/5-deletion.md) | 削除 | regular | `delete_post` 等のトグルOFF |
| [6](kinds/6-flag.md) | 通報(センシティブ等) | regular | `sens_vote` `set_sensitive` |
| [7](kinds/7-view.md) | 閲覧記録 | regular | `view_seen` |
| [8](kinds/8-follow.md) | フォロー | regular | `follow` |
| [9](kinds/9-bookmark.md) | ブックマーク(非公開) | regular | `bookmark` |
| [10000](kinds/10000-profile.md) | プロフィール | replaceable | `profile` `upload_image` `upload_deco` |
| [10001](kinds/10001-relay-list.md) | リレーリスト | replaceable | (新規) |
| [20000](kinds/20000-relay-auth.md) | リレー認証 | ephemeral | `login` |
| [20001](kinds/20001-http-auth.md) | HTTP認証 | ephemeral | セッション + CSRF |
| [30000](kinds/30000-emoji.md) | カスタム絵文字 | addressable | `emoji_add` `emoji_delete` |
| [30001](kinds/30001-setting.md) | 設定 | addressable | `noview` `notif_read` |
| [30002](kinds/30002-trends.md) | トレンド(リレー署名) | addressable | `trends` |

### features/ — 機能(1ファイル1機能)
| ファイル | 機能 |
|---|---|
| [identity-and-keys.md](features/identity-and-keys.md) | 鍵・ログイン・サインアップ |
| [handle.md](features/handle.md) | `@handle` と名前の確認 |
| [timeline.md](features/timeline.md) | 全体 / フォロー中 / おすすめ |
| [thread-and-reply.md](features/thread-and-reply.md) | スレッドと返信 |
| [quote.md](features/quote.md) | 引用 |
| [mention-and-hashtag.md](features/mention-and-hashtag.md) | メンション・ハッシュタグ |
| [media.md](features/media.md) | 画像・動画 |
| [sensitive-content.md](features/sensitive-content.md) | センシティブ(自己申告 + コミュニティ認定) |
| [custom-emoji.md](features/custom-emoji.md) | カスタム絵文字 |
| [profile-decoration-and-badges.md](features/profile-decoration-and-badges.md) | アイコンデコ・バッジ |
| [notifications.md](features/notifications.md) | 通知 |
| [counts.md](features/counts.md) | 件数(いいね数・フォロワー数など) |
| [search.md](features/search.md) | 検索 |
| [trends.md](features/trends.md) | トレンド |
| [recommendation-feed.md](features/recommendation-feed.md) | おすすめ順 |
| [view-history.md](features/view-history.md) | 閲覧者一覧 |
| [link-preview.md](features/link-preview.md) | リンクプレビュー(OGP) |
| [share-page.md](features/share-page.md) | シェア用ページ |
| [encryption.md](features/encryption.md) | 暗号化(DM・ブックマーク・設定) |
| [live-updates.md](features/live-updates.md) | リアルタイム更新(ポーリングの置換) |
| [rate-limit-and-abuse.md](features/rate-limit-and-abuse.md) | レート制限・スパム対策 |
| [multi-relay.md](features/multi-relay.md) | 複数リレー・分散 |

### docs/
- [api-mapping.md](docs/api-mapping.md) — 現行 `api.php` の全アクションとの対応表
- [migration-from-z.md](docs/migration-from-z.md) — 既存ユーザー・投稿の移行手順
- [minimal-implementation.md](docs/minimal-implementation.md) — クライアント署名・PHPリレー受理・共有ホスティング向けSSEの骨組み

## 表記

- MUST / SHOULD / MAY は RFC 2119 の意味。
- JSON例の `…` は省略。hex は小文字。時刻は UNIX秒(整数)。
