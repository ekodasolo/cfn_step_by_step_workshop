# Step 7: Launch Template + Auto Scaling + Conditions

> **目標**: EC2 直接定義を Launch Template + ASG に置き換え、Conditions で環境切り替えを実現する

## 学ぶこと

- `Conditions` — パラメータの値に応じて設定を切り替える
- `!If` — Conditions の結果に基づいて値を選択する
- Launch Template と Auto Scaling Group
- `UpdatePolicy` — ASG のローリング更新制御
- リソースの置き換え（Replace）が発生するスタック更新

## Conditions と !If

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

## 作業

`templates/application/application-step7.yaml` を開いてください（Step 6 完成済みの状態）。

このステップでは既存のリソースを**置き換える**大きな変更を行います。

### 7-1. Environment パラメータを追加

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

### 7-2. Conditions セクションを追加

`Parameters` と `Resources` の間に `Conditions` セクションを追加します。

```yaml
Conditions:
  IsProd: !Equals
    - !Ref Environment
    - "prod"
```

### 7-3. WebServerInstance を削除

EC2 インスタンスの直接定義（`WebServerInstance` リソース全体）を削除します。

### 7-4. Launch Template を追加

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

### 7-5. Auto Scaling Group を追加

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

### 7-6. TargetGroup の Targets を削除

Step 6 では `Targets` プロパティで EC2 を直接登録していましたが、ASG が自動登録するため不要になります。

```yaml
# TargetGroup から以下を削除
      Targets:
        - Id: !Ref WebServerInstance
```

## デプロイ（変更セットによるスタック更新）

1. `cfn-workshop-app` スタック → **「スタックアクション」** → **「既存スタックの変更セットを作成」**
2. 編集した `application-step7.yaml` をアップロード
3. パラメータ入力画面で `Environment` に **`dev`** を指定 → **「次へ」**
4. **「AWS CloudFormation によって IAM リソースが作成される場合があることを承認します」** にチェック → **「送信」**
5. 変更セットの **「変更」セクション** を確認 — WebServerInstance が **Remove**、LaunchTemplate・ASG が **Add** になっているはずです
6. **「変更セットを実行」**

> このステップでは EC2 インスタンスの置き換え（Replace）が発生します。イベントタブで古い EC2 が削除され、ASG 経由で新しい EC2 が起動する様子を観察してください。

## 確認

- EC2 コンソールで、Auto Scaling Group が作成されていることを確認
- ALB の DNS 名でアクセスし、ページが表示されることを確認
- （オプション）EC2 コンソールで ASG の **「希望するキャパシティ」** を `2` に変更し、インスタンスが増えることを確認
- ブラウザで何度かリロードし、Instance ID / Private IP が切り替わることを確認（ロードバランシングの実感）

---

[← Step 6: ALB を追加する](06-step6-alb.md) | [クリーンアップと学習のふりかえり →](08-cleanup.md)
