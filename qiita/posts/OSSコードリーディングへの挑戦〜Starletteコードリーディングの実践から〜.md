---
title: "OSSコードリーディングへの挑戦 〜Starletteコードリーディングの実践から〜"
qiita_id: "427f3e7567821da1d9b2"
url: "https://qiita.com/ikuoikuo/items/427f3e7567821da1d9b2"
created_at: "2025-12-16T16:26:56+09:00"
updated_at: "2025-12-16T16:58:51+09:00"
tags: ["Python","OSS","Starlette"]
---

<!-- Archived from Qiita: https://qiita.com/ikuoikuo/items/427f3e7567821da1d9b2 -->


## はじめに
OSS（オープンソースソフトウェア）のコードリーディングは重要だとわかっていても、  
「敷居が高い」「どこから読めばいいかわからない」「挫折した」  
そんな声をよく聞きます。  

実際、私自身も同じように感じていました。  しかし、今回 **Starlette** を題材に取り組んだことで、「OSSコードリーディングに挑むためのステップ」が少し見えてきたので共有します。


## 🚀TL; DR
OSSコードリーディングは「全部読む」のではなく、**目標を持って、最小構成を自作する**のが成功の近道
- 動く最小単位から積み上げるchibivue方式で進める。
- AIをフル活用する（構造の可視化、壁打ち）
- 仲間との週次報告でモチベーションを維持する

## 🏔️挑むステップ

### 1. 目標を決める
まずは「何をすれば達成なのか」を明確にします。  
ポイントは **定量的に達成がわかる目標を設定する** こと。  

今回の活動では、  
> Starlette のルーティング機能について、
公式ドキュメントの[最初のページ](https://starlette.dev/applications/)で紹介されている内容が
自分の実装として再現できていること

という目標にしました。

#### 作って学ぶスタイルの採用

このゴールを達成するために、今回は ChibiStarlette という簡易実装を自作し、作りながら理解するスタイルを採用しました。

この進め方は、[chibivue](https://book.chibivue.land/00-introduction/010-about.html) の思想を参考にしています。
chibivue とは Vue.js の仕組みを、自分の手で実装しながら理解するための極小構成の Vue.js 実装です。

https://book.chibivue.land/00-introduction/010-about.html

同様に ChibiStarlette でも、
- なぜこのクラスが必要なのか
- なぜこの処理順になるのか

を考えながら実装を進めることで、理解が深まりました。

#### 実装をゴールにする理由

実装をゴールにすると、目標が非常に明確になります。

- 曖昧な理解のままではコードが動かない
- 理解不足はそのままバグとして現れる

このため、「分かったつもり」を排除しながら進めることができ、結果として理解への最短距離になると感じました。

具体的には、次のコードが動作する状態を実装ゴールとしました。
Starlette ライブラリには一切依存せず、ルーティング機能を自前実装することで、Starlette のルーティングが内部でどのように動いているのかを再現しています。

```python
from chibistarlette.applications import ChibiStarlette
from chibistarlette.responses import PlainTextResponse, JSONResponse, HTMLResponse
from chibistarlette.requests import Request
from chibistarlette.routing import Route


async def homepage(request: Request):
    return PlainTextResponse('Hello, world!')

async def user_me(request: Request):
    username = "John Doe"
    return JSONResponse({'message': 'Hello, %s!' % username})
    
async def user(request: Request):
    username = request.path_params['username']
    return HTMLResponse('<html><body><h1>Hello, %s!</h1></body></html>' % username)

routes = [
    Route('/', homepage),
    Route('/user/me', user_me),
    Route('/user/{username}', user),
]

app = ChibiStarlette(routes=routes)
```

---

### 2. 全体構造の把握
目標が定まったら、まず全体の構造を把握することから着手しました。

Starlette では、複数のミドルウェアがエンドポイントを順にラップすることで、リクエスト処理の流れが構成されています。
この仕組みを最初に理解することで、「今どこを実装しているのか」「どこから手を付けるべきか」を判断しやすくなります。

なお、各ミドルウェアの役割や処理順の詳細については、本記事では踏み込まず、別の記事で解説する予定です。

ここで伝えたいのは、「いきなり細部に入るのではなく、**まず全体像を掴むこと**」の重要性です。
全体構造が見えていれば、現在取り組んでいる実装が「処理全体のどの責務を担っているのか」を常に意識しながら進めることができます。

最近ではコーディングエージェントを使用することによってコードをもとに処理を図式化してくれるので便利ですね。
※ 以下のフローは、コーディングエージェントによって図式化されたものです。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/fb96a61f-8b11-4e1d-9c9d-2ba943360628.png)





---

### 3. 依存関係が少ないところから作る
実装を進める際は、依存関係の少ない部分から手を付けるのがコツです。
今回の場合は、まず requests.py の実装から着手しました。

前節で全体像として説明した通り、Starlette では複数のミドルウェアが ASGI アプリケーションをラップし、最終的にエンドポイントが呼ばれます。
そのためまずは最終層のエンドポイントを作成することから手をつけ、エンドポイント内部でも処理の流れは次のようになっています。

>1. ASGI が渡す生データ（scope や receive）を Request クラスでラップし、Python 側で扱いやすい形に変換
>2. 定義した関数（エンドポイント）の処理を実行
>3. 結果を Response クラスで ASGI 形式に変換して返却

つまり、まず Request クラスの処理を理解し、実装することが依存関係の少ないスタート地点 となります。

その後は、Response クラスの実装、エンドポイント関数の実装と順に範囲を広げ、最小限の機能を真似ながらゼロから作るイメージで進めました。

---

### 4. 不必要な機能は削る
実装を理解する際、理解を妨げる要素は一旦省き、最小構成で進めるのが有効です。
たとえば validation のように、今回の目的に対して優先度の低い処理は省略することで、コードの流れを追いやすくなります。

以下の diff のように、Starlette クラスの定義においてもメンバ変数やバリデーション処理など、多くの機能を削っています。

```diff
-class Starlette:
-    def __init__(self, debug: bool = False, routes: Sequence[BaseRoute] | None = None):
-        self.debug = debug
-        self.router = Router(routes)
-        self.middleware_stack: ASGIApp | None = None
-        self.lifespan = Lifespan(self)
-        self.exception_handlers: dict[...] = {}
-        self.background_tasks: list[...] = []
-        # その他初期化処理
-
-    def build_middleware_stack(self) -> ASGIApp:
-        app = self.router
-        # ミドルウェアを順にラップ
-        app = ExceptionMiddleware(app, handlers=self.exception_handlers, debug=self.debug)
-        app = MiddlewareStack(app, self.middleware)
-        app = LifespanMiddleware(app, self.lifespan)
-        return app
-
-    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
-        scope["app"] = self
-        if self.middleware_stack is None:
-            self.middleware_stack = self.build_middleware_stack()
-        await self.middleware_stack(scope, receive, send)

+class ChibiStarlette:
+    def __init__(self, routes: Sequence[BaseRoute] | None = None) -> None:
+        self.router = Router(routes)
+        self.middleware_stack: ASGIApp | None = None
+
+    def build_middleware_stack(self) -> ASGIApp:
+        # ミドルウェアは省略し、Router だけ返す
+        return self.router
+
+    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
+        scope["app"] = self
+        if self.middleware_stack is None:
+            self.middleware_stack = self.build_middleware_stack()
+        await self.middleware_stack(scope, receive, send)
```

ただし、ファイル構成はそのまま残すことで、後から実際のコードと比較しやすくしています。
このように最小構成で進めることで、コード全体の流れや構造を把握しやすくなり、理解が格段に進みます。

どこまで最小構成にするかは悩ましいですが、**最初から完璧を目指す必要はない**と思います。まずは不要な部分を削って試しに実装し、全体を見ながら必要に応じて調整する、という形で進めると、自分にとって理解しやすい形で学習を進められます。

この「削る」作業の中で、クラスや関数、変数の役割の理解が深まりました。

---

### 5. 実際に動かす
作成したコードは逐一動かして挙動を確認することが重要です。
たとえば「この関数が呼ばれたら、どのようなレスポンスが返るのか」を実際に試すことで、理解が格段に深まりました。

- 関数やクラスを呼び出して出力を確認
- Request / Response の挙動を実際に触って検証

単体テストを利用するのも有効ですが、私は 新たにテスト用スクリプトを作成して動かす ことで理解を深めていました。たとえば以下は Request クラスの処理を検証する例です。

- ASGI アプリとして Request を呼び出し
- 実際にリクエスト情報がどのように処理されるかを確認

```python
from chibistarlette.requests import Request
import json

async def app(scope, receive, send):
    assert scope["type"] == "http"
    request = Request(scope, receive, send)

    headers_dict = {
        k.decode("latin-1"): v.decode("latin-1")
        for k, v in request.headers._list
    }
    
    # QueryParamsを辞書に変換
    query_dict = {
        k: v[0] if len(v) == 1 else v
        for k, v in request.query_params._params.items()
    } if request.query_params._params else {}

    info = {
        "method": request.method,
        "url": str(request.url),
        "headers": headers_dict,
        "query_params": query_dict,
        "path_params": request.path_params,
        "body": (await request.body()).decode(),
        "scope_type": request.scope["type"],
    }
    body_bytes = json.dumps(info, ensure_ascii=False, indent=2).encode("utf-8")
    
    await send({
        "type": "http.response.start",
        "status": 200,
        "headers": [(b"content-type", b"application/json; charset=utf-8")],
    })
    await send({
        "type": "http.response.body",
        "body": body_bytes,
    })
```

動作確認を通して、進捗を一歩ずつ積み上げていく感じで進めることができました。
どうしても全部を実装してから動作確認だと、途中で挫折してしまうことも多いのではないかと思います。


---

### 6. 抽象度を上げて考える
Starlette の compile_path 関数を読み解くと、URL パラメータの解析に「正規表現」が使われていることがわかります。この仕組みは他のフレームワークでも共通なのか気になり、Flask と比較してみました。

まず、Flask (Werkzeug) の正規表現を見ると、非常に多機能な設計であることがわかります。

```Plaintext
<
  (?:
    (?P<converter>[a-zA-Z_][a-zA-Z0-9_]*)   # コンバーター名（int, stringなど）
    (?:\((?P<arguments>.*?)\))?             # 引数（min, maxなど）
    :                                       # 区切り文字
  )?
  (?P<variable>[a-zA-Z_][a-zA-Z0-9_]*)      # 変数名
>
```

一方で、FastAPI (Starlette) の正規表現は以下の通りです。

```Plaintext
{([a-zA-Z_][a-zA-Z0-9_]*)(:[a-zA-Z_][a-zA-Z0-9_]*)?}
```

この実装の違いを比較することで、Python の歴史やそれぞれの違いが見えてきました。
1. Flask: 当時は Python 自体に標準的な型検査の仕組みがなかったため、フレームワーク側で独自のルール（<int:id> など）を作り、URL の定義の中で「型変換」や「バリデーション」まで完結させている
2. FastAPI: Python 3.5 以降で導入された型ヒントを前提に設計されており、独自の構文を覚える必要がない

FastAPIのドキュメントにも以下のような記述があります。

>"It's all based on standard Python type declarations (thanks to Pydantic). No new syntax to learn. Just standard modern Python.)
（すべては標準的なPythonの型宣言に基づいています。新しく覚えるべき構文はありません。ただの標準的なモダンPythonです。）

またAIとも壁打ちをすることで以下のように設計思想をまとめていただきました。
>Starletteがパス解析をシンプルに保ち、実際の型変換をPydanticやPython標準機能に委ねているのは、「**車輪の再発明をせず、言語のポテンシャルを最大限に活かす**」というモダンな設計思想の表れだと言えます。

この段階では、単なる実装の理解から設計の理解へとステップアップすることが目的です。
コードの細かい挙動だけでなく、設計思想や背景を意識することで、より深い理解につながります。

##  🏆成功要因

上記のステップを実践することで、ハードルの高かったコードリーディングを着実に進めることができました。
また、ステップ以外にも今回の成果につながった重要な要因が3点あります。

### 仲間との進捗共有（モチベーション維持） 
週に1回、同期と1時間ほどの進捗共有会を実施しました。「わからない部分を言語化して共有できる相手」がいることで、挫折せずに継続することができました。また仲間と進捗共有を定期的に行うことで目標・目的の再設計を柔軟に行うこともできました。

### AIのフル活用 
特に Roo CodeのAskモードが強力な武器になりました。

- **処理の図式化**: 「リクエストからレスポンスまでの流れはどうなっているか？」といった具体的な問いを投げ、全体像を把握。
- **コード解説と壁打ち**: 難解な箇所の解説や、抽象度を高めて概念的に理解するための相談相手として活用


##  🌀失敗要因
はじめはなかなか地に足がついている感じがせず、モチベーションも低下などもしました。
以下がその要因でした。

- **目標が曖昧なまま着手した**
「何をするか」が曖昧なまま、とりあえず読むことから始めると全体が見えず、モチベーションの低下につながりました。

- **最初から全コードをコピーして改良しようとした**
当初は、Starletteのコードをすべて手元にコピーし、そこから不要な部分を削っていくという方針で進めました。しかし、これが大きな苦戦の要因となりました。依存関係の迷路に迷いエラーの特定も困難でした。

この経験から、まずは 「**最小限の機能単位で再現・検証する**」 ことが、初学者にはよいのかと思いました。

##  ✍️まとめ
PRやIssueを読みながら進めるといった、まだまだ「最適な学び方」を模索している最中ではありますが、今回のステップを通して、OSSのコードリーディングの一歩目を踏み出せた実感があります。

次は **Uvicorn** あたりに挑戦したいと考えています。  
一緒にOSSの中身を読み解いていく仲間を募集中です！

