---
title: "Webhook署名をHMACからEd25519へ：Correct Hooksで「秘密鍵1本」の検証をPythonで実装する【検証済み】"
emoji: "🔏"
type: "tech"
topics: ["webhook", "python", "fastapi", "security", "typescript"]
published: true
published_at: "2026-10-21 08:00"
---

## 概要（Overview）

Webhook を送る側（producer）になると、署名の仕組みを決める必要があります。よく使われるのは Stripe や GitHub と同じ **HMAC（共通鍵）** 方式です。ただし HMAC では、送り先のエンドポイントごとに秘密鍵を発行し、保管し、ローテーションしなければなりません。顧客が増えるほど、管理する秘密鍵も増えていきます。

**公開鍵署名（非対称署名）** を使えば、送る側が持つ秘密鍵は 1 本で済みます。受け取る側（consumer）は、公開されている鍵セットを取得して検証するだけです。ただ、メッセージの組み立て方、鍵の配り方、ローテーションなどを自分で実装するのは手間がかかります。

[Correct Hooks](https://github.com/handlewithcarecollective/correct-hooks)（MIT）は、この部分をまとめて引き受けるライブラリです。Ed25519 による署名と検証、鍵セットの公開、ローテーションを、Python / TypeScript / Rust で同じ形式のまま扱えます。2026年10月に Show HN に投稿されたばかりで、日本語の解説はまだ見当たりません。

この記事では、ローカルで次のことを実際に確認しました。

1. FastAPI で **送信側（鍵セットの公開＋署名）** と **受信側（検証）** を作り、実際に HTTP で送る
2. 改ざん、別の顧客あての署名、別 URL あての署名、攻撃者の鍵、有効期限切れが、**すべて 403 で弾かれる** ことを確かめる
3. **ダウンタイムなしの鍵ローテーション** を試す
4. **Python ⇔ TypeScript** で、相互に署名を検証する

README どおりに書くと動かない箇所や、言語をまたぐと失敗するケースも見つかったので、後半にまとめています。

## 前提環境（Prerequisites）

検証環境は以下のとおりです（2026-10-19 時点）。

| ツール | バージョン |
| --- | --- |
| Python | 3.12.3 |
| correct_hooks（PyPI） | 0.1.1 |
| fastapi / uvicorn / httpx | 0.146.0 / 0.54.0 / 0.28.1 |
| Node.js | 22.23.3（TypeScript 版との相互検証に使用） |
| @handlewithcare/correct-hooks（npm） | 0.1.1 |
| OS | Linux |

```bash
python3 -m venv .venv
.venv/bin/pip install correct_hooks fastapi uvicorn
```

`correct_hooks` は `cryptography`・`pydantic`・`httpx` に依存しています。そのため、上のコマンドだけで httpx も入ります。

## 手順（Step-by-Step Code Guide）

### 1. 仕組みをざっくり把握する

Correct Hooks のリクエストには、次の 2 つのヘッダーが付きます。

| ヘッダー | 中身 |
| --- | --- |
| `correct-hooks-signature-components` | `v1,<webhook_id>.<鍵ID(kid)>.<署名時刻>`（それぞれ base64url） |
| `correct-hooks-signature` | `v1,<Ed25519 署名>` |

署名の対象は、**上の components ＋ 受信者 ID（recipient_id）＋ 送信先 URL ＋ HTTP メソッド ＋ ボディの生バイト** です。つまり、次のどれか 1 つでも食い違えば検証に失敗します。

- 別の顧客あてに作られた署名を流用した
- 同じ顧客の別エンドポイントに送り直した
- ボディを書き換えた

秘密鍵が 1 本でも安全に運用できるのは、このためです。

受信側は、送信側が公開している **鍵セット（JWKS に似た JSON）** を取得し、`kid` に対応する公開鍵で検証します。鍵セットは 300 秒間キャッシュされます。知らない `kid` の署名が届いた場合は、鍵セットを自動で取り直します。

### 2. 送信側（producer）：鍵セットの公開と署名

`generate_secret_signing_key()` で作った秘密鍵は、本番ではシークレットマネージャーに保管します。公開鍵セットは GET エンドポイントで配ります。

```python:producer.py
"""Webhook producer: one Ed25519 signing key for every customer + a public key set endpoint."""
import json
import uuid

import httpx
from fastapi import FastAPI
from correct_hooks import (
    SignArgs,
    create_public_verifying_key_set,
    generate_secret_signing_key,
    sign,
)

app = FastAPI()

# In production these come from a secret manager. Here they live in memory.
state = {
    "signing_key": generate_secret_signing_key(),
    "published_keys": [],
    "key_set_fetches": 0,
}
state["published_keys"] = [state["signing_key"]]


@app.get("/.well-known/webhook-keys")
def public_key_set():
    state["key_set_fetches"] += 1
    key_set = create_public_verifying_key_set(state["published_keys"])
    return json.loads(key_set.model_dump_json())


def deliver(event: dict, *, recipient_id: uuid.UUID, target_url: str, webhook_id=None):
    """Sign and POST one webhook. Call sign() again on every retry."""
    body = json.dumps(event).encode()
    headers = sign(SignArgs(
        webhook_id=webhook_id or uuid.uuid4(),
        recipient_id=recipient_id,  # from YOUR database, never from the consumer's input
        target_url=target_url,      # the exact string the consumer registered
        method="POST",
        request_body=body,
        secret_signing_key=state["signing_key"],
    ))
    return httpx.post(target_url, content=body, headers={
        "content-type": "application/json",
        "correct-hooks-signature-components": headers.correct_hooks_signature_components,
        "correct-hooks-signature": headers.correct_hooks_signature,
    })
```

ポイントは次の 3 つです。

- **`recipient_id` は送信側のデータベースが持つ値** を使います。受信側から渡された値を使ってはいけません（ライブラリの docstring にも明記されています）。
- **`target_url` は、受信側が登録した文字列そのまま** を使います。
- `sign()` は **再送のたびに呼び直し** ます。そうしないと署名時刻が古いままになり、後述する 5 分の有効期限に引っかかります。`webhook_id` は再送でも同じ値を使い、受信側の重複排除に使ってもらいます。

公開される鍵セットは、次のような形になります（`gotchas.py` の出力から抜粋）。秘密鍵の `d` は含まれません。

```json
{
  "keys": [
    {
      "kid": "5a7e80d3-e69f-4bf9-80aa-37f4b1bcfe24",
      "kty": "OKP",
      "crv": "Ed25519",
      "x": "mvuopsEIyCkQ-643wL9EJVJpIcrGnO2M_-pjfhw33gU"
    }
  ]
}
```

### 3. 受信側（consumer）：FastAPI で検証する

```python:consumer.py
"""Webhook consumer: verifies with the producer's public key set. No shared secret."""
import os
import uuid

from fastapi import FastAPI, HTTPException, Request
from correct_hooks import Verifier, VerifierConfig, VerifyArgs

# Both values come from the producer's dashboard / docs.
RECIPIENT_ID = uuid.UUID(os.environ["RECIPIENT_ID"])
KEY_SET_URL = os.environ.get("KEY_SET_URL", "http://127.0.0.1:8001/.well-known/webhook-keys")
# The exact URL string you registered with the producer (not request.url!).
TARGET_URL = os.environ.get("TARGET_URL", "http://localhost:8002/webhooks/orders")

app = FastAPI()
verifier = Verifier(VerifierConfig(
    recipient_id=RECIPIENT_ID,
    public_verifying_key_set_url=KEY_SET_URL,
))
seen_webhook_ids: set = set()  # use Redis etc. with a TTL >= 5 min in production
received: list = []


@app.post("/webhooks/orders")
async def orders(request: Request):
    try:
        verified = await verifier.verify(VerifyArgs(
            target_url=TARGET_URL,
            method=request.method,
            correct_hooks_signature_components=request.headers.get("correct-hooks-signature-components", ""),
            correct_hooks_signature=request.headers.get("correct-hooks-signature", ""),
            request_body=await request.body(),  # raw bytes, before any JSON parsing
        ))
    except Exception as e:
        raise HTTPException(status_code=403, detail=type(e).__name__)

    if verified.webhook_id in seen_webhook_ids:
        return {"status": "duplicate", "webhook_id": str(verified.webhook_id)}
    seen_webhook_ids.add(verified.webhook_id)
    received.append(await request.json())
    return {"status": "ok", "webhook_id": str(verified.webhook_id)}
```

ここでは次の 3 点に気をつけています。

- `Verifier` は **起動時に 1 回だけ作り**、使い回します。鍵セットのキャッシュは Verifier の中に持たれるためです。
- `target_url` には `request.url` ではなく、**登録した URL の文字列** を渡します（理由は「ハマりどころ」で説明します）。
- ボディは **`await request.body()` で受け取った生のバイト列** を渡します。JSON にパースするのは、検証が通ってからにします。

### 4. 実際に送って、攻撃も試す

送信側（:8001）と受信側（:8002）を uvicorn で実際に起動し、HTTP で送るスクリプトです。

:::details demo.py（全文）
```python:demo.py
"""End-to-end check: real producer + consumer servers, then attack and rotation scenarios."""
import json
import os
import threading
import time
import uuid

RECIPIENT_ID = uuid.uuid4()
os.environ["RECIPIENT_ID"] = str(RECIPIENT_ID)

import httpx
import uvicorn
from correct_hooks import SignArgs, VerifyArgs, decode_base64, generate_secret_signing_key, sign

import consumer
import producer

TARGET = consumer.TARGET_URL


def serve(app, port):
    server = uvicorn.Server(uvicorn.Config(app, host="127.0.0.1", port=port, log_level="warning"))
    threading.Thread(target=server.run, daemon=True).start()
    while not server.started:
        time.sleep(0.05)


serve(producer.app, 8001)
serve(consumer.app, 8002)

results = []


def check(name, response, want_status, want_detail=None):
    body = response.json()
    got = body.get("status") if response.status_code == 200 else body.get("detail")
    ok = response.status_code == want_status and (want_detail is None or got == want_detail)
    results.append(ok)
    print(f"{'PASS' if ok else 'FAIL'}  {name:<46} -> {response.status_code} {got}")


def post_raw(body: bytes, headers):
    return httpx.post(TARGET, content=body, headers={
        "content-type": "application/json",
        "correct-hooks-signature-components": headers.correct_hooks_signature_components,
        "correct-hooks-signature": headers.correct_hooks_signature,
    })


def sign_for(body: bytes, *, recipient_id=RECIPIENT_ID, target_url=TARGET, key=None, webhook_id=None):
    return sign(SignArgs(
        webhook_id=webhook_id or uuid.uuid4(), recipient_id=recipient_id, target_url=target_url,
        method="POST", request_body=body, secret_signing_key=key or producer.state["signing_key"],
    ))


print("== 1. normal delivery and retries")
event = {"type": "order.paid", "order_id": "ord_123", "amount": 4200}
wid = uuid.uuid4()
check("valid signature", producer.deliver(event, recipient_id=RECIPIENT_ID, target_url=TARGET, webhook_id=wid), 200, "ok")
check("retry with same webhook_id (re-signed)", producer.deliver(event, recipient_id=RECIPIENT_ID, target_url=TARGET, webhook_id=wid), 200, "duplicate")

print("== 2. attacks")
body = json.dumps(event).encode()
h = sign_for(body)
check("body tampered (amount 4200 -> 1)", post_raw(body.replace(b"4200", b"1"), h), 403, "InvalidSignature")
check("signed for another customer (recipient_id)", post_raw(body, sign_for(body, recipient_id=uuid.uuid4())), 403, "InvalidSignature")
check("signed for another URL (/webhooks/refunds)", post_raw(body, sign_for(body, target_url=TARGET.replace("orders", "refunds"))), 403, "InvalidSignature")
check("signed by an attacker's own key", post_raw(body, sign_for(body, key=generate_secret_signing_key())), 403, "PublicVerifyingKeyNotFound")
check("no signature headers", httpx.post(TARGET, content=body), 403, "InvalidSignature")

print("== 3. signature TTL (default 300s), via verify_with_time")
args = VerifyArgs(target_url=TARGET, method="POST", request_body=body,
                  correct_hooks_signature_components=h.correct_hooks_signature_components,
                  correct_hooks_signature=h.correct_hooks_signature)
# signed_at is the 3rd base64 part of the components header ("v1,<webhook_id>.<kid>.<signed_at>")
signed_at = int(decode_base64(h.correct_hooks_signature_components.split(",")[1].split(".")[2]))
for offset in (300, 301, -300, -301):
    try:
        consumer.verifier.verify_with_time_sync(args, now=signed_at + offset)
        outcome = "accepted"
    except Exception as e:
        outcome = type(e).__name__
    expected = "accepted" if abs(offset) <= 300 else "InvalidSignature"
    ok = outcome == expected
    results.append(ok)
    print(f"{'PASS' if ok else 'FAIL'}  {f'verify at signed_at{offset:+d}s':<46} -> {outcome}")

print("== 4. zero-downtime key rotation")
old_key = producer.state["signing_key"]
new_key = generate_secret_signing_key()
fetches = producer.state["key_set_fetches"]
producer.state["published_keys"] = [old_key, new_key]  # step 2: publish old + new
producer.state["signing_key"] = new_key                 # step 3: sign with new
check("first webhook signed with the new key", producer.deliver(event, recipient_id=RECIPIENT_ID, target_url=TARGET), 200, "ok")
print(f"      key set fetched {producer.state['key_set_fetches'] - fetches}x by the consumer (unknown kid -> refetch)")
check("in-flight retry still signed with old key", post_raw(body, sign_for(body, key=old_key)), 200, "ok")
producer.state["published_keys"] = [new_key]            # step 4: drop the old key
check("new key keeps working", producer.deliver(event, recipient_id=RECIPIENT_ID, target_url=TARGET), 200, "ok")
check("old key right after retirement (cached set)", post_raw(body, sign_for(body, key=old_key)), 200, "ok")

print("== 5. unknown kid triggers a key set refetch")
fetches = producer.state["key_set_fetches"]
for _ in range(5):
    post_raw(body, sign_for(body, key=generate_secret_signing_key()))
n = producer.state["key_set_fetches"] - fetches
print(f"      5 requests with random kids -> {n} key set fetches")
check("old key once the consumer has refetched", post_raw(body, sign_for(body, key=old_key)), 403, "PublicVerifyingKeyNotFound")

print(f"\n{sum(results)}/{len(results)} checks passed")
raise SystemExit(0 if all(results) else 1)
```
:::

実行結果です。

```bash
.venv/bin/python demo.py
```

```text
== 1. normal delivery and retries
PASS  valid signature                                -> 200 ok
PASS  retry with same webhook_id (re-signed)         -> 200 duplicate
== 2. attacks
PASS  body tampered (amount 4200 -> 1)               -> 403 InvalidSignature
PASS  signed for another customer (recipient_id)     -> 403 InvalidSignature
PASS  signed for another URL (/webhooks/refunds)     -> 403 InvalidSignature
PASS  signed by an attacker's own key                -> 403 PublicVerifyingKeyNotFound
PASS  no signature headers                           -> 403 InvalidSignature
== 3. signature TTL (default 300s), via verify_with_time
PASS  verify at signed_at+300s                       -> accepted
PASS  verify at signed_at+301s                       -> InvalidSignature
PASS  verify at signed_at-300s                       -> accepted
PASS  verify at signed_at-301s                       -> InvalidSignature
== 4. zero-downtime key rotation
PASS  first webhook signed with the new key          -> 200 ok
      key set fetched 1x by the consumer (unknown kid -> refetch)
PASS  in-flight retry still signed with old key      -> 200 ok
PASS  new key keeps working                          -> 200 ok
PASS  old key right after retirement (cached set)    -> 200 ok
== 5. unknown kid triggers a key set refetch
      5 requests with random kids -> 5 key set fetches
PASS  old key once the consumer has refetched        -> 403 PublicVerifyingKeyNotFound

16/16 checks passed
```

16 項目すべてが期待どおりの結果になりました。とくに次の点を確認できました。

- **改ざん、別の顧客あて、別 URL あて** の署名は、どれも `InvalidSignature` で弾かれます。
- **攻撃者が自分の鍵で署名** した場合は、公開鍵セットにその `kid` が無いので `PublicVerifyingKeyNotFound` になります。
- 署名の有効期限は **前後 300 秒** です。ちょうど 300 秒までは通り、301 秒で弾かれます。受信側の時計が送信側より遅れている場合（未来の署名時刻）も、同じ範囲で判定されます。
- 同じ `webhook_id` の再送は、署名を作り直しているので検証は通ります。そのうえで、受信側の重複チェックが `duplicate` を返します。

### 5. ダウンタイムなしの鍵ローテーション

README に書かれている手順は次の 4 ステップです。demo.py のセクション 4 でそのまま試しています。

1. 新しい秘密鍵を作る
2. **旧鍵と新鍵の両方** を鍵セットに載せて公開する
3. 新しい鍵で署名を始める
4. 鍵セットを **新鍵だけ** にする

受信側の設定は何も変えていません。それでも、新しい鍵で署名された最初のリクエストで、受信側が鍵セットを **1 回だけ取り直して** 検証に成功しました（ログの `key set fetched 1x`）。手順 2 と 3 の間に送られた旧鍵の再送も、そのまま通っています。

ただし、手順 4 の直後はまだ **旧鍵の署名も通ります**（`old key right after retirement (cached set) -> 200`）。これについては「ハマりどころ」で説明します。

### 6. Python ⇔ TypeScript で相互に検証する

送信側が Python、受信側が Node.js という組み合わせはよくあります。そこで、Python で署名したものを TypeScript の `Verifier` で検証し、その逆も試しました。

:::details cross_lang.py / ts/verify_py.mjs / ts/sign_ts.mjs（全文）
```python:cross_lang.py
"""Python <-> TypeScript interop.
  python cross_lang.py sign     -> fixtures/py_signed.json (+ key set) for ts/verify_py.mjs
  python cross_lang.py verify   -> verifies fixtures/ts_signed.json produced by ts/sign_ts.mjs
"""
import json
import sys
import time
import uuid
from pathlib import Path

from correct_hooks import (
    PublicVerifyingKeySet, SignArgs, VerifyArgs, create_public_verifying_key_set,
    generate_secret_signing_key, sign, verify_with_time,
)

FIX = Path(__file__).parent / "fixtures"

if sys.argv[1] == "sign":
    key = generate_secret_signing_key()
    recipient = uuid.uuid4()
    body = json.dumps({"type": "order.paid", "order_id": "ord_123"}).encode()
    url = "https://consumer.example.com/webhooks/orders"
    h = sign(SignArgs(webhook_id=uuid.uuid4(), recipient_id=recipient, target_url=url,
                      method="POST", request_body=body, secret_signing_key=key))
    FIX.joinpath("py_signed.json").write_text(json.dumps({
        "keySet": json.loads(create_public_verifying_key_set([key]).model_dump_json()),
        "recipientId": str(recipient), "targetUrl": url, "body": body.decode(),
        "components": h.correct_hooks_signature_components, "signature": h.correct_hooks_signature,
    }, indent=2))
    print("wrote fixtures/py_signed.json")
else:
    fx = json.loads(FIX.joinpath("ts_signed.json").read_text())
    key_set = PublicVerifyingKeySet.model_validate(fx["keySet"])
    for case in fx["cases"]:
        try:
            wid = verify_with_time(
                args=VerifyArgs(target_url=fx["targetUrl"], method="POST", request_body=fx["body"].encode(),
                                correct_hooks_signature_components=case["components"],
                                correct_hooks_signature=case["signature"]),
                recipient_id=uuid.UUID(fx["recipientId"]), signature_ttl_seconds=300,
                public_verifying_key_set=key_set, now=int(time.time())).webhook_id
            print(f"py verifies TS signature ({case['label']:<26}) -> ok {wid}")
        except Exception as e:
            print(f"py verifies TS signature ({case['label']:<26}) -> {type(e).__name__}")
```

```js:ts/verify_py.mjs
// Verify a Python-signed webhook with the TypeScript Verifier (key set served over HTTP).
import { createServer } from "node:http";
import { readFileSync } from "node:fs";
import { Verifier } from "@handlewithcare/correct-hooks";

const fx = JSON.parse(readFileSync(new URL("../fixtures/py_signed.json", import.meta.url)));
const server = createServer((_, res) => {
  res.writeHead(200, { "content-type": "application/json" });
  res.end(JSON.stringify(fx.keySet));
}).listen(8011);

for (const [label, recipientId] of [
  ["recipientId as given", fx.recipientId],
  ["recipientId uppercased", fx.recipientId.toUpperCase()],
]) {
  const verifier = new Verifier({
    recipientId,
    publicVerifyingKeySetUrl: new URL("http://127.0.0.1:8011/keys"),
  });
  try {
    const { webhookId } = await verifier.verify({
      targetUrl: fx.targetUrl,
      method: "POST",
      correctHooksSignatureComponents: fx.components,
      correctHooksSignature: fx.signature,
      requestBody: Buffer.from(fx.body),
    });
    console.log(`ts verifies Python signature (${label.padEnd(26)}) -> ok ${webhookId}`);
  } catch (e) {
    console.log(`ts verifies Python signature (${label.padEnd(26)}) -> ${e.name}`);
  }
}
server.close();
```

```js:ts/sign_ts.mjs
// Sign with the TypeScript library; cross_lang.py verify checks it in Python.
import { randomUUID } from "node:crypto";
import { writeFileSync } from "node:fs";
import { createPublicVerifyingKeySet, generateSecretSigningKey, sign } from "@handlewithcare/correct-hooks";

const key = generateSecretSigningKey();
const recipientId = randomUUID();
const targetUrl = "https://consumer.example.com/webhooks/orders";
const body = JSON.stringify({ type: "order.paid", order_id: "ord_123" });

const cases = [
  ["recipientId lowercase", recipientId],
  ["recipientId uppercase", recipientId.toUpperCase()],
].map(([label, rid]) => {
  const h = sign({
    webhookId: randomUUID(),
    recipientId: rid,
    targetUrl,
    method: "POST",
    requestBody: Buffer.from(body),
    secretSigningKey: key,
  });
  return { label, components: h.correctHooksSignatureComponents, signature: h.correctHooksSignature };
});

writeFileSync(
  new URL("../fixtures/ts_signed.json", import.meta.url),
  JSON.stringify({ keySet: createPublicVerifyingKeySet([key]), recipientId, targetUrl, body, cases }, null, 2),
);
console.log("wrote fixtures/ts_signed.json");
```
:::

```text
$ node -v
v22.23.3
wrote fixtures/py_signed.json
ts verifies Python signature (recipientId as given      ) -> ok ee6e595d-b7cb-4866-925e-1e7c9cc4db7b
ts verifies Python signature (recipientId uppercased    ) -> InvalidSignature
wrote fixtures/ts_signed.json
py verifies TS signature (recipientId lowercase     ) -> ok e2c453eb-a2ee-4ac7-afa4-02a16e27fd97
py verifies TS signature (recipientId uppercase     ) -> InvalidSignature
```

どちらの方向でも、検証は問題なく通りました。ただし、**recipientId の UUID を大文字にすると、どちらの方向でも失敗します**。これも「ハマりどころ」で説明します。

TypeScript 版の `VerifierConfig.publicVerifyingKeySetUrl` は、文字列ではなく **`URL` 型** で渡す必要があります。

### 7. 速度

README では、Rust で約 37,000 署名/秒とされています。Python 版で、5 KiB のボディを使って測ってみました（1 スレッド）。

:::details bench.py（全文）
```python:bench.py
"""Sign + verify throughput for a 5 KiB body on one CPU thread (vs. HMAC-SHA256 for reference)."""
import hashlib
import hmac
import os
import time
import uuid

from correct_hooks import (SignArgs, VerifyArgs, create_public_verifying_key_set,
                           generate_secret_signing_key, sign, verify_with_time)

N = 5000
key = generate_secret_signing_key()
key_set = create_public_verifying_key_set([key])
rid, url, body = uuid.uuid4(), "https://consumer.example.com/hooks", os.urandom(5 * 1024)


def rate(fn):
    t = time.perf_counter()
    for _ in range(N):
        fn()
    return N / (time.perf_counter() - t)


h = sign(SignArgs(webhook_id=uuid.uuid4(), recipient_id=rid, target_url=url, method="POST",
                  request_body=body, secret_signing_key=key))
va = VerifyArgs(target_url=url, method="POST", request_body=body,
                correct_hooks_signature_components=h.correct_hooks_signature_components,
                correct_hooks_signature=h.correct_hooks_signature)
now = int(time.time())
secret = os.urandom(32)
print(f"correct_hooks sign   : {rate(lambda: sign(SignArgs(webhook_id=uuid.uuid4(), recipient_id=rid, target_url=url, method='POST', request_body=body, secret_signing_key=key))):>9,.0f} /s")
print(f"correct_hooks verify : {rate(lambda: verify_with_time(args=va, recipient_id=rid, signature_ttl_seconds=300, public_verifying_key_set=key_set, now=now)):>9,.0f} /s")
print(f"hmac-sha256 (ref)    : {rate(lambda: hmac.new(secret, body, hashlib.sha256).digest()):>9,.0f} /s")
```
:::

```text
4
Model name:                              Intel(R) Core(TM) i5-8210Y CPU @ 1.60GHz
correct_hooks sign   :     7,882 /s
correct_hooks verify :     6,479 /s
hmac-sha256 (ref)    :    68,837 /s
```

1.6GHz のノート PC 向け CPU でも、**署名は約 7,900 回/秒、検証は約 6,500 回/秒** でした。HMAC と比べると 1 桁ほど遅いものの、Webhook の送受信がボトルネックになる量ではありません。

## ハマりどころ（Gotchas / Pitfalls）

以下の結果は、すべて次のスクリプトで再現できます（サーバーの起動は不要です）。

:::details gotchas.py（全文）
```python:gotchas.py
"""Pitfalls found while testing correct_hooks 0.1.1 (Python). No server needed."""
import asyncio
import json
import time
import uuid

from correct_hooks import (
    PublicVerifyingKeySet, SignArgs, Verifier, VerifierConfig, VerifyArgs,
    create_public_verifying_key_set, generate_secret_signing_key, sign, verify_with_time,
)

key = generate_secret_signing_key()
key_set = create_public_verifying_key_set([key])
recipient = uuid.uuid4()
url = "https://api.example.com/webhooks/orders"
body = json.dumps({"order_id": "ord_123", "amount": 4200}, separators=(",", ":")).encode()


def try_(label, fn):
    try:
        out = fn()
        print(f"{label:<54} -> ok {out if out is not None else ''}")
    except Exception as e:
        print(f"{label:<54} -> {type(e).__name__}: {(str(e).splitlines() or [""])[0][:70]}")


def signed(**over):
    args = dict(webhook_id=uuid.uuid4(), recipient_id=recipient, target_url=url,
                method="POST", request_body=body, secret_signing_key=key)
    args.update(over)
    return sign(SignArgs(**args))


def verify(h, *, target_url=url, method="POST", request_body=body, recipient_id=recipient):
    return verify_with_time(
        args=VerifyArgs(target_url=target_url, method=method, request_body=request_body,
                        correct_hooks_signature_components=h.correct_hooks_signature_components,
                        correct_hooks_signature=h.correct_hooks_signature),
        recipient_id=recipient_id, signature_ttl_seconds=300,
        public_verifying_key_set=key_set, now=int(time.time())).webhook_id


h = signed()
print("== A. README's Python verify() call shape (keyword args)")
verifier = Verifier(VerifierConfig(recipient_id=recipient, public_verifying_key_set_url="http://127.0.0.1:9/keys"))
try_("verifier.verify(target_url=..., ...)", lambda: asyncio.run(verifier.verify(
    target_url=url, method="POST", request_body=body,
    correct_hooks_signature_components=h.correct_hooks_signature_components,
    correct_hooks_signature=h.correct_hooks_signature)))

print("== B. target_url must be byte-for-byte identical")
try_("same string", lambda: verify(h))
try_("trailing slash", lambda: verify(h, target_url=url + "/"))
try_("request.url behind a proxy (http://10.0.0.5:8000/...)", lambda: verify(h, target_url="http://10.0.0.5:8000/webhooks/orders"))
try_("uppercase host", lambda: verify(h, target_url=url.replace("api.example.com", "API.example.com")))

print("== C. body must be the raw bytes")
try_("json.dumps(json.loads(body)) re-serialized", lambda: verify(h, request_body=json.dumps(json.loads(body)).encode()))
try_("body passed as str", lambda: verify(h, request_body=body.decode()))

print("== D. strict input types")
try_("sign(method='post')", lambda: signed(method="post"))
try_("sign(recipient_id=uuid1())", lambda: signed(recipient_id=uuid.uuid1()))
try_("sign(recipient_id='cust_42')", lambda: signed(recipient_id="cust_42"))
try_("sign(target_url='/webhooks/orders')", lambda: signed(target_url="/webhooks/orders"))

print("== E. what the public key set looks like (safe to serve)")
print(json.dumps(json.loads(key_set.model_dump_json()), indent=2))
print("secret key has 'd':", "d" in json.loads(key.model_dump_json()), "| public set has 'd':",
      any("d" in k for k in json.loads(key_set.model_dump_json())["keys"]))
```
:::

```text
== A. README's Python verify() call shape (keyword args)
verifier.verify(target_url=..., ...)                   -> TypeError: Verifier.verify() got an unexpected keyword argument 'target_url'
== B. target_url must be byte-for-byte identical
same string                                            -> ok 6705a5d3-7d47-46c0-a2fa-fd93131c6e76
trailing slash                                         -> InvalidSignature: 
request.url behind a proxy (http://10.0.0.5:8000/...)  -> InvalidSignature: 
uppercase host                                         -> InvalidSignature: 
== C. body must be the raw bytes
json.dumps(json.loads(body)) re-serialized             -> InvalidSignature: 
body passed as str                                     -> ValidationError: 1 validation error for VerifyArgs
== D. strict input types
sign(method='post')                                    -> InvalidArgs: 
sign(recipient_id=uuid1())                             -> ValidationError: 1 validation error for SignArgs
sign(recipient_id='cust_42')                           -> ValidationError: 1 validation error for SignArgs
sign(target_url='/webhooks/orders')                    -> InvalidArgs: 
== E. what the public key set looks like (safe to serve)
{
  "keys": [
    {
      "kid": "5a7e80d3-e69f-4bf9-80aa-37f4b1bcfe24",
      "kty": "OKP",
      "crv": "Ed25519",
      "x": "mvuopsEIyCkQ-643wL9EJVJpIcrGnO2M_-pjfhw33gU"
    }
  ]
}
secret key has 'd': True | public set has 'd': False
```

### 1. README の Python の verify 例は、0.1.1 ではそのまま動かない

README の Python クイックスタートでは、`verifier.verify(target_url=..., method=..., ...)` のようにキーワード引数で呼んでいます。しかし 0.1.1 の実際のシグネチャは `verify(self, args: VerifyArgs)` です。README どおりに書くと、`TypeError: Verifier.verify() got an unexpected keyword argument 'target_url'` になります。

```python
verified = await verifier.verify(VerifyArgs(
    target_url=TARGET_URL,
    method=request.method,
    correct_hooks_signature_components=...,
    correct_hooks_signature=...,
    request_body=await request.body(),
))
```

このように `VerifyArgs(...)` で包んで渡してください。同期版の `verify_sync()` もあります。

### 2. `target_url` は 1 文字でも違えば失敗する

URL は文字列のまま署名に含まれます。そのため、次のような違いでもすべて `InvalidSignature` になります。

- 末尾のスラッシュ
- ホスト名の大文字・小文字
- **リバースプロキシの裏で `request.url` を使った場合**（内部のホストやポートが入ってしまう）

受信側では、**送信側に登録した URL を設定値として持ち**、それを渡すのが確実です。

### 3. ボディは生バイトで渡す

`json.loads` → `json.dumps` で整形し直したボディは、空白の入り方が変わるので失敗します。FastAPI なら `await request.body()`、Express なら `express.raw()` で受け取ってください。Python 版は `request_body` に `str` を渡すと `ValidationError` になります（pydantic の strict モードのため）。

### 4. 入力の型がかなり厳しい（Python 版）

- `method="post"` は `InvalidArgs` になります。**大文字** にしてください。
- `recipient_id` と `webhook_id` は **UUID v4 のみ** です。`uuid1()` や `"cust_42"` のような独自 ID は `ValidationError` になります。既存の顧客 ID が UUID v4 でない場合は、Webhook 用の UUID を別途割り当てる必要があります。
- `target_url` に相対パスを渡すと `InvalidArgs` になります。

### 5. 言語をまたぐと、UUID の大文字・小文字で失敗する

Python 版は `recipient_id` を `uuid.UUID` に変換してから `str()` するため、署名メッセージには **常に小文字** の UUID が入ります。一方、TypeScript 版は渡された文字列をそのまま使います。そのため、片方で大文字の UUID を使うと `InvalidSignature` になります。言語をまたぐ場合は、**UUID を小文字に正規化して保存** しておきましょう。

### 6. 廃止した鍵は、キャッシュが切れるまで有効なまま

受信側は鍵セットを 300 秒キャッシュします。そのため、鍵セットから旧鍵を外しても、受信側がまだ取り直していなければ **旧鍵の署名が通ります**。ローテーション手順の 4 は「受信側から旧鍵が消える」ことまでは保証しません。

**漏洩した鍵を緊急で無効化** するときは、「最大 5 分間（＋CDN のキャッシュ時間）は、旧鍵の署名が通りうる」と考えておいてください。

### 7. 知らない `kid` が届くたびに、鍵セットを取り直す

デモのセクション 5 では、ランダムな鍵で署名したリクエストを 5 回送りました。その結果、**送信側の鍵セットのエンドポイントに 5 回アクセス** がありました。

受信側の Verifier は、知らない `kid` を見るたびに鍵セットを取り直します。並行して届いた分はロックでまとめられますが、順番に届くとそのたびに取得します。悪意のある第三者が受信側のエンドポイントにリクエストを送り続けると、受信側から送信側への GET が増えることになります。

送信側は、README にもあるとおり **鍵セットを CDN の裏に置く** のが安全です。受信側も、エンドポイントにレート制限をかけておくと安心です。

### 8. リプレイ対策は受信側の仕事

署名の有効期限内（最大 5 分）であれば、同じリクエストをそのまま送り直しても検証は通ります。`webhook_id` を Redis などに **TTL 5 分以上** で保存して、重複を弾いてください。この記事の consumer.py では、簡易的にプロセス内の set を使っています。

## まとめと次のステップ（Summary & Next Steps）

- Correct Hooks を使うと、**送信側は秘密鍵 1 本と公開 GET エンドポイント 1 つ** で Webhook 署名を運用できます。顧客ごとの共有シークレットは要りません。
- 署名には **受信者 ID・URL・メソッド・ボディ** が含まれるので、別の顧客あてやエンドポイント違いの流用は自動で弾かれます。
- 鍵ローテーションは、受信側の作業なしで切り替えられました。ただし、**旧鍵は最大 5 分残る** ことを前提に運用してください。
- 実装では、**URL は登録時の文字列、ボディは生バイト、UUID は小文字** の 3 点を守ればハマりません。

次のステップとしては、次のようなものが考えられます。

- **既存の HMAC 署名と並行運用する**：Correct Hooks のヘッダーを追加するだけなので、受信側が移行し終わるまで両方の署名を送れます（README の Migration の節）
- **Rust 版を試す**：`cargo add correct_hooks` で、同じ形式のまま高速に署名できます
- **再送とデッドレターを設計する**：README でも紹介されている [Standard Webhooks の運用上の注意](https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md#operational-considerations) が参考になります

---

*この記事のコードは、すべて上記の環境で実際に実行して確認しています。検証と執筆は Claude Code が行いました。*
