# CLAUDE.md — プロジェクト規約

## プロジェクト概要

CloudFormation テンプレートを段階的に拡張しながら学ぶハンズオンコンテンツの開発プロジェクト。
最小構成の VPC から始め、ALB + Auto Scaling + EC2 の構成を 2 スタックで完成させる。

---

## ユーザー（Yohei）について

- このプロジェクトの開発者を「Yohei」と呼ぶ
- Web アプリケーション開発は初心者のため、教育的な観点での解説やわかりやすい説明を心がける
- 技術的な判断をする際は、なぜそうするのかの背景も添える

---

## 仕様管理・ナレッジ管理

- **docs/cfn-workshop-outline.md**: ワークショップの構想・設計をまとめたファイル
- **TODOS.md**: タスクの内容を先にこのファイルにまとめてから、タスクを順番に消化する。タスクが一つ終わるごとにこのファイルを更新して進捗状況をまとめる
- **ISSUES.md**: 開発中に発生したエラーやトラブルの内容を記録する。発生状況・原因・対処法をセットで残す
- **KNOWLEDGE.md**: ISSUES.md での対応から得られたナレッジや TIPS をまとめる

---

## ディレクトリ構成

```
cfn_workshop/
├── CLAUDE.md              # このファイル（プロジェクト規約）
├── TODOS.md               # タスク管理
├── ISSUES.md              # エラー・トラブル記録
├── KNOWLEDGE.md           # ナレッジ・TIPS 集
├── app/                   # EC2 でホストするアプリ（カレンダーページ生成スクリプト）
├── templates/             # CloudFormation テンプレート
│   ├── network/           # ネットワークスタック
│   │   ├── network-final.yaml   # 完成形（Step 4 終了時点）
│   │   ├── network-step1.yaml   # Step 1 初期状態
│   │   ├── network-step2.yaml   # Step 2 初期状態
│   │   ├── network-step3.yaml   # Step 3 初期状態
│   │   └── network-step4.yaml   # Step 4 初期状態
│   └── application/       # アプリケーションスタック
│       ├── application-final.yaml  # 完成形（Step 7 終了時点）
│       ├── application-step5.yaml  # Step 5 初期状態
│       ├── application-step6.yaml  # Step 6 初期状態
│       └── application-step7.yaml  # Step 7 初期状態
└── docs/                  # 手順書・講師ガイド等
```

### テンプレートの作成方針

- **完成形（final）を先に作成**し、そこからリソースを削って**各ステップの初期状態を導出する**
- `stepN.yaml` は「Step N で参加者が手を動かす前の状態」= 前ステップの完成形

---

## コード・ドキュメントの言語方針

| 対象 | 言語 | 例 |
|---|---|---|
| ドキュメント（.md 等） | 日本語 | 手順書、講師ガイド |
| テンプレート内 Description | 英語 | `"Step 1 - VPC only"` |
| テンプレート内コメント | 日本語 | `# パブリックサブネット（AZ-a）` |
| リソース論理名 | 英語 | `PublicSubnet1`, `WebServerSG` |

基本方針: **システム的なものは英語、人間が説明を受けるためのものは日本語**

---

## バージョン管理

- Git で管理する
- リポジトリはこのプロジェクトディレクトリをルートとする

---

## 作業の進め方

- 作業が完了したら、その内容を **TODOS.md に反映**（該当タスクを `[x]` に更新）してから、Yohei に報告する
- 作業ごとに **Git のコミットを作成**する
