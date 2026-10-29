---
title: "farol入門：AIレビューの後に「このPRはどれだけ危ないか」を静的解析で出す【FastAPIで検証】"
emoji: "🗼"
type: "tech"
topics: ["farol", "python", "fastapi", "codereview", "ai"]
published: true
published_at: "2026-10-28 08:00"
---

## 概要（Overview）

AI にコードを書かせ、AI にレビューさせる。そこまで自動化しても、最後に「マージしてよいか」を決めるのは人間です。PR の数が増えると、人間のレビューは全部の PR に薄く広がり、流し読みになりがちです。

[farol](https://github.com/winnin/farol)（ポルトガル語で「灯台」）は、この **人間の承認者のための PR トリアージ CLI** です（MIT）。npm パッケージ `@winninjs/farol` の初版は 2026年10月22日で、2026年10月26日に Show HN に投稿されました。日本語の解説はまだ見当たりません。

farol は AI レビューの代わりにはなりません。AI レビューの **後** に動き、次の 3 つに答えます。

- **この変更はどこに届くか**: 変更された関数を呼んでいる既存のページ・API エンドポイント・ジョブを、呼び出し経路つきで列挙する
- **マージのリスクはどれくらいか**: *fast track* / *quick review* / *careful review* の判定と、0〜100 のリスクスコア
- **どこから読むべきか**: リスクの高いファイル順のリストと、手で試すべきエンドポイントのチェックリスト

LLM は使いません。TypeScript の型チェッカー、`go/types`、Python の `ast` で両方のツリーを静的に読むので、**同じ PR からは毎回同じ結果** が出ます。コードを実行することも、外部に送ることもありません。

この記事では、小さな FastAPI サービスに性質の違う 3 つの PR を用意し、farol が何をどう判定するかを実際に確かめます。

## 前提環境（Prerequisites）

検証環境は以下のとおりです（2026-10-26 時点）。

| ツール | バージョン |
| --- | --- |
| Node.js | 22.23.3（farol の `engines` は `>=22`） |
| farol (`@winninjs/farol`) | 0.1.3 |
| Python | 3.12.3 |
| FastAPI / Pydantic | 0.162.0 / 2.14.1 |
| pytest / pytest-cov | 9.1.1 / 7.1.0 |
| ruff | 0.17.2 |
| OS | Linux |

API キーもアカウントも要りません。ネットワークを使うのは npm と pip のインストールだけです。

```sh
npm i -D @winninjs/farol@0.1.3
python3 -m venv .venv && .venv/bin/pip install fastapi pydantic pytest pytest-cov httpx ruff
```

## Step 1: 題材の FastAPI サービス（main ブランチ）

商品一覧と見積もりを返すだけの小さな API です。ポイントは、価格の整形関数 `format_price` を **複数のルーターが共有している** ことです。

```python:app/pricing.py
def format_price(cents: int) -> str:
    return f"${cents / 100:.2f}"
```

```python:app/routers/items.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

from app.db import get_conn
from app.pricing import format_price

router = APIRouter()


class Item(BaseModel):
    id: int
    name: str
    price: str


@router.get("")
def list_items() -> list[Item]:
    rows = get_conn().execute("SELECT id, name, price FROM items").fetchall()
    return [Item(id=r[0], name=r[1], price=format_price(r[2])) for r in rows]


@router.get("/{item_id}")
def get_item(item_id: int) -> Item:
    row = get_conn().execute("SELECT id, name, price FROM items WHERE id = ?", (item_id,)).fetchone()
    if row is None:
        raise HTTPException(status_code=404)
    return Item(id=row[0], name=row[1], price=format_price(row[2]))
```

```python:app/routers/orders.py
from fastapi import APIRouter
from pydantic import BaseModel

from app.pricing import format_price

router = APIRouter()


class Quote(BaseModel):
    quantity: int
    total: str


@router.get("/quote")
def quote(unit_cents: int, quantity: int = 1) -> Quote:
    return Quote(quantity=quantity, total=format_price(unit_cents * quantity))
```

```python:app/main.py
from fastapi import FastAPI

from app.routers import health, items, orders

app = FastAPI(title="shop-api")
app.include_router(health.router)
app.include_router(items.router, prefix="/items")
app.include_router(orders.router, prefix="/orders")
```

```python:app/db.py
import sqlite3

ITEMS = [(1, "Keyboard", 4980), (2, "Mouse", 2480), (3, "Monitor", 32800)]


def get_conn() -> sqlite3.Connection:
    conn = sqlite3.connect(":memory:")
    conn.execute("CREATE TABLE items (id INTEGER PRIMARY KEY, name TEXT, price INTEGER)")
    conn.executemany("INSERT INTO items VALUES (?, ?, ?)", ITEMS)
    return conn
```

`app/routers/health.py` は `GET /health` で `{"status": "ok"}` を返すだけです。テストは `/health` と `/items` の 2 本です。

```python:tests/test_api.py
from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


def test_health():
    assert client.get("/health").json() == {"status": "ok"}


def test_list_items():
    body = client.get("/items").json()
    assert body[0] == {"id": 1, "name": "Keyboard", "price": "$49.80"}
```

farol の設定は 1 行で足ります。`pyproject.toml` には、後の Step で効いてくる設定を 2 つ入れておきます。

```json:farol.config.json
{ "language": "python", "surface": "backend", "python": { "roots": ["."] } }
```

```toml:pyproject.toml
[project]
name = "shop-api"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["fastapi>=0.115", "pydantic>=2"]

[tool.pytest.ini_options]
pythonpath = ["."]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]
```

### farol が見ているエントリーポイントを確認する

まず `routes` で、farol がどのエンドポイントを認識したかを確認します。

```sh
npx farol routes
```

```json
"key": "GET /health",
"key": "GET /items",
"key": "GET /items/{item_id}",
"key": "GET /orders/quote",
```

`include_router(..., prefix="/items")` のプレフィックスまで解決されています。farol はアプリを **import せず**、`ast` で読んでいるだけです。

## Step 2: 安全な PR → fast track

1 本目の PR は、`/version` エンドポイントのルーターとそのテストを **追加するだけ** です。

```python:app/routers/version.py
from fastapi import APIRouter

router = APIRouter()


@router.get("/version")
def version() -> dict[str, str]:
    return {"version": "0.2.0"}
```

```python:tests/test_version.py
from fastapi import FastAPI
from fastapi.testclient import TestClient

from app.routers import version

app = FastAPI()
app.include_router(version.router)


def test_version():
    assert TestClient(app).get("/version").json() == {"version": "0.2.0"}
```

カバレッジと ruff のレポートを作ってから analyze します。ruff の `S` ルールは flake8-bandit 相当で、farol はこれをセキュリティ所見として扱います。

```sh
git checkout pr-safe
mkdir -p reports
.venv/bin/pytest -q --cov=app --cov-report=json:reports/coverage.json
.venv/bin/ruff check --output-format json --extend-select S --exit-zero . > reports/ruff.json

npx farol analyze --base main --head HEAD \
  --coverage reports/coverage.json --eslint reports/ruff.json --out farol.json
npx farol report --in farol.json      # PR コメント用の Markdown
```

出力（抜粋）:

```
- **Review:** ✓ fast track — only new code, tests or low-risk changes; nothing existing is widely affected
- **Merge risk score:** 1/100 (low; higher is riskier)
- **Contracts:** no change to what endpoints receive or return (4 endpoint contracts read)
- **Diff coverage:** 100.0% (5 of 5 changed executable lines covered)
- **Lint on changed lines:** 0 errors, 0 warnings
```

既存コードに触れていないので、スコアは 1/100 です。ただし分類の詳細には次の 1 行があります。

```
- new file no route reaches (`new-file-disconnected`) `app/routers/version.py`: new file that no entry point reaches
```

この PR は `main.py` で `include_router` をしていません。つまり `/version` は本番では **存在しません**。farol はそれを「どのエントリーポイントからも届かない新規ファイル」として正しく指摘しますが、判定はあくまで fast track です。これについては「ハマりどころ」で詳しく触れます。

## Step 3: 危ない PR → careful review

2 本目の PR は、AI に「通貨を指定できるようにして、検索も足して」と頼んだら出てきそうな変更です。

```python:app/pricing.py（変更後）
def format_price(cents: int, currency: str = "USD") -> str:
    if currency == "JPY":
        return f"¥{cents:,}"
    return f"${cents / 100:,.2f}"
```

```python:app/routers/items.py（変更後）
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

from app.db import get_conn
from app.pricing import format_price

router = APIRouter()


class Item(BaseModel):
    id: int
    title: str
    price: str


@router.get("")
def list_items(currency: str) -> list[Item]:
    rows = get_conn().execute("SELECT id, name, price FROM items").fetchall()
    return [Item(id=r[0], title=r[1], price=format_price(r[2], currency)) for r in rows]


@router.get("/search")
def search_items(q: str) -> list[Item]:
    sql = f"SELECT id, name, price FROM items WHERE name LIKE '%{q}%'"
    rows = get_conn().execute(sql).fetchall()
    return [Item(id=r[0], title=r[1], price=format_price(r[2])) for r in rows]


@router.get("/{item_id}")
def get_item(item_id: int) -> Item:
    row = get_conn().execute("SELECT id, name, price FROM items WHERE id = ?", (item_id,)).fetchone()
    if row is None:
        raise HTTPException(status_code=404)
    return Item(id=row[0], title=row[1], price=format_price(row[2]))
```

テストも、新しいレスポンスに合わせて書き換えられています（`name` → `title`、`params={"currency": "USD"}` を追加）。

```python:tests/test_api.py（変更後の test_list_items）
def test_list_items():
    body = client.get("/items", params={"currency": "USD"}).json()
    assert body[0] == {"id": 1, "title": "Keyboard", "price": "$49.80"}
```

テストは通ります。差分だけを読むと、それなりにまともな PR に見えます。しかし問題は 4 つあります。

1. `format_price` の出力が変わった（`,` 区切りが入る）。`/orders/quote` もこの関数を使っている
2. レスポンスのフィールド `name` が `title` に変わった。既存のクライアントが壊れる
3. `currency` が **必須** のクエリパラメータになった。既存の `GET /items` がすべて 422 になる
4. `/search` が f-string で SQL を組み立てている（SQL インジェクション）

同じ手順で analyze すると、結果は次のとおりです。

```
- **Review:** ⚠ careful review — merge risk score 76/100; 1 high-severity security finding(s) on added lines; 2 endpoint(s) with breaking contract changes: GET /items, GET /items/{item_id} (check that clients are updated)
  - `app/routers/items.py` risk 48 · 56% of changed lines covered
  - `app/pricing.py` risk 21 · 75% of changed lines covered
- **Merge risk score:** 76/100 (high; higher is riskier)
- **Contracts:** 2 endpoints with breaking changes (5 endpoint contracts read)
  - 🔴 `GET /items`: `query:currency` new required parameter; `response[].name` field removed (+1 more) — callers unknown (no usage report)
  - 🔴 `GET /items/{item_id}`: `response.name` field removed (+1 more) — callers unknown (no usage report)
- **Risk:** 2 high, 0 medium, 0 low
  - 🔴 `app/pricing.py` 3/4 operations, 3 dependents
  - 🔴 `app/routers/items.py` 2/4 operations, 1 dependents
- **Security (static analysis of added lines):** 1 high, 0 medium, 0 low
  - `app/routers/items.py:24` high `bandit`: S608: Possible SQL injection vector through string-based query construction
- **Diff coverage:** 61.5% (8 of 13 changed executable lines covered) — uncovered ranges:
  - `app/pricing.py`: 3
  - `app/routers/items.py`: 24–26, 34
```

4 つの問題がすべて出ています。

- **影響範囲**: `app/pricing.py` は「4 つの既存オペレーションのうち 3 つ」から使われている、と出ます。差分に `orders.py` は出てこないのに、`GET /orders/quote` が影響を受けると分かります
- **契約の破壊**: Pydantic モデルと関数シグネチャの base と head を比べ、`response[].name` の削除と `query:currency` の必須化を **breaking** と判定しています
- **セキュリティ**: ruff の S608 が、追加された行の high 所見として拾われています
- **カバレッジ**: 追加した `/search`（24〜26 行）と、JPY の分岐（`pricing.py` 3 行目）にテストがないことが分かります

手で試すべき項目のチェックリストも付きます。

```
- [ ] new: `GET /items/search` (renders items.py)
- [ ] `GET /items` (via pricing.py)
- [ ] `GET /items/{item_id}` (via pricing.py)
- [ ] `GET /orders/quote` (via pricing.py)
```

### HTML レポート

`farol html` を使うと、1 ファイルの HTML レポートが出ます。ファイルごとに「レビュー済み」をチェックすると、残りのリスクスコアが減っていきます。

```sh
npx farol html --in farol.json --out farol.html
```

![farol の HTML レポート。Careful review、76/100、リスクの高い順に items.py と pricing.py が並ぶ](/images/farol-pr-risk-map-ja/farol-report.png)

### ある 1 ファイルの影響範囲だけを見る

`impact` を使うと、1 つのファイルからどのエンドポイントに届くかを経路つきで出せます。

```sh
npx farol impact app/pricing.py
```

```json
{
  "file": "app/pricing.py",
  "inGraph": true,
  "routes": [
    { "key": "GET /items", "via": ["app/routers/items.py", "app/pricing.py"] },
    { "key": "GET /items/search", "via": ["app/routers/items.py", "app/pricing.py"] },
    { "key": "GET /items/{item_id}", "via": ["app/routers/items.py", "app/pricing.py"] },
    { "key": "GET /orders/quote", "via": ["app/routers/orders.py", "app/pricing.py"] }
  ]
}
```

（実際の出力では `path` フィールドも付きます。紙幅の都合で省略しました。）

## Step 4: 修正後は `--since` で差分だけを見る

レビューで SQL インジェクションを指摘され、プレースホルダーに直してテストを足したとします。

```python:app/routers/items.py（search_items の修正）
@router.get("/search")
def search_items(q: str) -> list[Item]:
    rows = get_conn().execute(
        "SELECT id, name, price FROM items WHERE name LIKE ?", (f"%{q}%",)
    ).fetchall()
    return [Item(id=r[0], title=r[1], price=format_price(r[2])) for r in rows]
```

```python:tests/test_search.py
from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


def test_search_items():
    body = client.get("/items/search", params={"q": "Mou"}).json()
    assert [i["title"] for i in body] == ["Mouse"]


def test_search_is_not_injectable():
    assert client.get("/items/search", params={"q": "' OR '1'='1"}).json() == []
```

2 回目のレビューでは、前回見たコミット（`REVIEWED`）以降の差分だけを analyze します。

```sh
REVIEWED=<前回レビューした head の SHA>
npx farol analyze --base main --head HEAD --since "$REVIEWED" \
  --coverage reports/coverage.json --eslint reports/ruff.json --out since.json
```

```
- **Incremental: only the changes since b0d12724f276 (your last review). Links open the comparison of the new commits.**
- **Review:** ◐ quick review — existing code changed: 1 existing operation(s) run it
  - `app/routers/items.py` risk 5 · 100% of changed lines covered
- **Merge risk score:** 5/100 (low; higher is riskier)
- **Contracts:** no change to what endpoints receive or return (5 endpoint contracts read)
```

新しいコミットの影響は `GET /items/search` だけなので、quick review の 5/100 です。一方、PR 全体をもう一度 analyze すると次のようになります。

```
- **Review:** ⚠ careful review — merge risk score 60/100; 2 endpoint(s) with breaking contract changes: GET /items, GET /items/{item_id} (check that clients are updated); existing code changed: 3 existing operation(s) run it
```

セキュリティ所見は消えて 76 → 60 に下がりましたが、**破壊的な契約変更が残っているので careful review のまま** です。「修正が入ったからもう大丈夫」とはならないので、`name` → `title` を本当に意図しているか、クライアント側が追従しているかを人間が確かめる必要があります。

## ハマりどころ（Gotchas）

### 1. `include_router` 忘れは fast track のまま通る

Step 2 の PR は、ルーターを作ったのに `main.py` に登録していませんでした。farol は `new-file-disconnected`（どのエントリーポイントからも届かない新規ファイル）を **理由として** 出しますが、リスクとしては数えません。既存コードに影響しないので fast track、という判定自体は筋が通っています。ただし「機能が動かない」バグは見逃しやすくなります。fast track の PR でも、Classification の折りたたみ内の `new-file-disconnected` だけは目で確認する運用をおすすめします。

### 2. カバレッジが無いと「not verified」になる

最初に試したとき、`pyproject.toml` に `pythonpath = ["."]` を入れ忘れていて、pytest がテスト収集の段階で import エラーになりました。このときカバレッジの JSON はまともに出力されませんでしたが、farol は 0% とも合格とも言わず、**not verified** と表示します。

同じ危ない PR で比べると、スコアは次のように変わりました。

| 渡したレポート | スコア | Diff coverage |
| --- | --- | --- |
| なし | 79/100 | not verified |
| ruff のみ | 86/100 | not verified |
| ruff + coverage | 76/100 | 61.5% |

カバレッジを渡すと、テストされている行の分だけスコアが下がります。CI で pytest が落ちたときは、farol のスコアが高めに出る（安全側に倒れる）と覚えておくとよいです。

### 3. ruff の `S` ルールはテストの `assert` も拾う

`--extend-select S` を付けると、テストコードの `assert` が S101 として報告されます。farol はこれを「変更行上の lint エラー」として数えてしまうので、`[tool.ruff.lint.per-file-ignores]` で `"tests/**" = ["S101"]` を指定しておきましょう。

### 4. 判定は決定的

同じ PR を 2 回 analyze し、JSON を比べました。`analyzedAt` と `diagnostics.timingsMs`（処理時間）以外は完全に一致し、PR コメントの Markdown もバイト単位で同じでした。LLM レビューのように実行のたびに言い回しが変わることはありません。

### 5. Node 22 以上で動かす

`package.json` の `engines` は `>=22` です。この記事の検証はすべて Node 22.23.3 で行いました。nvm を使っている場合は `nvm use 22` を忘れないでください。

## 注意点・落とし穴（Pitfalls）

- **farol は承認も拒否もしない（shadow mode）**: CI のチェックを落とすことはなく、PR にコメントを付けるだけです。マージを止めたい場合は別の仕組みが必要です
- **静的に読めないものは見えない**: 実行時に組み立てるルートパスや `getattr`、DI コンテナ、Django REST Framework の router は読めません。読めない部分があると、PR は `unknown`（careful review）になります
- **呼び出し元のクライアントは分からない**: `callers unknown (no usage report)` と出ているとおり、どのサービスが壊れる API を呼んでいるかは、Datadog の usage レポートを渡さないと分かりません（今回は未検証）
- **スコアの閾値はヒューリスティック**: README によると、4 つのリポジトリの過去 662 コミットで調整されています。自分のリポジトリでは `farol calibrate` で確かめるのが前提です
- **GitHub Action は今回未検証**: `winnin/farol/action` による PR コメントの自動投稿は、ローカルでは試していません。この記事で確かめたのは CLI（`analyze` / `report` / `html` / `impact` / `routes`）だけです

## まとめと次のステップ（Summary & Next Steps）

- farol は、AI レビューの **後** に人間が「どれだけ真剣に読むべきか」を決めるための CLI です
- FastAPI では、ルートの解決、Pydantic モデルの契約差分、ruff の S ルール、coverage.py をまとめて、決定的な判定とスコアを出しました
- 共有関数の変更は、差分に出てこないエンドポイントへの影響まで呼び出し経路つきで示されました
- `--since` を使うと、2 回目のレビューは新しいコミットだけで済みます。ただし PR 全体の判定は別に確認しましょう

次のステップとして、次の 3 つを試してみてください。

1. 自分のリポジトリで `npx @winninjs/farol routes` を実行し、エントリーポイントが正しく読めているか確かめる
2. 既存の CI に pytest のカバレッジ JSON と `ruff --output-format json` を追加し、[公式の workflow 例](https://github.com/winnin/farol/tree/main/examples)をコピーする
3. `farol calibrate` で過去の履歴と照らし合わせ、スコアの閾値が自分のチームに合っているか確かめる

AI が書いたコードの「中身」を機械的に締めるなら、[ai-deslop](https://github.com/maharshi365/deslop) のような lint、「影響範囲」を人間に見せるなら farol、という使い分けになります。

