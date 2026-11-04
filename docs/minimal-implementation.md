# 最小実装の骨組み

仕様が「単純」であることの確認用スケッチ。そのまま動くレベルを目指すが、検証・制限は省略している。

## クライアント(JS): イベントを作って署名

```js
const enc = new TextEncoder();
const hex = (u8) => [...u8].map((b) => b.toString(16).padStart(2, '0')).join('');
const unhex = (h) => Uint8Array.from(h.match(/../g).map((x) => parseInt(x, 16)));

export async function makeEvent(sk /* CryptoKey(Ed25519) */, pubkey, kind, tags, content, created_at = Math.floor(Date.now() / 1000)) {
  const ser = JSON.stringify([0, pubkey, created_at, kind, tags, content]);   // 01-event.md の規則と一致
  const idBytes = new Uint8Array(await crypto.subtle.digest('SHA-256', enc.encode(ser)));
  const sig = new Uint8Array(await crypto.subtle.sign('Ed25519', sk, idBytes));
  return { id: hex(idBytes), pubkey, created_at, kind, tags, content, sig: hex(sig) };
}

// 鍵の生成(WebCrypto)
const kp = await crypto.subtle.generateKey({ name: 'Ed25519' }, true, ['sign', 'verify']);
const pubkey = hex(new Uint8Array(await crypto.subtle.exportKey('raw', kp.publicKey)));
// 秘密鍵の保存: exportKey('pkcs8', kp.privateKey) の末尾32バイトが seed。復元時に公開鍵が要るので、
// 公開鍵も一緒に保存するか、@noble/ed25519 の getPublicKey(seed) を使う。
```

### 送る・読む(HTTP+SSE の場合)

```js
const post = (msg) => fetch('/msg', { method: 'POST', body: JSON.stringify(msg) }).then((r) => r.json());

// 投稿
const ev = await makeEvent(sk, pubkey, 1, [['t', 'hello']], 'こんにちは #hello');
const [ok] = await post(['EVENT', ev]);                      // ["OK", id, true, ""]

// いいね(トグルの ON)
await post(['EVENT', await makeEvent(sk, pubkey, 3, [['e', postId], ['p', author]], '+')]);

// タイムライン
const res = await post(['REQ', 'tl', { kinds: [1], without: ['e'], limit: 20 }]);
const events = res.filter((m) => m[0] === 'EVENT').map((m) => m[2]);   // 検証してから使う

// ライブ
const b64url = (s) => btoa(String.fromCharCode(...enc.encode(s))).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');   // 日本語を含んでも安全
const es = new EventSource('/sse?sub=live&f=' + b64url(JSON.stringify([{ kinds: [1], since: now }])));
es.addEventListener('m', (e) => { const [t, , ev] = JSON.parse(e.data); if (t === 'EVENT') render(ev); });
```

## リレー(PHP): 受理の中心部

```php
function macaron_id(array $e): string {
    $s = json_encode([0, $e['pubkey'], $e['created_at'], $e['kind'], $e['tags'], $e['content']],
        JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_LINE_TERMINATORS);
    return hash('sha256', $s);
}
function macaron_verify(array $e): bool {
    foreach (['id','pubkey','sig','content'] as $k) if (!isset($e[$k]) || !is_string($e[$k])) return false;
    if (!is_int($e['created_at'] ?? null) || !is_int($e['kind'] ?? null) || !is_array($e['tags'] ?? null)) return false;
    if (!preg_match('/^[0-9a-f]{64}$/', $e['id']) || !preg_match('/^[0-9a-f]{64}$/', $e['pubkey']) || !preg_match('/^[0-9a-f]{128}$/', $e['sig'])) return false;
    if (!hash_equals(macaron_id($e), $e['id'])) return false;
    return sodium_crypto_sign_verify_detached(hex2bin($e['sig']), hex2bin($e['id']), hex2bin($e['pubkey']));
}

function macaron_store(PDO $db, array $e): array {          // 02-kind-ranges.md の保存ルール
    $k = $e['kind'];
    if ($k >= 20000 && $k < 30000) return [true, ''];     // ephemeral: 保存しない(配信だけ)
    $d = null;
    if ($k >= 30000 && $k < 40000) {                          // addressable: d タグが無ければ空文字
        $d = '';
        foreach ($e['tags'] as $t) if (($t[0] ?? '') === 'd') { $d = (string)($t[1] ?? ''); break; }
    }
    if ($db->query('SELECT 1 FROM tombstones WHERE id=' . $db->quote($e['id']))->fetch()) return [true, 'duplicate: deleted'];
    $db->beginTransaction();
    if (($k >= 10000 && $k < 20000) || $d !== null) {         // replaceable / addressable: 新しい方だけ残す
        $old = $db->prepare('SELECT seq, created_at, id FROM events WHERE pubkey=? AND kind=? AND d IS ?');
        $old->execute([$e['pubkey'], $k, $d]);
        if ($o = $old->fetch(PDO::FETCH_ASSOC)) {
            $newer = $e['created_at'] > (int)$o['created_at'] || ($e['created_at'] === (int)$o['created_at'] && $e['id'] < $o['id']);
            if (!$newer) { $db->rollBack(); return [true, 'duplicate: older']; }
            $db->prepare('DELETE FROM events WHERE seq=?')->execute([$o['seq']]);
            $db->prepare('DELETE FROM tags WHERE seq=?')->execute([$o['seq']]);
        }
    }
    $ins = $db->prepare('INSERT OR IGNORE INTO events(id,pubkey,created_at,kind,d,json) VALUES(?,?,?,?,?,?)');
    $ins->execute([$e['id'], $e['pubkey'], $e['created_at'], $k, $d, json_encode($e, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES)]);
    if (!$ins->rowCount()) { $db->rollBack(); return [true, 'duplicate: have it']; }
    $seq = (int)$db->lastInsertId();
    $tg = $db->prepare('INSERT OR IGNORE INTO tags(seq,name,value,created_at) VALUES(?,?,?,?)');
    foreach ($e['tags'] as $t) if (is_array($t) && isset($t[0], $t[1]) && strlen($t[0]) === 1) $tg->execute([$seq, $t[0], (string)$t[1], $e['created_at']]);
    $db->commit();
    return [true, ''];
}
```
(`kind 5` の削除処理・kind別検証・レート制限は [08-relay](../core/08-relay.md) の手順に沿って足す。)

### 共有ホスティング向け SSE(1接続を短く)

```php
// GET /sse?f=…  — 50秒だけ開いて閉じる。EventSource が Last-Event-ID 付きで再接続する
session_write_close(); ignore_user_abort(false); set_time_limit(70);
header('Content-Type: text/event-stream'); header('Cache-Control: no-store'); header('X-Accel-Buffering: no');
while (ob_get_level()) ob_end_flush();
$last = (int)($_SERVER['HTTP_LAST_EVENT_ID'] ?? 0);
echo "retry: 1000\n\n";
for ($t = 0; $t < 50 && !connection_aborted(); $t++) {
    foreach (query_events_after($last, $filters) as $row) {       // seq > $last かつフィルタに一致
        $last = $row['seq'];
        echo "id: {$row['seq']}\nevent: m\ndata: " . json_encode(['EVENT', $sub, json_decode($row['json'])], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES) . "\n\n";
    }
    echo ": ping\n\n"; flush(); sleep(1);
}
```
