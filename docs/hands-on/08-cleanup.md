# クリーンアップと学習のふりかえり

## クリーンアップ

ワークショップで作成したリソースを削除します。**アプリケーションスタックから先に削除**してください。

> ネットワークスタックの Export 値をアプリケーションスタックが参照しているため、先にネットワークスタックを削除しようとするとエラーになります。

1. CloudFormation コンソールで **`cfn-workshop-app`** を選択 → **「削除」**
2. `DELETE_COMPLETE` を確認
3. **`cfn-workshop-network`** を選択 → **「削除」**
4. `DELETE_COMPLETE` を確認

> スタック間に依存関係がある場合、参照される側（ネットワーク）は参照する側（アプリケーション）を先に削除しないと削除できません。これも CloudFormation の重要な運用知識です。

---

## 学習のふりかえり

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
| 変更セット作成・確認・実行 | Step 2〜4, Step 6〜7 |
| クロススタック参照 | Step 4〜5 |
| リソース置き換え | Step 7 |
| スタック削除（依存順序） | クリーンアップ |

---

[← Step 7: Launch Template + Auto Scaling + Conditions](07-step7-asg-conditions.md) | [概要に戻る →](00-overview.md)
