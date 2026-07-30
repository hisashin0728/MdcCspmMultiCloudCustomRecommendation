# 4. AWS のクエリ例20個

## 4.1 使い方

各ファイルは1つのカスタム推奨事項候補です。`Identifiers.Type` と `Record` のパスを実環境で確認し、正常・異常の既知資産でテストしてください。型名は製品の `ec2.instance`、`cloudtrail.trail` 形式に統一しています。製品内参考 KQL で確認できない型は Categories の値と照合してください。

製品内の [AWS 参考 KQL](../MDC側で提供されている参考KQL%28AWS%29.md) は、API レスポンスを `Record.VpcId`、`Record.Routes` のように直接参照します。本章の例も同じ構造です。追加例では EC2、EBS、SSM、ECR、S3、ELB、IAM、RDS、KMS、CloudTrail、VPC の本体型に加え、権限、ステータス、ポリシー、属性、関連付けを表す補助型が使用されています。

補助型は `join`、`leftouter`、`leftanti`、`in` の入力にできます。ただし、同じ `elasticloadbalancing.loadbalancer` 型に本体行と属性行が含まれる例もあるため、「1つの型は1つの `Record` 形状」という前提では設計しません。

## 4.2 一覧

| # | 検査 | 主な証拠 | クエリ |
|---:|---|---|---|
| 1 | EC2 のパブリック IPv4 | `Association.PublicIp` | [01-ec2-instance-public-ip.kql](../queries/aws/01-ec2-instance-public-ip.kql) |
| 2 | 使用中 EBS の暗号化 | `State`, `Encrypted` | [02-ebs-volume-encryption.kql](../queries/aws/02-ebs-volume-encryption.kql) |
| 3 | 未接続 EBS | `State` | [03-ebs-volume-in-use.kql](../queries/aws/03-ebs-volume-in-use.kql) |
| 4 | EFS の暗号化 | `Encrypted` | [04-efs-encryption.kql](../queries/aws/04-efs-encryption.kql) |
| 5 | RDS の非公開化 | `PubliclyAccessible` | [05-rds-no-public-access.kql](../queries/aws/05-rds-no-public-access.kql) |
| 6 | RDS ストレージ暗号化 | `StorageEncrypted` | [06-rds-storage-encryption.kql](../queries/aws/06-rds-storage-encryption.kql) |
| 7 | RDS Multi-AZ | `MultiAZ` | [07-rds-multi-az.kql](../queries/aws/07-rds-multi-az.kql) |
| 8 | RDS 自動マイナー更新 | `AutoMinorVersionUpgrade` | [08-rds-auto-minor-upgrade.kql](../queries/aws/08-rds-auto-minor-upgrade.kql) |
| 9 | RDS 削除保護 | `DeletionProtection` | [09-rds-deletion-protection.kql](../queries/aws/09-rds-deletion-protection.kql) |
| 10 | RDS Enhanced Monitoring | `MonitoringInterval` | [10-rds-enhanced-monitoring.kql](../queries/aws/10-rds-enhanced-monitoring.kql) |
| 11 | RDS ログエクスポート | `EnabledCloudwatchLogsExports` | [11-rds-log-exports.kql](../queries/aws/11-rds-log-exports.kql) |
| 12 | CloudTrail マルチリージョン | `IsMultiRegionTrail` | [12-cloudtrail-multi-region.kql](../queries/aws/12-cloudtrail-multi-region.kql) |
| 13 | CloudTrail ログ検証 | `LogFileValidationEnabled` | [13-cloudtrail-log-validation.kql](../queries/aws/13-cloudtrail-log-validation.kql) |
| 14 | CloudTrail KMS 暗号化 | `KmsKeyId` | [14-cloudtrail-kms-encryption.kql](../queries/aws/14-cloudtrail-kms-encryption.kql) |
| 15 | CloudTrail と CloudWatch Logs の統合 | `CloudWatchLogsLogGroupArn` | [15-cloudtrail-cloudwatch-integration.kql](../queries/aws/15-cloudtrail-cloudwatch-integration.kql) |
| 16 | GuardDuty の有効化 | `Status` | [16-guardduty-enabled.kql](../queries/aws/16-guardduty-enabled.kql) |
| 17 | CloudWatch Logs の KMS 暗号化 | `KmsKeyId` | [17-cloudwatch-log-group-kms.kql](../queries/aws/17-cloudwatch-log-group-kms.kql) |
| 18 | Lambda の VPC 配置 | `VpcConfig.VpcId` | [18-lambda-inside-vpc.kql](../queries/aws/18-lambda-inside-vpc.kql) |
| 19 | Lambda の DLQ | `DeadLetterConfig.TargetArn` | [19-lambda-dead-letter-queue.kql](../queries/aws/19-lambda-dead-letter-queue.kql) |
| 20 | Secrets Manager の自動ローテーション | `RotationEnabled` | [20-secrets-manager-rotation.kql](../queries/aws/20-secrets-manager-rotation.kql) |

## 4.3 適用時の注意

- 3、18、19はワークロード要件によって例外が多いコントロールです。タグやアカウントで対象を明示してください。
- 11は DB エンジンごとに利用可能なログが異なります。配列が空でないことだけでなく、必要なログ名を検査する派生課題を推奨します。
- 12と16は本来アカウント・リージョン全体の存在性を評価するコントロールです。サンプルは収集済みの個別資産を評価します。資産が1件も収集されない状態は検出できないため、AWS Security Hub または AWS Config の組織集約と併用してください。
- 14と17は AWS 管理の既定暗号化ではなく、KMS キーの関連付けを求める組織向けです。
- 20はシークレットの用途に応じて最大ローテーション間隔も追加評価してください。
- 製品内参考例の資産名だけを使う結合は、複数アカウント・リージョンで衝突しないか確認し、必要に応じて `Identifiers.AmazonAccountId` と `AdditionalData.Region` をキーへ追加してください。
- 製品内参考例の多くは必須7列の `project` を省略しています。直接登録せず、HELP の出力契約へ正規化してください。
- IAMユーザー／グループへロールを直接割り当てるAWS IAMモデルではありません。`iam.policyuser`、`iam.policygroup`、`iam.policyversion` から `sts:AssumeRole` の許可を追跡し、ロール信頼ポリシーの収集有無を別途確認してください。構成上のAllowだけで実効権限を断定しないでください。

## 4.4 取得できる情報の範囲

AWS コネクタの対応型には EC2、RDS、CloudTrail、IAM、S3、EKS、ECS、Lambda、KMS、GuardDuty などがあります。ただし、AWS API 上で別 API となる設定は別資産型、同じ型の異なる形の行、または未収集情報になる場合があります。存在しない資産、アカウント全体の否定条件、ライブ疎通、パスワード強度などは、単一の資産レコードだけでは証明できません。補助型との関連付けで判定範囲は広がりますが、履歴、割り当て範囲外、未収集データを参照できるようになるわけではありません。
