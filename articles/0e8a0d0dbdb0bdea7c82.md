---
title: "Angular SSR on Cloudflare Pages + Hono で tRPC を体感しよう！"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Angular", "Cloudflare", "Hono"]
published: false
---

tRPC を体感してみたかったので、今回は Hono の RPC モードを使ってみました。

今回のサンプリコードはこちらです。

https://github.com/fusho-takahashi/ng-cfp-rpc

## C3(create-cloudflare-cli) で Angular SSR アプリを作る

Cloudflare 公式の Framework guides は情報が古くなっているようなので注意してください。

```bash
npm create cloudflare@latest
```

上記コマンドを実行すると質問されますので、下記のように答えましょう。

```bash
╭ Create an application with Cloudflare Step 1 of 3
│
├ In which directory do you want to create your application?
│ dir ./ng-cfp-rpc <- ここは任意のアプリケーション名に置き換えてください
│
├ What type of application do you want to create?
│ type Website or web app
│
├ Which development framework do you want to use?
│ framework Angular
│
╰ Continue with Angular via `npx @angular/cli@16.2.2 new ng-cfp-rpc --standalone`
```

Angular 最新でしかも standalone フラグを書いてくれているところに好感を持てますね！
このあと Step 2 で SSR や Cloudflare Pages の設定もしてくれます。
最後に Step 3 では Cloudflare Pages に deploy するか聞かれるのですが、今回は tRPC を体験したいだけなので `No` にしました。

アプリケーションの作成が終わったら `npm start` してみましょう。いつもと変わらない、見慣れた景色が広がっています。

![Angular 初期画面](/images/app_is_running.png)

## Hono で API を書く

まず Hono を npm install します。

```bash
npm i hono
```

それからアプリケーションの root に `functions` ディレクトリを作成し下記内容の `index.ts` を置きます。

```ts:index.ts
import { Hono } from 'hono';

export const apiApp = new Hono();

const apiRoute = apiApp.get('/title', (c) => {
  return c.jsonT({ title: 'Hono RPC mode' });
});

export type ApiRoute = typeof apiRoute;

```

Pages Functions の実装は、 `c.jsonT()` で値を返し、`apiRoute` の型を `export` します。この `apiRoute` の型を Angular アプリケーション側で `import` することで、サーバー・クライアント間で同じ型を使って開発ができるというわけです。

## Angular から Pages Functions にアクセスできるようにする

現状すべてのパスを Angular でハンドルしてしまうので、せっかく作った API にアクセスすることができません。 `main.server.ts` を変更して、`/api` から始まる URL は Hono でハンドルできるようにします。

```diff ts:main.server.ts

(globalThis as any).__workerFetchHandler = async function fetch(
	request: Request,
	env: Env
) {
	const url = new URL(request.url);
	console.log("render SSR", url.href);

+  if (url.pathname.startsWith("/api")) {
+    const app = new Hono({ strict: false }).route("/api", apiApp);
+    return await app.fetch(request, env);
+  }

	// Get the root `index.html` content.
	const indexUrl = new URL("/", url);
	const indexResponse = await env.ASSETS.fetch(new Request(indexUrl));
	const document = await indexResponse.text();
```

## Angular で API をコールする

ついに Angular から API をコールする準備ができました！早速試してみましょう！
今回いつも使ってる HttpClient は使いません。代わりに Hono の client を作成します。

```ts:app.component.ts
@Component({
  ...
})
export class AppComponent implements OnInit {
  title = 'ng-cfp-rpc';
  client = hc<ApiRoute>('http://127.0.0.1:8788/api');
}
```

作成した client を使って `ngOnInit` の中で API をコールしてみましょう。

```ts:app.component.ts
async ngOnInit() {
  const res = await this.client.title.$get();
  const data = await res.json();
  this.title = data.title;
}
```

皆様、お気づきになったでしょうか？

![パスがコード補完される様子](/images/IntelliSense_path.png)

![プロパティがコード補完される様子](/images/IntelliSense_property.png)

URL のパスや、レスポンスのプロパティのコード補完が聞いています！感動！

そして `npm start` してみると、きちんと表示が変わっています。

![API から取得した文字列を表示](/images/rpc_is_running.png)

## おわりに

無事 tRPC を体感することができましたね！今回はやっていないですが、post の場合は body のコード補完もきちんと
効きますし、開発体験バク上がり間違いなしです！
さらに Cloudflare D1 と繋げばフロントエンドからバックエンドから DB まで、まさにオールインワンな構成を実現できます。そちらの記事も近いうちに書きたいと思っておりますので、お楽しみに！

みなさんもぜひ tRPC を体験してみてください！
