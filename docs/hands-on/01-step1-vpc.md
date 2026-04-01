# Step 1: VPC を作成する

> **目標**: テンプレートの基本構造を理解し、最初のスタックを作成する

## 学ぶこと

- テンプレートの基本構造（`AWSTemplateFormatVersion`, `Description`, `Resources`）
- リソースの書き方（`Type`, `Properties`）
- マネジメントコンソールからのスタック作成
- スタックの状態遷移（`CREATE_IN_PROGRESS` → `CREATE_COMPLETE`）

## CloudFormation テンプレートの基本構造

CloudFormation テンプレートは YAML 形式で記述します。最も基本的な構造は以下の通りです。

```yaml
AWSTemplateFormatVersion: "2010-09-09"   # テンプレートのバージョン（固定値）
Description: "テンプレートの説明"          # スタック一覧に表示される説明文

Resources:                                # 作成するリソースを定義するセクション
  リソースの論理名:                        # テンプレート内でこのリソースを参照する名前
    Type: AWS::サービス名::リソースタイプ   # 作成するリソースの種類
    Properties:                           # リソースの設定
      プロパティ名: 値
```

`Resources` セクションだけが必須で、他はオプションです。

## 作業

`templates/network/network-step1.yaml` を開いてください。TODO コメントの箇所に VPC リソースを追加します。

**📖 リファレンス**: AWS ドキュメントで `AWS::EC2::VPC` のページを開き、プロパティを確認してください。

以下の内容を `Resources:` の下に記述してください。

```yaml
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: "cfn-workshop-vpc"
```

> **ポイント**
> - `CidrBlock` は VPC のアドレス範囲です。`/16` は 65,536 個の IP アドレスを使えることを意味します
> - `EnableDnsSupport` と `EnableDnsHostnames` を `true` にしておくと、VPC 内のリソースが DNS 名前解決できるようになります
> - `Tags` でリソースに名前を付けておくと、コンソールで識別しやすくなります

## デプロイ

1. マネジメントコンソールで **CloudFormation** を開く
2. **「スタックの作成」** → **「新しいリソースを使用（標準）」** をクリック
3. **「テンプレートファイルのアップロード」** を選択し、編集した `network-step1.yaml` をアップロード
4. スタック名: **`cfn-workshop-network`** と入力
5. 他の項目はデフォルトのまま **「次へ」** → **「次へ」** → **「送信」**

## 確認

- CloudFormation コンソールでスタックのステータスが **`CREATE_COMPLETE`** になることを確認
- **「イベント」タブ** でリソース作成の流れを確認
- **VPC コンソール** に移動して、作成された VPC を確認

---

[← 概要に戻る](00-overview.md) | [Step 2: サブネットを追加する →](02-step2-subnets.md)
