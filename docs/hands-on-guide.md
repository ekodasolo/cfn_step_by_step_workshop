# CloudFormation ハンズオン 手順書

## ワークショップ概要

最小構成のテンプレートから段階的にリソースを追加し、**テンプレートを育てながら CloudFormation を学ぶ**ハンズオンです。

### 完成形アーキテクチャ

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

これを **ネットワークスタック** と **アプリケーションスタック** の 2 スタックに分けて作ります。

### 進め方

- テンプレートファイル（YAML）をテキストエディタで編集します
- 各ステップでは **TODO コメント** がある箇所にリソースやプロパティを追記します
- プロパティの書き方は **AWS 公式 CloudFormation ドキュメント** のリソースリファレンスを参照してください
- 編集が終わったらマネジメントコンソールからスタックを作成 / 更新します

### 前提

- AWS マネジメントコンソールにログイン済み
- リージョン: **ap-northeast-1（東京）**
- テキストエディタ（VS Code 推奨）が使える状態

---

# Part 1: ネットワークスタック

---

## Step 1: VPC を作成する

> **目標**: テンプレートの基本構造を理解し、最初のスタックを作成する

### 学ぶこと

- テンプレートの基本構造（`AWSTemplateFormatVersion`, `Description`, `Resources`）
- リソースの書き方（`Type`, `Properties`）
- マネジメントコンソールからのスタック作成
- スタックの状態遷移（`CREATE_IN_PROGRESS` → `CREATE_COMPLETE`）

### CloudFormation テンプレートの基本構造

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

### 作業

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

### デプロイ

1. マネジメントコンソールで **CloudFormation** を開く
2. **「スタックの作成」** → **「新しいリソースを使用（標準）」** をクリック
3. **「テンプレートファイルのアップロード」** を選択し、編集した `network-step1.yaml` をアップロード
4. スタック名: **`cfn-workshop-network`** と入力
5. 他の項目はデフォルトのまま **「次へ」** → **「次へ」** → **「送信」**

### 確認

- CloudFormation コンソールでスタックのステータスが **`CREATE_COMPLETE`** になることを確認
- **「イベント」タブ** でリソース作成の流れを確認
- **VPC コンソール** に移動して、作成された VPC を確認

---

## Step 2: サブネットを追加する

> **目標**: Parameters で値を外部化し、組み込み関数を使ってリソースを定義する

### 学ぶこと

- `Parameters` セクション（テンプレートにパラメータを定義し、デプロイ時に値を渡す）
- `!Ref` — パラメータやリソースの値を参照する
- `!Select` + `!GetAZs` — アベイラビリティゾーンを動的に取得する
- スタック更新の実行

### Parameters とは

デプロイのたびに異なる値を使いたい場合、値をテンプレートにハードコードせず `Parameters` で外部化します。

```yaml
Parameters:
  パラメータ名:
    Type: String            # パラメータの型
    Default: "デフォルト値"  # デプロイ時に指定しなかった場合の値
    Description: "説明"
```

パラメータの値をリソースのプロパティで使うには **`!Ref パラメータ名`** と書きます。

### 組み込み関数: !Ref, !Select, !GetAZs

```yaml
# パラメータやリソースの値を参照する
!Ref VpcCidr           # → パラメータ VpcCidr の値（例: "10.0.0.0/16"）
!Ref VPC               # → リソース VPC の ID（例: "vpc-0abc123..."）

# リージョンの AZ 一覧から指定番目を取得する
!GetAZs ""             # → ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
!Select [0, !GetAZs ""]  # → "ap-northeast-1a"（0番目）
!Select [1, !GetAZs ""]  # → "ap-northeast-1c"（1番目）
```

### 作業

`templates/network/network-step2.yaml` を開いてください（Step 1 完成済みの状態）。

#### 2-1. Parameters セクションを追加

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

#### 2-2. VPC の CidrBlock をパラメータ参照に変更

VPC リソースの `CidrBlock` を、ハードコードされた値からパラメータ参照に変更します。

```yaml
# 変更前
CidrBlock: "10.0.0.0/16"

# 変更後
CidrBlock: !Ref VpcCidr
```

#### 2-3. サブネットリソースを追加

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

### デプロイ（スタック更新）

1. CloudFormation コンソールで **`cfn-workshop-network`** スタックを選択
2. **「更新」** をクリック
3. **「既存テンプレートを置き換える」** → 編集した `network-step2.yaml` をアップロード
4. パラメータはデフォルト値のまま **「次へ」** → **「次へ」** → **「送信」**

### 確認

- スタックのステータスが **`UPDATE_COMPLETE`** になることを確認
- **「イベント」タブ** で、VPC は変更なし・サブネットが新規作成されたことを確認
- VPC コンソールの **「サブネット」** で 2 つのサブネットが異なる AZ に作成されていることを確認

---

## Step 3: インターネット接続を構成する

> **目標**: IGW とルートテーブルを追加し、リソース間の依存関係を理解する

### 学ぶこと

- リソース間の依存関係（暗黙的依存 vs `DependsOn`）
- `!Sub` — 文字列の中に変数を埋め込む
- スタック更新でリソースを追加する（既存リソースには影響なし）

### 依存関係と DependsOn

CloudFormation は `!Ref` や `!GetAtt` で参照されたリソースを自動的に先に作成します（**暗黙的依存**）。

しかし、明示的な参照がないのに作成順序を制御したい場合は `DependsOn` を使います。

```yaml
PublicRoute:
  Type: AWS::EC2::Route
  DependsOn: VPCGatewayAttachment    # ← IGW のアタッチが完了してからルートを作る
  Properties:
    RouteTableId: !Ref PublicRouteTable
    DestinationCidrBlock: "0.0.0.0/0"
    GatewayId: !Ref InternetGateway
```

> この例では `GatewayId: !Ref InternetGateway` で IGW は参照していますが、IGW が VPC に**アタッチ**されていないとルートは作れません。アタッチを行う `VPCGatewayAttachment` との間に直接の参照がないため、`DependsOn` で順序を保証します。

### 組み込み関数: !Sub

`!Sub` は文字列の中に `${変数名}` で値を埋め込む関数です。

```yaml
# パラメータやリソースの値を埋め込む
Value: !Sub "${AWS::StackName}-igw"
# → "cfn-workshop-network-igw"（AWS::StackName は疑似パラメータ）
```

`!Ref` は単独の値しか返せませんが、`!Sub` なら文字列の中に組み込めます。

### 作業

`templates/network/network-step3.yaml` を開いてください（Step 2 完成済みの状態）。

TODO コメントの箇所に以下のリソースを追加します。

#### 3-1. Internet Gateway

**📖 リファレンス**: `AWS::EC2::InternetGateway`

```yaml
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-igw"
```

#### 3-2. IGW を VPC にアタッチ

**📖 リファレンス**: `AWS::EC2::VPCGatewayAttachment`

```yaml
  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
```

#### 3-3. ルートテーブルとルート

**📖 リファレンス**: `AWS::EC2::RouteTable`, `AWS::EC2::Route`

ルートテーブルを作り、インターネット向けのデフォルトルートを追加します。

以下のプロパティを自分でリファレンスを見ながら指定してください。

- Route の **`DestinationCidrBlock`** — どの宛先へのルートか（`"0.0.0.0/0"` = 全トラフィック）
- Route の **`GatewayId`** — トラフィックの転送先（`!Ref InternetGateway`）

```yaml
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-public-rt"

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: # ← 全トラフィック宛のルート
      GatewayId: # ← IGW を参照
```

#### 3-4. サブネットとルートテーブルの関連付け

**📖 リファレンス**: `AWS::EC2::SubnetRouteTableAssociation`

```yaml
  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable
```

### デプロイ（スタック更新）

Step 2 と同じ手順でスタックを更新します。

### 確認

- **「イベント」タブ** で、既存の VPC やサブネットには変更が発生せず、新しいリソースだけが作成されたことを確認
- VPC コンソールの **「ルートテーブル」** で、デフォルトルート（`0.0.0.0/0` → IGW）が設定されていることを確認

---

## Step 4: セキュリティグループと Outputs

> **目標**: セキュリティグループを追加し、Outputs でクロススタック参照の準備をする

### 学ぶこと

- `Outputs` セクション — スタック外に値を公開する
- `Export` — 他のスタックから参照できるようにする
- `!GetAtt` — リソースの属性を取得する
- セキュリティグループの Ingress 設計

### Outputs と Export

`Outputs` はスタックの作成結果を外部に公開する仕組みです。`Export` を付けると他のスタックから `!ImportValue` で参照できます。

```yaml
Outputs:
  VpcId:
    Description: "VPC ID"
    Value: !Ref VPC
    Export:
      Name: !Sub "${AWS::StackName}-VpcId"
      # → "cfn-workshop-network-VpcId" という名前でエクスポート
```

### 組み込み関数: !GetAtt

`!Ref` がリソースの ID を返すのに対し、`!GetAtt` は ID 以外の属性を取得できます。

```yaml
!Ref ALBSecurityGroup                    # → "sg-0abc123..."（セキュリティグループ ID）
!GetAtt ALBSecurityGroup.GroupId         # → "sg-0abc123..."（GroupId 属性）
```

> `!Ref` で返る値はリソースタイプによって異なります。セキュリティグループの場合は `!Ref` でも ID が返りますが、`!GetAtt` を使うと明示的に属性名を指定できるため、意図が明確になります。

### 作業

`templates/network/network-step4.yaml` を開いてください（Step 3 完成済みの状態）。

#### 4-1. セキュリティグループを追加

**📖 リファレンス**: `AWS::EC2::SecurityGroup`

2 つのセキュリティグループを作成します。

- **ALB 用**: インターネットからの HTTP（ポート 80）を許可
- **EC2 用**: ALB からの HTTP（ポート 80）のみ許可

以下のプロパティを自分でリファレンスを見ながら指定してください。

- `SecurityGroupIngress` のルール定義（`IpProtocol`, `FromPort`, `ToPort`）
- EC2 用 SG の `SourceSecurityGroupId`（ALB の SG を `!GetAtt` で参照）

```yaml
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Security group for ALB - allows HTTP from internet"
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: # ← プロトコル
          FromPort: # ← 開始ポート
          ToPort: # ← 終了ポート
          CidrIp: "0.0.0.0/0"
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-alb-sg"

  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Security group for EC2 - allows HTTP from ALB only"
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: # ← プロトコル
          FromPort: # ← 開始ポート
          ToPort: # ← 終了ポート
          SourceSecurityGroupId: # ← ALB の SG を !GetAtt で参照
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web-sg"
```

> **ポイント**
> - ALB 用 SG は `CidrIp: "0.0.0.0/0"` でインターネット全体からのアクセスを許可
> - EC2 用 SG は `SourceSecurityGroupId` で **ALB の SG からのみ** アクセスを許可（CIDR ではなく SG 間参照）
> - これにより EC2 に直接アクセスはできず、必ず ALB を経由する構成になります

#### 4-2. Outputs セクションを追加

テンプレートの末尾に `Outputs` セクションを追加します。アプリケーションスタックから参照する 5 つの値をエクスポートします。

```yaml
Outputs:
  VpcId:
    Description: "VPC ID"
    Value: !Ref VPC
    Export:
      Name: !Sub "${AWS::StackName}-VpcId"

  PublicSubnet1Id:
    Description: "Public subnet 1 ID"
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub "${AWS::StackName}-PublicSubnet1Id"

  PublicSubnet2Id:
    Description: "Public subnet 2 ID"
    Value: !Ref PublicSubnet2
    Export:
      Name: !Sub "${AWS::StackName}-PublicSubnet2Id"

  ALBSecurityGroupId:
    Description: "ALB security group ID"
    Value: !GetAtt ALBSecurityGroup.GroupId
    Export:
      Name: !Sub "${AWS::StackName}-ALBSecurityGroupId"

  WebServerSecurityGroupId:
    Description: "Web server security group ID"
    Value: !GetAtt WebServerSecurityGroup.GroupId
    Export:
      Name: !Sub "${AWS::StackName}-WebServerSecurityGroupId"
```

### デプロイ（スタック更新）

Step 2, 3 と同じ手順でスタックを更新します。

### 確認

- CloudFormation コンソールの **「出力」タブ** で、5 つの値がエクスポートされていることを確認
- VPC コンソールの **「セキュリティグループ」** で、2 つの SG が作成されていることを確認
- EC2 用 SG のインバウンドルールが、ALB の SG を参照していることを確認

> **ここまでのまとめ**: ネットワークスタックが完成しました。VPC + サブネット + IGW + ルートテーブル + SG が揃い、Outputs で値をエクスポートしています。次はこれらを参照してアプリケーションスタックを作ります。

---

# Part 2: アプリケーションスタック

---

## Step 5: EC2 インスタンスを作成する

> **目標**: 2 つ目のスタックを作成し、クロススタック参照と Mappings を使う

### 学ぶこと

- `!ImportValue` — 他のスタックの Export 値を参照する（クロススタック参照）
- `Mappings` セクション — リージョンなどに応じた値のマッピングを定義する
- `!FindInMap` — Mappings から値を取得する
- `UserData` — EC2 起動時にスクリプトを実行する

### クロススタック参照: !ImportValue

Step 4 でネットワークスタックから `Export` した値を、アプリケーションスタックで `!ImportValue` を使って参照します。

```yaml
# ネットワークスタック（Export 側）
Outputs:
  VpcId:
    Value: !Ref VPC
    Export:
      Name: !Sub "${AWS::StackName}-VpcId"     # → "cfn-workshop-network-VpcId"

# アプリケーションスタック（Import 側）
VpcId: !ImportValue
  Fn::Sub: "${NetworkStackName}-VpcId"          # → "cfn-workshop-network-VpcId"
```

> `!ImportValue` の中で `!Sub` を使う場合、短縮構文を入れ子にできないため `Fn::Sub` と正式構文で書きます。

### Mappings と !FindInMap

リージョンごとに異なる AMI ID を管理するために `Mappings` を使います。

```yaml
Mappings:
  RegionAMI:
    ap-northeast-1:
      AMI: ami-0599b6e53ca798bb2
    us-east-1:
      AMI: ami-0953476d60561c955

# 使用例: 現在のリージョンの AMI ID を取得
ImageId: !FindInMap
  - RegionAMI            # マッピング名
  - !Ref "AWS::Region"   # 第1キー（リージョン）
  - AMI                  # 第2キー
```

### 作業

`templates/application/application-step5.yaml` を開いてください。

#### 5-1. Parameters を追加

```yaml
Parameters:
  NetworkStackName:
    Type: String
    Description: "Name of the network stack (for cross-stack references)"
```

#### 5-2. Mappings を追加

**📖 リファレンス**: CloudFormation ドキュメントの「Mappings」セクション

```yaml
Mappings:
  RegionAMI:
    ap-northeast-1:
      AMI: ami-0599b6e53ca798bb2
    ap-northeast-3:
      AMI: ami-0a897c44e83f91dfc
    us-east-1:
      AMI: ami-0953476d60561c955
```

#### 5-3. IAM ロールとインスタンスプロファイルを追加

SSM Session Manager で EC2 に接続するために、IAM ロールが必要です。

```yaml
  WebServerRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "${AWS::StackName}-web-server-role"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: "sts:AssumeRole"
      ManagedPolicyArns:
        - "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web-server-role"

  WebServerInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref WebServerRole
```

#### 5-4. EC2 インスタンスを追加

**📖 リファレンス**: `AWS::EC2::Instance`

以下のプロパティを自分でリファレンスを見ながら指定してください。

- **`ImageId`** — `!FindInMap` で Mappings から AMI ID を取得
- **`SubnetId`** — `!ImportValue` でネットワークスタックのサブネットを参照
- **`SecurityGroupIds`** — `!ImportValue` でネットワークスタックの SG を参照

```yaml
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: # ← !FindInMap で RegionAMI から取得
      InstanceType: "t2.micro"
      IamInstanceProfile: !Ref WebServerInstanceProfile
      SubnetId: # ← !ImportValue でネットワークスタックの PublicSubnet1Id を参照
      SecurityGroupIds:
        - # ← !ImportValue でネットワークスタックの WebServerSecurityGroupId を参照
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web-server"
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          set -euo pipefail
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd

          TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
            -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
          INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
            http://169.254.169.254/latest/meta-data/instance-id)
          PRIVATE_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
            http://169.254.169.254/latest/meta-data/local-ipv4)

          cat > /var/www/html/index.html <<HTMLEOF
          <html><body>
          <h1>CloudFormation Workshop</h1>
          <p>Instance ID: ${!INSTANCE_ID}</p>
          <p>Private IP: ${!PRIVATE_IP}</p>
          </body></html>
          HTMLEOF
```

> **ポイント**
> - `UserData` 内の `${!INSTANCE_ID}` は CloudFormation の `!Sub` ではなくシェル変数として展開されます。`${!変数名}` と書くことで `!Sub` のエスケープになります
> - 上記は簡易版です。完成形テンプレートではカレンダー表示付きのリッチなページを生成します

#### 5-5. DummyWaitHandle を削除

元のテンプレートにあった `DummyWaitHandle` リソースは不要になったので削除してください。

#### 5-6. Outputs を追加

```yaml
Outputs:
  InstanceId:
    Description: "EC2 instance ID"
    Value: !Ref WebServerInstance
```

### デプロイ（新しいスタック作成）

1. CloudFormation コンソールで **「スタックの作成」** をクリック
2. 編集した `application-step5.yaml` をアップロード
3. スタック名: **`cfn-workshop-app`**
4. パラメータ `NetworkStackName` に **`cfn-workshop-network`** と入力
5. **「AWS CloudFormation によって IAM リソースが作成される場合があることを承認します」** にチェックを入れる
6. **「送信」**

### 確認

- スタックのステータスが **`CREATE_COMPLETE`** になることを確認
- EC2 コンソールで Web サーバーインスタンスが起動していることを確認
- **SSM Session Manager** で EC2 に接続し、`curl localhost` でページが返ることを確認

---

## Step 6: ALB を追加する

> **目標**: ALB を追加し、ロードバランサー経由でアクセスできるようにする

### 学ぶこと

- ALB の構成要素（ALB → Listener → TargetGroup → Targets）
- スタック更新でリソースを追加する
- ALB の DNS 名で動作確認する

### ALB の構成

ALB は 3 つのリソースで構成されます。

```
ブラウザ → [ALB] → [Listener :80] → [TargetGroup] → [EC2 インスタンス]
```

- **ALB (LoadBalancer)**: ロードバランサー本体。サブネットと SG を指定
- **Listener**: ALB がリッスンするポートとプロトコル。受け取ったリクエストの転送先を指定
- **TargetGroup**: リクエストの転送先グループ。ヘルスチェックの設定も含む

### 作業

`templates/application/application-step6.yaml` を開いてください（Step 5 完成済みの状態）。

TODO コメントの箇所に ALB 関連のリソースを追加します。

#### 6-1. ALB

**📖 リファレンス**: `AWS::ElasticLoadBalancingV2::LoadBalancer`

```yaml
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub "${AWS::StackName}-alb"
      Scheme: internet-facing
      Type: application
      Subnets:
        - !ImportValue
            Fn::Sub: "${NetworkStackName}-PublicSubnet1Id"
        - !ImportValue
            Fn::Sub: "${NetworkStackName}-PublicSubnet2Id"
      SecurityGroups:
        - !ImportValue
            Fn::Sub: "${NetworkStackName}-ALBSecurityGroupId"
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-alb"
```

#### 6-2. ターゲットグループ

**📖 リファレンス**: `AWS::ElasticLoadBalancingV2::TargetGroup`

以下のプロパティを自分でリファレンスを見ながら指定してください。

- **`HealthCheckPath`** — ヘルスチェックで叩くパス
- **`HealthCheckProtocol`** — ヘルスチェックのプロトコル

```yaml
  WebServerTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub "${AWS::StackName}-tg"
      Protocol: HTTP
      Port: 80
      VpcId: !ImportValue
        Fn::Sub: "${NetworkStackName}-VpcId"
      TargetType: instance
      Targets:
        - Id: !Ref WebServerInstance
      HealthCheckProtocol: # ← ヘルスチェックのプロトコル
      HealthCheckPath: # ← ヘルスチェックのパス
      HealthCheckIntervalSeconds: 30
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 5
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-tg"
```

#### 6-3. リスナー

**📖 リファレンス**: `AWS::ElasticLoadBalancingV2::Listener`

以下のプロパティを自分でリファレンスを見ながら指定してください。

- **`DefaultActions`** — リクエストをどう処理するか（`Type: forward` でターゲットグループに転送）

```yaml
  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Protocol: HTTP
      Port: 80
      DefaultActions: # ← Type: forward で TargetGroup に転送する設定を書く
```

#### 6-4. Outputs を更新

`Outputs` に ALB の DNS 名を追加します。

```yaml
Outputs:
  ALBDnsName:
    Description: "ALB DNS name - access the application via this URL"
    Value: !Sub "http://${ApplicationLoadBalancer.DNSName}"
```

### デプロイ（スタック更新）

Step 2〜4 と同じ手順でアプリケーションスタックを更新します。

### 確認

- CloudFormation コンソールの **「出力」タブ** で ALB の DNS 名を確認
- ブラウザで ALB の DNS 名にアクセスし、カレンダーページが表示されることを確認
- 何度かリロードしても同じインスタンスの情報が表示される（まだ 1 台しかないため）

---

## Step 7: Launch Template + Auto Scaling + Conditions

> **目標**: EC2 直接定義を Launch Template + ASG に置き換え、Conditions で環境切り替えを実現する

### 学ぶこと

- `Conditions` — パラメータの値に応じて設定を切り替える
- `!If` — Conditions の結果に基づいて値を選択する
- Launch Template と Auto Scaling Group
- `UpdatePolicy` — ASG のローリング更新制御
- リソースの置き換え（Replace）が発生するスタック更新

### Conditions と !If

環境（本番/開発）によってインスタンスサイズを変えたい場合などに使います。

```yaml
Parameters:
  Environment:
    Type: String
    Default: "dev"
    AllowedValues: ["dev", "prod"]

Conditions:
  IsProd: !Equals [!Ref Environment, "prod"]  # Environment が "prod" なら true

Resources:
  # ...
    InstanceType: !If
      - IsProd          # 条件名
      - "t2.small"      # true の場合
      - "t2.micro"      # false の場合
```

### 作業

`templates/application/application-step7.yaml` を開いてください（Step 6 完成済みの状態）。

このステップでは既存のリソースを**置き換える**大きな変更を行います。

#### 7-1. Environment パラメータを追加

`Parameters` セクションに `Environment` パラメータを追加します。

```yaml
  Environment:
    Type: String
    Default: "dev"
    AllowedValues:
      - "dev"
      - "prod"
    Description: "Environment type (dev or prod)"
```

#### 7-2. Conditions セクションを追加

`Parameters` と `Resources` の間に `Conditions` セクションを追加します。

```yaml
Conditions:
  IsProd: !Equals
    - !Ref Environment
    - "prod"
```

#### 7-3. WebServerInstance を削除

EC2 インスタンスの直接定義（`WebServerInstance` リソース全体）を削除します。

#### 7-4. Launch Template を追加

**📖 リファレンス**: `AWS::EC2::LaunchTemplate`

EC2 の設定を Launch Template にまとめます。`!If` でインスタンスタイプを環境ごとに切り替えます。

```yaml
  WebServerLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Sub "${AWS::StackName}-launch-template"
      LaunchTemplateData:
        ImageId: !FindInMap
          - RegionAMI
          - !Ref "AWS::Region"
          - AMI
        InstanceType: !If
          - IsProd
          - "t2.small"
          - "t2.micro"
        IamInstanceProfile:
          Arn: !GetAtt WebServerInstanceProfile.Arn
        SecurityGroupIds:
          - !ImportValue
              Fn::Sub: "${NetworkStackName}-WebServerSecurityGroupId"
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: !Sub "${AWS::StackName}-web-server"
        UserData:
          # （Step 6 の WebServerInstance にあった UserData をそのまま移動）
```

> UserData の内容は Step 6 の EC2 インスタンスと同じものをそのまま使います。

#### 7-5. Auto Scaling Group を追加

**📖 リファレンス**: `AWS::AutoScaling::AutoScalingGroup`

以下のプロパティを自分でリファレンスを見ながら指定してください。

- **`MinSize`**, **`MaxSize`**, **`DesiredCapacity`** — `!If` で環境ごとに台数を変える
- **`UpdatePolicy`** の `AutoScalingRollingUpdate` — ローリング更新の設定

```yaml
  WebServerASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: !Sub "${AWS::StackName}-asg"
      LaunchTemplate:
        LaunchTemplateId: !Ref WebServerLaunchTemplate
        Version: !GetAtt WebServerLaunchTemplate.LatestVersionNumber
      MinSize: # ← !If で prod:2, dev:1
      MaxSize: # ← !If で prod:4, dev:2
      DesiredCapacity: # ← !If で prod:2, dev:1
      VPCZoneIdentifier:
        - !ImportValue
            Fn::Sub: "${NetworkStackName}-PublicSubnet1Id"
        - !ImportValue
            Fn::Sub: "${NetworkStackName}-PublicSubnet2Id"
      TargetGroupARNs:
        - !Ref WebServerTargetGroup
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web-server"
          PropagateAtLaunch: true
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: # ← 更新中に維持する最小台数
        MaxBatchSize: # ← 同時に更新する最大台数
        PauseTime: PT5M
        WaitOnResourceSignals: false
```

> **ポイント**
> - `VPCZoneIdentifier` に 2 つのサブネットを指定することで、ASG がインスタンスを 2 つの AZ に分散配置します
> - `TargetGroupARNs` を指定すると、ASG が新しいインスタンスを自動でターゲットグループに登録します
> - `UpdatePolicy` はスタック更新時の ASG の振る舞いを制御します（一度に何台まで入れ替えるか等）

#### 7-6. TargetGroup の Targets を削除

Step 6 では `Targets` プロパティで EC2 を直接登録していましたが、ASG が自動登録するため不要になります。

```yaml
# TargetGroup から以下を削除
      Targets:
        - Id: !Ref WebServerInstance
```

### デプロイ（スタック更新）

1. スタックを更新（`application-step7.yaml` をアップロード）
2. パラメータ入力画面で `Environment` に **`dev`** を指定
3. **「AWS CloudFormation によって IAM リソースが作成される場合があることを承認します」** にチェック
4. **「送信」**

> このステップでは EC2 インスタンスの置き換え（Replace）が発生します。イベントタブで古い EC2 が削除され、ASG 経由で新しい EC2 が起動する様子を観察してください。

### 確認

- EC2 コンソールで、Auto Scaling Group が作成されていることを確認
- ALB の DNS 名でアクセスし、ページが表示されることを確認
- （オプション）EC2 コンソールで ASG の **「希望するキャパシティ」** を `2` に変更し、インスタンスが増えることを確認
- ブラウザで何度かリロードし、Instance ID / Private IP が切り替わることを確認（ロードバランシングの実感）

---

# クリーンアップ

ワークショップで作成したリソースを削除します。**アプリケーションスタックから先に削除**してください。

> ネットワークスタックの Export 値をアプリケーションスタックが参照しているため、先にネットワークスタックを削除しようとするとエラーになります。

1. CloudFormation コンソールで **`cfn-workshop-app`** を選択 → **「削除」**
2. `DELETE_COMPLETE` を確認
3. **`cfn-workshop-network`** を選択 → **「削除」**
4. `DELETE_COMPLETE` を確認

> スタック間に依存関係がある場合、参照される側（ネットワーク）は参照する側（アプリケーション）を先に削除しないと削除できません。これも CloudFormation の重要な運用知識です。

---

# 学習のふりかえり

このワークショップで学んだ CloudFormation の要素をまとめます。

### テンプレートのセクション

| セクション | 役割 | 学んだステップ |
|---|---|---|
| `Resources` | 作成するリソースを定義（必須） | Step 1 |
| `Parameters` | デプロイ時に外部から値を渡す | Step 2 |
| `Outputs` / `Export` | スタック外に値を公開・エクスポート | Step 4 |
| `Mappings` | リージョン等に応じた値のマッピング | Step 5 |
| `Conditions` | パラメータに応じた条件分岐 | Step 7 |

### 組み込み関数（短縮構文）

| 関数 | 役割 | 学んだステップ |
|---|---|---|
| `!Ref` | パラメータ / リソースの値を参照 | Step 2 |
| `!Select` + `!GetAZs` | AZ を動的に取得 | Step 2 |
| `!Sub` | 文字列に変数を埋め込む | Step 3 |
| `!GetAtt` | リソースの属性を取得 | Step 4 |
| `!ImportValue` | 他のスタックの Export 値を参照 | Step 5 |
| `!FindInMap` | Mappings から値を取得 | Step 5 |
| `!If` | Conditions の結果で値を切り替え | Step 7 |

### スタック操作

| 操作 | 体験したステップ |
|---|---|
| スタック作成 (Create) | Step 1, Step 5 |
| スタック更新 (Update) | Step 2〜4, Step 6〜7 |
| クロススタック参照 | Step 4〜5 |
| リソース置き換え | Step 7 |
| スタック削除（依存順序） | クリーンアップ |
