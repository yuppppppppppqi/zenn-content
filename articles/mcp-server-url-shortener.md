---
title: "個人開発サービスにMCPを実装してリンク管理を自動化した話"
emoji: "🔗"
type: "tech"
topics: ["MCP", "個人開発", "API", "ClaudeCode", "JavaScript"]
published: true
---

## はじめに

URL短縮サービス [Picoli](https://picoli.site/z-picoli-0001a) を個人開発している。APIでリンクの作成・統計取得はできていたが、毎回ブラウザで管理画面を開くか、ターミナルで `curl` を叩く必要があった。

MCPサーバーを実装してClaude Codeに登録したところ、**エディタから一歩も出ずにリンク管理が完結するようになった**。開発中の「ちょっとリンク作りたい」が自然言語で済むのは想像以上に快適だった。

この記事では、個人開発のWebサービスにMCPサーバーを実装する流れを、Picoliの事例をもとに書く。

## MCPとは（30秒で）

MCP（Model Context Protocol）はAnthropicが策定したプロトコルで、AIアシスタントと外部ツールを接続する標準仕様。Claude Codeなどのクライアントから、自作ツールの機能を呼び出せるようになる。

詳しい仕様は公式ドキュメントに譲る。ここでは実装にフォーカスする。

## なぜMCPを実装しようと思ったか

個人開発をしていると、作業中に短縮リンクが必要になる場面が地味に多い。

- READMEにデモURLを貼りたい
- ブログ記事にトラッキング用のリンクを入れたい
- リリース告知用にチャネル別のリンクを分けたい

そのたびに「ブラウザを開く → Picoliにログイン → リンク作成 → コピー → エディタに戻る」をやっていた。1回30秒程度だが、1日に何度もやると積み重なる。

MCPなら「このURL短縮して」で終わる。その差は大きい。

## 実装したツール一覧

Picoliでは5つのMCPツールを実装した。

| ツール名 | 機能 | 主な用途 |
|---------|------|---------|
| `shorten_url` | 1件の短縮リンク作成 | 日常的なリンク作成 |
| `shorten_urls` | 最大500件の一括作成 | キャンペーン・リリース告知 |
| `get_link_stats` | スラッグ指定でクリック数取得 | 効果測定 |
| `list_links` | リンク一覧取得 | 管理・確認 |
| `get_analytics` | 全体の分析データ取得 | ダッシュボード代わり |

## 実装の流れ

### 1. MCP SDKのセットアップ

```javascript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new McpServer({
  name: "picoli",
  version: "1.0.0",
});
```

`@modelcontextprotocol/sdk` を使う。stdioトランスポートが最もシンプルで、Claude Codeとの連携に向いている。

### 2. ツールの定義

各ツールは `server.tool()` で定義する。重要なのは **descriptionを丁寧に書くこと**。Claude Codeはこのdescriptionを読んでツールを選択するため、曖昧だと意図しないツールが呼ばれる。

```javascript
server.tool(
  "shorten_url",
  "Create a short URL using picoli.site. Optionally specify a custom slug. Returns the shortened URL and slug.",
  {
    url: { type: "string", description: "The destination URL to shorten" },
    slug: { type: "string", description: "Optional custom slug (e.g. 'my-link')" },
  },
  async ({ url, slug }) => {
    const result = await picoliApi.shortenUrl(url, slug);
    return {
      content: [{ type: "text", text: JSON.stringify(result, null, 2) }],
    };
  }
);
```

### 3. APIラッパー

MCPツールのハンドラからは、既存のREST APIを叩くだけ。新しいバックエンドは不要。

```javascript
class PicoliApi {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.baseUrl = "https://picoli.site/api";
  }

  async shortenUrl(url, slug) {
    const body = { url };
    if (slug) body.slug = slug;

    const res = await fetch(`${this.baseUrl}/links`, {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${this.apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });
    return res.json();
  }
}
```

MCPサーバーはAPIのクライアントに過ぎない。既にREST APIがあるサービスなら、MCPの実装は本質的に **APIクライアント + ツール定義のdescription** だけで済む。

### 4. Claude Codeへの登録

```bash
claude mcp add-json picoli '{
  "type": "stdio",
  "command": "node",
  "args": ["/path/to/picoli-mcp/index.js"],
  "env": { "PICOLI_API_KEY": "your-api-key" }
}'
```

これで登録完了。Claude Codeを再起動すると使えるようになる。

## 使ってみる

### リンク作成

```
自分: 「https://example.com/my-app を短縮して、スラッグは demo-app で」
Claude: 作成しました → https://picoli.site/demo-app
```

### 一括作成

```
自分: 「以下の3つのURLを短縮して。スラッグはそれぞれ app-twitter, app-qiita, app-zenn で」
     - https://example.com/my-app (Twitter用)
     - https://example.com/my-app (Qiita用)
     - https://example.com/my-app (Zenn用)

Claude: 3件作成しました
  - https://picoli.site/app-twitter
  - https://picoli.site/app-qiita
  - https://picoli.site/app-zenn
```

### クリック数の確認

```
自分: 「app-twitter と app-qiita のクリック数を教えて」
Claude:
  - app-twitter: 142 clicks
  - app-qiita: 89 clicks
```

チャネル別の流入を確認するのに、ブラウザもターミナルも不要。

## 実装して気づいたこと

### descriptionが全て

MCPツールの使い勝手は、コードの品質よりも **descriptionの質** で決まる。

最初は `"Shorten a URL"` とだけ書いていた。するとClaude Codeが「URLを短くしたい」と言われた時にこのツールを呼ばないことがあった。

`"Create a short URL using picoli.site. Optionally specify a custom slug. Returns the shortened URL and slug."` に変えたら安定した。**何ができて、何が返るか** を明記するのが重要。

### エラーハンドリングは手厚く

MCPツールがエラーを返すと、Claude Codeが「うまくいきませんでした」としか言えなくなる。エラー時にも **何が原因か** をテキストで返すようにすると、AIが自力でリトライや代替案を提示できる。

```javascript
async ({ url, slug }) => {
  try {
    const result = await picoliApi.shortenUrl(url, slug);
    return { content: [{ type: "text", text: JSON.stringify(result, null, 2) }] };
  } catch (e) {
    return {
      content: [{ type: "text", text: `Error: ${e.message}. Check if the API key is valid and the URL format is correct.` }],
      isError: true,
    };
  }
}
```

### REST APIがあれば実装は軽い

Picoliの場合、MCPサーバーの実装は **1日で終わった**。既にREST APIが整備されていたから。

逆に言えば、MCPを実装する前にまずREST APIをちゃんと作るのが先。APIの設計が良ければ、MCPは薄いラッパーを書くだけで済む。

## 個人開発者にMCPを勧める理由

自分のサービスにMCPを実装する最大のメリットは **自分が一番のヘビーユーザーになれる** こと。

ブラウザの管理画面を自分で使っていると、UIの不備に気づいて直したくなる。するとフロントエンドの改修に時間を吸われる。MCPならUIを経由しないので、**バックエンドのAPI品質だけに集中できる**。

個人開発は時間が有限。MCPで管理画面の開発をスキップして、本質的な機能に集中するのは合理的な選択だと思う。

## まとめ

- 既にREST APIがあるサービスなら、MCPサーバーの実装は1日で終わる
- descriptionを丁寧に書くのが一番大事
- 個人開発者こそMCPを実装すべき。自分が一番のユーザーになれる

Picoliのサイトはこちら。MCP対応のURL短縮サービスとして使える。

https://picoli.site/z-picoli-0001b
