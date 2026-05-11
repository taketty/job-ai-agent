# 設計ドキュメント — job-ai-agent

> AI 駆動開発ライフサイクル（AI-DLC）準拠

---

## 目次

1. [プロジェクト概要](#1-プロジェクト概要)
2. [AI-DLC フェーズ定義](#2-ai-dlc-フェーズ定義)
3. [要件定義](#3-要件定義)
4. [AIシステム設計](#4-aiシステム設計)
5. [アーキテクチャ設計](#5-アーキテクチャ設計)
6. [コンポーネント詳細設計](#6-コンポーネント詳細設計)
7. [データ設計](#7-データ設計)
8. [外部連携設計](#8-外部連携設計)
9. [セキュリティ設計](#9-セキュリティ設計)
10. [テスト戦略](#10-テスト戦略)
11. [LLMOps・モニタリング](#11-llmopsモニタリング)

---

## 1. プロジェクト概要

### サービス概要

| 項目 | 内容 |
|------|------|
| サービス名 | job-ai-agent |
| コンセプト | 就活の全工程を AI が代行する全自動就活エージェント OS |
| ターゲット | 就職・転職活動中の求職者 |
| 提供価値 | 初回チャット以降の就活工程をゼロにし、対人バイアスを排除した本質的マッチングを実現 |

### ユーザー体験フロー

```
① 初期ヒアリング（チャット）
      ↓  ユーザーの作業はここで終了
② ES 自動生成
      ↓
③ 面接日程の自動調整（Google Calendar）
      ↓
④ 面接 AI 代行（求職者 AI ↔ 企業 AI）
      ↓
⑤ 内定レコメンド
      ↓
⑥ 辞退メール自動送信（Gmail）
```

---

## 2. AI-DLC フェーズ定義

本プロジェクトは以下の AI 駆動開発ライフサイクルに従い進行します。

```
┌─────────────────────────────────────────────────────────┐
│                     AI-DLC サイクル                      │
│                                                         │
│  ① Discovery  →  ② Design  →  ③ Build  →  ④ Evaluate  │
│       ↑                                       │        │
│       └───────────── ⑤ Iterate ←─────────────┘        │
└─────────────────────────────────────────────────────────┘
```

| フェーズ | 内容 | AI の活用方法 |
|---------|------|--------------|
| ① Discovery | 要件・課題の発見 | Claude による要件整理・ユーザーストーリー生成 |
| ② Design | システム設計 | Claude によるアーキテクチャ提案・プロンプト設計支援 |
| ③ Build | 実装 | Claude Code による AI 駆動コーディング |
| ④ Evaluate | 品質評価 | LLM-as-a-Judge による出力品質評価、自動テスト生成 |
| ⑤ Iterate | 継続改善 | フィードバックループによるプロンプト・ロジック改善 |

---

## 3. 要件定義

### 3.1 機能要件

#### FR-01 初期ヒアリング

- ユーザーはチャット UI から希望条件（企業の軸、年収、休日等）と経歴を入力する
- AI は会話を通じて不明点を深掘りし、「初期パラメータ」として構造化して保存する
- 入力完了後、ユーザーに確認画面を提示し承認を得る

#### FR-02 ES 自動生成

- 初期パラメータをもとに、企業情報（企業名・事業内容・求める人物像）に応じたESを自動生成する
- 生成した ES は S3 に保存し、ユーザーが閲覧できる

#### FR-03 面接日程の自動調整

- 企業からの面接打診メール（Gmail）を AI が自動検知する
- ユーザーの Google Calendar の空き時間を取得し、最適な候補日を選定する
- 企業への返信メールを自動生成・送信する
- 確定した面接予定を Google Calendar に自動登録する

#### FR-04 面接 AI 代行

- 求職者 AI（ユーザーの初期パラメータを基に動作）が企業の質問に回答する
- 企業 AI（Bedrock Agent）が求職者 AI に対して面接を実施する
- 面接ログを構造化してDynamoDB に保存する

#### FR-05 内定レコメンド

- 複数の内定情報と面接ログを分析し、初期パラメータとの適合スコアを算出する
- スコアと理由をユーザーに提示し、最も適した1社を推薦する

#### FR-06 辞退メール自動生成・送信

- ユーザーが承認した企業を除く全社へ、Gmail API で辞退メールを自動生成・送信する

### 3.2 非機能要件

| 項目 | 要件 |
|------|------|
| 可用性 | 99.9% 以上（AWS マネージドサービスの SLA に依拠） |
| レスポンス | チャット応答：ストリーミングで初回トークン 2 秒以内 |
| スケーラビリティ | サーバレス構成によりリクエスト数に応じて自動スケール |
| コスト | 使用量課金モデルで固定費ゼロ |
| セキュリティ | ユーザーの個人情報・OAuth トークンを暗号化して管理 |

### 3.3 制約条件

- AWS サーバレスサービスのみ使用（EC2 / RDS / Aurora / ECS / EKS は使用禁止）
- 外部連携は Google Calendar API と Gmail API のみ
- AI 基盤は Amazon Bedrock を使用

---

## 4. AIシステム設計

### 4.1 マルチエージェント構成

Amazon Bedrock Agents のマルチエージェントオーケストレーションを採用します。

```
┌────────────────────────────────────────────────────────────────┐
│                    Orchestrator Agent                          │
│              （全体ワークフローの制御・意思決定）                   │
└───────┬───────────┬──────────────┬───────────────┬────────────┘
        ↓           ↓              ↓               ↓
┌──────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────────┐
│ Profile  │ │   ES     │ │  Interview   │ │Recommendation│
│  Agent   │ │Generator │ │    Agent     │ │    Agent     │
│          │ │  Agent   │ │              │ │              │
│ユーザー  │ │ESを企業  │ │面接の        │ │内定の        │
│プロファイ│ │ごとに    │ │シミュレーシ  │ │適合スコアを  │
│ル分析   │ │最適化生成│ │ョン・代行    │ │算出・推薦    │
└──────────┘ └──────────┘ └──────────────┘ └──────────────┘
                                ↕
                    ┌──────────────────────┐
                    │  Scheduler Agent     │
                    │  （Google Calendar   │
                    │   / Gmail 連携）     │
                    └──────────────────────┘
```

### 4.2 エージェント定義

#### Orchestrator Agent

- **モデル**: Claude 3.5 Sonnet（amazon.claude-3-5-sonnet-20241022-v2:0）
- **役割**: ユーザーの就活フェーズを管理し、適切なサブエージェントに処理を委譲する
- **入力**: ユーザーの現在フェーズ、初期パラメータ、外部イベント（メール受信等）
- **ツール**: Step Functions 起動、DynamoDB Read/Write

#### Profile Agent

- **モデル**: Claude 3.5 Haiku
- **役割**: チャット形式でユーザーの希望条件・経歴を収集し、構造化データに変換する
- **入力**: ユーザーの発話テキスト
- **出力**: 初期パラメータ JSON（希望職種、年収、勤務地、企業規模、価値観、経歴等）

#### ES Generator Agent

- **モデル**: Claude 3.5 Sonnet
- **役割**: 初期パラメータと企業情報から最適化された ES を生成する
- **入力**: 初期パラメータ、企業情報（企業名・求める人物像・事業内容）
- **出力**: ES テキスト（志望動機、自己PR、ガクチカ等）

#### Interview Agent

- **モデル**: Claude 3.5 Sonnet（求職者 AI）/ Claude 3 Haiku（企業 AI）
- **役割**: 求職者 AI と企業 AI が往復で面接を実施し、双方の「本音」を引き出す
- **入力**: 初期パラメータ（求職者 AI）、企業情報（企業 AI）、面接ログ
- **出力**: 構造化面接ログ（Q&A、評価コメント）

#### Scheduler Agent

- **モデル**: Claude 3.5 Haiku
- **役割**: Google Calendar の空き時間取得、面接日程調整、メール送受信を担う
- **ツール**: Google Calendar API Tool、Gmail API Tool
- **入力**: 面接候補日リクエスト、空き時間情報
- **出力**: 確定日時、送信メール文面

#### Recommendation Agent

- **モデル**: Claude 3.5 Sonnet
- **役割**: 全内定と面接ログを初期パラメータと照合し、適合スコアを算出・推薦する
- **入力**: 内定リスト、面接ログ、初期パラメータ
- **出力**: 推薦企業 + 推薦理由 + 各社スコアレポート

### 4.3 プロンプト設計方針

| 原則 | 内容 |
|------|------|
| ロール定義 | 各エージェントに明確なペルソナとゴールを定義 |
| 構造化出力 | JSON Schema を用いて出力フォーマットを固定 |
| Few-shot | 期待する出力例を2〜3件プロンプトに含める |
| ガードレール | 個人情報の外部漏洩・不適切な文章生成を防ぐ制約を明記 |
| チェーン | エージェント間の受け渡しは構造化 JSON で行い、情報の欠落を防ぐ |

### 4.4 AI 評価指標（LLMOps）

| エージェント | 評価指標 | 目標値 |
|------------|---------|--------|
| Profile Agent | パラメータ抽出完全率 | 95% 以上 |
| ES Generator | 人間評価スコア（1-5） | 4.0 以上 |
| Interview Agent | 面接カバレッジ率（必須質問網羅） | 90% 以上 |
| Recommendation Agent | 推薦理由の根拠引用率 | 100% |

---

## 5. アーキテクチャ設計

### 5.1 全体構成図

```
                          ┌─────────────────┐
                          │   ユーザー       │
                          └────────┬────────┘
                                   │ HTTPS
                    ┌──────────────┴──────────────┐
                    ↓                             ↓
           ┌────────────────┐           ┌──────────────────┐
           │  AWS Amplify   │           │   API Gateway    │
           │（フロントエンド）│           │  REST / WebSocket│
           └────────────────┘           └────────┬─────────┘
                                                 │
                          ┌──────────────────────┼──────────────────────┐
                          ↓                      ↓                      ↓
                 ┌─────────────────┐   ┌──────────────────┐   ┌───────────────┐
                 │ Lambda          │   │ Lambda           │   │  Lambda       │
                 │ (Chat Handler)  │   │ (Event Handler)  │   │ (Auth Handler)│
                 └────────┬────────┘   └────────┬─────────┘   └───────┬───────┘
                          │                     │                     │
                          ↓                     ↓                     ↓
                 ┌─────────────────┐   ┌──────────────────┐   ┌───────────────┐
                 │  Bedrock Agents │   │  Step Functions  │   │    Cognito    │
                 │ (Multi-Agent)   │   │ (Job App Flow)   │   │   User Pool   │
                 └────────┬────────┘   └────────┬─────────┘   └───────────────┘
                          │                     │
                          ↓                     ↓
                 ┌─────────────────────────────────────────┐
                 │              DynamoDB                   │
                 │  users / conversations / applications   │
                 └─────────────────────────────────────────┘
                          │
                          ↓
                 ┌─────────────────────────────────────────┐
                 │                 S3                      │
                 │    生成 ES / 面接ログ / 一時ファイル     │
                 └─────────────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ↓                               ↓
┌──────────────────────┐       ┌──────────────────────┐
│  Google Calendar API │       │      Gmail API       │
│  （面接日程管理）     │       │  （メール送受信）     │
└──────────────────────┘       └──────────────────────┘
```

### 5.2 使用 AWS サービス一覧

| サービス | 用途 | 選定理由 |
|---------|------|---------|
| **Amazon Bedrock** | LLM（Claude）+ エージェントオーケストレーション | マルチエージェント対応、サーバレス、従量課金 |
| **AWS Lambda** | ビジネスロジック全般 | イベント駆動、完全サーバレス |
| **AWS Step Functions** | 就活ワークフローのオーケストレーション | 状態管理、エラーハンドリング、可視化 |
| **Amazon API Gateway** | REST API / WebSocket API | Lambda との統合、スケール自動化 |
| **Amazon DynamoDB** | ユーザーデータ・会話履歴・選考状況の保存 | サーバレス NoSQL、従量課金 |
| **Amazon S3** | 生成 ES・面接ログのファイルストレージ | 低コスト、耐久性 |
| **Amazon Cognito** | ユーザー認証・認可 | Google OAuth 連携、サーバレス |
| **AWS Secrets Manager** | Google API 認証情報の安全な管理 | 自動ローテーション対応 |
| **Amazon SQS** | 非同期処理キュー（面接処理等） | デカップリング、リトライ制御 |
| **Amazon EventBridge** | メール定期チェック・スケジュールトリガー | イベント駆動の自動化 |
| **AWS Amplify** | Next.js フロントエンドのホスティング・CI/CD | Git 連携、自動デプロイ |
| **Amazon CloudWatch** | ログ収集・メトリクス監視・アラート | AWS 統合監視 |
| **AWS X-Ray** | 分散トレーシング | マルチエージェント間のトレース |

---

## 6. コンポーネント詳細設計

### 6.1 フロントエンド

- **フレームワーク**: Next.js 15（App Router）
- **スタイリング**: Tailwind CSS
- **ホスティング**: AWS Amplify
- **認証**: Amazon Cognito（Google OAuth 連携）

#### 主要画面

| 画面 | 説明 |
|------|------|
| チャット画面 | 初期ヒアリングのメインUI。WebSocket でリアルタイムストリーミング表示 |
| ダッシュボード | 選考状況の一覧表示（企業名・フェーズ・面接日時・ステータス） |
| 内定レポート画面 | 内定企業の比較・AI推薦結果・スコアの可視化 |
| 設定画面 | Google アカウント連携・通知設定 |

### 6.2 バックエンド Lambda 関数

| 関数名 | トリガー | 役割 |
|--------|---------|------|
| `chat-handler` | API Gateway WebSocket | チャットメッセージを受信し Bedrock Agents に委譲、ストリーミング返却 |
| `workflow-trigger` | API Gateway REST | Step Functions ジョブ申し込みワークフローを起動 |
| `gmail-poller` | EventBridge（5分ごと） | Gmail API で受信メールを確認し、面接打診を検知・SQS に投入 |
| `interview-processor` | SQS | 面接ワークフロー（Bedrock Agents）を非同期実行 |
| `auth-callback` | API Gateway REST | Google OAuth コールバック処理、トークンを Secrets Manager に保存 |

### 6.3 Step Functions ワークフロー

就活プロセス全体を状態機械として管理します。

```
[Start]
   ↓
[Profile Analysis]         ← Profile Agent 呼び出し
   ↓
[ES Generation]            ← ES Generator Agent 呼び出し
   ↓
[Wait for Interview]       ← Gmail ポーリング待機（EventBridge）
   ↓
[Schedule Interview]       ← Scheduler Agent（Calendar / Gmail）
   ↓
[Run Interview]            ← Interview Agent（SQS 経由で非同期）
   ↓
[Evaluate Offer]           ← Recommendation Agent 呼び出し
   ↓
[Send Decline Mails]       ← Scheduler Agent（Gmail）
   ↓
[Complete]
```

---

## 7. データ設計

### 7.1 DynamoDB テーブル設計

#### `users` テーブル

| 属性名 | 型 | キー | 説明 |
|--------|-----|-----|------|
| `userId` | String | PK | Cognito ユーザーID |
| `initialParams` | Map | - | 初期パラメータ（希望条件・経歴） |
| `googleAccessToken` | String | - | Google OAuth アクセストークン（暗号化） |
| `createdAt` | String | - | ISO8601 |

#### `conversations` テーブル

| 属性名 | 型 | キー | 説明 |
|--------|-----|-----|------|
| `userId` | String | PK | ユーザーID |
| `messageId` | String | SK | メッセージID（タイムスタンプ降順） |
| `role` | String | - | `user` / `assistant` |
| `content` | String | - | メッセージ本文 |
| `createdAt` | String | - | ISO8601 |

#### `applications` テーブル

| 属性名 | 型 | キー | 説明 |
|--------|-----|-----|------|
| `userId` | String | PK | ユーザーID |
| `applicationId` | String | SK | 選考ID |
| `companyName` | String | - | 企業名 |
| `phase` | String | - | `es_sent` / `interview_scheduled` / `interview_done` / `offered` / `declined` |
| `interviewAt` | String | - | 面接日時（ISO8601） |
| `calendarEventId` | String | - | Google Calendar イベントID |
| `esS3Key` | String | - | 生成 ES の S3 キー |
| `interviewLogS3Key` | String | - | 面接ログの S3 キー |
| `offerScore` | Number | - | AI 適合スコア（0-100） |
| `updatedAt` | String | - | ISO8601 |

### 7.2 S3 バケット構造

```
job-ai-agent-documents/
├── {userId}/
│   ├── es/
│   │   └── {applicationId}.pdf
│   └── interview-logs/
│       └── {applicationId}.json
```

### 7.3 初期パラメータ JSON スキーマ

```json
{
  "desiredConditions": {
    "jobTypes": ["string"],
    "industries": ["string"],
    "annualSalaryMin": "number",
    "workStyle": "string",
    "holidays": "string",
    "location": ["string"],
    "companySize": "string",
    "values": ["string"]
  },
  "career": {
    "education": "string",
    "workHistory": [
      {
        "company": "string",
        "role": "string",
        "period": "string",
        "achievements": "string"
      }
    ],
    "skills": ["string"],
    "selfPR": "string"
  }
}
```

---

## 8. 外部連携設計

### 8.1 Google 認証フロー

```
[ユーザー] → [フロントエンド] → [Cognito Hosted UI]
                                       ↓ Google OAuth 2.0
                              [Google 認証画面]
                                       ↓ Authorization Code
                              [Lambda auth-callback]
                                       ↓ Token Exchange
                              [Secrets Manager に保存]
                                       ↓
                              [フロントエンドにリダイレクト]
```

- スコープ: `https://www.googleapis.com/auth/calendar` / `https://www.googleapis.com/auth/gmail.modify`
- トークンは Secrets Manager に暗号化保存（ユーザー ID をキーに分離管理）
- アクセストークンの有効期限切れ時はリフレッシュトークンで自動更新

### 8.2 Google Calendar 連携

| 処理 | API | Lambda 関数 |
|------|-----|------------|
| 空き時間取得 | `freebusy.query` | `interview-processor` |
| イベント作成 | `events.insert` | `interview-processor` |
| イベント更新 | `events.update` | `interview-processor` |
| イベント削除 | `events.delete` | `interview-processor` |

### 8.3 Gmail 連携

| 処理 | API | Lambda 関数 |
|------|-----|------------|
| 受信メール一覧取得 | `messages.list` | `gmail-poller` |
| メール本文取得 | `messages.get` | `gmail-poller` |
| メール送信 | `messages.send` | `interview-processor` |
| ラベル付与 | `messages.modify` | `gmail-poller` |

#### 面接打診メール検知ロジック

`gmail-poller` が 5 分ごとに以下の条件でメールを検索します。

```
query: "label:INBOX is:unread subject:(面接 OR 選考 OR interview)"
```

検知後は AI（Bedrock）でメール本文を解析し、面接打診か否かを判定して SQS に投入します。

---

## 9. セキュリティ設計

### 9.1 認証・認可

- 全 API は Cognito JWT による認証を必須とする
- Lambda は最小権限の IAM ロールを個別に付与する
- Google OAuth トークンは Secrets Manager で管理し、Lambda 環境変数に直接保存しない

### 9.2 データ保護

| データ種別 | 保護方式 |
|-----------|---------|
| DynamoDB | AWS 管理キーによる保存時暗号化 |
| S3 | SSE-S3 による暗号化、パブリックアクセスブロック |
| Google トークン | Secrets Manager（KMS 暗号化）に保存 |
| 通信 | 全通信を TLS 1.2 以上で暗号化 |

### 9.3 個人情報保護

- ユーザーの経歴・希望条件は DynamoDB に保存し、本人のみアクセス可能
- AI エージェントへの入力時は userId を匿名化したプロンプトを使用
- データ削除機能を実装し、退会時に全データを削除する

### 9.4 Bedrock ガードレール

Amazon Bedrock Guardrails を設定し、以下を防止します。

- 差別的・不適切な文章の生成
- 個人情報の意図しない外部送信
- 事実と異なる経歴・実績の捏造

---

## 10. テスト戦略

AI-DLC に基づき、AI コンポーネントと通常のソフトウェアテストを並行して実施します。

### 10.1 テスト種別

| テスト種別 | 対象 | ツール |
|-----------|------|--------|
| ユニットテスト | Lambda 関数・ユーティリティ | Jest |
| 統合テスト | API Gateway ↔ Lambda ↔ DynamoDB | AWS SAM Local |
| E2E テスト | フロントエンド全体フロー | Playwright |
| AI 品質評価 | 各エージェントの出力品質 | LLM-as-a-Judge（Claude 3.5 Sonnet） |
| プロンプト回帰テスト | プロンプト変更時の出力一貫性 | カスタム評価スクリプト |

### 10.2 AI 品質評価（LLM-as-a-Judge）

各エージェントの出力を Claude 3.5 Sonnet が自動評価します。

```
評価プロセス:
  入力: エージェント出力 + 評価基準プロンプト
  出力: スコア（1-5）+ 評価理由 + 改善提案

評価基準（ES Generator の例）:
  - 企業の求める人物像との整合性（0-2点）
  - 初期パラメータの反映度（0-2点）
  - 文章の自然さ・読みやすさ（0-1点）
```

---

## 11. LLMOps・モニタリング

### 11.1 プロンプト管理

- プロンプトはコードと分離し、DynamoDB または S3 で管理する
- バージョン管理を行い、ロールバック可能な構成とする
- A/B テストにより継続的にプロンプトを最適化する

### 11.2 モニタリング指標

| 指標 | 収集方法 | アラート閾値 |
|------|---------|------------|
| Bedrock API レイテンシ | CloudWatch Metrics | p95 > 10s |
| Lambda エラー率 | CloudWatch Logs | > 1% |
| Step Functions 失敗数 | CloudWatch Metrics | > 0（即通知） |
| AI 評価スコア平均 | カスタムメトリクス | < 3.5 |
| Google API 認証エラー | CloudWatch Logs Insights | 検知次第通知 |

### 11.3 フィードバックループ

```
[本番ユーザーの操作ログ]
       ↓
[CloudWatch Logs Insights で集計]
       ↓
[低評価ケースをサンプリング]
       ↓
[プロンプト改善 → 評価 → デプロイ]
       ↓
[改善効果を CloudWatch カスタムメトリクスで追跡]
```

---

*本ドキュメントは AI 駆動開発ライフサイクル（AI-DLC）の Discovery・Design フェーズの成果物です。*
