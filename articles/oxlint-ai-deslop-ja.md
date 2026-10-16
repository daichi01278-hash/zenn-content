---
title: "ai-deslop入門：AIが書いたTypeScriptの「雑さ」をOxlintで機械的に弾く【検証済み】"
emoji: "🧹"
type: "tech"
topics: ["oxlint", "typescript", "lint", "ai", "claudecode"]
published: true
---

## 概要（Overview）

AI に TypeScript を書かせると、型は通るのにレビューで引っかかるコードが出てきます。たとえば次のようなものです。

- `input as unknown as User` のような二重キャスト
- `unknown` や `object` を受け取るだけの関数
- `filter().map()` の連鎖
- `...(cond ? { x } : {})` のような条件付きスプレッド
- モックが呼ばれたかしか確かめないテスト

[ai-deslop](https://github.com/maharshi365/deslop) は、こうした「根拠の薄い（low-evidence）」書き方を検出する **Oxlint の JS プラグイン**です（MIT）。2026年10月14日に Show HN に投稿されたばかりで、日本語の解説はまだ見当たりません。

この記事では次の流れで、実際に手を動かして確認した結果をまとめます。

1. AI が書きそうなコードをわざと用意し、ルールが **何を、なぜ** 指摘するかを見る
2. 指摘どおりに直して、エラーを 0 件にする
3. 既存リポジトリに **段階的に** 入れる設定を作る

README に書かれていないハマりどころも、検証中にいくつか見つかりました（後半で紹介します）。

## 前提環境（Prerequisites）

検証環境は以下のとおりです（2026-10-14 時点）。

| ツール | バージョン |
| --- | --- |
| Node.js | 22.23.3（**22.18 以上が必要**。理由は後述） |
| oxlint | 1.88.0 |
| ai-deslop | 0.4.6 |
| oxlint-tsgolint | 7.0.2004（型情報を使うルールに必要） |
| OS | Linux |

LLM の API キーやクラウドのアカウントは不要です。すべてローカルで動きます。

## 手順（Step-by-Step Code Guide）

### 1. インストール

```bash
mkdir deslop-demo && cd deslop-demo
npm init -y
npm pkg set type=module
npm i -D oxlint ai-deslop oxlint-tsgolint
```

ポイントは2つあります。

- `oxlint-tsgolint` を入れ忘れると、`Failed to find tsgolint executable.` と表示されて **終了コード 1** で止まります。ai-deslop の推奨設定は型情報を使うルール（`typeAware: true`）を有効にするので、このパッケージが必要です
- `"type": "module"` がないと、TS の設定ファイルを読み込むたびに `MODULE_TYPELESS_PACKAGE_JSON` の警告が出ます

型情報を使うルールのために `tsconfig.json` も置いておきます。

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noEmit": true,
    "types": []
  },
  "include": ["src", "test"]
}
```

### 2. 設定ファイル

README のとおり、推奨設定をそのまま読み込みます。

```ts:oxlint.config.ts
import { defineConfig } from "oxlint";
import { configs as deslopConfigs } from "ai-deslop";

export default defineConfig({
  jsPlugins: ["ai-deslop"],
  options: deslopConfigs.recommended.options,
  rules: deslopConfigs.recommended.rules,
});
```

`recommended` の中身を出力して確認すると、deslop のルール 21 個と標準ルール 5 個（`typescript/no-floating-promises` など）が **すべて `error`** で、`options` は `{ "typeAware": true }` でした。

### 3. 「AI っぽい」コードを書く

1つのファイルに、よく見かけるパターンを詰め込みました。

```ts:src/user.ts
export type UserShape = { id: string; name: string; plan?: "free" | "pro" };
export type UserId = string;

export function parseUser(input: unknown): UserShape {
  if (typeof input === "object" && input !== null) {
    return input as unknown as UserShape;
  }
  throw new Error("invalid user");
}

export function toPayload(user: UserShape, includePlan: boolean) {
  return {
    id: user.id,
    ...(includePlan ? { plan: user.plan } : {}),
  };
}

export function activeNames(users: UserShape[]): string[] {
  return users.filter((u) => u.plan === "pro").map((u) => u.name);
}

export function logMeta(meta: object): void {
  console.log(Reflect.get(meta, "id"));
}

export function byId(users: UserShape[]) {
  return users.reduce((acc, u) => Object.assign({}, acc, { [u.id]: u }), {} as Record<string, UserShape>);
}

export type Account = UserShape;
```

```ts:src/jobs.ts
export async function sendMail(to: string): Promise<void> {
  await new Promise((r) => setTimeout(r, 1));
  console.log("sent", to);
}

export function onSignup(email: string): void {
  sendMail(email);
}

export function label(n: number): string {
  if (n !== null) return `#${n}`;
  return "none";
}
```

```ts:test/user.test.ts
import { describe, it, expect, vi } from "vitest";
vi.mock("../src/jobs.js");
import { sendMail, onSignup } from "../src/jobs.js";

describe("onSignup", () => {
  it("sends mail", () => {
    onSignup("a@example.com");
    expect(sendMail).toHaveBeenCalled();
  });
  it("sends mail to the user", () => {
    onSignup("a@example.com");
    expect(sendMail).toHaveBeenCalledWith("a@example.com");
  });
  it("message", () => {
    const MSG = "welcome to pro plan";
    expect(MSG).toContain("pro");
  });
});
```

### 4. 実行する

```bash
npx oxlint src test
```

結果は **16 種類のルールで 25 件のエラー**、終了コードは 1 でした。出力の一部を載せます。

```text
  x deslop(no-chained-type-assertions): This assertion chain discards type evidence.
    Keep the original precise type, or parse untrusted input at its boundary before narrowing it.
   ,-[src/user.ts:6:12]
 5 |   if (typeof input === "object" && input !== null) {
 6 |     return input as unknown as UserShape;
   :            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 7 |   }
   `----

  x typescript(no-floating-promises): Promises must be awaited, add void operator to ignore.
   ,-[src/jobs.ts:7:3]
 6 | export function onSignup(email: string): void {
 7 |   sendMail(email);
   :   ^^^^^^^|^^^^^^^^
   :          `-- This unhandled promise-like value has type `Promise<void>`.
 8 | }
   `----
```

検出されたルールと、それぞれの意図は次のとおりです。

| ルール | 検出箇所 | 意図 |
| --- | --- | --- |
| `no-unknown-parameters` / `no-object-parameters` | `input: unknown`、`meta: object` | 型の境界をあいまいにしない。パースは I/O 境界で一度だけ行う |
| `no-runtime-typeof` | `typeof input === "object"` | その場しのぎの絞り込みではなく、境界でパースする |
| `no-chained-type-assertions` | `as unknown as` | 型の根拠を捨てる二重キャストを禁止する |
| `require-safety-comment-for-type-assertion` | `as` 全般 | キャストするなら `// SAFETY:` コメントで、成り立つ前提を書く |
| `no-conditional-empty-object-spread` | `...(c ? {x} : {})` | プロパティの省略を空オブジェクトで隠さない |
| `no-array-filter-map` | `.filter().map()` | 配列を2回走査しない |
| `no-reduce-accumulator-copy` | `Object.assign({}, acc, …)` | 毎回コピーすると計算量が O(n²) になる |
| `no-reflect-get` | `Reflect.get` | 型付きのプロパティアクセスを使う |
| `no-shape-in-symbol-names` | `UserShape` | 名前には構造（shape）ではなく、ドメイン上の役割を付ける |
| `no-pass-through-type-alias` | `type Account = UserShape` | ただの別名を増やさない |
| `typescript/no-floating-promises` | `sendMail(email);` | await し忘れた Promise を検出する |
| `typescript/no-unnecessary-condition` | `n !== null`（`n: number`） | 常に同じ結果になる条件を検出する |
| `no-module-mocking` | `vi.mock(...)` | モジュールのモックではなく依存性注入を使う |
| `no-call-only-assertions` | `toHaveBeenCalled()` だけのテスト | 「呼ばれたか」だけを確かめるテストを禁止する |
| `no-poor-substring-tests` | 定数に対する `toContain` | 意味のない部分一致テストを禁止する |

実際に試すと、ルールが **意外と見逃してくれる** ケースもありました。

- `type UserId = string` は検出されません。`no-pass-through-type-alias` は「別の名前付きの型への別名」だけを対象にしています
- `toHaveBeenCalledWith("a@example.com")` だけのテストは許可されます。引数まで確かめていれば、実データを検証していると判断されるためです

このように「何でも禁止」ではなく、線引きが考えられています。

### 5. 指摘どおりに直す

方針は「`unknown` を受け取るのは I/O 境界のパース関数だけにして、その先はドメイン型で扱う」です。

```ts:fixed/user.ts
export type Plan = "free" | "pro";
export type User = { id: string; name: string; plan?: Plan };
export type UserPayload = { id: string; plan?: Plan };

// I/O 境界で一度だけパースする
export function parseUser(json: string): User {
  // SAFETY: 以下ですべてのフィールドを検証してから User として返す
  const raw = JSON.parse(json) as Partial<Record<keyof User, string>>;
  const { id, name, plan } = raw;
  if (id === undefined || name === undefined) throw new Error("invalid user");
  if (plan !== undefined && plan !== "free" && plan !== "pro") throw new Error("invalid plan");
  const user: User = { id, name };
  if (plan !== undefined) user.plan = plan;
  return user;
}

export function toPayload(user: User, includePlan: boolean): UserPayload {
  const payload: UserPayload = { id: user.id };
  if (includePlan && user.plan !== undefined) payload.plan = user.plan;
  return payload;
}

export function activeNames(users: User[]): string[] {
  return users.flatMap((u) => (u.plan === "pro" ? [u.name] : []));
}

export function logMeta(user: User): void {
  console.log(user.id);
}

export function byId(users: User[]): Map<string, User> {
  const map = new Map<string, User>();
  for (const u of users) map.set(u.id, u);
  return map;
}
```

```bash
$ npx oxlint fixed
Found 0 warnings and 0 errors.
Finished in 304ms on 1 file with 135 rules using 4 threads.
```

なお、`// SAFETY:` の行を消すと `require-safety-comment-for-type-assertion` のエラーが 1 件出ます。**キャストそのものは禁止せず、前提を言葉で書かせる**というルールです。

### 6. 期待どおりに動くかをスクリプトで確かめる

`-f json` で出力し、期待したルールがすべて検出されているかを確認するスクリプトです。ルールのバージョンを上げたときの回帰テストにも使えます。

```js:verify.mjs
import { execFileSync } from "node:child_process";

function lint(...paths) {
  try {
    execFileSync("npx", ["oxlint", "-f", "json", ...paths], { encoding: "utf8" });
    return { code: 0, diagnostics: [] };
  } catch (e) {
    return { code: e.status, diagnostics: JSON.parse(e.stdout).diagnostics };
  }
}

const expected = {
  "src/user.ts": [
    "no-shape-in-symbol-names", "no-unknown-parameters", "no-runtime-typeof",
    "no-chained-type-assertions", "require-safety-comment-for-type-assertion",
    "no-conditional-empty-object-spread", "no-array-filter-map", "no-object-parameters",
    "no-reflect-get", "no-reduce-accumulator-copy", "no-pass-through-type-alias",
  ],
  "src/jobs.ts": ["no-floating-promises", "no-unnecessary-condition"],
  "test/user.test.ts": ["no-module-mocking", "no-call-only-assertions", "no-poor-substring-tests"],
};

let failed = 0;
const bad = lint("src", "test");
for (const [file, rules] of Object.entries(expected)) {
  const hit = new Set(bad.diagnostics.filter((d) => d.filename === file).map((d) => d.code.replace(/^.*\((.*)\)$/, "$1")));
  for (const rule of rules) {
    const ok = hit.has(rule);
    if (!ok) failed++;
    console.log(`${ok ? "PASS" : "FAIL"} ${file}: ${rule}`);
  }
}
console.log(`sloppy code: exit ${bad.code}, ${bad.diagnostics.length} diagnostics`);
if (bad.code !== 1) failed++;

const good = lint("fixed");
console.log(`${good.code === 0 ? "PASS" : "FAIL"} fixed/: exit ${good.code}, ${good.diagnostics.length} diagnostics`);
if (good.code !== 0) failed++;

process.exit(failed ? 1 : 0);
```

```text
PASS src/user.ts: no-shape-in-symbol-names
...（中略・16 項目すべて PASS）...
PASS test/user.test.ts: no-poor-substring-tests
sloppy code: exit 1, 25 diagnostics
PASS fixed/: exit 0, 0 diagnostics
```

3 ディレクトリ・4 ファイルの lint は、実時間 0.7 秒ほどで終わりました（型情報を使うルールを含みます）。

### 7. 既存リポジトリへ段階的に入れる

既存のコードにいきなり全ルールを `error` で入れると、CI が真っ赤になります。そこで、まずは **deslop のルールを warn にして、件数の上限で「増やさない」ことだけを保証する** 設定を作りました。

```ts:oxlint.gradual.config.ts
import { defineConfig } from "oxlint";
import { configs as deslopConfigs } from "ai-deslop";

const asWarn = Object.fromEntries(
  Object.keys(deslopConfigs.recommended.rules).map((rule) => [rule, "warn"]),
);

export default defineConfig({
  jsPlugins: ["ai-deslop"],
  options: deslopConfigs.recommended.options,
  rules: {
    ...asWarn,
    // 実害が大きいものだけは最初から error
    "typescript/no-floating-promises": "error",
  },
});
```

| 実行内容 | 結果 | 終了コード |
| --- | --- | --- |
| `--max-warnings=30`（警告 23 件） | 上限以内なので通る | 0 |
| `--max-warnings=5`（警告 23 件） | `Exceeded maximum number of warnings.` | 1 |
| await し忘れた Promise を含む | エラーが 1 件出る | 1 |

上限を現在の件数に合わせておき、直すたびに下げていく、という運用ができます。

GitHub Actions では `-f github` を付けると、PR の差分の該当行に注釈が付きます。

```yaml:.github/workflows/lint.yml
- uses: actions/setup-node@v4
  with:
    node-version: 22
- run: npm ci
- run: npx oxlint -c oxlint.gradual.config.ts --max-warnings=23 -f github  # 上限は現在の警告件数に合わせる
```

```text
::error file=src/jobs.ts,line=7,endLine=7,col=3,endColumn=18,title=typescript(no-floating-promises)::src/jobs.ts:7:3: Promises must be awaited, add void operator to ignore.
```

（`-f github` の出力形式はローカルで確認済みです。Actions 上での実行は、この記事では試していません。）

## ハマりどころ（Gotchas / Pitfalls）

### 1. `oxlint-tsgolint` を入れないと何も検査されない

推奨設定は型情報を使うルールを有効にするので、`oxlint-tsgolint` がないと全体が `Failed to find tsgolint executable.` で止まります。README のインストール手順には、このパッケージが書かれていません。

### 2. AI エージェントの中で動かすと出力形式が勝手に変わる

検証中に一番驚いたのがこれです。環境変数 `CLAUDECODE`・`AI_AGENT`・`CODEX_SANDBOX` のいずれかが設定されていると、oxlint は **コード抜粋なしの 1 行形式（`agent` 形式）** に自動で切り替わります。

```text
src/jobs.ts:7:3: error typescript(no-floating-promises): Promises must be awaited, add void operator to ignore. help: ...
```

Claude Code や Codex に修正させるときはトークンの節約になります。ただ、同じコマンドなのに手元とエージェントで出力が違うので、ログを解析する側では戸惑います。CI やスクリプトでは `-f json` や `-f github` を **明示** しましょう（`CI=true` だけでは切り替わりませんでした）。

### 3. TS の設定ファイルは Node 22.18 以上で使う

Node 20.20.2 で実行すると、次のエラーになりました。

```text
Error: Unknown file extension ".ts" for .../oxlint.config.ts
TypeScript config files require Node.js ^20.19.0 || >=22.18.0.
Detected Node.js v20.20.2.
```

メッセージには「^20.19.0 なら対応」と書いてあるのに、20.20.2 で失敗します。Node 22.18 以上を使うか、設定を `.oxlintrc.json` に書くのが確実です。

### 4. ルールは「意見が強い（opinionated）」

`no-shape-in-symbol-names` は `UserShape` のような名前を一律でエラーにしますし、`no-module-mocking` は `vi.mock` を全面的に禁止します。チームの方針と合わないルールは、`rules` で `"off"` にして構いません。推奨設定は出発点であって、正解ではありません。

### 5. JS プラグインはまだアルファ版

Oxlint の JS プラグイン機能は、公式に **alpha** とされています。また、ai-deslop は `@oxlint/plugins` を固定バージョンで依存に持っていて、README にも「oxlint のバージョンと揃えること」と書かれています。`oxlint` と `ai-deslop` は一緒にアップデートし、`verify.mjs` のような回帰テストで確認するのが安全です。

## まとめと次のステップ（Summary & Next Steps）

- ai-deslop を入れると、「型は通るけれど根拠の薄いコード」を **機械的に、しかも 1 秒以内で** 検出できます
- 指摘メッセージには「I/O 境界でパースする」「`SAFETY:` で前提を書く」といった **直し方の方針** まで書かれているので、AI に修正させるときの指示としてもそのまま使えます
- 既存リポジトリには、warn と `--max-warnings` で段階的に入れるのがおすすめです

次のステップとしては、次のようなものが考えられます。

- **AI の修正ループに組み込む**：Claude Code の hooks などで、ファイルを編集するたびに `oxlint` を走らせ、指摘を AI に返して直させる
- **オプトインのルールを試す**：`no-duplicate-types`（同じ型定義の重複を検出）や `canonical-class-names`（Tailwind v4 のクラス名を正しい表記にそろえる）
- **自作ルールを作る**：Oxlint の JS プラグインは ESLint v9 互換の API なので、チーム独自の「AI にやってほしくないこと」もルールにできる

---

*この記事のコードは、すべて上記の環境で実際に実行して確認しています。検証と執筆は Claude Code が行いました。*
