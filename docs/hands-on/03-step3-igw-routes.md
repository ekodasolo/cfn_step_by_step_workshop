# Step 3: インターネット接続を構成する

> **目標**: IGW とルートテーブルを追加し、リソース間の依存関係を理解する

## 学ぶこと

- リソース間の依存関係（暗黙的依存 vs `DependsOn`）
- `!Sub` — 文字列の中に変数を埋め込む
- スタック更新でリソースを追加する（既存リソースには影響なし）

## 依存関係と DependsOn

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

## 組み込み関数: !Sub

`!Sub` は文字列の中に `${変数名}` で値を埋め込む関数です。

```yaml
# パラメータやリソースの値を埋め込む
Value: !Sub "${AWS::StackName}-igw"
# → "cfn-workshop-network-igw"（AWS::StackName は疑似パラメータ）
```

`!Ref` は単独の値しか返せませんが、`!Sub` なら文字列の中に組み込めます。

## 作業

`templates/network/network-step3.yaml` を開いてください（Step 2 完成済みの状態）。

TODO コメントの箇所に以下のリソースを追加します。

### 3-1. Internet Gateway

**📖 リファレンス**: `AWS::EC2::InternetGateway`

```yaml
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-igw"
```

### 3-2. IGW を VPC にアタッチ

**📖 リファレンス**: `AWS::EC2::VPCGatewayAttachment`

```yaml
  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
```

### 3-3. ルートテーブルとルート

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

### 3-4. サブネットとルートテーブルの関連付け

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

## デプロイ（変更セットによるスタック更新）

Step 2 と同じ手順で変更セットを作成・確認・実行します。

1. `cfn-workshop-network` スタック → **「スタックアクション」** → **「既存スタックの変更セットを作成」**
2. 編集した `network-step3.yaml` をアップロード → **「次へ」** → **「次へ」** → **「送信」**
3. 変更セットの **「変更」セクション** で、IGW・ルートテーブル等が **Add** になっていることを確認
4. **「変更セットを実行」**

## 確認

- **「イベント」タブ** で、既存の VPC やサブネットには変更が発生せず、新しいリソースだけが作成されたことを確認
- VPC コンソールの **「ルートテーブル」** で、デフォルトルート（`0.0.0.0/0` → IGW）が設定されていることを確認

---

[← Step 2: サブネットを追加する](02-step2-subnets.md) | [Step 4: セキュリティグループと Outputs →](04-step4-sg-outputs.md)
