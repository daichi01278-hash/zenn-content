---
title: "Node 22で消せるdevDependencies：tsx・jest・dotenv・better-sqlite3なしでTypeScript CLIを作る【検証済み】"
emoji: "🪶"
type: "tech"
topics: ["nodejs", "typescript", "sqlite", "test", "node22"]
published: true
published_at: "2026-11-04 08:00"
---

## 概要（Overview）

Node.js 20 は **2026年4月30日にサポートが終了**しました。Node 22 以降に上げたプロジェクトは多いと思いますが、`package.json` の中身は Node 18 の頃から変わっていない、ということはないでしょうか。

Node 22.18 以降では、次のパッケージの役割をほぼ Node 本体だけでこなせます。

| これまで入れていたもの | Node 22.18+ の組み込み機能 |
| --- | --- |
| `tsx` / `ts-node` | `.ts` をそのまま実行（type stripping） |
| `better-sqlite3` | `node:sqlite`（FTS5 も使える） |
| `jest` / `vitest` | `node:test`（snapshot、フェイクタイマー、カバレッジのしきい値） |
| `dotenv` | `--env-file` / `--env-file-if-exists` |
| `chalk` | `util.styleText` |
| `minimist` / `yargs` | `util.parseArgs` |
| `npm run` | `node --run` |

機能ごとの解説はすでにいくつもあります。この記事では、**これらを全部まとめて 1 つの小さな CLI を作り、`dependencies` も `devDependencies` も 0 件のまま、テストとカバレッジの判定まで通します。** 実際に動かしてみて分かったハマりどころ（6 つ再現しました）も紹介します。

作るものは `linkstash` という、ブックマークを保存して全文検索する CLI です。

## 前提環境（Prerequisites）

| 項目 | バージョン |
| --- | --- |
| Node.js | 22.23.3（**22.18.0 以上が必要**） |
| OS | Linux |
| npm パッケージ | **なし**（`npm install` は一度も実行しません） |

検証日は 2026-11-02 です。Node 24 / 26 にも同じ機能がありますが、この記事で動作を確かめたのは 22.23.3 だけです。

ディレクトリ構成は次のとおりです。

```text
linkstash/
├── package.json
├── .env.example
├── src/
│   ├── cli.ts      # 引数解析と表示
│   ├── store.ts    # node:sqlite + FTS5
│   └── format.ts   # 相対時刻と色付け
└── test/
    ├── store.test.ts
    ├── store.test.ts.snapshot   # 自動生成
    └── format.test.ts
```

## Step-by-Step Code Guide

### Step 1: package.json — scripts だけを書く

```json:package.json
{
  "name": "linkstash",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22.18" },
  "bin": { "linkstash": "./src/cli.ts" },
  "scripts": {
    "start": "node --env-file-if-exists=.env --disable-warning=ExperimentalWarning src/cli.ts",
    "test": "node --test --test-reporter=spec \"test/**/*.test.ts\"",
    "test:snap": "node --test --test-update-snapshots \"test/**/*.test.ts\"",
    "coverage": "node --test --experimental-test-coverage --test-coverage-lines=90 --test-coverage-include=\"src/**\" \"test/**/*.test.ts\""
  }
}
```

押さえておきたい点は次の 4 つです。

- `"type": "module"` を指定し、ESM として扱います。
- `engines` に `>=22.18` と書いておきます。type stripping がフラグなしで有効になったのが **v22.18.0** だからです。
- `--env-file-if-exists` は、`.env` がなくてもエラーになりません（`--env-file` はファイルがないと失敗します）。
- scripts は `node --run start` のように呼び出します。`npm run` より起動が速く、npm がなくても動きます。

### Step 2: node:sqlite + FTS5 でストアを作る

```ts:src/store.ts
import { DatabaseSync } from 'node:sqlite';

export interface Link {
  id: number;
  url: string;
  title: string;
  tags: string;
  createdAt: number; // epoch ms
}

export interface SearchHit extends Pick<Link, 'id' | 'url' | 'title'> {
  snippet: string;
}

export class Store {
  #db: DatabaseSync;

  constructor(path: string) {
    this.#db = new DatabaseSync(path);
    this.#db.exec(`
      CREATE TABLE IF NOT EXISTS links (
        id INTEGER PRIMARY KEY,
        url TEXT NOT NULL UNIQUE,
        title TEXT NOT NULL,
        tags TEXT NOT NULL DEFAULT '',
        created_at INTEGER NOT NULL
      );
      CREATE VIRTUAL TABLE IF NOT EXISTS links_fts
        USING fts5(title, tags, content='links', content_rowid='id');
      CREATE TRIGGER IF NOT EXISTS links_ai AFTER INSERT ON links BEGIN
        INSERT INTO links_fts(rowid, title, tags) VALUES (new.id, new.title, new.tags);
      END;
    `);
  }

  add(url: string, title: string, tags: string[] = [], now = Date.now()): Link {
    const row = this.#db
      .prepare(
        `INSERT INTO links (url, title, tags, created_at) VALUES (?, ?, ?, ?)
         RETURNING id, url, title, tags, created_at AS createdAt`,
      )
      .get(url, title, tags.join(' '), now);
    return row as unknown as Link;
  }

  list(limit = 20): Link[] {
    return this.#db
      .prepare(
        `SELECT id, url, title, tags, created_at AS createdAt
         FROM links ORDER BY created_at DESC, id DESC LIMIT ?`,
      )
      .all(limit) as unknown as Link[];
  }

  search(query: string): SearchHit[] {
    return this.#db
      .prepare(
        `SELECT l.id, l.url, l.title,
                snippet(links_fts, 0, '[', ']', '…', 8) AS snippet
         FROM links_fts JOIN links l ON l.id = links_fts.rowid
         WHERE links_fts MATCH ? ORDER BY bm25(links_fts)`,
      )
      .all(query) as unknown as SearchHit[];
  }

  close(): void {
    this.#db.close();
  }
}
```

- `DatabaseSync` は同期 API で、`better-sqlite3` とほぼ同じ使い心地です。
- Node に組み込まれた SQLite では **FTS5 がそのまま使えます**。`bm25()` で並べ替え、`snippet()` で一致箇所をハイライトしています。
- `RETURNING` も使えるので、INSERT した行を 1 回のクエリで受け取れます。
- 後で説明するとおり、コンストラクタ引数に `private` を付ける書き方（parameter properties）は使えません。そのため ES の `#db` フィールドで書いています。

### Step 3: styleText で色を付ける

```ts:src/format.ts
import { styleText } from 'node:util';
import type { Link } from './store.ts';

const UNITS: [Intl.RelativeTimeFormatUnit, number][] = [
  ['day', 86_400_000],
  ['hour', 3_600_000],
  ['minute', 60_000],
];

const rtf = new Intl.RelativeTimeFormat('en', { numeric: 'auto' });

export function ago(createdAt: number, now = Date.now()): string {
  const diff = createdAt - now;
  for (const [unit, ms] of UNITS) {
    if (Math.abs(diff) >= ms) return rtf.format(Math.round(diff / ms), unit);
  }
  return 'just now';
}

export function formatLink(link: Link): string {
  const tags = link.tags ? ' ' + styleText('cyan', `#${link.tags.split(' ').join(' #')}`) : '';
  return `${styleText('bold', link.title)}${tags}\n  ${styleText('dim', link.url)} · ${ago(link.createdAt)}`;
}
```

`import type { Link } from './store.ts'` の **`type` と `.ts` 拡張子はどちらも省略できません**。理由はハマりどころの節で説明します。

### Step 4: parseArgs で CLI にする

```ts:src/cli.ts
#!/usr/bin/env node
import { parseArgs, styleText } from 'node:util';
import { Store } from './store.ts';
import { formatLink } from './format.ts';

const { values, positionals } = parseArgs({
  allowPositionals: true,
  options: {
    tag: { type: 'string', short: 't', multiple: true, default: [] },
    limit: { type: 'string', short: 'n', default: '20' },
  },
});

const [command, ...rest] = positionals;
const store = new Store(process.env.LINKSTASH_DB ?? 'linkstash.db');

try {
  switch (command) {
    case 'add': {
      const [url, ...title] = rest;
      if (!url) throw new Error('usage: linkstash add <url> [title...] [-t tag]');
      const link = store.add(url, title.join(' ') || url, values.tag);
      console.log(styleText('green', `saved #${link.id}`), link.title);
      break;
    }
    case 'list':
      for (const link of store.list(Number(values.limit))) console.log(formatLink(link));
      break;
    case 'search':
      for (const hit of store.search(rest.join(' '))) {
        console.log(`${hit.snippet}\n  ${styleText('dim', hit.url)}`);
      }
      break;
    default:
      throw new Error('commands: add | list | search');
  }
} catch (err) {
  console.error(styleText('red', (err as Error).message));
  process.exitCode = 1;
} finally {
  store.close();
}
```

`.env` には DB のパスだけを書きます。

```bash:.env.example
LINKSTASH_DB=demo.db
```

実際に動かしたときのログです（`cp .env.example .env` を実行したあと）。

```text
$ node --run start -- add https://nodejs.org/api/sqlite.html "node:sqlite reference" -t node -t sqlite
saved #1 node:sqlite reference
$ node --run start -- add https://nodejs.org/api/typescript.html "Modules: TypeScript" -t node -t typescript
saved #2 Modules: TypeScript
$ node --run start -- add https://nodejs.org/api/sqlite.html dup
UNIQUE constraint failed: links.url
(exit=1)
$ node --run start -- list
Modules: TypeScript #node #typescript
  https://nodejs.org/api/typescript.html · just now
node:sqlite reference #node #sqlite
  https://nodejs.org/api/sqlite.html · just now
$ node --run start -- search typescript
Modules: [TypeScript]
  https://nodejs.org/api/typescript.html
```

`demo.db` が作られたので、`.env` が読み込まれていることも確認できました。

### Step 5: node:test で snapshot とフェイクタイマーを使う

```ts:test/store.test.ts
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { Store } from '../src/store.ts';

test('add + list returns newest first', () => {
  const store = new Store(':memory:');
  store.add('https://nodejs.org', 'Node.js', ['runtime'], 1_000);
  store.add('https://sqlite.org', 'SQLite', ['db'], 2_000);
  assert.deepEqual(
    store.list().map((l) => l.title),
    ['SQLite', 'Node.js'],
  );
  store.close();
});

test('duplicate url is rejected by UNIQUE', () => {
  const store = new Store(':memory:');
  store.add('https://nodejs.org', 'Node.js');
  assert.throws(() => store.add('https://nodejs.org', 'again'), /UNIQUE constraint failed/);
  store.close();
});

test('full-text search ranks with bm25 and highlights the match', (t) => {
  const store = new Store(':memory:');
  store.add('https://a.example', 'Type stripping in Node 22', ['typescript']);
  store.add('https://b.example', 'SQLite FTS5 tutorial', ['sqlite']);
  store.add('https://c.example', 'Running TypeScript without a build step', ['typescript', 'node']);
  t.assert.snapshot(store.search('typescript'));
  store.close();
});
```

```ts:test/format.test.ts
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { ago, formatLink } from '../src/format.ts';

test('ago() with explicit now', () => {
  const now = Date.UTC(2026, 10, 2);
  assert.equal(ago(now - 30_000, now), 'just now');
  assert.equal(ago(now - 5 * 60_000, now), '5 minutes ago');
  assert.equal(ago(now - 86_400_000, now), 'yesterday');
});

test('formatLink() uses the mocked clock', (t) => {
  t.mock.timers.enable({ apis: ['Date'], now: Date.UTC(2026, 10, 2, 12) });
  const out = formatLink({
    id: 1,
    url: 'https://nodejs.org',
    title: 'Node.js',
    tags: 'runtime js',
    createdAt: Date.UTC(2026, 10, 2, 9),
  });
  // styleText drops colors when stdout is not a TTY / NO_COLOR is set
  assert.match(out, /Node\.js/);
  assert.match(out, /#runtime #js/);
  assert.match(out, /3 hours ago$/);
});
```

- `t.assert.snapshot()` は Jest の `toMatchSnapshot()` に相当します。初回は `node --run test:snap` で `.snapshot` ファイルを生成します。
- `t.mock.timers.enable({ apis: ['Date'] })` を使うと `Date.now()` を固定できます。テストが終わると自動で元に戻ります。

生成された snapshot は次のとおりです。FTS5 が `bm25` で順位を付けています。タイトルに "TypeScript" を含む c が 1 位、タグにだけ含む a が 2 位です。

```text:test/store.test.ts.snapshot
exports[`full-text search ranks with bm25 and highlights the match 1`] = `
[
  {
    "id": 3,
    "url": "https://c.example",
    "title": "Running TypeScript without a build step",
    "snippet": "Running [TypeScript] without a build step"
  },
  {
    "id": 1,
    "url": "https://a.example",
    "title": "Type stripping in Node 22",
    "snippet": "Type stripping in Node 22"
  }
]
`;
```

テストの実行結果です。

```text
$ node --run test
✔ ago() with explicit now (1.247322ms)
✔ formatLink() uses the mocked clock (1.252259ms)
✔ add + list returns newest first (2.798374ms)
✔ duplicate url is rejected by UNIQUE (1.184229ms)
✔ full-text search ranks with bm25 and highlights the match (2.957474ms)
ℹ tests 5
ℹ pass 5
ℹ fail 0
```

### Step 6: カバレッジのしきい値で CI を止める

```text
$ node --run coverage
# file       | line % | branch % | funcs % | uncovered lines
#  format.ts | 100.00 |    85.71 |  100.00 |
#  store.ts  | 100.00 |   100.00 |  100.00 |
# all files  | 100.00 |    92.86 |  100.00 |
```

しきい値を超えられないと exit 1 になるので、そのまま CI のゲートとして使えます。

```text
$ node --test --experimental-test-coverage --test-coverage-branches=95 \
    --test-coverage-include="src/**" "test/**/*.test.ts"
# Error: 92.86% branch coverage does not meet threshold of 95%.
(exit=1)
```

ここまでの手順で、`package.json` の依存は **0 件**です。

## ハマりどころ（Gotchas）

以下の 1〜6 は、空のディレクトリで 1 つずつ再現しました（Node 22.23.3）。

| # | コード | 結果 |
| --- | --- | --- |
| 1 | `import { User, VERSION } from './types.ts'`（User は interface） | `SyntaxError: The requested module './types.ts' does not provide an export named 'User'` |
| 2 | `import { VERSION } from './types'`（拡張子なし） | `ERR_MODULE_NOT_FOUND` |
| 3 | `enum Color { Red, Green }` | `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX: TypeScript enum is not supported in strip-only mode` |
| 4 | `constructor(private x: number) {}` | `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX: TypeScript parameter property is not supported ...` |
| 5 | `const n: number = 'not a number'` | **そのまま動く**（exit 0） |
| 6 | node_modules 内のパッケージが `.ts` を export している | `ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING` |

1. **型を import するときは `type` が必要です。** Node は型を削除するだけで、import されたものが型なのか値なのかは判断しません。`import { type User, VERSION }` と書けば動きました。
2. **相対 import には `.ts` 拡張子が必須です。** バンドラーのように拡張子を補完してくれません。
3. **enum は書けません。** `--experimental-transform-types` を付ければ動きました（ExperimentalWarning は出ます）。ただ、`as const` のオブジェクトに置き換えるほうが安全です。
4. **parameter properties も書けません。** フィールドを宣言して、コンストラクタ内で代入します。
5. **型チェックは行われません。** 型エラーがあっても普通に実行されます。型チェックは引き続き `tsc --noEmit` で行う必要があり、`typescript` パッケージだけは devDependencies に残ることになります（この記事では `tsc` は実行していません）。Node の公式ドキュメントでは、`erasableSyntaxOnly` と `verbatimModuleSyntax` を有効にした tsconfig が推奨されています。こうしておくと、1・3・4 を型チェックの段階で検出できます。
6. **配布するライブラリは JS にビルドしてから publish します。** node_modules 内の `.ts` は型が削除されません。

ほかに、テストまわりで気づいたことが 4 つあります。

- **snapshot がない状態で初回実行すると失敗します。** `Cannot read snapshot file ... Missing snapshots can be generated by rerunning the command with the --test-update-snapshots flag.` と表示されます。Jest のように自動で書き出されないので、生成した `.snapshot` は必ずコミットします。
- **ExperimentalWarning が毎回表示されます。** Node 22 では `node:sqlite` と MockTimers がまだ experimental 扱いで、実行のたびに警告が stderr に出ます。CLI では `--disable-warning=ExperimentalWarning` で消せます。テストでは「本当に experimental な API を使っている」ことが見えるように、あえて残しています。
- **カバレッジにはテストが import したファイルしか含まれません。** 上の表に `cli.ts` がないのはそのためです。`--test-coverage-include` は「集計に含める範囲」を絞る指定で、読み込まれていないファイルを 0% として数えることはしません。「100%」という数字を見て安心しないようにしてください。
- **Node 20 では何も動きません。** `node --run` は `bad option`、`.ts` は `ERR_UNKNOWN_FILE_EXTENSION` になりました。`engines` を書いておくのはこのためです。

## まとめと次のステップ（Summary & Next Steps）

- Node 22.18 以降なら、TS の実行・SQLite（FTS5 も含む）・テスト（snapshot、フェイクタイマー、カバレッジのしきい値）・.env・色付け・引数解析まで、**npm パッケージなしで**揃います。
- 制約は「型を削除するだけ」という点にあります。`import type`、`.ts` 拡張子、enum を使わない、の 3 つを tsconfig で強制し、型チェックは `tsc --noEmit` に任せる、という分担が現実的です。
- 依存が減れば `npm audit` の対象も減り、CI での `npm ci` も不要になります。小さな社内ツールや AI エージェント用のスクリプトから試してみるのがおすすめです。

次のステップとしては、次のようなものが考えられます。

- `node --test --watch` で TDD をする
- `process.loadEnvFile()` で、実行時に .env を読み込む
- `node:sqlite` の `backup()` を使ってバックアップ用のスクリプトを作る

参考リンク:

- [Node.js — Modules: TypeScript](https://nodejs.org/api/typescript.html)
- [Node.js — SQLite](https://nodejs.org/api/sqlite.html)
- [Node.js — Test runner](https://nodejs.org/api/test.html)
- [Node.js Release schedule](https://github.com/nodejs/Release)
