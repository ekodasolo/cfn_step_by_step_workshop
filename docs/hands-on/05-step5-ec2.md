# Step 5: EC2 インスタンスを作成する

> **目標**: 2 つ目のスタックを作成し、クロススタック参照と Mappings を使う

## 学ぶこと

- `!ImportValue` — 他のスタックの Export 値を参照する（クロススタック参照）
- `Mappings` セクション — リージョンなどに応じた値のマッピングを定義する
- `!FindInMap` — Mappings から値を取得する
- `UserData` — EC2 起動時にスクリプトを実行する

## クロススタック参照: !ImportValue

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

## Mappings と !FindInMap

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

## 作業

`templates/application/application-step5.yaml` を開いてください。

### 5-1. Parameters を追加

```yaml
Parameters:
  NetworkStackName:
    Type: String
    Description: "Name of the network stack (for cross-stack references)"
```

### 5-2. Mappings を追加

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

### 5-3. IAM ロールとインスタンスプロファイルを追加

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

### 5-4. EC2 インスタンスを追加

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

### 5-5. DummyWaitHandle を削除

元のテンプレートにあった `DummyWaitHandle` リソースは不要になったので削除してください。

### 5-6. Outputs を追加

```yaml
Outputs:
  InstanceId:
    Description: "EC2 instance ID"
    Value: !Ref WebServerInstance
```

## デプロイ（新しいスタック作成）

1. CloudFormation コンソールで **「スタックの作成」** をクリック
2. 編集した `application-step5.yaml` をアップロード
3. スタック名: **`cfn-workshop-app`**
4. パラメータ `NetworkStackName` に **`cfn-workshop-network`** と入力
5. **「AWS CloudFormation によって IAM リソースが作成される場合があることを承認します」** にチェックを入れる
6. **「送信」**

## 確認

- スタックのステータスが **`CREATE_COMPLETE`** になることを確認
- EC2 コンソールで Web サーバーインスタンスが起動していることを確認
- **SSM Session Manager** で EC2 に接続し、`curl localhost` でページが返ることを確認

---

[← Step 4: セキュリティグループと Outputs](04-step4-sg-outputs.md) | [Step 6: ALB を追加する →](06-step6-alb.md)
