# 06 トランスポート: WebSocket

## 接続

```
wss://relay.example.com/ws
```

- テキストフレーム1つ = [メッセージ](05-messages.md) 1つ(JSON配列)。バイナリフレームは使わない。
- 1接続で複数の購読(`sub-id`)を同時に持てる。上限は `limits.max_subscriptions`(目安 20)。
- WebSocket の Ping/Pong を30秒ごとに使い、応答のない接続は閉じる。
- 切断後の再接続は指数バックオフ(1s → 2s → … 最大30s)。再接続したら**購読を張り直し**、`since` に「最後に受け取った `created_at`」を指定して取りこぼしを埋める。

## 認証が必要なとき

リレーが `["AUTH", <challenge>]` を送ってくる。クライアントは [kind 20000](../kinds/20000-relay-auth.md) を署名して `["AUTH", <event>]` で返す。認証は接続単位。

## 向いている環境

- 常駐プロセスが使えるサーバー(Node / Go / Rust / PHP + Swoole・Ratchet など)。
- 共有ホスティングの素のPHP(Apache + mod_php / FPM)では常駐接続が難しいので、[07-transport-sse.md](07-transport-sse.md) を使う。

## 最小クライアント例(ブラウザ)

```js
const ws = new WebSocket('wss://relay.example.com/ws');
ws.onopen = () => ws.send(JSON.stringify(['REQ', 'tl', { kinds: [1], without: ['e'], limit: 20 }]));
ws.onmessage = (m) => {
  const [type, ...rest] = JSON.parse(m.data);
  if (type === 'EVENT') render(rest[1]);        // 検証してから描画
  if (type === 'EOSE')  hideSpinner();
};
```
