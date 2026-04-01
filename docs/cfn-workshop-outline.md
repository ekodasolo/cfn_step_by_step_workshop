# CloudFormation ワークショップ 構想まとめ

## コンセプト

最小構成のテンプレートから段階的に拡張し、**テンプレートを育てながら CloudFormation を学ぶ**ワークショップ。
スタック更新を繰り返す過程で、テンプレートの要素・組み込み関数・スタック運用を自然に習得する。

---

## 完成形アーキテクチャ

```
            Internet
               │
         ┌─────┴─────┐
         │    IGW     │
         └─────┬─────┘
               │
    ┌──────────┴──────────┐
    │        VPC          │
    │  ┌───────┬────────┐ │
    │  │Pub-1a │ Pub-1c │ │  ← パブリックサブネット × 2 (Multi-AZ)
    │  │       │        │ │
    │  │  EC2  │  EC2   │ │  ← Auto Scaling Group で管理
    │  └───┬───┴───┬────┘ │
    │      └───┬───┘      │
    │      ┌───┴───┐      │
    │      │  ALB  │      │
    │      └───────┘      │
    └─────────────────────┘
```

- **2 スタック構成**: ネットワークスタック + アプリケーションスタック
- **DB 不使用**: EC2 でスタティックな Web サイトをホスト
- EC2 の UserData で簡単な HTML を配信する（nginx or httpd）

---

## スタック構成と分担

| スタック | 含むリソース |
|---|---|
| **ネットワークスタック** | VPC / サブネット / IGW / ルートテーブル / セキュリティグループ |
| **アプリケーションスタック** | EC2 / ALB + TargetGroup + Listener / Launch Template / Auto Scaling Group |

---

## 段階的拡張ステップ

### ネットワークスタック編

#### Step 1: VPC だけのテンプレート

**作るもの**: VPC 1 つだけ

**学ぶこと**:
- テンプレートの基本構造（`AWSTemplateFormatVersion`, `Description`, `Resources`）
- リソースの書き方（Type, Properties）
- マネジメントコンソールからのスタック作成
- スタックの状態遷移（CREATE_IN_PROGRESS → CREATE_COMPLETE）

**テンプレート要素**:
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "Step 1 - VPC only"
Resources:
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

---

#### Step 2: サブネットを追加

**作るもの**: パブリックサブネット × 2（Multi-AZ）

**学ぶこと**:
- `Parameters`（CIDR や AZ を外部化）
- 組み込み関数: `Ref`, `Fn::Select`, `Fn::GetAZs`
- スタック更新の実行（テンプレートを差し替えて Update Stack）
- 更新時のイベントログの読み方

**新たに登場するテンプレート要素**:
- `Parameters` セクション
- `Ref` でパラメータやリソースを参照
- `Fn::Select` + `Fn::GetAZs` で AZ を動的に取得

---

#### Step 3: インターネット接続を構成

**作るもの**: Internet Gateway / ルートテーブル / ルート / サブネット関連付け

**学ぶこと**:
- リソース間の依存関係（暗黙的依存 vs `DependsOn`）
- `Fn::Sub` を使った文字列組み立て（タグの動的命名など）
- 「リソース追加 = 既存リソースに影響なし」の安全な更新パターン
- CloudFormation が作成順序を自動解決する仕組み

**新たに登場するテンプレート要素**:
- `DependsOn`（IGW アタッチ → ルート作成の順序制御）
- `Fn::Sub`

---

#### Step 4: セキュリティグループと Outputs

**作るもの**: ALB 用 SG / EC2 用 SG（ALB からのみ許可）

**学ぶこと**:
- `Outputs` セクション（値をスタック外に公開する）
- `Export` によるクロススタック参照の準備
- セキュリティグループの Ingress/Egress 設計
- ここまでの完成形をマネジメントコンソールで確認

**新たに登場するテンプレート要素**:
- `Outputs` + `Export`
- `Fn::GetAtt`（リソースの属性を取得）

---

### アプリケーションスタック編

#### Step 5: EC2 インスタンス 1 台

**作るもの**: EC2 1 台（UserData で httpd を起動し、簡単な HTML を配信）

**学ぶこと**:
- 2 つ目のスタック作成（別テンプレートから新しいスタック）
- `Fn::ImportValue` によるクロススタック参照（ネットワークスタックの出力を使う）
- `Fn::Base64` + `Fn::Sub` による UserData の記述
- `Mappings` で AMI ID をリージョンごとに管理
- EC2 に直接アクセスして動作確認

**新たに登場するテンプレート要素**:
- `Fn::ImportValue`（クロススタック参照）
- `Fn::Base64`
- `Mappings` セクション

---

#### Step 6: ALB を追加

**作るもの**: ALB / ターゲットグループ / リスナー

**学ぶこと**:
- ALB の構成要素（ALB → Listener → TargetGroup → Targets）
- EC2 をターゲットグループに登録する
- スタック更新で既存 EC2 に ALB を追加する流れ
- ALB の DNS 名を `Outputs` で取得し、ブラウザからアクセス確認

**ポイント**:
- EC2 に直接アクセスしていたのを ALB 経由に切り替える体験

---

#### Step 7: Launch Template + Auto Scaling

**作るもの**: Launch Template / Auto Scaling Group（EC2 の直接定義を置き換え）

**学ぶこと**:
- Launch Template と直接 EC2 定義の違い
- Auto Scaling Group のパラメータ（Min/Max/Desired）
- **リソースの置き換え（Replace）** が発生するスタック更新
- 更新ポリシー（`UpdatePolicy`）によるローリング更新
- スケーリングの動作確認（Desired を変更して台数増減）
- `Conditions`（オプション: 本番/開発でインスタンスサイズを切り替え）

**新たに登場するテンプレート要素**:
- `UpdatePolicy`（Auto Scaling のローリング更新制御）
- `Conditions`（オプション）

---

## 学習要素の対応表

| Step | テンプレート要素 | 組み込み関数 | スタック操作 |
|---|---|---|---|
| 1 | Resources, Type, Properties, Tags | — | Create Stack |
| 2 | Parameters | Ref, Fn::Select, Fn::GetAZs | Update Stack |
| 3 | DependsOn | Fn::Sub | Update Stack（リソース追加） |
| 4 | Outputs, Export | Fn::GetAtt | Update Stack + コンソール確認 |
| 5 | Mappings, Fn::ImportValue | Fn::Base64, Fn::ImportValue, Fn::Sub | Create Stack（2つ目） |
| 6 | — (構成パターン) | — | Update Stack（ALB追加） |
| 7 | UpdatePolicy, (Conditions) | — | Update Stack（リソース置換） |

---

## CloudFormation 運用面の学習ポイント

各ステップを通じて自然に学べる運用知識:

| 運用トピック | 学ぶタイミング |
|---|---|
| スタックの状態遷移 | Step 1（作成時） |
| スタック更新の流れ | Step 2 以降（毎回） |
| イベントログの読み方 | Step 2〜（更新時のリソース変更追跡） |
| 変更セット（Change Sets） | Step 3 or 4（安全な更新確認として紹介） |
| クロススタック参照 | Step 4〜5（Export → ImportValue） |
| リソースの置き換え vs 更新 | Step 7（EC2 → Launch Template + ASG） |
| ロールバック | エラー発生時（意図的に体験させるのもアリ） |
| スタック削除の順序 | 最後（App → Network の順で削除が必要） |

---

## 前提条件・制約

- **参加者**: AWS の基礎知識あり（VPC, EC2, SG の概念は知っている前提）
- **使用ツール**: AWS マネジメントコンソール（CLI は使わない or 補助的に使用）
- **リージョン**: 固定（例: ap-northeast-1）
- **テンプレート形式**: YAML
- **EC2 でホストするもの**: httpd + 簡単な HTML（「CloudFormation Workshop」的なページ）

---

## 未決定事項

- [ ] ワークショップの所要時間（半日？1日？）
- [ ] 参加者は自分で手を動かすか、講師のデモを見る形式か
- [ ] EC2 のキーペア管理（SSM Session Manager にするか）
- [ ] 意図的にエラーを起こしてロールバックを体験するステップを入れるか
- [ ] Conditions を Step 7 に含めるか、オプション扱いにするか
- [ ] このワークショップ用のディレクトリを flask_workshop 配下に作るか、別リポジトリにするか
