# 5. GCP のクエリ例20個

## 5.1 使い方

GCP コネクタ固有の `Identifiers.Type` は公開された固定スキーマとして確認できないため、全例の `Identifiers.Type` は意図的に `REPLACE_WITH_*` になっています。[データモデル](02-data-model.md) の探索クエリまたは製品の Categories で Compute Engine、Cloud Storage、GKE の型を取得して置換してください。

フィールドは Google Security Health Analytics の検出器が示す Cloud Asset メタデータを基準にし、`Record.canIpForward` のように API レスポンスを `Record` 直下から参照します。Defender for Cloud が同じフィールドを収集している場合にだけ使用できます。

## 5.2 一覧

| # | SHA カテゴリ | 主な証拠 | クエリ |
|---:|---|---|---|
| 1 | `PUBLIC_IP_ADDRESS` | `networkInterfaces[].accessConfigs` | [01-compute-public-ip.kql](../queries/gcp/01-compute-public-ip.kql) |
| 2 | `IP_FORWARDING_ENABLED` | `canIpForward` | [02-compute-ip-forwarding.kql](../queries/gcp/02-compute-ip-forwarding.kql) |
| 3 | `COMPUTE_SECURE_BOOT_DISABLED` | `shieldedInstanceConfig.enableSecureBoot` | [03-compute-secure-boot.kql](../queries/gcp/03-compute-secure-boot.kql) |
| 4 | `SHIELDED_VM_DISABLED` | `enableIntegrityMonitoring` | [04-compute-integrity-monitoring.kql](../queries/gcp/04-compute-integrity-monitoring.kql) |
| 5 | `SHIELDED_VM_DISABLED` | `enableVtpm` | [05-compute-vtpm.kql](../queries/gcp/05-compute-vtpm.kql) |
| 6 | `COMPUTE_PROJECT_WIDE_SSH_KEYS_ALLOWED` | metadata item | [06-compute-block-project-ssh-keys.kql](../queries/gcp/06-compute-block-project-ssh-keys.kql) |
| 7 | `COMPUTE_SERIAL_PORTS_ENABLED` | metadata item | [07-compute-serial-port-disabled.kql](../queries/gcp/07-compute-serial-port-disabled.kql) |
| 8 | `DEFAULT_SERVICE_ACCOUNT_USED` | `serviceAccounts[].email` | [08-compute-no-default-service-account.kql](../queries/gcp/08-compute-no-default-service-account.kql) |
| 9 | `BUCKET_POLICY_ONLY_DISABLED` | `uniformBucketLevelAccess.enabled` | [09-storage-uniform-access.kql](../queries/gcp/09-storage-uniform-access.kql) |
| 10 | `OBJECT_VERSIONING_DISABLED` | `versioning.enabled` | [10-storage-versioning.kql](../queries/gcp/10-storage-versioning.kql) |
| 11 | `BUCKET_LOGGING_DISABLED` | `logging.logBucket` | [11-storage-access-logging.kql](../queries/gcp/11-storage-access-logging.kql) |
| 12 | `LOCKED_RETENTION_POLICY_NOT_SET` | `retentionPolicy.isLocked` | [12-storage-retention-policy-locked.kql](../queries/gcp/12-storage-retention-policy-locked.kql) |
| 13 | `BUCKET_CMEK_DISABLED` | `encryption.defaultKmsKeyName` | [13-storage-cmek.kql](../queries/gcp/13-storage-cmek.kql) |
| 14 | `CLUSTER_LOGGING_DISABLED` | `loggingService` | [14-gke-logging.kql](../queries/gcp/14-gke-logging.kql) |
| 15 | `CLUSTER_MONITORING_DISABLED` | `monitoringService` | [15-gke-monitoring.kql](../queries/gcp/15-gke-monitoring.kql) |
| 16 | `LEGACY_AUTHORIZATION_ENABLED` | `legacyAbac.enabled` | [16-gke-legacy-abac.kql](../queries/gcp/16-gke-legacy-abac.kql) |
| 17 | `PRIVATE_CLUSTER_DISABLED` | `enablePrivateNodes` | [17-gke-private-nodes.kql](../queries/gcp/17-gke-private-nodes.kql) |
| 18 | `NETWORK_POLICY_DISABLED` | `networkPolicy.enabled` | [18-gke-network-policy.kql](../queries/gcp/18-gke-network-policy.kql) |
| 19 | `WORKLOAD_IDENTITY_DISABLED` | `workloadIdentityConfig.workloadPool` | [19-gke-workload-identity.kql](../queries/gcp/19-gke-workload-identity.kql) |
| 20 | `CLUSTER_SECRETS_ENCRYPTION_DISABLED` | `databaseEncryption.state` | [20-gke-secrets-encryption.kql](../queries/gcp/20-gke-secrets-encryption.kql) |

## 5.3 適用時の注意

- `coalesce(..., false)` を使う例では、Google Cloud のリソース表現上「プロパティなし」が既定の無効状態を意味します。Defender 側の収集漏れと区別するため、フィールド充足率を先に確認してください。
- 1の `accessConfigs` は配列です。環境の JSON がオブジェクト配列なら、`natIP` を内側の `mv-apply` で評価するよう調整します。
- 6と7はプロジェクトメタデータによる継承も確認してください。インスタンスレコードだけでは有効設定の全体を判断できない場合があります。
- 10と12はログ格納先など、保護対象のバケットにスコープを限定します。
- 13と20は CMEK を要求する組織ポリシーがある場合に適用します。
- GKE Autopilot やサービス管理リソースでは、ユーザーが変更できない設定があります。クラスターモードを例外条件に含めてください。

## 5.4 取得できる情報の範囲

Security Health Analytics の一部検出器は、Cloud Asset Inventory に加えて別 API、IAM ポリシー、組織ポリシー、関連資産を読みます。単一の `RawEntityMetadata` 行で同じ検出を完全再現できるとは限りません。公開アクセス、IAM ロール分離、ログシンクの存在など複数資産にまたがる条件は、組み込みの GCP 検出器との併用を推奨します。
