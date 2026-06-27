# Claude Platform on AWS ガイド

> Anthropic の Claude ネイティブ API プラットフォームを AWS アカウント経由で利用するためのガイドです。

## 目次

- [概要](#概要)
- [主な特徴](#主な特徴)
- [利用可能な機能](#利用可能な機能)
- [Amazon Bedrock との違い](#amazon-bedrock-との違い)
- [ユースケース・想定ユーザー](#ユースケース想定ユーザー)
- [セットアップ手順](#セットアップ手順)
  - [前提条件](#前提条件)
  - [Step 1: AWS Console でサインアップ](#step-1-aws-console-でサインアップ)
  - [Step 2: Anthropic 組織のセットアップ](#step-2-anthropic-組織のセットアップ)
  - [Step 3: Workspace ID を確認](#step-3-workspace-id-を確認)
  - [Step 4: Claude Console にサインイン](#step-4-claude-console-にサインイン)
- [SDK のインストール](#sdk-のインストール)
- [認証方式](#認証方式)
- [トラブルシューティング](#トラブルシューティング)
- [参考リンク](#参考リンク)

---

## 概要

**Claude Platform on AWS** は、Anthropic が運営する Claude のネイティブ API プラットフォームを、AWS アカウント経由で利用できるサービスです。2026年5月11日に一般提供（GA）が開始されました。

> 💡 ひと言でいうと：「Anthropic の Claude API を、そのまま AWS の認証・課金・監査の仕組みに乗せて使えるようにしたもの」

### アーキテクチャの概要

```
┌──────────────────────────────────────────────────────────────┐
│                     開発者 / アプリケーション                  │
└──────────────────┬───────────────────────────────────────────┘
                   │ API リクエスト
                   │ (SigV4 署名 or API キー)
                   ▼
┌──────────────────────────────────────────────────────────────┐
│                    AWS (認証・課金・監査)                      │
│  • IAM / SigV4 認証                                           │
│  • AWS Marketplace 課金                                       │
│  • CloudTrail ログ                                            │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│              Anthropic (インフラ運用・推論実行)                 │
│  • Claude モデル推論                                           │
│  • Agent Skills / Code Execution                              │
│  • Claude Console                                             │
└──────────────────────────────────────────────────────────────┘
```

---

## 主な特徴

| 観点 | 内容 |
|------|------|
| **運営主体** | Anthropic がインフラを運用（推論も Anthropic 側で実行） |
| **API** | Anthropic Messages API（`/v1/messages`）をそのまま利用 |
| **認証** | AWS IAM / SigV4 — 既存の AWS クレデンシャルでアクセス可能 |
| **課金** | AWS Marketplace 経由で AWS の請求に統合（Anthropic と別契約不要） |
| **監査** | AWS CloudTrail にアクティビティが記録される |
| **機能提供タイミング** | Anthropic が新モデル・新機能をリリースした**当日**に利用可能 |

---

## 利用可能な機能

- **Messages API** — ストリーミング／バッチ処理対応
- **Agent Skills** — PowerPoint・Excel・Word・PDF の文書生成等のプリビルトスキル
- **Code Execution** — Anthropic のマネージドサンドボックスでコード実行
- **Extended Thinking** — 高度な推論用の拡張思考
- **Files API** — ファイルアップロード＆リクエスト間での参照
- **Prompt Caching** — ツール、システムプロンプト、メッセージ履歴のキャッシュ
- **Claude Console** — プロンプト開発・評価のための UI
- **Claude Managed Agents (beta)** — マネージドエージェント
- **MCP Connector (beta)** — MCP プロトコル接続
- **Web Search / Web Fetch** — Web 検索・取得
- **Citations** — 引用機能

---

## Amazon Bedrock との違い

|  | Claude Platform on AWS | Amazon Bedrock |
|--|------------------------|----------------|
| **運営者** | Anthropic | AWS |
| **API** | Anthropic Messages API | Bedrock API（Converse / InvokeModel） |
| **ベース URL** | `aws-external-anthropic.{region}.api.aws` | `bedrock-runtime.{region}.amazonaws.com` |
| **データ処理** | Anthropic ＋ AWS（データは AWS 外に出る可能性あり） | AWS のみ（データは AWS 内に留まる） |
| **新機能の利用** | 当日提供 | AWS 側の統合待ち |
| **Agent Skills** | ✅ 利用可能 | ❌ 利用不可 |
| **Beta 機能** | `anthropic-beta` ヘッダーで利用可能 | AWS 側のサポートが必要 |
| **レート制限** | Anthropic が管理 | AWS が管理 |
| **コンプライアンス** | AWS の SOC / ISO / HIPAA 等は**対象外** | FedRAMP High, SOC, ISO, HIPAA 等で対象 |
| **SDK クライアント** | `AnthropicAWS`（Python 等） | `AnthropicBedrock` / Bedrock SDK |
| **コンソール** | Claude Console（`platform.claude.com`） | Bedrock Console |

---

## ユースケース・想定ユーザー

### Claude Platform on AWS が向いているケース

- ✅ 最新の Claude 機能に**即日アクセス**したい（Beta 含む）
- ✅ 既存の AWS アカウント・IAM・請求にそのまま統合したい
- ✅ Anthropic と別途契約を結びたくない
- ✅ Agent Skills や Code Execution などネイティブ機能を使いたい

### Amazon Bedrock が向いているケース

- ✅ **厳格なデータ所在要件**（リージョン内保持必須）がある
- ✅ AWS が唯一のデータ処理者である必要がある
- ✅ AWS Guardrails、Knowledge Bases、PrivateLink などが必要
- ✅ AWS のコンプライアンスプログラム（SOC, ISO, HIPAA 等）のカバレッジが必要

---

## セットアップ手順

### 前提条件

| # | 要件 |
|---|------|
| 1 | 有効な AWS アカウント |
| 2 | AWS CLI がインストール・設定済み |
| 3 | **Outbound Web Identity Federation** が AWS アカウントで有効化されていること（初回のみ） |
| 4 | セットアップ完了後に取得する Workspace ID |

---

### Step 1: AWS Console でサインアップ

1. [AWS Console](https://console.aws.amazon.com/) を開き、**Claude Platform on AWS** のサービスページに移動
2. **「Sign up」** を選択
3. **「Continue」** を選択

> [!NOTE]
> サインアップには数分かかります。ページに「Sign-up in progress」バナーが表示されるので、そのまま待ちます。AWS が裏側で AWS Marketplace サブスクリプションを自動的にプロビジョニングし、完了後にリダイレクトされます。

> [!TIP]
> 既に Anthropic からの **Private Offer** がある場合は、サインアップ前に Anthropic または AWS のアカウント担当者に連絡してください。割引は遡及適用されません。

---

### Step 2: Anthropic 組織のセットアップ

サインアップ完了後、`platform.claude.com/partner-signup` にリダイレクトされます。

1. 組織のオーナーの**メールアドレス**を入力 → **「Get started」**
2. そのメールに届くセットアップリンクを開く
3. 組織の詳細情報を入力（組織名、法人種別、国、利用用途）→ **「Complete setup」**

> [!IMPORTANT]
> この手順で作成される Anthropic 組織は、既存の Anthropic 組織（直接契約や Claude Enterprise）とは**別の組織**です。API キー、ワークスペース、Claude Console の設定は引き継がれません。

---

### Step 3: Workspace ID を確認

- サインアップ時に**デフォルトのワークスペース**が自動作成されます
- AWS Console の Claude Platform on AWS サービスページ → **「Workspaces」** で確認
- 形式: `wrkspc_XXXXXXXX`（英数字の識別子）
- リージョンごとに追加ワークスペースを作成可能

---

### Step 4: Claude Console にサインイン

1. `aws-external-anthropic:AssumeConsole` 権限を持つ **IAM ロールを Assume** する
2. AWS Console の Claude Platform on AWS ページ → **「Sign in」** を選択
3. AWS が JWT を発行し、`platform.claude.com` にリダイレクト
4. 初回サインイン時にメールアドレスを入力 → Console ユーザーが自動プロビジョニング

> サインイン後、Claude Console のサイドバーに **「Account managed by AWS」** と表示されます。

---

## SDK のインストール

Anthropic の公式 SDK は Claude Platform on AWS をサポートしています（7言語対応、現在 Beta）。

```bash
# Python
pip install -U "anthropic[aws]"

# TypeScript
npm install @anthropic-ai/aws-sdk

# C#
dotnet add package Anthropic.Aws

# Go
go get github.com/anthropics/anthropic-sdk-go

# Java (Gradle)
implementation("com.anthropic:anthropic-java-aws:2.27.0")

# PHP
composer require anthropic-ai/sdk aws/aws-sdk-php

# Ruby
gem install anthropic aws-sdk-core
```

### Python での基本的な使い方

```python
from anthropic import AnthropicAWS

client = AnthropicAWS()  # AWS クレデンシャルを自動解決

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Hello, Claude!"}
    ]
)
print(message.content[0].text)
```

---

## 認証方式

| 方式 | 説明 |
|------|------|
| **AWS IAM + SigV4**（推奨） | 既存の AWS クレデンシャルで署名リクエストを送信 |
| **API キー** | Bearer トークンとして提示 |

**エンドポイント:**

```
https://aws-external-anthropic.<region>.api.aws
```

---

## トラブルシューティング

| 症状 | 対処 |
|------|------|
| 「Failed to enable OutboundWebIdentityFederation」 | もう一度 **Continue** を押す（IAM 伝播に時間がかかる場合がある） |
| サインアップ中プログレスバーが出ない | 正常動作。数分待つ |
| 「Signed in as a different account」 | **Log out and continue** を選択 |
| 「Not found」メッセージ | リダイレクト中に一時的に表示される場合あり。無視して可 |
| Usage ページにデータが出ない | 最初の API 呼出後、数分で反映される |

---

## 参考リンク

| リソース | URL |
|----------|-----|
| 公式プロダクトページ | https://aws.amazon.com/claude-platform/ |
| AWS 公式ドキュメント | https://docs.aws.amazon.com/claude-platform/latest/userguide/welcome.html |
| セットアップガイド | https://docs.aws.amazon.com/claude-platform/latest/userguide/setup.html |
| 認証 | https://docs.aws.amazon.com/claude-platform/latest/userguide/authentication.html |
| Bedrock との比較 | https://docs.aws.amazon.com/claude-platform/latest/userguide/cpa-vs-bedrock.html |
| 発表ブログ（AWS） | https://aws.amazon.com/blogs/machine-learning/introducing-claude-platform-on-aws-anthropics-native-platform-through-your-aws-account/ |
| 発表ブログ（Anthropic） | https://claude.com/blog/claude-platform-on-aws |
| Getting Started ノートブック | https://github.com/aws-samples/anthropic-on-aws/blob/main/claude-platform-on-aws/notebooks/ |
| SDK ドキュメント | https://platform.claude.com/docs/en/api/client-sdks |

---

## ライセンス・利用規約

- [AWS Service Terms](https://aws.amazon.com/service-terms/)
- [Anthropic Commercial Terms of Service](https://www.anthropic.com/policies/terms)
- [Anthropic Data Processing Addendum](https://www.anthropic.com/policies/data-processing-addendum)
- [Anthropic Usage Policy](https://www.anthropic.com/policies/usage-policy)

---

*最終更新: 2026年6月*
