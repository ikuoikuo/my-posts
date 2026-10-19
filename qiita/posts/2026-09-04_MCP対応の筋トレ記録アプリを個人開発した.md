---
title: "MCP対応の筋トレ記録アプリを個人開発した"
qiita_id: "aeae10406c4730ab67cf"
url: "https://qiita.com/ikuoikuo/items/aeae10406c4730ab67cf"
created_at: "2026-09-04T17:23:49+09:00"
updated_at: "2026-09-04T17:31:00+09:00"
tags: ["MCP","cloudflare","multica"]
---

<!-- Archived from Qiita: https://qiita.com/ikuoikuo/items/aeae10406c4730ab67cf -->
## TL;DR

- 日々の筋トレを記録できる Web アプリを個人開発した
- 最大の特徴は **MCP 対応**。AI が直接記録を読み書きできるので、AI をパーソナルトレーナーにできる
- 実装は [Multica](https://multica.ai/) でエンジニアチームを組んで開発した

## 作ったもの

https://muscle-training-mcp.tsukutakun.workers.dev/

最近、筋トレの記録を管理するWebアプリを個人開発しました。

一見するとよくある筋トレ記録アプリですが、最大の特徴は MCPに対応していること です。AI がトレーニング履歴を直接読み書きできるため、単なる記録アプリではなく「AIを自分専用の**パーソナルトレーナー**として活用できる環境」を作ることができます。

![スクリーンショット 2026-09-04 16.47.38.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/b03cb75a-dde5-41c1-9aa0-07421dcbec85.png)

例えば AI に対して、
> 前回のベンチプレスの記録をもとに今日のメニューを考えて

と依頼したり、

> 過去3ヶ月のトレーニング頻度をヒートマップで可視化して

といった分析を行わせたりすることができます。

筋トレの記録を保存するだけでなく、記録をもとにAIがコーチングや分析を行えるのが特徴です。

![スクリーンショット 2026-09-04 16.47.45.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/12a8c7c7-47a8-4936-8b1d-24b18116236b.png)

画面上では Claude から操作している例を紹介していますが、Claude に限らず、MCP に対応したクライアントであれば接続して利用できます。

## アーキテクチャ

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/e5bf5fbc-9f75-4074-ae79-2961d51813ce.png)

Web UI と MCP サーバーは同一の **Cloudflare Worker** 上で動作しています。
認証基盤は Google アカウントを中心に構成しており、Google アカウントを持っていれば誰でも利用できます。

また、運用コストですが、
- Cloudflare Workers
- Cloudflare D1

の無料枠を利用することで、現時点では**無料**で運用できています。

| 領域 | 採用 |
|---|---|
| 言語 | TypeScript |
| 実行環境 | Cloudflare Workers |
| ルーティング | Hono |
| DB | Cloudflare D1（SQLite）+ Drizzle ORM |
| UI | React + Vite（SPA） |
| MCP 認証 | OAuth 2.1（Worker が認可サーバー、上流 IdP は Google） |
| UI 認証 | Google ログイン |

## 開発方法
最近流行り（？）の[Multica](https://multica.ai/) を使用して開発を行ってみました。

実装は Multica 上でリーダー / フロントエンド / バックエンド / デザイナーの役割を分けたエンジニアチームを組み、タスクを並行して進める形で作りました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/805c37fa-b51d-49c3-851f-e41fe518cf41.png)

実際の開発では私が Web エンジニアリーダー役のエージェントにタスクを依頼し、そのエージェントが適切な担当エージェントへタスクを振り分ける形で進行します。
以下はリーダーが切ったイシューにデザイナーエージェントが取り組む様子です。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/97efed54-9f1e-460b-b20a-cdb4c0b1c3b7.png)

Cloudflare へのデプロイについても、Cloudflare の MCP をエージェントに接続することでスムーズに行うことができました。

## まとめ

筋トレ記録アプリ自体は珍しくありませんが、MCP を介して AI が直接データを読み書きできるようにしたことで、

- 記録
- 分析
- メニュー作成
- 振り返り

までを AI と対話しながら行えるようになりました。

また、開発手法としてはMultica を使用しました。
Multica を利用し始めてからしばらく経ち、自分なりの活用方法も見えてきたため、最近どのように使っているか、詳細は別途記事にまとめようと思います。

