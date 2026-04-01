# Step 4: セキュリティグループと Outputs

> **目標**: セキュリティグループを追加し、Outputs でクロススタック参照の準備をする

## 学ぶこと

- `Outputs` セクション — スタック外に値を公開する
- `Export` — 他のスタックから参照できるようにする
- `!GetAtt` — リソースの属性を取得する
- セキュリティグループの Ingress 設計

## Outputs と Export

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

## 組み込み関数: !GetAtt

`!Ref` がリソースの ID を返すのに対し、`!GetAtt` は ID 以外の属性を取得できます。

```yaml
!Ref ALBSecurityGroup                    # → "sg-0abc123..."（セキュリティグループ ID）
!GetAtt ALBSecurityGroup.GroupId         # → "sg-0abc123..."（GroupId 属性）
```

> `!Ref` で返る値はリソースタイプによって異なります。セキュリティグループの場合は `!Ref` でも ID が返りますが、`!GetAtt` を使うと明示的に属性名を指定できるため、意図が明確になります。

## 作業

`templates/network/network-step4.yaml` を開いてください（Step 3 完成済みの状態）。

### 4-1. セキュリティグループを追加

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

### 4-2. Outputs セクションを追加

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

## デプロイ（変更セットによるスタック更新）

Step 2, 3 と同じ手順で変更セットを作成・確認・実行します。

1. `cfn-workshop-network` スタック → **「スタックアクション」** → **「既存スタックの変更セットを作成」**
2. 編集した `network-step4.yaml` をアップロード → **「次へ」** → **「次へ」** → **「送信」**
3. 変更セットで SG × 2 が **Add** になっていることを確認
4. **「変更セットを実行」**

## 確認

- CloudFormation コンソールの **「出力」タブ** で、5 つの値がエクスポートされていることを確認
- VPC コンソールの **「セキュリティグループ」** で、2 つの SG が作成されていることを確認
- EC2 用 SG のインバウンドルールが、ALB の SG を参照していることを確認

> **ここまでのまとめ**: ネットワークスタックが完成しました。VPC + サブネット + IGW + ルートテーブル + SG が揃い、Outputs で値をエクスポートしています。次はこれらを参照してアプリケーションスタックを作ります。

---

[← Step 3: インターネット接続を構成する](03-step3-igw-routes.md) | [Step 5: EC2 インスタンスを作成する →](05-step5-ec2.md)
