---
title: "TerraformでAWS bedrockを使用したLINE botを作成する"
qiita_id: "9812654517306142ddb3"
url: "https://qiita.com/ikuoikuo/items/9812654517306142ddb3"
created_at: "2025-08-27T20:54:49+09:00"
updated_at: "2025-08-27T20:54:49+09:00"
tags: ["Python","AWS","Terraform","bedrock"]
---

<!-- Archived from Qiita: https://qiita.com/ikuoikuo/items/9812654517306142ddb3 -->
## 概要
筋トレの内容を報告すると、ものすごく褒めてくれるLINE botを作成しました。
やっぱり人間誰しも褒められたいですよね。。。

本当は女の子に頭を撫でてもらうのがベストですが、女性でもこのボットを活用できるようにかわいい子供から褒めてもらう仕様にしています。

![LINE_capture_777986062.662590.JPG](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/239fbdda-fc96-4891-9af7-751a59cedd46.jpeg)

## アーキテクチャ
LINEのMessagingAPIのwebhookを使用してAPI Gateway経由でLambdaを呼び出しています。bedrockではモデルをClaude 3.5 sonnetに設定しています。
![messagingAPI_architecture.drawio (2).png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3086181/c2a3cf4b-11f2-498c-9b23-a4a4d7b20558.png)

## lambda関数
今回は個人チャットではなくLINEグループに招待する予定なので、メンションされた時のみ反応するように実装しています。

個人チャットまたはボットがいるLINEグループにメッセージが送信されると、webhookでLambdaが呼び出されますが、実際に返信するのは、そのイベントでメンションされたユーザーIDがボットのユーザーIDと一致する場合のみです。

```py
import os, json, hmac, hashlib, base64, urllib.request
import boto3

bedrock = boto3.client("bedrock-runtime", region_name=os.environ["REGION"])

def _verify_line_signature(channel_secret, body, signature):
    mac = hmac.new(channel_secret.encode("utf-8"), body.encode("utf-8"), hashlib.sha256).digest()
    expected = base64.b64encode(mac).decode("utf-8")
    return hmac.compare_digest(expected, signature)

def _call_bedrock(model_id, prompt_text):
    payload = {
        "anthropic_version": "bedrock-2023-05-31",
        "messages": [
            {"role":"user","content":[{"type":"text","text": prompt_text}]}
        ],
        "max_tokens": 300,
        "temperature": 0.6
    }
    resp = bedrock.invoke_model(
        modelId=model_id,
        body=json.dumps(payload).encode("utf-8"),
        contentType="application/json",
        accept="application/json"
    )
    out = json.loads(resp["body"].read())
    return out["content"][0]["text"]

def _reply_line(access_token, reply_token, messages):
    data = json.dumps({"replyToken": reply_token, "messages": messages}).encode("utf-8")
    req = urllib.request.Request(
        "https://api.line.me/v2/bot/message/reply",
        data=data,
        headers={
            "Content-Type": "application/json",
            "Authorization": f"Bearer {access_token}"
        },
        method="POST"
    )

    try:
        with urllib.request.urlopen(req) as r:
            r.read()
    except urllib.error.HTTPError as e:
        error_content = e.read().decode()
        print(f"HTTP Error {e.code}: {error_content}")
        raise  
    except urllib.error.URLError as e:
        print(f"URL Error: {e.reason}")
        raise

def _is_bot_mentioned(event, bot_user_id):
    """ボットがメンションされているかチェック"""
    source_type = event.get("source", {}).get("type")
    
    if source_type == "user":
        return False
    
    if source_type in ["group", "room"]:
        message = event.get("message", {})
        mention = message.get("mention", {})
        if "mentionees" in mention:
            mentionees = mention["mentionees"]
            return any(m.get("userId") == bot_user_id for m in mentionees)
    
    return False

def lambda_handler(event, context):
    body = event.get("body") or ""
    headers = {k.lower():v for k,v in (event.get("headers") or {}).items()}
    signature = headers.get("x-line-signature", "")

    access_token = os.environ["LINE_CHANNEL_ACCESS_TOKEN"]
    channel_secret = os.environ["LINE_CHANNEL_SECRET"]
    bot_user_id = os.environ["BOT_USER_ID"]

    if not _verify_line_signature(channel_secret, body, signature):
        return {"statusCode": 403, "body": "invalid signature"}

    payload = json.loads(body)
    for ev in payload.get("events", []):
        if ev.get("type") == "message" and ev.get("message",{}).get("type") == "text":
            
            if not _is_bot_mentioned(ev, bot_user_id):
                continue
                
            reply_token = ev["replyToken"]
            text = ev["message"]["text"]
            
            message = ev.get("message", {})
            mention = message.get("mention", {})
            if "mentionees" in mention:
                for mention_info in mention["mentionees"]:
                    if mention_info.get("userId") == bot_user_id:
                        start = mention_info.get("index", 0)
                        length = mention_info.get("length", 0)
                        if start is not None and length is not None:
                            text = text[:start] + text[start + length:]
                            break
            
            text = text.strip()

            prompt = "hogehoge"

            ai = _call_bedrock(os.environ["BEDROCK_MODEL_ID"], prompt)
            messages = [{"type":"text","text": ai}]
            _reply_line(access_token, reply_token, messages)

    return {"statusCode": 200, "body": "ok"}
```
## 注意点
### LINE公式アカウントの設定方法
以下を参考にしてボットを作成できます。MessagingAPIを使ったボットの作成にはLINE公式アカウントが必要になります。

https://developers.line.biz/ja/docs/messaging-api/building-bot/

### Bedrockでのモデル呼び出しにおいて
Bedrockのモデルの中には直接モデルを呼び出すことができず、推論プロファイルを通じてアクセスする必要があるモデルが存在します。（2025/08/27現在Claude 3 Sonnetは直接呼び出し不可）
Bedrockのモデルアクセスページから確認できます。

https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html

## プロンプト
現在のおじさんのような少年のような返信を返してくれるシステムプロンプトはこちらになります。
```py
prompt = (
    "あなたは幼い子供のようにタメ口で、優しく筋トレを褒めてくれるキャラクターです。\n"
    "ユーザーが送ったメッセージから、どの部位を鍛えたかを判断してください。\n"
    "かわいい言葉を適度に入れて褒めてください。\n"
    "最後は必ず「引き続き頑張りまちょう💪」で締めてください。\n\n"
    f"ユーザーメッセージ: 『{text}』\n\n"
    "出力例: 「わー！胸すごい頑張ったね〜えらいえらい〜引き続き頑張りまちょう💪」\n"
    "注意: 上から目線は避ける、100文字程度で簡潔にしてください"
)
```

## terraform
LINEのアクセストークンやボットのユーザーIDなどは、terraform.tfvarsに格納しています。
複雑な構成ではないため、モジュール化せずにmain.tfにまとめて記述しています。

```hcl main.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# ---- IAM for Lambda ----
data "aws_iam_policy_document" "lambda_assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda_role" {
  name               = "line-bedrock-bot-role"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume.json
}

resource "aws_iam_role_policy" "lambda_policy" {
  name = "line-bedrock-bot-inline"
  role = aws_iam_role.lambda_role.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "bedrock:InvokeModel",
          "bedrock:InvokeModelWithResponseStream"
        ]
        Resource = "*"  # 必要に応じてモデルARNで絞れます
      }
    ]
  })
}

# ---- Lambda ----
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/lambda"
  output_path = "${path.module}/lambda.zip"
}

resource "aws_lambda_function" "bot" {
  function_name = "line-bedrock-bot"
  role          = aws_iam_role.lambda_role.arn
  handler       = "app.lambda_handler"
  runtime       = "python3.11"
  filename      = data.archive_file.lambda_zip.output_path
  timeout       = 15

  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  environment {
    variables = {
      LINE_CHANNEL_ACCESS_TOKEN = var.line_channel_access_token
      LINE_CHANNEL_SECRET       = var.line_channel_secret
      BEDROCK_MODEL_ID          = var.bedrock_model_id
      REGION                    = var.aws_region
      BOT_USER_ID              = var.bot_user_id
    }
  }
}

# ---- API Gateway HTTP API ----
resource "aws_apigatewayv2_api" "httpapi" {
  name          = "line-webhook"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda" {
  api_id                 = aws_apigatewayv2_api.httpapi.id
  integration_type       = "AWS_PROXY"
  integration_uri        = aws_lambda_function.bot.arn
  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "webhook" {
  api_id    = aws_apigatewayv2_api.httpapi.id
  route_key = "POST /webhook"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}

resource "aws_apigatewayv2_stage" "prod" {
  api_id      = aws_apigatewayv2_api.httpapi.id
  name        = "$default"
  auto_deploy = true
}

resource "aws_lambda_permission" "allow_apigw" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.bot.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.httpapi.execution_arn}/*/*"
}

output "webhook_url" {
  value = "${aws_apigatewayv2_api.httpapi.api_endpoint}/webhook"
}

```

## コード
https://github.com/ikuoikuo/musclemate-linebot

## まとめ
生成AIとIaCの組み合わせによって簡単にサービスを構築、デプロイできるようになったなと感じます。
これからも自分の欲しいものをいっぱい作っていきたいですね。引き続き頑張りまちょう💪

