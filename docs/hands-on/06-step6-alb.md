# Step 6: ALB を追加する

> **目標**: ALB を追加し、ロードバランサー経由でアクセスできるようにする

## 学ぶこと

- ALB の構成要素（ALB → Listener → TargetGroup → Targets）
- スタック更新でリソースを追加する
- ALB の DNS 名で動作確認する

## ALB の構成

ALB は 3 つのリソースで構成されます。

```
ブラウザ → [ALB] → [Listener :80] → [TargetGroup] → [EC2 インスタンス]
```

- **ALB (LoadBalancer)**: ロードバランサー本体。サブネットと SG を指定
- **Listener**: ALB がリッスンするポートとプロトコル。受け取ったリクエストの転送先を指定
- **TargetGroup**: リクエストの転送先グループ。ヘルスチェックの設定も含む

## 作業

`templates/application/application-step6.yaml` を開いてください（Step 5 完成済みの状態）。

TODO コメントの箇所に ALB 関連のリソースを追加します。

### 6-1. ALB

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

### 6-2. ターゲットグループ

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

### 6-3. リスナー

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

### 6-4. Outputs を更新

`Outputs` に ALB の DNS 名を追加します。

```yaml
Outputs:
  ALBDnsName:
    Description: "ALB DNS name - access the application via this URL"
    Value: !Sub "http://${ApplicationLoadBalancer.DNSName}"
```

## デプロイ（変更セットによるスタック更新）

1. `cfn-workshop-app` スタック → **「スタックアクション」** → **「既存スタックの変更セットを作成」**
2. 編集した `application-step6.yaml` をアップロード → **「次へ」** → **「次へ」** → **「送信」**
3. 変更セットで ALB・TargetGroup・Listener が **Add** になっていることを確認
4. **「変更セットを実行」**

## 確認

- CloudFormation コンソールの **「出力」タブ** で ALB の DNS 名を確認
- ブラウザで ALB の DNS 名にアクセスし、カレンダーページが表示されることを確認
- 何度かリロードしても同じインスタンスの情報が表示される（まだ 1 台しかないため）

---

[← Step 5: EC2 インスタンスを作成する](05-step5-ec2.md) | [Step 7: Launch Template + Auto Scaling + Conditions →](07-step7-asg-conditions.md)
