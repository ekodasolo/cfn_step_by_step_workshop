# Step 2: サブネットを追加する

> **目標**: Parameters で値を外部化し、組み込み関数を使ってリソースを定義する

## 学ぶこと

- `Parameters` セクション（テンプレートにパラメータを定義し、デプロイ時に値を渡す）
- `!Ref` — パラメータやリソースの値を参照する
- `!Select` + `!GetAZs` — アベイラビリティゾーンを動的に取得する
- 変更セット（Change Set）によるスタック更新

## Parameters とは

デプロイのたびに異なる値を使いたい場合、値をテンプレートにハードコードせず `Parameters` で外部化します。

```yaml
Parameters:
  パラメータ名:
    Type: String            # パラメータの型
    Default: "デフォルト値"  # デプロイ時に指定しなかった場合の値
    Description: "説明"
```

パラメータの値をリソースのプロパティで使うには **`!Ref パラメータ名`** と書きます。

## 組み込み関数: !Ref, !Select, !GetAZs

```yaml
# パラメータやリソースの値を参照する
!Ref VpcCidr           # → パラメータ VpcCidr の値（例: "10.0.0.0/16"）
!Ref VPC               # → リソース VPC の ID（例: "vpc-0abc123..."）

# リージョンの AZ 一覧から指定番目を取得する
!GetAZs ""             # → ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
!Select [0, !GetAZs ""]  # → "ap-northeast-1a"（0番目）
!Select [1, !GetAZs ""]  # → "ap-northeast-1c"（1番目）
```

## 作業

`templates/network/network-step2.yaml` を開いてください（Step 1 完成済みの状態）。

### 2-1. Parameters セクションを追加

`AWSTemplateFormatVersion` と `Resources` の間に `Parameters` セクションを追加します。

```yaml
Parameters:
  VpcCidr:
    Type: String
    Default: "10.0.0.0/16"
    Description: "VPC CIDR block"

  PublicSubnet1Cidr:
    Type: String
    Default: "10.0.1.0/24"
    Description: "Public subnet 1 CIDR block"

  PublicSubnet2Cidr:
    Type: String
    Default: "10.0.2.0/24"
    Description: "Public subnet 2 CIDR block"
```

### 2-2. VPC の CidrBlock をパラメータ参照に変更

VPC リソースの `CidrBlock` を、ハードコードされた値からパラメータ参照に変更します。

```yaml
# 変更前
CidrBlock: "10.0.0.0/16"

# 変更後
CidrBlock: !Ref VpcCidr
```

### 2-3. サブネットリソースを追加

**📖 リファレンス**: AWS ドキュメントで `AWS::EC2::Subnet` のページを開き、プロパティを確認してください。

TODO コメントの箇所に 2 つのサブネットを追加します。以下のプロパティを自分でリファレンスを見ながら指定してください。

- **`VpcId`** — どの VPC に作るか（`!Ref VPC` で参照）
- **`CidrBlock`** — サブネットのアドレス範囲（パラメータを `!Ref` で参照）
- **`AvailabilityZone`** — どの AZ に配置するか（`!Select` + `!GetAZs` で動的に取得）

```yaml
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: # ← !Ref で VPC を参照
      CidrBlock: # ← !Ref で PublicSubnet1Cidr を参照
      AvailabilityZone: # ← !Select と !GetAZs で 0 番目の AZ を取得
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-public-1a"

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: # ← !Ref で VPC を参照
      CidrBlock: # ← !Ref で PublicSubnet2Cidr を参照
      AvailabilityZone: # ← !Select と !GetAZs で 1 番目の AZ を取得
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-public-1c"
```

> **ポイント**
> - `MapPublicIpOnLaunch: true` にすると、このサブネットに起動した EC2 にパブリック IP が自動で付与されます
> - `!Sub "${AWS::StackName}-public-1a"` は `!Sub` でスタック名を動的に埋め込んでいます（次のステップで詳しく学びます）

## 変更セット（Change Set）とは

スタックを更新するとき、いきなり適用するのではなく**変更セット**を作成して「何がどう変わるか」を事前に確認できます。本番運用でも必ず行う安全な更新手順です。

```
テンプレート変更 → 変更セット作成 → 内容確認 → 変更セット実行 → スタック更新完了
```

変更セットには各リソースの変更内容が表示されます。

| アクション | 意味 |
|---|---|
| **Add** | 新規作成されるリソース |
| **Modify** | 設定が変更されるリソース |
| **Remove** | 削除されるリソース |

## デプロイ（変更セットによるスタック更新）

1. CloudFormation コンソールで **`cfn-workshop-network`** スタックを選択
2. **「スタックアクション」** → **「既存スタックの変更セットを作成」** をクリック
3. **「既存テンプレートを置き換える」** → 編集した `network-step2.yaml` をアップロード
4. パラメータはデフォルト値のまま **「次へ」** → **「次へ」** → **「送信」**
5. 変更セットのステータスが **`CREATE_COMPLETE`** になるまで待つ
6. **「変更」セクション** で、追加されるリソース（サブネット × 2）を確認する
7. 内容に問題なければ **「変更セットを実行」** をクリック

> **ポイント**
> - 変更セットは「プレビュー」です。実行するまでスタックには何も変更されません
> - この手順はこの後の全てのスタック更新で使います

## 確認

- スタックのステータスが **`UPDATE_COMPLETE`** になることを確認
- **「イベント」タブ** で、VPC は変更なし・サブネットが新規作成されたことを確認
- VPC コンソールの **「サブネット」** で 2 つのサブネットが異なる AZ に作成されていることを確認

---

[← Step 1: VPC を作成する](01-step1-vpc.md) | [Step 3: インターネット接続を構成する →](03-step3-igw-routes.md)
