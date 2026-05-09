# YAML記法チートシート

このチートシートは、CloudFormationワークショップで使用するYAML記法の基本をまとめたものです。  
YAMLは「YAML Ain't Markup Language」の略で、データをシンプルに表現するためのフォーマットです。

## 1. コメント

```yaml
# これはコメントです
# 行の先頭に # を付けるとコメントになります

Resources:  # 行末にもコメントを書けます
  VPC:      # このようにインラインコメントも可能
```

## 2. キーと値（Key-Value）

YAMLの基本は「キー: 値」のペアです。

### 文字列
```yaml
# クォートなし
Description: This is a simple string

# ダブルクォート（特殊文字をエスケープできる）
Name: "vpc-stack"

# シングルクォート（文字列をそのまま扱う）
Path: '/path/to/file'

# 数字で始まる文字列は必ずクォートで囲む
Version: "2010-09-09"
```

### 数値
```yaml
Port: 80          # 整数
MinSize: 1
MaxSize: 10
Timeout: 300      # 秒数など
```

### ブール値
```yaml
EnableDnsSupport: true
MapPublicIpOnLaunch: false
```

## 3. インデント（階層構造）

YAMLではインデント（字下げ）で階層を表現します。  
**重要**: スペースを使います（タブは使えません）。通常2スペースまたは4スペース。

```yaml
Resources:                      # 第1階層
  VPC:                         # 第2階層（2スペース）
    Type: AWS::EC2::VPC        # 第3階層（4スペース）
    Properties:                # 第3階層
      CidrBlock: 10.0.0.0/16  # 第4階層（6スペース）
      EnableDnsSupport: true   # 第4階層
```

## 4. リスト（配列）

ハイフン（-）でリストを表現します。

### シンプルなリスト
```yaml
AvailabilityZones:
  - ap-northeast-1a
  - ap-northeast-1c
  - ap-northeast-1d
```

### オブジェクトのリスト
```yaml
Tags:
  - Key: Name                  # リストの1つ目の要素
    Value: my-vpc              # Keyと同じインデントレベル
  - Key: Environment           # リストの2つ目の要素
    Value: production
```

### ネストしたリスト
```yaml
# リストの中にリストを含む例
Servers:
  - Name: web-server
    Ports:
      - 80
      - 443
    IpAddresses:
      - 10.0.1.10
      - 10.0.1.11
  - Name: db-server
    Ports:
      - 3306
    IpAddresses:
      - 10.0.2.10
```

## 5. 複数行の文字列

### パイプ記号（|）- 改行を保持
```yaml
UserData: |
  #!/bin/bash
  yum update -y
  yum install -y httpd
  systemctl start httpd
  systemctl enable httpd
```
改行がそのまま保持されます。スクリプトを書く時に便利。

### 大なり記号（>）- 改行を空白に変換
```yaml
Description: >
  This is a very long description
  that spans multiple lines
  but will be treated as a single line.
```
複数行が1行にまとめられます（改行が空白になる）。

## 6. 辞書（マッピング/オブジェクト）

キーと値のペアの集合です。

```yaml
Mappings:
  RegionAMI:                    # 第1レベルのキー
    ap-northeast-1:             # 第2レベルのキー
      AMI: ami-12345678        # 第3レベルのキーと値
    us-east-1:
      AMI: ami-87654321
```

## 7. 組み合わせの例

実際のCloudFormationテンプレートでよく見る構造：

```yaml
# リソース定義（辞書の中に辞書）
Resources:
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for web server
      VpcId: vpc-12345678
      # リストの要素として辞書を含む
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
      # タグのリスト
      Tags:
        - Key: Name
          Value: WebServerSG
        - Key: Environment
          Value: Production
```

## 8. 特殊文字のエスケープ

### ダブルクォートで囲む場合
```yaml
# 特殊文字を含む場合
Path: "C:\\Users\\Documents"    # バックスラッシュ
Message: "He said \"Hello\""    # ダブルクォート
```

### シングルクォートで囲む場合
```yaml
# シングルクォート内ではエスケープ不要
Path: 'C:\Users\Documents'
# ただしシングルクォート自体は '' で表現
Message: 'It''s a nice day'
```

## 9. 空の値

```yaml
# null値（これらはすべて同じ意味）
Value1: null
Value2: ~
Value3:         # 値なし

# 空文字列
EmptyString: ""
```

## 10. YAMLの注意点

### よくある間違い

```yaml
# タブを使ってしまう（エラーになる）
Resources:
	VPC:    # ← タブはNG

# インデントが揃っていない
Tags:
  - Key: Name
   Value: test    # ← インデントがズレている

# コロンの後にスペースがない
Type:AWS::EC2::VPC    # ← Type: AWS::EC2::VPC が正しい
```

### 正しい書き方

```yaml
# スペースでインデント（2スペースまたは4スペース）
Resources:
  VPC:
    Type: AWS::EC2::VPC

# インデントを揃える
Tags:
  - Key: Name
    Value: test

# コロンの後にスペース
Type: AWS::EC2::VPC
```

## まとめ

本ワークショップで重要なYAML記法：

1. **インデント**: 必ずスペース（2または4）を使い、階層を正確に表現
2. **リスト**: ハイフン（-）で項目を列挙
3. **キー:値**: コロンの後には必ずスペース
4. **複数行**: スクリプトには`|`を使用
5. **コメント**: `#`で説明を追加

これらの基本を押さえれば、CloudFormationテンプレートの読み書きに困ることはありません。