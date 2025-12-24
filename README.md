# バレエ教室向け自動翻訳LINE Bot

AWS Lambda と Amazon Bedrock を活用した、子供バレエ教室向けの多言語自動翻訳LINE Botです。ポーランド人講師と日本人保護者の円滑なコミュニケーションを支援します。

## 概要

このプロジェクトは、子供バレエ教室で働くポーランド人講師と日本人保護者の間の言語の壁を解消するために開発されました。バレエ専門用語を理解し、教室特有のコンテキストを考慮した高精度な翻訳を提供します。

### 主な機能

- **双方向自動翻訳**
  - 日本語 → 英語 + ポーランド語
  - 英語/ポーランド語 → 日本語

- **バレエ専門用語対応**
  - プリエ、アラベスク、ピルエットなどのバレエ用語を正確に翻訳
  - 50語以上の専門用語辞書を実装

- **コンテキスト認識翻訳**
  - 子供バレエ教室特有の文脈を理解
  - 丁寧で温かみのある表現を使用

### 技術スタック

- **AWS Lambda**: サーバーレス実行環境
- **Amazon Bedrock**: AI翻訳エンジン（Claude 3.5 Sonnet）
- **Amazon API Gateway**: Webhookエンドポイント
- **LINE Messaging API**: LINE Botインターフェース
- **Python 3.12**: 実装言語

## アーキテクチャ

```
LINE App (ユーザー)
    ↓
LINE Messaging API
    ↓
Amazon API Gateway
    ↓
AWS Lambda (翻訳処理)
    ├─ 言語検出
    ├─ バレエ用語辞書
    └─ Amazon Bedrock (Claude 3.5 Sonnet)
    ↓
LINE App (翻訳結果)
```

## 翻訳例

### 日本語から英語・ポーランド語への翻訳

**入力（保護者）:**
```
明日のバーレッスンは何時からですか？発表会の練習もありますか？
```

**出力（講師へ）:**
```
🇬🇧 English:
What time does tomorrow's barre work start? Will there also be rehearsal for the recital?

🇵🇱 Polski:
O której godzinie rozpoczyna się jutrzejsza praca przy drążku? Czy będzie również próba do recitalu?
```

### 英語・ポーランド語から日本語への翻訳

**入力（講師）:**
```
Next week we will practice arabesque and pirouette. Please bring pointe shoes.
```

**出力（保護者へ）:**
```
🇯🇵 日本語:
来週はアラベスクとピルエットの練習をします。トウシューズをご持参ください。
```

## ドキュメント

プロジェクトに関する詳細なドキュメントは `docs/` ディレクトリにあります:

- **[プロジェクト企画書](docs/project-proposal.md)**: プロジェクトの目的、背景、機能要件
- **[技術仕様書](docs/technical-specification.md)**: システムアーキテクチャ、API仕様、実装詳細
- **[実装計画書](docs/implementation-plan.md)**: 4週間の開発スケジュール、タスク、マイルストーン

## プロジェクト構成（予定）

```
Bedrock-linebot/
├── docs/                          # ドキュメント
│   ├── project-proposal.md        # 企画書
│   ├── technical-specification.md # 技術仕様書
│   └── implementation-plan.md     # 実装計画書
├── src/
│   ├── lambda/                    # Lambda関数
│   │   ├── lambda_function.py     # メインハンドラー
│   │   ├── config.py              # 設定管理
│   │   ├── services/              # ビジネスロジック
│   │   ├── models/                # データモデル
│   │   └── utils/                 # ユーティリティ
│   └── tests/                     # テストコード
├── infrastructure/                # IaC（Terraform/SAM）
└── README.md                      # このファイル
```

## バレエ用語辞書（一部）

| 日本語 | 英語 | ポーランド語 |
|-------|------|-------------|
| プリエ | plié | plié |
| アラベスク | arabesque | arabesque |
| ピルエット | pirouette | piruet |
| バーレッスン | barre work | praca przy drążku |
| 発表会 | recital | recital |
| トウシューズ | pointe shoes | buty na palcach |

完全な辞書は実装時に追加されます。

## コスト見積もり

### 月間運用コスト（100メッセージ/日として）

- AWS Lambda: 無料枠内
- Amazon Bedrock: 約¥4,000/月
- API Gateway: 無料枠内
- CloudWatch Logs: 約¥100/月

**合計: 約¥4,100/月**

## 開発スケジュール

- **Phase 1**: 環境構築と基本実装（Week 1）
- **Phase 2**: 機能拡張とバレエ用語対応（Week 2）
- **Phase 3**: テスト・パフォーマンス最適化（Week 3）
- **Phase 4**: 本番リリースと運用準備（Week 4）

詳細は[実装計画書](docs/implementation-plan.md)を参照してください。

## セキュリティ

- LINE Messaging APIの署名検証
- AWS IAMロールによる最小権限の原則
- 機密情報はParameter Storeで暗号化保存
- メッセージログは保存しない（プライバシー保護）

## 品質目標

- **翻訳精度**: バレエ用語95%以上、一般文章85%以上
- **応答時間**: 5秒以内
- **稼働率**: 99%以上
- **ユーザー満足度**: 80%以上

## 今後の拡張計画

- 音声メッセージの文字起こし＋翻訳
- 画像内テキスト（OCR）翻訳
- スケジュール管理機能
- よくある質問（FAQ）の自動応答
- 他の習い事への展開（音楽教室、スポーツ教室など）

## ライセンス

このプロジェクトは、子供バレエ教室向けに開発されています。

## 貢献

このプロジェクトへの貢献を歓迎します。プルリクエストを送信する前に、以下を確認してください:

- コードがPEP 8に準拠している
- 適切なテストが追加されている
- ドキュメントが更新されている

## サポート

質問や問題がある場合は、GitHubのIssuesセクションで報告してください。

---

**開発状況**: 企画・設計フェーズ完了、実装開始準備中
