---
title: "HyperFrames入門：HTMLテンプレート1枚とJSONでMP4動画を一括生成する【検証済み】"
emoji: "🎬"
type: "tech"
topics: ["hyperframes", "gsap", "ffmpeg", "nodejs", "動画生成"]
published: true
---

## 概要（Overview）

[HyperFrames](https://github.com/heygen-com/hyperframes) は、HeyGen が公開している **HTML / CSS / JS アニメーションを決定論的に MP4 へレンダリングする OSS（Apache-2.0）** です。今週の GitHub Trending 上位に入っていて、スターは 58k を超えています。「AIエージェントのための動画フレームワーク」として注目されています。

ただ、紹介記事の多くは `init → preview → render` で終わっていて、自動化で本当に役立つ機能まで触れていません。その機能がこの2つです。

- **`data-composition-variables`**：HTMLテンプレートに型付きの変数を宣言する
- **`render --batch rows.json`**：**JSONの1行ごとにMP4を1本**レンダリングする

この2つを組み合わせると、次のような動画を **HTMLファイル1枚** から量産できます。React も動画編集ソフトも要りません。

- CHANGELOG からリリース告知動画を作る
- 顧客ごとのオンボーディング動画を作る
- SNS用の動画カードを作る

本記事では5秒の「リリース告知」テンプレートを作り、JSONの3行から3本の動画を一括生成します。**掲載しているコマンドと出力は、すべてローカルで実際に実行したもの** です（HyperFrames v0.8.140）。

![バッチ生成した2本の動画から切り出したフレーム](/images/hyperframes/contact-sheet.png)

## 前提環境（Prerequisites）

| ツール | 検証バージョン | 備考 |
|---|---|---|
| Node.js | 22.23.3 | **22以上が必須**。Node 20 では CLI が起動しません |
| FFmpeg / ffprobe | 6.1.1 | `sudo apt install ffmpeg` / `brew install ffmpeg` |
| hyperframes CLI | 0.8.140 | `npx` で実行（グローバルインストール不要） |
| Chrome Headless Shell | 152 | `npx hyperframes browser ensure` で取得 |

```bash
node -v                                  # 22以上であること
npx hyperframes@0.8.140 doctor           # FFmpeg・Chrome・メモリ・/dev/shm をまとめてチェック
npx hyperframes@0.8.140 browser ensure   # ヘッドレスChromeを初回だけダウンロード
```

## 手順（Step-by-Step Code Guide）

### 1. ディレクトリ構成

```text
hyperframes-batch-video/
├── index.html          # テンプレート（コンポジション）
├── hyperframes.json    # プロジェクト設定（hyperframes init で生成）
├── package.json
├── data/
│   ├── one.json        # 変数オブジェクト1件
│   └── releases.json   # 行の配列 → 1行ごとに動画1本
└── verify.mjs          # ffprobe で出力を検証
```

雛形は `npx hyperframes@0.8.140 init my-video --example blank --non-interactive` で作れます。生成された `index.html` を以下の内容に置き換えてください。

### 2. 変数を宣言してテンプレートを作る

変数は **`<html>` 要素に JSON 配列で宣言** します。ページ内で `window.__hyperframes.getVariables()` を呼ぶと、マージ済みの値が返ります。宣言したデフォルト値の上に、CLI やバッチ行で渡した値が上書きされます。

```html
<!doctype html>
<html
  lang="en"
  data-composition-variables='[
    {"id":"product","type":"string","label":"Product","default":"Acme CLI"},
    {"id":"version","type":"string","label":"Version","default":"v1.0.0"},
    {"id":"headline","type":"string","label":"Headline","default":"Faster than ever"},
    {"id":"features","type":"string","label":"Features (| separated)","default":"New API|Smaller bundle|Bug fixes"},
    {"id":"accent","type":"color","label":"Accent color","default":"#6366f1"}
  ]'
>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=1280, height=720" />
    <script src="https://cdn.jsdelivr.net/npm/gsap@3.14.2/dist/gsap.min.js"></script>
    <style>
      * { margin: 0; padding: 0; box-sizing: border-box; }
      html, body { width: 1280px; height: 720px; overflow: hidden; background: #0b0b10; }
      #root { position: relative; width: 100%; height: 100%; font-family: ui-sans-serif, system-ui, sans-serif; color: #f4f4f5; }
      .clip { position: absolute; inset: 0; display: flex; flex-direction: column; justify-content: center; padding: 0 96px; }
      #badge { align-self: flex-start; padding: 8px 20px; border-radius: 999px; font-size: 28px; font-weight: 700; color: #0b0b10; }
      #product { font-size: 40px; opacity: 0.7; margin-top: 24px; }
      #headline { font-size: 80px; font-weight: 800; letter-spacing: -0.03em; line-height: 1.05; }
      #features li { list-style: none; font-size: 44px; margin: 14px 0; }
      #features li::before { content: "→ "; color: var(--accent); }
    </style>
  </head>
  <body>
    <div id="root" data-composition-id="main" data-start="0" data-duration="5" data-width="1280" data-height="720">
      <section id="intro" class="clip" data-start="0" data-duration="2.5" data-track-index="0">
        <span id="badge"></span>
        <h1 id="headline"></h1>
        <p id="product"></p>
      </section>
      <section id="list" class="clip" data-start="2.5" data-duration="2.5" data-track-index="0">
        <ul id="features"></ul>
      </section>
    </div>
    <script>
      // 1) Read merged variables (defaults < --variables / batch row)
      const v = window.__hyperframes.getVariables();
      document.documentElement.style.setProperty("--accent", v.accent);
      document.querySelector("#badge").textContent = v.version;
      document.querySelector("#badge").style.background = v.accent;
      document.querySelector("#headline").textContent = v.headline;
      document.querySelector("#product").textContent = v.product;
      for (const f of v.features.split("|")) {
        const li = document.createElement("li");
        li.textContent = f; // textContent: data rows never become HTML
        document.querySelector("#features").append(li);
      }

      // 2) Build a PAUSED timeline and register it under the composition id
      const tl = gsap.timeline({ paused: true });
      tl.from("#badge", { opacity: 0, y: 20, duration: 0.4 }, 0.1)
        .from("#headline", { opacity: 0, y: 40, duration: 0.6 }, 0.3)
        .from("#product", { opacity: 0, duration: 0.4 }, 0.8)
        .from("#features li", { opacity: 0, x: -40, duration: 0.4, stagger: 0.25 }, 2.6);
      window.__timelines["main"] = tl;
      tl.seek(0);
    </script>
  </body>
</html>
```

ポイント：

- `data-composition-id="main"` と `data-duration="5"` で、5秒の動画になります。
- 各 `.clip` の `data-start` / `data-duration`（秒）に従って、HyperFrames が要素の表示・非表示を切り替えます。
- GSAP タイムラインは **必ず `paused: true`** で作り、`window.__timelines["<composition-id>"]` に登録します。HyperFrames はタイムラインを再生するのではなく、**フレームごとに seek** して書き出します。これが「決定論的レンダリング」の仕組みです。

### 3. lint とランタイムチェック

```bash
npx hyperframes@0.8.140 check
```

実際の出力（抜粋）：

```text
Lint      0 error(s), 2 warning(s), 0 info(s)
Runtime   ◇ 0 errors, 0 warnings
Layout    ◇ 0 issues across 9 sample(s)
Motion    ◇ 0 errors, 0 warnings
Contrast  0 error(s), 1 warning(s), 0 info(s)
◇  Check passed
```

lint の警告2件については「ハマりどころ」で説明します。

### 4. 変数を渡して1本レンダリング

`data/one.json`：

```json
{ "product": "billshock", "version": "v0.2.0", "headline": "Catch cloud bill spikes in CI", "features": "GitHub Action|Slack alerts|JSON output", "accent": "#f97316" }
```

```bash
npx hyperframes@0.8.140 render \
  --variables-file data/one.json --strict-variables \
  -o renders/single.mp4
```

```text
◇  renders/single.mp4
   275.2 KB · 5.0s video · rendered in 12.4s
   screenshot capture · software gpu · ... capture 10.0s · encode (during capture) 9.9s · assemble 0.1s
```

GPU なしの4コアノートPC（i5-8210Y）で、150フレームを約12秒で書き出せました。

### 5. バッチレンダリング：JSONの1行 → MP4 1本

`data/releases.json`：

```json
[
  { "product": "Acme CLI", "version": "v2.0.0", "headline": "Rewritten in Rust", "features": "10x faster startup|Single binary|Plugin API", "accent": "#22c55e" },
  { "product": "Acme CLI", "version": "v2.1.0", "headline": "Windows support", "features": "PowerShell completions|MSI installer", "accent": "#3b82f6" },
  { "product": "Acme Cloud", "version": "v0.9.0", "headline": "Usage-based billing", "features": "Stripe metering|Per-seat caps|CSV export", "accent": "#e11d48" }
]
```

```bash
npx hyperframes@0.8.140 render \
  --batch data/releases.json \
  --strict-variables --json \
  -o "renders/{index}-{version}.mp4"
```

`--json` を付けると、結果が機械可読な JSON 1つで出力されます。実際の出力（rows は抜粋）：

```json
{"type":"batch-complete","manifestPath":".../renders/manifest.json","total":3,"completed":3,"failed":0,"skipped":0,
 "rows":[{"index":0,"outputPath":".../renders/0-v2.0.0.mp4","status":"completed","durationMs":5000,"renderTimeMs":11985, ...}, ...]}
```

```text
renders/0-v2.0.0.mp4
renders/1-v2.1.0.mp4
renders/2-v0.9.0.mp4
renders/manifest.json
```

### 6. 出力を自動検証する

量産した動画を目視で全部確認するのは現実的ではありません。そこで `verify.mjs` を用意しました。`manifest.json` を読み、`ffprobe` でコーデック・解像度・尺をチェックしたうえで、目視確認用に各動画から2フレームを書き出します。

```js
// Verifies every rendered MP4 with ffprobe and dumps two frames per video.
import { execFileSync } from "node:child_process";
import { readFileSync, mkdirSync } from "node:fs";

const manifest = JSON.parse(readFileSync("renders/manifest.json", "utf8"));
mkdirSync("renders/frames", { recursive: true });
let failed = 0;

for (const row of manifest.rows) {
  const probe = JSON.parse(
    execFileSync("ffprobe", [
      "-v", "error", "-select_streams", "v:0",
      "-show_entries", "stream=width,height,codec_name,r_frame_rate:format=duration",
      "-of", "json", row.outputPath,
    ]).toString(),
  );
  const s = probe.streams[0];
  const duration = Number(probe.format.duration);
  const ok = row.status === "completed" && s.width === 1280 && s.height === 720 && Math.abs(duration - 5) < 0.1;
  if (!ok) failed++;
  console.log(
    `${ok ? "PASS" : "FAIL"} row=${row.index} ${row.variables.version} ` +
      `${s.codec_name} ${s.width}x${s.height} ${s.r_frame_rate}fps ${duration.toFixed(2)}s`,
  );
  for (const t of [1.5, 4.5]) {
    execFileSync("ffmpeg", [
      "-v", "error", "-y", "-ss", String(t), "-i", row.outputPath,
      "-frames:v", "1", `renders/frames/${row.index}-${t}s.png`,
    ]);
  }
}
console.log(failed === 0 ? `All ${manifest.rows.length} videos verified.` : `${failed} video(s) failed.`);
process.exit(failed === 0 ? 0 : 1);
```

```text
$ node verify.mjs
PASS row=0 v2.0.0 h264 1280x720 30/1fps 5.00s
PASS row=1 v2.1.0 h264 1280x720 30/1fps 5.00s
PASS row=2 v0.9.0 h264 1280x720 30/1fps 5.00s
All 3 videos verified.
```

## ハマりどころ（Gotchas / Pitfalls）

1. **Node 20 では動かない**
   `HyperFrames requires Node.js >= 22` と表示されて終了します。nvm を使っているなら `nvm install 22 && nvm use 22` で切り替えてください。`package.json` に `"engines": {"node": ">=22"}` を書いておくと安心です。

2. **`-o` を指定しないと、バッチの出力ファイル名がタイムスタンプになる**
   デフォルトでは `myproject_2026-10-08_12-34-23_0.mp4` のような名前になります。`-o` には `{index}` や `{変数id}` のプレースホルダーが使えます（例：`-o "renders/{index}-{version}.mp4"`）。
   - 複数の行が同じパスになる場合は、レンダリング前に "Batch output collision" エラーで止まります。
   - スペースやスラッシュを含む変数はファイル名に使わないのが無難です。

3. **変数キーのタイポは、デフォルトでは警告止まり**
   `--variables '{"headlne":"..."}'` と打ち間違えても、レンダリングはそのまま成功します。見出しには **デフォルト値** が使われ、`Variable "headlne" is not declared` という警告が出るだけです。CI では `--strict-variables` を付けて、exit code 1 で失敗させましょう。
   ```text
   ✗  Variable validation failed
      Aborting render due to variable issues (--strict-variables mode).
   ```

4. **タイムラインは「停止状態」で、composition id と完全一致するキーに登録する**
   `gsap.timeline({ paused: true })` で作り、`window.__timelines["main"] = tl` で登録します。キーの `"main"` は `data-composition-id` と同じ値にしてください。paused になっていない、またはキーがずれていると、HyperFrames がアニメーションを seek できません。その結果、プレビューと書き出し結果が一致しなくなります。

5. **バッチ行のデータは「信頼できない文字列」として扱う**
   行データは CHANGELOG や CRM、ユーザー入力から来ることが多いはずです。`innerHTML` ではなく `textContent` を使ってください。そうしないと、行に `<img src=x onerror=...>` のようなマークアップが混ざったとき、レンダラーの Chrome 内でそれが実行されてしまいます。

6. **lint 警告 `nested_structure_needs_subcomposition`**
   時間指定した `<section>` の中に複数の要素を入れても、レンダリング結果は正常です。ただし lint は「シーンごとにサブコンポジション（`data-composition-src="compositions/intro.html"`）に分けるべき」と警告します。
   - 1ファイルのテンプレートなら無視して構いません。
   - 規模が大きくなったら、Studio のタイムラインで要素ごとに1行ずつ表示されるよう、警告に従って分割しましょう。

7. **CDN から読み込むスクリプトは、レンダリング時にネットワークが必要**
   今回のテンプレートは GSAP を jsDelivr から読み込んでいます。ネットワークが制限された CI では、`gsap.min.js` を `index.html` と同じ場所に置き、相対パスで読み込んでください。

8. **フォントは環境によって変わる**
   ローカルでのレンダリングにはシステムフォントが使われます。マシンが変わっても同じピクセルで出力したいなら `render --docker`（Docker が必要）を使ってください。Chrome のバージョンとフォントが固定されます。

9. **テレメトリはデフォルトで有効**
   CLI は匿名の利用統計を送信します。CI では `HYPERFRAMES_NO_TELEMETRY=1`（`DO_NOT_TRACK=1` も有効）を設定しておきましょう。また `init` は GitHub 上の AI スキルを確認しにいきますが、`HYPERFRAMES_SKIP_SKILLS=1` でスキップできます。

10. **GPU がなくても動くが、遅くなる**
    WebGL が使えない環境では SwiftShader とスクリーンショット方式にフォールバックします。今回の環境では、5秒のクリップ1本に約12秒かかりました。
    - バッチの各行はデフォルトで順番に処理されます（`--batch-concurrency 1`）。1本のレンダリングがすでに複数のワーカーで並列化されているためです。
    - 並列数を上げるのは RAM に余裕があるときだけにしてください。Chrome ワーカー1つあたり約256MB使います。

## まとめと次のステップ（Summary & Next Steps）

- **HTML 1枚 + `data-composition-variables` + `render --batch`** で、データから動画を作る再現性のあるパイプラインが組めます。動画編集ソフトも、レンダリングごとの課金も要りません。
- `--strict-variables`、プレースホルダー付きの `-o`、ffprobe による検証の3点を揃えると、CI に組み込んでも安心です。

発展させるなら：

- **CI でリリース動画を作る**：`gh release list --json` から `releases.json` を生成し、タグを打つたびにレンダリングする
- **縦型・SNS向けの書き出し**：`--resolution portrait` や `square` のプリセットで、同じテンプレートを使い回す
- **別フォーマットでの書き出し**：`--format webm|gif|mov|hls`。透過 WebM / MOV はオーバーレイ素材にも使えます
- **エージェント連携**：Claude Code / Cursor 用のスキル（`npx skills add heygen-com/hyperframes`）が同梱されています。テンプレートの編集は AI に任せ、データは JSON で管理するという分担ができます

本記事のコード（`index.html` / `data/*.json` / `verify.mjs`）はすべて上に全文掲載しており、そのままコピーして動かせます。
