# 情報源と参考 URL

## 製品内リファレンス

- [How to build a query](../HowToBuildaQuery.md): クエリ構造、`Record`、最新レコード、スコープ、複数環境の制約
- [MDC が提供する AWS 参考 KQL](../MDC側で提供されている参考KQL%28AWS%29.md): 単一資産、配列、集合、`leftouter`、`leftanti`、ポリシーJSON、日時比較の参考ユースケース。抽出内容には重複・途切れた見出しがあり、多くの例でHELP必須の最終 `project` が省略されているため、登録用完成形ではなく構造リファレンスとして使用する

## Microsoft

- [Microsoft Defender for Cloud でカスタム標準と推奨事項を作成する](https://learn.microsoft.com/azure/defender-for-cloud/create-custom-recommendations)
- [Azure Resource Graph がサポートするリソース型](https://learn.microsoft.com/azure/governance/resource-graph/reference/supported-tables-resources)
- [Azure Resource Graph のテーブルとリソース型](https://learn.microsoft.com/azure/governance/resource-graph/concepts/query-language)
- [KQL の dynamic データ型](https://learn.microsoft.com/kusto/query/scalar-data-types/dynamic)
- [KQL の mv-apply 演算子](https://learn.microsoft.com/kusto/query/mv-apply-operator)

## AWS

- [AWS Security Hub controls reference](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-controls-reference.html)
- [AWS Foundational Security Best Practices standard](https://docs.aws.amazon.com/securityhub/latest/userguide/fsbp-standard.html)
- [AWS Config managed rules](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html)
- [AWS Well-Architected Framework: Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html)
- [IAM roles: terms and concepts](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_terms-and-concepts.html)
- [IAM user groups](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups.html)
- [IAM policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)
- [Amazon EC2 API Reference](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/Welcome.html)
- [Amazon RDS API Reference](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/Welcome.html)

## Google Cloud

- [Security Health Analytics の脆弱性検出結果](https://cloud.google.com/security-command-center/docs/concepts-vulnerabilities-findings)
- [Security Health Analytics の概要](https://cloud.google.com/security-command-center/docs/concepts-security-health-analytics)
- [Cloud Asset Inventory がサポートする資産型](https://cloud.google.com/asset-inventory/docs/supported-asset-types)
- [Google Cloud Well-Architected Framework: Security, privacy, and compliance](https://cloud.google.com/architecture/framework/security)
- [Compute Engine REST resources](https://cloud.google.com/compute/docs/reference/rest/v1)
- [Cloud Storage JSON API: Buckets](https://cloud.google.com/storage/docs/json_api/v1/buckets)
- [GKE REST API: projects.locations.clusters](https://cloud.google.com/kubernetes-engine/docs/reference/rest/v1/projects.locations.clusters)

## 読み方

Microsoft の資料はカスタム推奨事項の入力・出力契約と Defender 側の運用に使用します。AWS と Google Cloud の資料はコントロールの意図、対象資産、設定フィールド、修復方法の根拠に使用します。コントロール ID、フィールド、既定値は変更されるため、カスタム推奨事項のレビュー時にリンク先の最新版と再照合してください。
