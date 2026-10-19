---
title: "ContextForge使ってみた"
qiita_id: "5fdb79e0c4347e513624"
url: "https://qiita.com/ikuoikuo/items/5fdb79e0c4347e513624"
created_at: "2026-06-12T14:58:58+09:00"
updated_at: "2026-06-12T14:58:58+09:00"
tags: ["MCP","ContextForge"]
---

<!-- Archived from Qiita: https://qiita.com/ikuoikuo/items/5fdb79e0c4347e513624 -->
## ContextForge とは
エージェント開発が進むと、こういう問題が出てきます。

- MCP サーバーが増えすぎてどこに何があるかわからない
- LangGraph、CrewAI、AutoGen など異なるフレームワークが別々にツールを呼んでいる
- 「誰がどのツールを呼んだか」の管理やセキュリティが分散する

ContextForge はこれらを 1つのゲートウェイに集約することで解決します。

ContextForge はIBM発のOSSで、バラバラなツール・エージェント・APIを1つの入口に集約し、「ガバナンス」「ディスカバリ」「オブザーバビリティ」を提供します。

> （公式説明）
An open source registry and proxy that federates MCP, A2A, and REST/gRPC APIs with centralized **governance**, **discovery**, and **observability**. Optimizes Agent & Tool calling, and supports plugins.

| カテゴリー        | 説明             |
| ---------------- | ------------------------ |
| ガバナンス        | 認証・認可（誰が使えるか）<br>レート制限やポリシーの適用|
| ディスカバリ       | 使えるツール・エージェント等を検索・一覧 |
| オブザーバビリティ | トレースやメトリクス、利用状況の可視化 |


https://ibm.github.io/mcp-context-forge/latest/

## 概要
簡単な例として、ContextForge に自作MCPサーバーを2つ登録し、Claude Desktop から呼び出す例を紹介します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/b9d11b93-b710-4951-8747-4ffa13b21c9c.png)

挨拶ツールと足し算ツールを作り、ContextForgeに登録してみました。
AdminUIから登録済みのツールを一覧で見ることができます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/24d5d520-1075-4027-8dd6-b8b685f174a9.png)

登録したツールをClaude Desktopから呼び出します。
ContextForgeを通すことで認証認可やレート制限などを適用できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/115a3e35-c92f-481e-b84b-d359fcefb6f6.png)


## 手順

### 1. ContextForge の立ち上げ
基本的にはREADME に従って立ち上げを行います。

```bash
git clone https://github.com/IBM/mcp-context-forge.git
cd mcp-context-forge
```

Admin UIからツールの登録等を行う場合は、.env のCSRF_ENABLEDパラメータをfalseにしてから、以下コマンドの実行を行います。(あくまでローカル検証用の設定です。)

```bash
# .env のシークレット(標準ライブラリで生成して設定)
python3 -c "import secrets;print(secrets.token_urlsafe(48))"
#   → .env の JWT_SECRET_KEY= と AUTH_ENCRYPTION_SECRET= に貼る

# 依存関係のインストール
make install-dev

# DB 初期化(alembic は mcpgateway/ 配下から実行すること)
cd mcpgateway && uv run python -m alembic upgrade head 

# 起動
cd ../ && make serve
```

### 2. ローカルMCPサーバーの構築
FastMCP を使用して2つのツールを実装します。

```server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo-tools")


@mcp.tool()
def add(a: float, b: float) -> float:
    """2つの数を足し算して返します。"""
    return a + b


@mcp.tool()
def greet(name: str) -> str:
    """名前を受け取り、日本語の挨拶を返します。"""
    return f"こんにちは、{name}さん! ContextForge 経由の自作 MCP ツールです。"


if __name__ == "__main__":
    mcp.run()

```

### 3. MCPサーバーをHTTPで公開する
mcpgateway.translate を使って、STDIOベースの MCP サーバーを Streamable HTTP エンドポイントとして起動します。この後、Admin UI からこのエンドポイントを ContextForge に登録します。

```bash
uv run python -m mcpgateway.translate --stdio "uv run python /path/to/server.py" \
  --expose-streamable-http --port 9100
```

### 4. ContextForge への登録（Admin UI）

#### 4-1. MCPサーバーの追加
Admin UI から、先ほど起動した http://localhost:9100 を MCP サーバーとして登録します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/d87d79d3-9e9c-49ac-b922-8314f270767a.png)

登録が完了すると、サーバーに含まれるツール（greet・add）が自動的に検出・追加されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/f40ac17b-781f-4907-89e1-56ce1d5a4f48.png)

#### 4-2. Virtual Serverの追加
次に、登録したツールをまとめて公開するための Virtual Server を作成します。Virtual Server は、複数のツールやサーバーを1つのエンドポイントとして束ねる仕組みです。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/70d244a4-343a-481c-b719-291fc3d954c2.png)


### 5. Claude Desktop 連携
設定ファイル の `mcpServers` に以下のように追加します。
serverId やトークンはAdmin UIから取得した値に書き換える必要があります。

```json
"contextforge-demo": {
  "command": "/opt/homebrew/bin/npx",
  "args": [
    "-y", "mcp-remote",
    "http://localhost:4444/servers/<serverId>/sse",
    "--header", "Authorization: Bearer <トークン>",
    "--transport", "sse-only"
  ]
}
```

手順は以上です。
あとはClaude Desktop を再起動して、命令を出すと適切なツールを使用して回答を返してくれます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/72ff4fbb-80f0-4663-bd90-1a04038f9196.png)


## まとめ
今回はツール登録とClaude Desktop連携という最小構成を試しました。ContextForgeにはこの他にも、A2A対応エージェントの登録、プラグインによるポリシー適用、OpenTelemetryによるトレーシング等さまざまな機能があります。
野良MCPが増えていく中で、ゲートウェイ層で「ガバナンス」「ディスカバリ」「オブザーバビリティ」をまとめて確保できるContextForgeは、エンタープライズ導入の有力な選択肢だと感じました。

