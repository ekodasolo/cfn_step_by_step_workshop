# CloudFormation ワークショップ 講師ガイド

## 概要

このドキュメントは講師向けの進行ガイドです。各ステップの補足説明、つまずきやすいポイント、タイムキープの目安をまとめています。

---

## 事前準備

### 環境準備

- [ ] 参加者全員が AWS マネジメントコンソールにログインできることを確認
- [ ] リージョンが **ap-northeast-1（東京）** に統一されていることを確認
- [ ] 参加者にテンプレートファイル一式を配布（`templates/` ディレクトリの `stepN.yaml` ファイル）
- [ ] Mappings の AMI ID が最新であることを確認（EC2 コンソール → AMI カタログ → Amazon Linux 2023）

### 配布物

| ファイル | 用途 |
|---|---|
| `templates/network/network-step1.yaml` | Step 1 の出発点 |
| `templates/network/network-step2.yaml` | Step 2 の出発点（Step 1 が間に合わなかった場合のリカバリ用も兼ねる） |
| `templates/network/network-step3.yaml` | Step 3 の出発点 |
| `templates/network/network-step4.yaml` | Step 4 の出発点 |
| `templates/application/application-step5.yaml` | Step 5 の出発点 |
| `templates/application/application-step6.yaml` | Step 6 の出発点 |
| `templates/application/application-step7.yaml` | Step 7 の出発点 |
| `templates/network/network-final.yaml` | ネットワークスタック完成形（答え合わせ用） |
| `templates/application/application-final.yaml` | アプリケーションスタック完成形（答え合わせ用） |
| `docs/hands-on-guide.md` | 参加者用テキスト |

### リカバリ戦略

参加者がステップの途中でつまずいた場合、**次のステップの初期テンプレート**（= 現ステップの完成形）をそのままデプロイすることで追いつけます。

例: Step 2 でつまずいた場合 → `network-step3.yaml`（Step 2 完成済み）をアップロードしてスタック更新

---

## タイムテーブル

| 時間 | 内容 | 講師のアクション |
|---|---|---|
| 0:00 - 0:15 | イントロ | `introduction.md` に沿って IaC の利点・テンプレート = パラメータシートのアナロジーを説明 |
| 0:15 - 0:45 | Step 1〜2 | テンプレート構造、Parameters、!Ref を解説しながら進行 |
| 0:45 - 1:15 | Step 3 | DependsOn、!Sub の概念を解説。IGW+ルートテーブルはリソースが多いので丁寧に |
| 1:15 - 1:45 | Step 4 | SG と Outputs を解説。ここでネットワークスタック完成 |
| 1:45 - 2:00 | **休憩** | — |
| 2:00 - 2:40 | Step 5 | クロススタック参照、Mappings を解説。SSM 接続確認 |
| 2:40 - 3:15 | Step 6 | ALB の構成要素を図示しながら解説。ブラウザで動作確認 |
| 3:15 - 3:50 | Step 7 | Conditions、ASG を解説。リロードでインスタンス切り替わりを実演 |
| 3:50 - 4:00 | まとめ | ふりかえり、クリーンアップ、質疑応答 |

### タイムキープのコツ

- **Step 1〜2 は速めに進める**（30 分）。テンプレートの基本構造は説明が中心で、記述量は少ない
- **Step 3 は時間がかかりやすい**（30 分確保）。リソースが 5 種類あり、依存関係の概念も新しい
- **Step 5 は山場**（40 分確保）。新しいスタック作成 + クロススタック参照 + UserData と新概念が多い
- 時間が押している場合は、Step 7 の `UpdatePolicy` を省略可能（ASG の基本だけ説明）

---

## ステップ別 講師メモ

### Step 1: VPC だけのテンプレート

**説明のポイント**:
- 「CloudFormation = インフラをコードで定義するサービス」を最初に伝える
- テンプレートの 3 要素（`AWSTemplateFormatVersion`, `Description`, `Resources`）を図示
- 「Resources だけが必須、他は全部オプション」を強調

**つまずきポイント**:
- YAML のインデントミス（スペース 2 つで統一。タブは使わない）
- スタック名に大文字やアンダースコアを使うとリソース名で問題が出ることがある → **小文字・ハイフン区切り**を推奨

**デプロイ後に見せるもの**:
- CloudFormation コンソールのイベントタブ（`CREATE_IN_PROGRESS` → `CREATE_COMPLETE` の流れ）
- VPC コンソールで実際に VPC が作られていること

---

### Step 2: サブネットを追加

**説明のポイント**:
- 「なぜ Parameters を使うのか」— 同じテンプレートを別の環境（開発/本番）で使い回すため
- `!Ref` は「パラメータの値」と「リソースの ID」の 2 つを返す関数
- `!GetAZs ""` の `""` は「現在のリージョン」を意味する

**つまずきポイント**:
- `!Select` の書き方:

  ```yaml
  # 正しい（YAML リスト形式）
  AvailabilityZone: !Select
    - 0
    - !GetAZs ""

  # 正しい（インライン形式）
  AvailabilityZone: !Select [0, !GetAZs ""]
  ```

- Parameters セクションの配置場所を間違える（`Resources` の中に書いてしまう等）

**デプロイ後に見せるもの**:
- イベントタブで「VPC は変更なし、サブネットだけ新規作成」を確認 → **スタック更新は差分だけ適用される**

---

### Step 3: インターネット接続を構成

**説明のポイント**:
- IGW → VPCGatewayAttachment → RouteTable → Route → SubnetRouteTableAssociation の関係を図示
- `DependsOn` が必要な理由: 「IGW は VPC にアタッチされてからでないとルーティングに使えない」
- 暗黙的依存の例: `VpcId: !Ref VPC` と書くだけで VPC → サブネットの順序が保証される

**つまずきポイント**:
- リソースが 5 種類あり混乱しやすい → 1 つずつ追加してもよい（全部まとめて追加する必要はない）
- `DestinationCidrBlock: "0.0.0.0/0"` の引用符忘れ

**補足説明**:
- 「今回はパブリックサブネットだけなのでルートテーブルは 1 つだが、プライベートサブネットがある場合は NAT Gateway 用の別ルートテーブルが必要」

---

### Step 4: セキュリティグループと Outputs

**説明のポイント**:
- SG 間参照の意味: 「ALB の SG を持つリソースからのトラフィックだけを許可する」＝ 「ALB 経由のアクセスだけ通す」
- `Outputs` + `Export` の目的: 「このスタックの値を、別のスタックから参照できるようにする」
- Export 名はリージョン内でユニークでなければならない → `${AWS::StackName}` を prefix にする理由

**つまずきポイント**:
- `SourceSecurityGroupId` と `CidrIp` を混同する
- `!GetAtt ALBSecurityGroup.GroupId` の `.GroupId` を忘れる

**デプロイ後に見せるもの**:
- CloudFormation コンソールの「出力」タブ → エクスポートされた値を確認
- 「これが次のスタックから参照される」と予告

---

### Step 5: EC2 インスタンスを作成する

**説明のポイント**:
- 「ここから 2 つ目のスタック」— ネットワークとアプリケーションを分離する理由（ライフサイクルが異なる、チーム分担）
- `!ImportValue` + `Fn::Sub` の書き方が少し複雑 → 「短縮構文は入れ子にできないから正式構文を使う」
- UserData の `${!変数名}` エスケープ → 「`!Sub` がシェル変数を CloudFormation 変数と勘違いしないようにするため」

**つまずきポイント**:
- `NetworkStackName` パラメータに正確なスタック名（`cfn-workshop-network`）を入力しないとエラー
- IAM リソースの承認チェックを忘れる → スタック作成が失敗する
- `!FindInMap` の引数の順序（マッピング名、第 1 キー、第 2 キー）

**デプロイ後に見せるもの**:
- SSM Session Manager での接続手順を実演
- `curl localhost` でページが返ることを確認

---

### Step 6: ALB を追加

**説明のポイント**:
- ALB の 3 層構造を図示: ALB → Listener → TargetGroup → EC2
- 「Listener はポートとプロトコルで受け、DefaultActions で転送先を決める」
- 「TargetGroup のヘルスチェックが通らないとトラフィックが転送されない」

**つまずきポイント**:
- `DefaultActions` の書き方:

  ```yaml
  DefaultActions:
    - Type: forward
      TargetGroupArn: !Ref WebServerTargetGroup
  ```

  `Type` と `TargetGroupArn` のインデントを間違えやすい
- ALB 作成には数分かかる → 「待ち時間にヘルスチェックの仕組みを説明する」

**デプロイ後に見せるもの**:
- 「出力」タブの ALB DNS 名をブラウザで開く
- カレンダーページが表示されること
- 「まだ 1 台なのでリロードしても同じインスタンス → 次のステップで解決」

---

### Step 7: Launch Template + Auto Scaling + Conditions

**説明のポイント**:
- 「EC2 直接定義 → Launch Template + ASG」への移行は実運用でもよくあるパターン
- Conditions は「テンプレートの if 文」— 同じテンプレートで開発/本番を使い分ける
- UpdatePolicy: 「全台一斉入れ替えではなく、1 台ずつ入れ替えることでダウンタイムをなくす」

**つまずきポイント**:
- WebServerInstance の削除忘れ → Launch Template と競合する
- TargetGroup の `Targets` プロパティの削除忘れ → ASG と競合する
- `!If` の書き方（リスト形式: `[条件名, true値, false値]`）

**デプロイ後に見せるもの**:
- イベントタブで EC2 の Replace が発生する様子
- ALB DNS 名でアクセスし、リロードで Instance ID / Private IP が切り替わること → **ロードバランシングの実感**
- （時間があれば）EC2 コンソールで DesiredCapacity を変更し、インスタンスが増減する様子

---

## クリーンアップ

**必ず確認すること**:
- 全参加者がアプリケーションスタック → ネットワークスタックの順で削除したこと
- 削除が `DELETE_COMPLETE` になったこと
- 削除に失敗した場合はリソースの手動削除が必要になる場合がある（SG が他リソースに参照されている等）

---

## よくある質問

### Q: YAML のインデントを間違えてスタック作成がエラーになった

テンプレートの構文エラーは CloudFormation がアップロード時に検出します。エラーメッセージに行番号が出るので、該当箇所のインデントを確認してください。スペース 2 つで統一し、タブは使わないでください。

### Q: スタック更新がロールバックされた

今回のワークショップではハッピーパターンのみ扱っていますが、テンプレートの記述ミスで更新が失敗するとロールバックが発生します。イベントタブでどのリソースが失敗したかを確認し、テンプレートを修正して再度更新してください。

### Q: !ImportValue でエラーが出る

`NetworkStackName` パラメータの値がネットワークスタックの名前と完全に一致しているか確認してください。大文字小文字も区別されます。

### Q: EC2 にアクセスできない / ページが表示されない

- セキュリティグループの設定を確認（ポート 80 が開いているか）
- ヘルスチェックが通っているかターゲットグループで確認
- EC2 の UserData が正しく実行されたか SSM Session Manager で `systemctl status httpd` を確認
