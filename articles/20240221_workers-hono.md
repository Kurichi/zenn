---
title: "Cloudflare Pages で Stripe を動かす"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['cloudflare', 'stripe', 'hono']
published: true
---

先日、ハッカソンで Hono × Cloudflare Pages × Stripe という技術構成で決済を含むサービスを作ったのですが、その際に Node.js ランタイム依存の API 周りで詰まったのでメモを残しておきます。

# 要約
Cloudflare Pages で Hono テンプレートを使って Stripe を動かしたかったが、Vite の Rollup によって Node.js ランタイムへの依存が発生し、以下のエラーが発生。
```
  _worker.js:1:17: ERROR: Could not resolve "crypto"
  _worker.js:1:17: ERROR: Could not resolve "crypto"
  _worker.js:1:56: ERROR: Could not resolve "events"
  _worker.js:1:82: ERROR: Could not resolve "http"
  _worker.js:1:106: ERROR: Could not resolve "https"
  _worker.js:1:128: ERROR: Could not resolve "util"
```
~~`vite.config.js` で `build.rollupOptions.external` に `stripe` を設定することで回避した。~~
**2024/02/29 追記：`vite.config.js` で `ssr.target` を `webworker` に設定することで回避した．**

# やったこと
まずは、エラーが発生するまでの流れを書いておきます。

### プロジェクトの作成
まず、hono の Pages テンプレートを用いてプロジェクトを作成します。
```sh
$ pnpm create hono@latest pages-hono-stripe
✔ Using target directory … pages-hono-stripe
✔ Which template do you want to use? › cloudflare-pages
cloned honojs/starter#main to /Users/kurichi/repos/pages-hono-stripe
✔ Copied project files
```

### Stripe のセットアップ
次に、Stripe のライブラリをインストールします。
```sh
$ pnpm add stripe
```
そして，`.dev.vars` に Stripe のシークレットキーを設定します。
```:.dev.vars
STRIPE_SECRET_KEY=sk_test_aaaaaaaaaaaaaaaaaaaaaaaa
```

### Stripe のライブラリを使う
`src/index.tsx` に以下のようなコードを書いて、Stripe のインスタンスを生成します。
```tsx:src/index.tsx
import { Hono } from "hono";
import { renderer } from "./renderer";
import Stripe from "stripe";

type Bindings = {
  STRIPE_SECRET_KEY: string;
}

const app = new Hono<{Bindings: Bindings}>();

app.get("*", renderer);

app.get("/", (c) => {
	const stripe = new Stripe(c.env.STRIPE_SECRET_KEY);
	return c.render(<h1>Hello!</h1>);
});

export default app;
```

### ビルド & プレビュー
最後に、ビルドしてプレビューを確認します。
```sh
$ pnpm build
$ pnpm preview
```

### エラー
すると、プレビュー時に次のようなエラーが発生し、サーバーが起動しません。
```
  _worker.js:1:17: ERROR: Could not resolve "crypto"
  _worker.js:1:17: ERROR: Could not resolve "crypto"
  _worker.js:1:56: ERROR: Could not resolve "events"
  _worker.js:1:82: ERROR: Could not resolve "http"
  _worker.js:1:106: ERROR: Could not resolve "https"
  _worker.js:1:128: ERROR: Could not resolve "util"
```
:::details 出力全文　
```sh
$ pnpm build

> @ build /Users/kurichi/repos/pages-stripe
> vite build

vite v5.1.3 building SSR bundle for production...
✓ 227 modules transformed.
dist/_worker.js  133.79 kB
✓ built in 378ms
$ pnpm preview

> @ preview /Users/kurichi/repos/pages-hono-stripe
> wrangler pages dev dist

▲ [WARNING] No compatibility_date was specified. Using today's date: 2024-02-21.

  Pass it in your terminal:
  --compatibility-date=2024-02-21

  See https://developers.cloudflare.com/workers/platform/compatibility-dates/ for more information.


✘ [ERROR] 6 error(s) and 0 warning(s) when compiling Worker.


▲ [WARNING] Failed to bundle _worker.js. Error: Build failed with 6 errors:

  _worker.js:1:17: ERROR: Could not resolve "crypto"
  _worker.js:1:56: ERROR: Could not resolve "events"
  _worker.js:1:82: ERROR: Could not resolve "http"
  _worker.js:1:106: ERROR: Could not resolve "https"
  _worker.js:1:128: ERROR: Could not resolve "util"
  ...
      at failureErrorWithLog
  (/Users/kurichi/repos/pages-hono-stripe/node_modules/.pnpm/esbuild@0.17.19/node_modules/esbuild/lib/main.js:1636:15)
      at
  /Users/kurichi/repos/pages-hono-stripe/node_modules/.pnpm/esbuild@0.17.19/node_modules/esbuild/lib/main.js:1048:25
      at
  /Users/kurichi/repos/pages-hono-stripe/node_modules/.pnpm/esbuild@0.17.19/node_modules/esbuild/lib/main.js:1512:9
      at process.processTicksAndRejections (node:internal/process/task_queues:95:5) {
    errors: [
      {
        detail: undefined,
        id: '',
        location: [Object],
        notes: [Array],
        pluginName: '',
        text: 'Could not resolve "crypto"'
      },
      {
        detail: undefined,
        id: '',
        location: [Object],
        notes: [Array],
        pluginName: '',
        text: 'Could not resolve "events"'
      },
      {
        detail: undefined,
        id: '',
        location: [Object],
        notes: [Array],
        pluginName: '',
        text: 'Could not resolve "http"'
      },
      {
        detail: undefined,
        id: '',
        location: [Object],
        notes: [Array],
        pluginName: '',
        text: 'Could not resolve "https"'
      },
      {
        detail: undefined,
        id: '',
        location: [Object],
        notes: [Array],
        pluginName: '',
        text: 'Could not resolve "util"'
      },
      {
        detail: undefined,
        id: '',
        location: [Object],
        notes: [Array],
        pluginName: '',
        text: 'Could not resolve "child_process"'
      }
    ],
    warnings: []
  }
```
:::

# 試したこと
色々調べてみたところ、こちらの公式ドキュメントを見つけました。
https://developers.cloudflare.com/workers/runtime-apis/nodejs/

この記事に従い、以下のコマンドを実行してみました。
```sh
$ pnpm wrangler pages dev dist --compatibility-flags=nodejs_compat
```
しかし、同じエラーが発生しました。

# 解決策
~~最終的に、`vite.config.js` に `build.rollupOptions.external` を設定することで解決しました。~~
**2024/02/29 追記：`vite.config.js` で `ssr.target` を `webworker` に設定することで解決した．**
```diff js:vite.config.js
import build from "@hono/vite-cloudflare-pages";
import devServer from "@hono/vite-dev-server";
import { defineConfig } from "vite";

export default defineConfig({
	plugins: [
		build(),
		devServer({
			entry: "src/index.tsx",
		}),
	],
-	build: {
-		rollupOptions: {
-			external: ["stripe"],
-		},
-	},
+	ssr: {
+		target: "webworker",
+	},
});
```

# さいごに
今回の解決策でとりあえず動かすことができましたが、もっと良い方法があるかもしれません。
Rollup Plugin の設定を変更することで解決できるかなとは思っているのですが、Vite や Rollup についての知識がまだ浅いので、もっと良い方法があれば教えていただきたいです。
