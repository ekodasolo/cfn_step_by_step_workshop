# CloudFormation ワークショップ

最小構成の VPC テンプレートから段階的にリソースを追加し、CloudFormation を学ぶ 4 時間のハンズオン教材。

## 完成形

VPC + ALB + Auto Scaling + EC2（カレンダーアプリ）を 2 スタック構成で構築する。

## ディレクトリ構成

```
templates/
  network/       ネットワークスタック（step1〜4 + final）
  application/   アプリケーションスタック（step5〜7 + final）
app/             EC2 でホストするカレンダーページ生成スクリプト
docs/
  introduction.md          IaC / CloudFormation の導入資料
  hands-on/                参加者用手順書（チャプター別）
  instructor-guide.md      講師ガイド
  cfn-workshop-outline.md  ワークショップ構想・設計
```

## ステップ一覧

| Step | 内容 | 主な学習要素 |
|------|------|-------------|
| 1 | VPC | テンプレート基本構造 |
| 2 | サブネット | Parameters, !Ref, 変更セット |
| 3 | IGW + ルートテーブル | DependsOn, !Sub |
| 4 | SG + Outputs | Outputs/Export, !GetAtt |
| 5 | EC2 (新スタック) | !ImportValue, Mappings, !FindInMap |
| 6 | ALB | ALB/Listener/TargetGroup 構成 |
| 7 | Launch Template + ASG | Conditions, !If, UpdatePolicy |
