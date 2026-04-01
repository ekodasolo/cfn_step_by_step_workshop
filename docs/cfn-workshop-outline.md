# CloudFormation ワークショップ 構想まとめ

## コンセプト

最小構成のテンプレートから段階的に拡張し、**テンプレートを育てながら CloudFormation を学ぶ**ワークショップ。
スタック更新を繰り返す過程で、テンプレートの要素・組み込み関数・スタック運用を自然に習得する。

---

## ワークショップ基本情報

| 項目 | 内容 |
|---|---|
| 所要時間 | 4 時間 |
| 形式 | ハンズオン（参加者が手を動かす） |
| テンプレート形式 | YAML（短縮構文 `!Ref`, `!Sub` 等を使用） |
| 進め方 | テキストの指示に沿い、AWS 公式 CloudFormation ドキュメントのリソースリファレンスを参照してプロパティを記述する |
| 作業範囲 | 全プロパティを書かせるのではなく、キーポイントとなるプロパティだけ自分でリファレンスを見て指定する |
| エラーハンドリング | ハッピーパターンのみ（ロールバック体験は含めない） |

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
- EC2 の UserData で httpd を起動し、シェルスクリプトでカレンダーページを生成
- **表示内容**: 当月カレンダー + EC2 メタデータ（Instance ID / Private IP）
  - カレンダー部分は全インスタンス共通、メタデータ部分がインスタンスごとに異なる
  - ALB でリロードすると別インスタンスに振り分けられ、メタデータの変化でロードバランシングを実感できる
- **SSM Session Manager** でアクセス（キーペア不使用）

---

## スタック構成と分担

| スタック | 含むリソース |
|---|---|
| **ネットワークスタック** | VPC / サブネット / IGW / ルートテーブル / セキュリティグループ |
| **アプリケーションスタック** | IAM ロール（SSM 用） / EC2 / ALB + TargetGroup + Listener / Launch Template / Auto Scaling Group |

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

**参加者が書くキーポイント**:
- `AWS::EC2::VPC` の `CidrBlock` プロパティ

---

#### Step 2: サブネットを追加

**作るもの**: パブリックサブネット × 2（Multi-AZ）

**学ぶこと**:
- `Parameters` セクション（CIDR を外部化）
- 短縮構文: `!Ref`, `!Select`, `!GetAZs`
- スタック更新の実行（テンプレートを差し替えて Update Stack）
- 更新時のイベントログの読み方

**参加者が書くキーポイント**:
- `AWS::EC2::Subnet` の `VpcId`, `CidrBlock`, `AvailabilityZone` プロパティ
- `!Ref` でパラメータやリソースを参照する書き方
- `!Select` + `!GetAZs` で AZ を動的に取得する書き方

**新たに登場するテンプレート要素**:
- `Parameters`
- `!Ref`, `!Select`, `!GetAZs`

---

#### Step 3: インターネット接続を構成

**作るもの**: Internet Gateway / VPC Gateway Attachment / ルートテーブル / ルート / サブネットルートテーブル関連付け

**学ぶこと**:
- リソース間の依存関係（暗黙的依存 vs `DependsOn`）
- `!Sub` を使った文字列組み立て（タグの動的命名）
- 「リソース追加 = 既存リソースに影響なし」の安全な更新パターン
- CloudFormation が作成順序を自動解決する仕組み

**参加者が書くキーポイント**:
- `AWS::EC2::InternetGateway` の定義
- `AWS::EC2::Route` の `GatewayId`, `DestinationCidrBlock` プロパティ
- `!Sub` による動的なタグ名の組み立て

**新たに登場するテンプレート要素**:
- `DependsOn`
- `!Sub`

---

#### Step 4: セキュリティグループと Outputs

**作るもの**: ALB 用 SG（HTTP 80 を公開）/ EC2 用 SG（ALB からのみ HTTP 許可）

**学ぶこと**:
- `Outputs` セクション（値をスタック外に公開する）
- `Export` によるクロススタック参照の準備
- `!GetAtt` でリソースの属性を取得
- セキュリティグループの Ingress 設計（SG 間参照）

**参加者が書くキーポイント**:
- `SecurityGroupIngress` のルール定義（ポート、ソース SG）
- `Outputs` + `Export` の書き方
- `!GetAtt` の使い方

**新たに登場するテンプレート要素**:
- `Outputs` + `Export`
- `!GetAtt`

---

### アプリケーションスタック編

#### Step 5: EC2 インスタンス 1 台

**作るもの**: IAM ロール（SSM 用）/ インスタンスプロファイル / EC2 1 台（UserData で httpd + HTML を配信）

**学ぶこと**:
- 2 つ目のスタック作成（別テンプレートから新しいスタック）
- `!ImportValue` によるクロススタック参照（ネットワークスタックの Outputs を使う）
- `Mappings` でリージョンごとの AMI ID を管理
- `!FindInMap` で Mappings から値を取得
- UserData による EC2 の初期設定
- SSM Session Manager で EC2 に接続して動作確認

**参加者が書くキーポイント**:
- `!ImportValue` でネットワークスタックのサブネット ID・SG ID を参照
- `Mappings` セクションの定義
- `!FindInMap` の書き方
- `UserData` の httpd インストール・起動スクリプト

**新たに登場するテンプレート要素**:
- `Mappings`
- `!ImportValue`, `!FindInMap`

---

#### Step 6: ALB を追加

**作るもの**: ALB / ターゲットグループ / リスナー

**学ぶこと**:
- ALB の構成要素（ALB → Listener → TargetGroup → Targets）
- EC2 をターゲットグループに登録する
- スタック更新で既存 EC2 に ALB を追加する流れ
- ALB の DNS 名を `Outputs` で取得し、ブラウザからアクセス確認

**参加者が書くキーポイント**:
- `AWS::ElasticLoadBalancingV2::TargetGroup` のヘルスチェック設定
- `AWS::ElasticLoadBalancingV2::Listener` の `DefaultActions` 定義
- ALB の `Subnets` に `!ImportValue` でサブネットを指定

---

#### Step 7: Launch Template + Auto Scaling + Conditions

**作るもの**: Launch Template / Auto Scaling Group（EC2 の直接定義を置き換え）/ Conditions による環境切り替え

**学ぶこと**:
- Launch Template と直接 EC2 定義の違い
- Auto Scaling Group のパラメータ（Min/Max/Desired）
- `Conditions` で環境（prod/dev）によるインスタンスサイズの切り替え
- `!If` による条件分岐
- **リソースの置き換え（Replace）** が発生するスタック更新
- 更新ポリシー（`UpdatePolicy`）によるローリング更新
- スケーリングの動作確認（Desired を変更して台数増減）

**参加者が書くキーポイント**:
- `Conditions` セクションの定義
- `!If` による `InstanceType` の切り替え
- `AWS::AutoScaling::AutoScalingGroup` の `MinSize`, `MaxSize`, `DesiredCapacity`
- `UpdatePolicy` の `AutoScalingRollingUpdate` 設定

**新たに登場するテンプレート要素**:
- `Conditions`
- `!If`
- `UpdatePolicy`

---

## 学習要素の対応表

| Step | テンプレート要素 | 組み込み関数（短縮構文） | スタック操作 |
|---|---|---|---|
| 1 | Resources, Type, Properties, Tags | — | Create Stack |
| 2 | Parameters | !Ref, !Select, !GetAZs | Update Stack |
| 3 | DependsOn | !Sub | Update Stack（リソース追加） |
| 4 | Outputs, Export | !GetAtt | Update Stack + コンソール確認 |
| 5 | Mappings | !ImportValue, !FindInMap | Create Stack（2つ目） |
| 6 | — (ALB 構成パターン) | — | Update Stack（ALB 追加） |
| 7 | Conditions, UpdatePolicy | !If | Update Stack（リソース置換） |

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
| スタック削除の順序 | 最後（App → Network の順で削除が必要） |

---

## 前提条件

- **参加者**: AWS の基礎知識あり（VPC, EC2, SG の概念は知っている前提）
- **使用ツール**: AWS マネジメントコンソール
- **参照資料**: AWS 公式 CloudFormation ドキュメント（リソースリファレンス）
- **リージョン**: ap-northeast-1（東京）
- **テンプレート形式**: YAML（短縮構文を使用）
- **EC2 接続**: SSM Session Manager（キーペア不使用）
- **EC2 でホストするもの**: httpd + シェルスクリプト生成のカレンダーページ（`app/generate-page.sh`）

---

## 時間配分（目安: 4 時間）

| 時間 | 内容 |
|---|---|
| 0:00 - 0:15 | イントロ・CloudFormation の概要説明 |
| 0:15 - 0:45 | Step 1〜2（VPC + サブネット） |
| 0:45 - 1:15 | Step 3（IGW + ルートテーブル） |
| 1:15 - 1:45 | Step 4（SG + Outputs） |
| 1:45 - 2:00 | 休憩 |
| 2:00 - 2:40 | Step 5（EC2 + SSM 接続確認） |
| 2:40 - 3:15 | Step 6（ALB + ブラウザ確認） |
| 3:15 - 3:50 | Step 7（Launch Template + ASG + Conditions） |
| 3:50 - 4:00 | まとめ・スタック削除・クリーンアップ |
