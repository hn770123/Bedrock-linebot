# 技術仕様書 - バレエ教室向け自動翻訳LINE Bot

## 1. システムアーキテクチャ

### 1.1 全体構成

```
┌─────────────────┐
│   LINE Users    │
│  (講師・保護者)  │
└────────┬────────┘
         │
         │ HTTPS
         ▼
┌─────────────────────────┐
│  LINE Messaging API     │
│  (Webhook)              │
└────────┬────────────────┘
         │
         │ Webhook POST
         ▼
┌─────────────────────────┐
│  Amazon API Gateway     │
│  (REST API)             │
└────────┬────────────────┘
         │
         │ Invoke
         ▼
┌─────────────────────────┐
│    AWS Lambda           │
│  ┌──────────────────┐   │
│  │ Translation      │   │
│  │ Handler          │   │
│  └────────┬─────────┘   │
│           │             │
│  ┌────────▼─────────┐   │
│  │ Language         │   │
│  │ Detector         │   │
│  └────────┬─────────┘   │
│           │             │
│  ┌────────▼─────────┐   │
│  │ Ballet Terms     │   │
│  │ Dictionary       │   │
│  └────────┬─────────┘   │
└───────────┼─────────────┘
            │
            │ API Call
            ▼
┌─────────────────────────┐
│   Amazon Bedrock        │
│   (Claude 3.5 Sonnet)   │
└─────────────────────────┘
```

### 1.2 データフロー

1. **受信処理**
   ```
   LINE Message → API Gateway → Lambda (Webhook Handler)
   ```

2. **翻訳処理**
   ```
   Message → Language Detection → Dictionary Lookup →
   Bedrock Translation → Format Response
   ```

3. **送信処理**
   ```
   Translation Result → LINE Messaging API → User
   ```

## 2. AWS Lambda関数仕様

### 2.1 関数構成

#### メインハンドラー: `lambda_handler(event, context)`

**入力パラメータ:**
```python
{
    "body": str,  # LINE Webhookのペイロード（JSON文字列）
    "headers": {
        "x-line-signature": str  # LINE署名
    }
}
```

**出力:**
```python
{
    "statusCode": int,
    "body": str
}
```

### 2.2 モジュール構成

```
lambda/
├── lambda_function.py      # メインハンドラー
├── config.py               # 設定管理
├── services/
│   ├── __init__.py
│   ├── line_service.py     # LINE API連携
│   ├── translation_service.py  # 翻訳サービス
│   └── language_detector.py    # 言語検出
├── models/
│   ├── __init__.py
│   └── ballet_dictionary.py    # バレエ用語辞書
└── utils/
    ├── __init__.py
    ├── logger.py           # ロギング
    └── validators.py       # バリデーション
```

### 2.3 環境変数

| 変数名 | 説明 | 例 |
|--------|------|-----|
| `LINE_CHANNEL_SECRET` | LINEチャネルシークレット | (Parameter Store参照) |
| `LINE_CHANNEL_ACCESS_TOKEN` | LINEアクセストークン | (Parameter Store参照) |
| `BEDROCK_MODEL_ID` | Bedrockモデル識別子 | `anthropic.claude-3-5-sonnet-20241022-v2:0` |
| `AWS_REGION` | AWSリージョン | `us-east-1` |
| `LOG_LEVEL` | ログレベル | `INFO` |

### 2.4 Lambda設定

- **ランタイム**: Python 3.12
- **メモリ**: 512 MB
- **タイムアウト**: 30秒
- **同時実行数**: 10（初期設定）
- **IAMロール**: LambdaExecutionRole（後述）

## 3. Amazon Bedrock連携

### 3.1 使用モデル

- **モデルID**: `anthropic.claude-3-5-sonnet-20241022-v2:0`
- **推奨理由**:
  - 高い翻訳精度
  - 多言語対応
  - コンテキスト理解能力が高い

### 3.2 API呼び出し仕様

#### リクエスト形式

```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

request_body = {
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 1000,
    "temperature": 0.3,  # 翻訳の一貫性を重視
    "messages": [
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": f"{prompt}"
                }
            ]
        }
    ]
}

response = bedrock.invoke_model(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    body=json.dumps(request_body)
)
```

### 3.3 翻訳プロンプトテンプレート

#### 日本語→英語・ポーランド語

```python
PROMPT_JA_TO_EN_PL = """あなたはバレエ教室の専門翻訳者です。
子供バレエ教室で、日本人の保護者が講師（ポーランド人）に送るメッセージを翻訳してください。

【翻訳ルール】
1. 丁寧で温かみのある表現を使用
2. バレエ用語は専門用語として正確に翻訳
3. 子供（幼児〜小学生）に関する内容であることを考慮
4. 文化的な配慮を含める

【バレエ用語の例】
- プリエ → plié
- アラベスク → arabesque
- 発表会 → recital
- バーレッスン → barre work / praca przy drążku

【翻訳対象テキスト】
{message}

【出力形式】
以下のJSON形式で出力してください:
{{
  "english": "英語翻訳結果",
  "polish": "ポーランド語翻訳結果"
}}
"""
```

#### 英語・ポーランド語→日本語

```python
PROMPT_EN_PL_TO_JA = """あなたはバレエ教室の専門翻訳者です。
子供バレエ教室で、講師（ポーランド人）が日本人の保護者に送るメッセージを翻訳してください。

【翻訳ルール】
1. 丁寧で親しみやすい日本語に翻訳
2. バレエ用語は日本で一般的なカタカナ表記を使用
3. 子供（幼児〜小学生）に関する内容であることを考慮
4. 敬語を適切に使用

【バレエ用語の例】
- plié → プリエ
- arabesque → アラベスク
- recital → 発表会
- barre work → バーレッスン

【翻訳対象テキスト】
{message}

【出力形式】
翻訳結果のみを出力してください（JSON不要）。
"""
```

## 4. 言語検出

### 4.1 検出ロジック

```python
import re

def detect_language(text: str) -> str:
    """
    テキストの言語を検出

    Returns:
        'ja': 日本語
        'en_or_pl': 英語またはポーランド語
    """
    # 日本語文字（ひらがな、カタカナ、漢字）の検出
    japanese_pattern = re.compile(r'[\u3040-\u309F\u30A0-\u30FF\u4E00-\u9FFF]')

    if japanese_pattern.search(text):
        return 'ja'
    else:
        return 'en_or_pl'
```

### 4.2 言語別処理分岐

| 検出言語 | 翻訳先 | 出力形式 |
|----------|--------|----------|
| 日本語 | 英語 + ポーランド語 | 2言語表示 |
| 英語/ポーランド語 | 日本語 | 1言語表示 |

## 5. バレエ用語辞書

### 5.1 辞書データ構造

```python
BALLET_TERMS = {
    "ja_to_en": {
        "プリエ": "plié",
        "ルルベ": "relevé",
        "アラベスク": "arabesque",
        "ピルエット": "pirouette",
        "グランバットマン": "grand battement",
        "ポールドブラ": "port de bras",
        "タンデュ": "tendu",
        "デガジェ": "dégagé",
        "バーレッスン": "barre work",
        "センターレッスン": "center work",
        "発表会": "recital",
        "リハーサル": "rehearsal",
        "レオタード": "leotard",
        "トウシューズ": "pointe shoes",
        "バレエシューズ": "ballet shoes",
        "タイツ": "tights"
    },
    "ja_to_pl": {
        "プリエ": "plié",
        "ルルベ": "releve",
        "アラベスク": "arabesque",
        "ピルエット": "piruet",
        "バーレッスン": "praca przy drążku",
        "発表会": "recital",
        "レオタード": "trykot",
        "トウシューズ": "buty na palcach"
    },
    "en_to_ja": {
        "plié": "プリエ",
        "relevé": "ルルベ",
        "arabesque": "アラベスク",
        "pirouette": "ピルエット",
        "grand battement": "グランバットマン",
        "port de bras": "ポールドブラ",
        "tendu": "タンデュ",
        "dégagé": "デガジェ",
        "barre work": "バーレッスン",
        "center work": "センターレッスン",
        "recital": "発表会",
        "rehearsal": "リハーサル",
        "leotard": "レオタード",
        "pointe shoes": "トウシューズ",
        "ballet shoes": "バレエシューズ",
        "tights": "タイツ"
    }
}
```

### 5.2 辞書活用方法

- プロンプトに用語例を含める
- 翻訳後の用語検証（オプション）
- ユーザーフィードバックによる辞書拡張

## 6. LINE Messaging API連携

### 6.1 Webhook検証

```python
import hashlib
import hmac
import base64

def validate_signature(body: str, signature: str, channel_secret: str) -> bool:
    """
    LINE Webhookの署名を検証
    """
    hash_digest = hmac.new(
        channel_secret.encode('utf-8'),
        body.encode('utf-8'),
        hashlib.sha256
    ).digest()

    expected_signature = base64.b64encode(hash_digest).decode('utf-8')

    return signature == expected_signature
```

### 6.2 メッセージ返信

```python
import requests
import json

def send_reply(reply_token: str, message: str, access_token: str):
    """
    LINEにメッセージを返信
    """
    url = 'https://api.line.me/v2/bot/message/reply'

    headers = {
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {access_token}'
    }

    payload = {
        'replyToken': reply_token,
        'messages': [
            {
                'type': 'text',
                'text': message
            }
        ]
    }

    response = requests.post(url, headers=headers, data=json.dumps(payload))
    return response
```

### 6.3 メッセージフォーマット

#### 日本語→英語・ポーランド語の返信

```
🇬🇧 English:
{english_translation}

🇵🇱 Polski:
{polish_translation}
```

#### 英語・ポーランド語→日本語の返信

```
🇯🇵 日本語:
{japanese_translation}
```

## 7. エラーハンドリング

### 7.1 エラー分類と対応

| エラー種別 | HTTPステータス | ユーザー通知 | ログレベル |
|------------|----------------|--------------|-----------|
| 署名検証失敗 | 401 | なし | WARNING |
| メッセージ形式不正 | 400 | 「メッセージを処理できませんでした」 | WARNING |
| Bedrock API エラー | 500 | 「翻訳サービスで問題が発生しました」 | ERROR |
| LINE API エラー | 500 | なし（リトライ） | ERROR |
| タイムアウト | 504 | 「処理に時間がかかっています」 | WARNING |

### 7.2 リトライポリシー

```python
from botocore.config import Config

bedrock_config = Config(
    retries={
        'max_attempts': 3,
        'mode': 'adaptive'
    },
    connect_timeout=5,
    read_timeout=30
)

bedrock = boto3.client('bedrock-runtime', config=bedrock_config)
```

### 7.3 エラーメッセージ例

```python
ERROR_MESSAGES = {
    'translation_failed': '申し訳ございません。翻訳処理でエラーが発生しました。もう一度お試しください。',
    'unsupported_message': 'テキストメッセージのみ対応しています。',
    'service_unavailable': 'サービスが一時的に利用できません。しばらくしてからお試しください。'
}
```

## 8. セキュリティ仕様

### 8.1 IAMロール設定

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel"
      ],
      "Resource": [
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter"
      ],
      "Resource": [
        "arn:aws:ssm:us-east-1:*:parameter/linebot/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

### 8.2 Parameter Store設定

| パラメータ名 | タイプ | 説明 |
|--------------|--------|------|
| `/linebot/channel-secret` | SecureString | LINEチャネルシークレット |
| `/linebot/access-token` | SecureString | LINEアクセストークン |

### 8.3 API Gatewayセキュリティ

- **HTTPS強制**: 全エンドポイント
- **リクエストボディサイズ制限**: 10MB
- **レート制限**: 100リクエスト/秒
- **WAF（オプション）**: DDoS保護

## 9. ログとモニタリング

### 9.1 ログ出力

```python
import logging
import json

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def log_translation(user_id: str, source_lang: str, target_lang: str,
                   input_length: int, output_length: int, duration: float):
    """
    翻訳処理のログ出力
    """
    log_data = {
        'event': 'translation',
        'user_id': user_id,  # ハッシュ化推奨
        'source_language': source_lang,
        'target_language': target_lang,
        'input_length': input_length,
        'output_length': output_length,
        'duration_ms': duration
    }
    logger.info(json.dumps(log_data))
```

### 9.2 CloudWatch メトリクス

- **カスタムメトリクス**:
  - 翻訳リクエスト数（言語別）
  - 平均レスポンス時間
  - エラー率
  - Bedrockトークン使用量

### 9.3 アラート設定

| メトリクス | 閾値 | アクション |
|-----------|------|-----------|
| エラー率 | 5%以上 | SNS通知 |
| 平均レスポンス時間 | 10秒以上 | SNS通知 |
| Lambda同時実行数 | 制限の80%以上 | SNS通知 |

## 10. パフォーマンス最適化

### 10.1 Lambda最適化

- **コールドスタート対策**:
  - グローバル変数でAWS SDKクライアントを初期化
  - Lambda SnapStart有効化（Python 3.12対応後）

```python
import boto3

# グローバルスコープで初期化
bedrock_client = boto3.client('bedrock-runtime', region_name='us-east-1')
ssm_client = boto3.client('ssm', region_name='us-east-1')

def lambda_handler(event, context):
    # 既に初期化済みのクライアントを使用
    pass
```

### 10.2 Bedrock最適化

- **トークン削減**:
  - 不要な空白の削除
  - プロンプトの簡潔化
  - max_tokensの適切な設定

- **キャッシング**（将来実装）:
  - 同一メッセージの翻訳結果をDynamoDBにキャッシュ

### 10.3 レスポンス時間目標

| 処理 | 目標時間 |
|------|----------|
| Webhook受信〜Lambda起動 | <500ms |
| 言語検出 | <50ms |
| Bedrock翻訳 | <3秒 |
| LINE返信 | <500ms |
| **合計** | **<5秒** |

## 11. テスト仕様

### 11.1 単体テスト

```python
# tests/test_language_detector.py
import pytest
from services.language_detector import detect_language

def test_detect_japanese():
    assert detect_language("こんにちは") == "ja"
    assert detect_language("レッスンは何時ですか？") == "ja"

def test_detect_english_polish():
    assert detect_language("Hello, how are you?") == "en_or_pl"
    assert detect_language("Dzień dobry") == "en_or_pl"
```

### 11.2 統合テスト

- LINE Webhook イベントのシミュレーション
- Bedrock APIモックによる翻訳テスト
- エンドツーエンドフロー検証

### 11.3 負荷テスト

- 同時リクエスト: 10, 50, 100
- 想定シナリオ: 発表会前の集中問い合わせ

## 12. デプロイメント

### 12.1 IaC（Infrastructure as Code）

- **推奨ツール**: AWS SAM または Terraform
- **環境分離**: dev, staging, production

### 12.2 CI/CD

```yaml
# 例: GitHub Actions
name: Deploy Lambda

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.12'
      - name: Install dependencies
        run: pip install -r requirements.txt -t .
      - name: Deploy to AWS Lambda
        run: |
          zip -r function.zip .
          aws lambda update-function-code \
            --function-name ballet-translation-bot \
            --zip-file fileb://function.zip
```

## 13. 依存ライブラリ

### requirements.txt

```
boto3>=1.34.0
requests>=2.31.0
```

### レイヤー管理

- AWS SDK (boto3)は Lambda標準で提供されるため、最小限のパッケージングが可能

## 14. 今後の技術拡張

### 14.1 機能追加候補

- **DynamoDB統合**: 翻訳履歴の保存
- **S3統合**: バレエ用語辞書の外部管理
- **EventBridge統合**: 定期メンテナンスタスク
- **Amazon Polly統合**: 音声翻訳対応

### 14.2 スケーラビリティ

- Lambda同時実行数の段階的増加
- DynamoDB DAXによる高速キャッシング
- CloudFrontによるAPIレスポンス最適化（必要に応じて）

---

以上が、バレエ教室向け自動翻訳LINE Botの技術仕様書です。
