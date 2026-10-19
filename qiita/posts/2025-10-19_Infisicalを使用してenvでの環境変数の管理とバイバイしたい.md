---
title: "infisical を使用してenvでの環境変数の管理とバイバイしたい"
qiita_id: "d7024d53b95b9ef287bf"
url: "https://qiita.com/ikuoikuo/items/d7024d53b95b9ef287bf"
created_at: "2025-10-19T19:22:44+09:00"
updated_at: "2025-10-19T19:22:44+09:00"
tags: ["Infisical"]
---

<!-- Archived from Qiita: https://qiita.com/ikuoikuo/items/d7024d53b95b9ef287bf -->
## この記事でやること
Infisicalの設定からFlaskでの実装までを順を追って解説します。

## 背景
- infisicalは2025年10月時点で日本語での解説ドキュメントが少ない
- .env ファイルによる環境変数管理では、チーム間での運用に課題があった

## infisicalとは
[Infisical](https://infisical.com)は、APIキーやデータベース接続情報などの機密情報（Secrets）を安全に管理・共有するためのプラットフォームです。
Web UIやCLI、SDKを通じて環境変数を簡単に取得でき、チームや環境ごとの権限管理も行えます。

## infisical を使用する手順
flask でinfisicalに登録した環境変数を読み込めるまでのステップを示します。

### 0. 前提条件
- Flaskが動作するPython環境が整っていること
- Infisicalのアカウント登録が完了していること

### 1.プロジェクトの作成
1. Infisicalのダッシュボードから Overview → Add new Project をクリックします
2. 「Project Type」は Secrets Management に設定します

作成が完了すると、Overviewページに新しいプロジェクトが表示されます。
次に、Project Settings に移動し、Project ID をコピーしておきます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/c63aa9ab-fdcd-46e7-8ca8-2be8efe66255.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/f7e8f6d8-c278-4b6d-ae45-eec246a6c7bd.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/5e638a0d-748a-4da3-9337-b0890433542d.png)

### 2. secretの追加
1. プロジェクトページで Add Secret をクリックします
2. 以下のように入力します：
    - Key: NAME
    - Value: 任意の値（例: "Infisical User"）
    - Environment: Development

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/b9a54fc3-7af0-4770-a249-2c4926c30fd6.png)


![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/e5aae122-7ce4-4456-b3ea-4cd8179fb9f7.png)

### 3. Access コントロール設定
次に、アプリケーションがInfisicalにアクセスできるように認証情報を設定します。
（ここは公式ドキュメントでも分かりづらい部分なので、詳しく説明します。）

1. Organization Access → Identities に移動します
2. 新しいIdentityを作成します
3. 作成したIdentityの詳細ページで Authentication タブを開きます
4. 作成した Client ID と Client Secret を控えておきます

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/04a8e06d-7812-4772-bfa9-2bd0e2d719c5.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/a9cc51d1-a147-42a3-b88e-2b2684cf1478.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/66120ef2-5969-4656-924a-7179a7cd060b.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/064b2a6f-1c69-4683-8436-d6f85d560d00.png)

### 4. FlaskでInfisicalを利用する
Infisical SDKをPythonにインストールします。
```
pip install infisicalsdk
```

ローカルサーバーを起動してアクセスすると、Infisicalに登録した値が表示されます。

```python
from flask import Flask
from infisical_sdk import InfisicalSDKClient

app = Flask(__name__)

client = InfisicalSDKClient(host="https://app.infisical.com") 
client.auth.universal_auth.login(
    "your_client_id",
    "your_client_secrets"
)

@app.route("/")
def hello_world():
    name = client.secrets.get_secret_by_name(
        secret_name="NAME",
        project_id="your_project_id",
        environment_slug="dev",
        secret_path="/"
    )

    return f"Hello! My name is: {name.secretValue}"
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/d984e893-5442-4b98-a08a-0dbbf62df125.png)


## 終わりに
Infisicalなどのシークレット管理ツールを使用すれば環境ごとの設定ミスなども防げそうで、安全にシークレット管理できそうですね。

## 参考
[公式ドキュメント](https://infisical.com/docs/documentation/getting-started/overview)

