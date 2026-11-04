# 機能: 件数(いいね数・リポスト数・フォロワー数など)

- 使う メッセージ: `COUNT`([05-messages](../core/05-messages.md)) / 拡張フィルタ `group`
- 現行: `decorate()` が `GROUP BY` で数えて返していたもの(`rep` `rt` `likes` `rx` `qt` `sflag`)と、`user` の `followers` `following` `posts`

## COUNT

```
C: ["COUNT", "c1", {"kinds":[8],"#p":["<pk>"]}]
R: ["COUNT", "c1", {"count": 42}]
```

- フィルタは `REQ` と同じ([04-filters](../core/04-filters.md))。`limit` は無視。
- 複数フィルタを渡すと**フィルタごとではなく全体の合計**(OR で数える)。フィルタごとに数えたいときは `COUNT` を分ける。
- 重い集計の近似・キャッシュは許可(返値に `"approximate": true`)。

## group(拡張)

1回で投稿ごと・種類ごとに集計するための項目。タイムライン1ページ分(20投稿)の表示に必要な件数を、**3〜5回の COUNT** で取るために使う。

```
C: ["COUNT","rx",{"kinds":[3],"#e":["a","b","c"],"group":["e","content"]}]
R: ["COUNT","rx",{"count":7,"groups":{"a":{"+":3,"🔥":1},"b":{"+":2},"c":{":party:":1}}}]
```

- `group` は最大2段。指定できるキーは `e` `p` `q` `kind` `content` `pubkey`(最後は「誰が」)。
- `groups` はキーの値 → 数(2段なら入れ子)。

## 画面ごとの件数の取り方(現行 `decorate` の置き換え)

| 表示 | COUNT |
|---|---|
| 返信数 `rep` | `{"kinds":[1],"#e":ids,"group":["e"]}` |
| 引用数 `qt` | `{"kinds":[1],"#q":ids,"group":["q"]}` |
| リポスト数 `rt` | `{"kinds":[2],"#e":ids,"group":["e"]}` |
| いいね・リアクション `likes` `rx` | `{"kinds":[3],"#e":ids,"group":["e","content"]}`(`+` がいいね) |
| 報告数 `sflag` | `{"kinds":[6],"#e":ids,"group":["e"]}` |
| フォロワー数 | `{"kinds":[8],"#p":[pk]}` |
| フォロー数 | `{"kinds":[8],"authors":[pk]}` |
| ポスト数 | `{"kinds":[1],"authors":[pk],"without":["e"]}` |

## 自分の状態(`liked` `reposted` `bookmarked` `myrx`)

件数ではなく**自分のイベント**を `REQ` する:

```
{"kinds":[3],"authors":["<me>"],"#e":ids}      → いいね・リアクション済みの種類
{"kinds":[2],"authors":["<me>"],"#e":ids}      → リポスト済み
{"kinds":[9],"authors":["<me>"]}               → ブックマーク(復号して照合)
```
解除に使うため、**イベントidも保持する**([kind 5](../kinds/5-deletion.md))。

## 楽観的更新

押した瞬間に画面の数字を +1 し、`OK false` なら戻す。署名済みイベントはその場で作れるので、通信の待ち時間が見えない。
