# タスク一覧

## 凡例

- `[ ]` 未着手
- `[x]` 完了

---

## 0. プロジェクト基盤

- [x] リポジトリ初期化・ディレクトリ構成作成
- [x] ワークショップ構想の詳細化（未決定事項の確定）
- [x] EC2 ホスティング用カレンダーページ生成スクリプト作成

---

## 1. 完成形テンプレート作成

完成形を先に作り、動作確認の基準にする。

- [x] 1-1. ネットワークスタック完成形（`templates/network/network-final.yaml`）
  - VPC / サブネット×2 / IGW / ルートテーブル / SG×2 / Outputs+Export
- [x] 1-2. アプリケーションスタック完成形（`templates/application/application-final.yaml`）
  - IAM ロール / Launch Template / ASG / ALB / TargetGroup / Listener / Conditions / Mappings
- [ ] 1-3. 完成形テンプレートの動作確認（AWS にデプロイして検証）

---

## 2. ステップ別テンプレート作成（ネットワークスタック）

完成形から巻き戻し、各ステップの初期状態を導出する。

- [x] 2-1. `network-step4.yaml`（Step 4 初期状態 = Step 3 完成形）
  - Step 3 まで完成済み、SG と Outputs がまだない状態
- [x] 2-2. `network-step3.yaml`（Step 3 初期状態 = Step 2 完成形）
  - VPC + サブネット完成済み、IGW・ルートテーブルがまだない状態
- [x] 2-3. `network-step2.yaml`（Step 2 初期状態 = Step 1 完成形）
  - VPC のみ完成済み、サブネットがまだない状態
- [x] 2-4. `network-step1.yaml`（Step 1 初期状態 = 空テンプレート）
  - テンプレートの骨格のみ、リソースなし

---

## 3. ステップ別テンプレート作成（アプリケーションスタック）

- [x] 3-1. `application-step7.yaml`（Step 7 初期状態 = Step 6 完成形）
  - EC2 + ALB 完成済み、Launch Template・ASG・Conditions がまだない状態
- [x] 3-2. `application-step6.yaml`（Step 6 初期状態 = Step 5 完成形）
  - EC2 1台のみ完成済み、ALB がまだない状態
- [x] 3-3. `application-step5.yaml`（Step 5 初期状態 = 空テンプレート）
  - テンプレートの骨格のみ、リソースなし

---

## 4. ドキュメント作成

- [ ] 4-1. ハンズオン手順書（参加者用テキスト）
- [ ] 4-2. 講師ガイド（進行メモ・補足説明・トラブル対応）
