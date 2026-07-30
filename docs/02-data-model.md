# 2. `RawEntityMetadata` とフィールド

## 2.1 入力列

カスタム推奨事項の KQL は `RawEntityMetadata` から始めます。主に使用する列は次のとおりです。

| 列 | 用途 |
|---|---|
| `Id` | Defender for Cloud 内の資産識別子 |
| `Name` | 表示名 |
| `Environment` | `AWS`、`GCP`、`Azure` などの環境 |
| `Identifiers` | クラウド固有 ID、資産型などの識別情報 |
| `AdditionalData` | コネクタや資産に関する追加情報 |
| `Record` | AWS/GCP API から返された構造を保持する資産レコード |

`Identifiers`、`AdditionalData`、`Record` は `dynamic` 値です。JSON の大文字小文字と配列構造を実データで確認してください。

製品内 HELP と AWS 参考 KQL では、`Identifiers.Type` は `ec2.instance`、`cloudtrail.trail` のような製品カテゴリの値です。Azure Resource Graph の `microsoft.awsconnector/*` とは別の名前空間です。また、設定値は `Record.State.Name.Value` や `Record.VpcId` のように `Record` 直下から参照します。

同じ `Identifiers.Type` でも、すべての行が同じ `Record` の形を持つとは限りません。製品内の ELB 例では `elasticloadbalancing.loadbalancer` に、ロードバランサー本体の行と `Key` / `Value` を持つ属性行が含まれます。型だけでスキーマを決めず、判定に使う識別フィールドと候補プロパティの充足率を確認してください。また、S3 の `Record.Policy` や ECR の `Record.PolicyText` のように、`Record` 内の値が JSON 文字列の場合は `parse_json(tostring(...))` で再解析します。

## 2.2 必須出力

判定クエリは次の7列を返します。

```kusto
| project Id, Name, Environment, Identifiers, AdditionalData, Record, HealthStatus
```

`HealthStatus` は文字列の `HEALTHY` または `UNHEALTHY` です。集計だけを返したり、不健全な行だけを `where` で残したりすると、資産ごとの状態を評価できません。

## 2.3 資産型を探索する

最初に環境ごとの型と件数を確認します。

```kusto
RawEntityMetadata
| where Environment in~ ('AWS', 'GCP')
| extend ResourceType = tostring(Identifiers.Type)
| summarize Resources = count(), Samples = make_set(Name, 3) by Environment, ResourceType
| order by Environment asc, Resources desc
```

型名は推測せず、カスタム推奨事項画面の Categories またはこの結果からコピーします。AWS の Azure Resource Graph 表現にある `microsoft.awsconnector/*` を転用しないでください。

## 2.4 プロパティを探索する

対象型を1つに絞り、プロパティのキーを列挙します。

```kusto
let TargetEnvironment = 'AWS';
let TargetType = 'REPLACE_WITH_DISCOVERED_TYPE';
RawEntityMetadata
| where Environment =~ TargetEnvironment
| where tostring(Identifiers.Type) =~ TargetType
| extend PropertyName = bag_keys(Record)
| mv-expand PropertyName
| summarize PresentOnResources = dcount(Id) by PropertyName = tostring(PropertyName)
| order by PropertyName asc
```

次に、評価候補フィールドの値と型を確認します。

```kusto
let TargetEnvironment = 'AWS';
let TargetType = 'REPLACE_WITH_DISCOVERED_TYPE';
RawEntityMetadata
| where Environment =~ TargetEnvironment
| where tostring(Identifiers.Type) =~ TargetType
| project Name, Value = Record.REPLACE_WITH_PROPERTY
| extend RuntimeType = gettype(Value)
| take 20
```

## 2.5 null と欠損

次の3状態を区別します。

| 状態 | 意味の例 | 扱い |
|---|---|---|
| 明示的な `false` | 機能が無効 | ベストプラクティスに反するなら `UNHEALTHY` |
| `null` がサービス仕様上の値 | KMS キー未指定など | 仕様を確認して判定に使用 |
| フィールド自体が未収集 | コネクタ差異、権限不足、型違い | 自動的に設定不良としない |

本番登録前に、対象件数と候補フィールドの充足率を確認します。

```kusto
let TargetType = 'REPLACE_WITH_DISCOVERED_TYPE';
RawEntityMetadata
| where Environment =~ 'AWS' and tostring(Identifiers.Type) =~ TargetType
| extend Candidate = Record.REPLACE_WITH_PROPERTY
| summarize Total = count(), Present = countif(isnotnull(Candidate))
| extend CoveragePercent = round(100.0 * Present / Total, 1)
```

## 2.6 Azure Resource Graph との関係

接続された一部の AWS 資産は Azure Resource Graph の `resources` でも参照できます。`securityresources` は Defender for Cloud の評価結果を保持します。ただし、カスタム推奨事項の編集画面で評価される入力契約は `RawEntityMetadata` です。ARG のリソース型や `properties` のパスを転用しないでください。

## 2.7 フィールドから可能なこと

| 判定 | 必要な情報 | 例 |
|---|---|---|
| 単一資産の設定値 | `Record` 内のフィールド | 暗号化、公開フラグ、削除保護 |
| 配列内の設定 | `Record` 内の配列 | セキュリティグループ、ルート、サービスアカウント |
| 関連資産との突合 | 両方の型が `RawEntityMetadata` に存在 | VPC と Flow Log、Trail と Trail Status |
| 任意の関連レコードとの突合 | 親資産の行と関連レコードの共通キーがある | Snapshot と CreateVolumePermission |
| 現在値に含まれる日時との比較 | API レコード内の日時 | 停止から30日超の EC2 |
| コンテキストを含む相関 | `Identifiers`、`AdditionalData` にスコープ情報がある | アカウント、リージョン、資産 ID の複合キー |
| 割り当て範囲内の例外 | `Identifiers`、`AdditionalData`、タグ等 | 特定リージョンや承認済みラベルの除外 |

## 2.8 フィールドから仕様上できないこと

1. **複数環境の同時評価**: 1つの推奨事項で複数環境は扱えません。AWS用とGCP用を分けます。
2. **任意期間の履歴検索**: Timespan フィルターは不要で、評価サービスが最新レコードを選びます。これは構成履歴テーブルではありません。
3. **未収集情報の推論**: `Record` にない設定、データ内容、通信の到達性、実際の認証成功などは判定できません。
4. **起点資産が存在しない状態の出力**: 出力する資産行が必要です。親資産があれば関連レコード不在を判定できますが、アカウント全体に何もないことを架空行として返すことはできません。
5. **割り当て外スコープの参照保証**: クエリは割り当て資産に対して評価されます。固定 ARN やアカウント ID は適用範囲を狭めます。
6. **null の意味の自動判定**: API既定値、アクセス権不足、コネクタ未収集を KQL だけで常に区別することはできません。
7. **修復操作**: `RawEntityMetadata` は判定入力です。KQL は AWS/GCP の設定を変更しません。
8. **型名だけによる一意なスキーマ保証**: 同じ `Identifiers.Type` に異なる形の `Record` が含まれる例があります。型名だけからフィールドの存在や意味を確定できません。

## 2.9 製品内資料との対応

- [How to build a query](../HowToBuildaQuery.md): 先頭の環境・型フィルター、`iff`、必須出力、最新レコード、スコープの制約を説明します。
- [MDC が提供する AWS 参考 KQL](../MDC側で提供されている参考KQL%28AWS%29.md): 単一資産、配列展開、`join`、`leftanti`、`union`、日時比較の実例です。

## 2.10 `Identifiers.Type` の詳細

### 2.10.1 何を表す項目か

`Identifiers` は `dynamic` 値で、資産のクラウド固有 ID・スコープ・資産型などの識別情報を保持します。そのうち `Identifiers.Type` は **その1行がどの資産クラスの情報なのかを示す型 (discriminator)** です。カスタム推奨事項のクエリ先頭で環境と組み合わせて指定し、「この推奨事項がどの資産を評価するか」を確定します。

```kusto
| where Environment == 'AWS' and Identifiers.Type == 'ec2.volume'
```

命名は `<サービス>.<リソース>` の小文字ドット記法です(例: `ec2.volume`、`rds.dbinstance`、`cloudtrail.trail`)。これは AWS/GCP コネクタが収集した資産を分類する **Defender for Cloud 独自のカテゴリ体系** であり、次のいずれとも別物です。混同して転用しないでください。

- Azure Resource Graph の `microsoft.awsconnector/*`(ARG 表現)
- AWS CloudFormation / Config の `AWS::EC2::Volume` 形式
- GCP の `compute.googleapis.com/Instance` 形式

### 2.10.2 何を絞り込めるか(役割)

- **評価対象の資産クラスを1つに固定する**: `Identifiers.Type` が、推奨事項が見る資産の母集団を決めます。
- **`Record` のスキーマがほぼ決まる**: 型が決まると `Record` が保持する AWS/GCP API レスポンスの形状が定まり、どのプロパティで健全性を判定できるかが決まります(下表)。
- **相関のキーになる**: 親資産と関連レコードを別々の型として取得し、`join` / `leftanti` で突合できます(例: `cloudtrail.trail` と Trail Status、VPC と Flow Log)。

ただし 2.1・2.8 のとおり、**同じ `Identifiers.Type` でも `Record` の形が一様とは限りません**(本体行と属性行が混在する例があります)。型名だけでスキーマを確定せず、判定に使うフィールドの充足率(2.5)を必ず確認してください。

### 2.10.3 粒度に関する注意 — 本体資産だけではない

1つの AWS サービスが複数の `Identifiers.Type` に分かれます。粒度は **AWS API の Describe/Get 呼び出し単位に近く**、資産本体だけでなく、その付随メタデータも独立した型として収集されます。テスト環境で件数が多く見えるのはこのためです。例(IAM):

| Identifiers.Type | 収集単位 | 含まれる主な情報 |
|---|---|---|
| `iam.user` / `iam.role` / `iam.group` | プリンシパル本体 | 名前、ARN、作成日時、アタッチ状況 |
| `iam.policy` | 管理ポリシー本体 | ポリシー名、ARN、既定バージョン ID |
| `iam.policyversion` | ポリシーの各バージョン | 権限ステートメント本体(`Document`)。実際の許可判定に使用 |
| `iam.accessadvisor` | サービス最終アクセス情報 | サービスごとの最終アクセス日時。未使用権限の検出に使用 |

`ec2.instanceType`、`iam.accessadvisor`、`iam.policyversion` のように、資産本体(`ec2.instance`、`iam.role`)より細かい型が多数現れるのはこの設計によるものです。

### 2.10.4 テスト環境 (ZAVA) での確認結果

- ZAVA テナント (Tenant ID `0527ecb7-06fb-4769-b324-fd4a3bb865eb`) には AWS コネクタ **`AlpineSkiHouseAWS`** と **`c2c-demo-connector`** が接続され、AWS 資産が収集済みであることを確認しました。
- `Identifiers.Type` の**実値一覧そのものは Azure Resource Graph / `az graph` からは取得できません**。`RawEntityMetadata` はカスタム推奨事項エディター専用のテーブルで、ARG では `Environment` 列すら解決できません(`Operator_FailedToResolveEntity`)。したがって `Identifiers.Type` の正確な文字列の列挙は、2.10.5 の探索クエリを**ポータルのカスタム推奨事項作成画面**で実行して確認します。
- 一方、**ZAVA で現在どの AWS 資産型が見えているか**は、`securityresources`(評価結果)の `resourceDetails.ResourceType` から実測できます。その一覧を 2.10.7 に掲載しています。

### 2.10.5 実値を列挙する探索クエリ(ポータルで実行)

型名は推測せず、次のクエリの結果か、カスタム推奨事項画面の Categories からコピーします。

```kusto
RawEntityMetadata
| where Environment == 'AWS'
| extend ResourceType = tostring(Identifiers.Type)
| summarize Resources = count(), Samples = make_set(Name, 3) by ResourceType
| order by Resources desc
```

特定の型の中身(プロパティのキー)を確認するには 2.4 の探索クエリを使います。

### 2.10.6 代表的な AWS `Identifiers.Type` と含まれる情報

次の型は本リポジトリの検証済みクエリ([../queries/aws/](../queries/aws/))で実際に使用しているもので、`Record` の実パスもそこに準拠します。

| Identifiers.Type | 資産 | 判定に使う代表的な `Record` パス | 対応クエリ |
|---|---|---|---|
| `ec2.networkinterface` | ENI(EC2 のネットワークインターフェイス) | `Record.Association.PublicIp` | [01](../queries/aws/01-ec2-instance-public-ip.kql) |
| `ec2.volume` | EBS ボリューム | `Record.State`、`Record.Encrypted` | [02](../queries/aws/02-ebs-volume-encryption.kql), [03](../queries/aws/03-ebs-volume-in-use.kql) |
| `efs.filesystem` | EFS ファイルシステム | `Record.Encrypted` | [04](../queries/aws/04-efs-encryption.kql) |
| `rds.dbinstance` | RDS DB インスタンス | `Record.PubliclyAccessible`、`Record.StorageEncrypted`、`Record.MultiAZ`、`Record.DeletionProtection`、`Record.MonitoringInterval`、`Record.EnabledCloudwatchLogsExports` | [05](../queries/aws/05-rds-no-public-access.kql)〜[11](../queries/aws/11-rds-log-exports.kql) |
| `cloudtrail.trail` | CloudTrail 証跡 | `Record.IsMultiRegionTrail`、`Record.LogFileValidationEnabled`、`Record.KmsKeyId`、`Record.CloudWatchLogsLogGroupArn` | [12](../queries/aws/12-cloudtrail-multi-region.kql)〜[15](../queries/aws/15-cloudtrail-cloudwatch-integration.kql) |
| `guardduty.detector` | GuardDuty 検出器 | `Record.Status` | [16](../queries/aws/16-guardduty-enabled.kql) |
| `logs.loggroup` | CloudWatch Logs ロググループ | `Record.KmsKeyId` | [17](../queries/aws/17-cloudwatch-log-group-kms.kql) |
| `lambda.function` | Lambda 関数 | `Record.VpcConfig.VpcId`、`Record.DeadLetterConfig` | [18](../queries/aws/18-lambda-inside-vpc.kql), [19](../queries/aws/19-lambda-dead-letter-queue.kql) |
| `secretsmanager.secret` | Secrets Manager シークレット | `Record.RotationEnabled` | [20](../queries/aws/20-secrets-manager-rotation.kql) |

参考(IAM 関連の型で扱える情報の例。実在の型名と `Record` パスはポータルで確認):

| Identifiers.Type | 含まれる情報の例 | 判定できることの例 |
|---|---|---|
| `iam.policyversion` | 権限ステートメント本体(`Record.Document`) | 過剰な `Action: *` / `Resource: *` の検出 |
| `iam.accessadvisor` | サービス最終アクセス日時 | 一定期間未使用の権限の検出 |
| `ec2.instance` | インスタンス構成(状態、IAM プロファイル、メタデータオプション等) | IMDSv2 必須化、停止インスタンスの棚卸し |

> 上表の型名・`Record` パスは環境やコネクタで変わり得ます。本番登録前に、必ず 2.10.5 の探索クエリと 2.5 の充足率チェックで実データを確認してください。

### 2.10.7 ZAVA 環境で現在見えている AWS 資産型(実測)

次の表は、ZAVA テナントの全 AWS コネクタ配下で **現在評価対象になっている AWS 資産型**を実測したものです(2026-07-30 時点)。`RawEntityMetadata` は ARG から引けないため、代わりに ARG の `securityresources`(評価結果)から `resourceDetails.ResourceType` を集計しています。`ResourceType` の末尾セグメントは `Identifiers.Type` と 1 対 1 ではありませんが、ほぼ対応します(下表「対応する `Identifiers.Type`(推定)」)。**正確な `Identifiers.Type` 文字列は 2.10.5 のポータル探索クエリで確定**してください。

実測に使用したクエリ(ZAVA の全サブスクリプションを対象、`az graph` / Resource Graph Explorer):

```kusto
securityresources
| where type == 'microsoft.security/assessments'
| where tostring(properties.resourceDetails.Source) == 'AWS'
| extend ResourceType = tostring(properties.resourceDetails.ResourceType)
| summarize Resources = dcount(tostring(properties.resourceDetails.NativeCloudUniqueIdentifier)) by ResourceType
| order by Resources desc
```

| ResourceType(`microsoft.security/securityconnectors/` 以下) | 対応する `Identifiers.Type`(推定) | 資産 | ZAVA の実リソース数 |
|---|---|---|---|
| `keyspacestable` | `cassandra.table` | Amazon Keyspaces テーブル | 656 |
| `ec2subnet` | `ec2.subnet` | サブネット | 65 |
| `ec2securitygroup` | `ec2.securitygroup` | セキュリティグループ | 36 |
| `iamrole` | `iam.role` | IAM ロール | 35 |
| `ec2routetable` | `ec2.routetable` | ルートテーブル | 24 |
| `ec2vpc` | `ec2.vpc` | VPC | 19 |
| `ec2network-acl` | `ec2.networkacl` | ネットワーク ACL | 19 |
| `eventbridgeeventbus` | `events.eventbus` | EventBridge イベントバス | 17 |
| `athenaworkgroup` | `athena.workgroup` | Athena ワークグループ | 17 |
| `ecrrepository` | `ecr.repository` | ECR リポジトリ | 12 |
| `iampolicy` | `iam.policy` | IAM 管理ポリシー | 11 |
| `s3bucket` | `s3.bucket` | S3 バケット | 11 |
| `iamcredentialreport` | `iam.credentialreport` | IAM 認証情報レポート | 11 |
| `cloudformationstack` | `cloudformation.stack` | CloudFormation スタック | 11 |
| `iamuser` | `iam.user` | IAM ユーザー | 10 |
| `sqsqueue` | `sqs.queue` | SQS キュー | 7 |
| `ekscluster` | `eks.cluster` | EKS クラスター | 6 |
| `ecsservice` | `ecs.service` | ECS サービス | 6 |
| `kmskey` | `kms.key` | KMS キー | 5 |
| `ecstaskdefinition` | `ecs.taskdefinition` | ECS タスク定義 | 5 |
| `ec2volumes` | `ec2.volume` | EBS ボリューム | 4 |
| `ec2instance` | `ec2.instance` | EC2 インスタンス | 4 |
| `lambdafunction` | `lambda.function` | Lambda 関数 | 3 |
| `cloudtrailtrail` | `cloudtrail.trail` | CloudTrail 証跡 | 3 |
| `elasticloadbalancingloadbalancer` | `elasticloadbalancing.loadbalancer` | ELB(ロードバランサー) | 2 |
| `autoscalinggroup` / `autoscalingautoscalinggroupname` | `autoscaling.autoscalinggroup` | Auto Scaling グループ | 2 |
| `ec2vpc-flow-log` | `ec2.vpcflowlog` | VPC フローログ | 2 |
| `stsaccount` / `stsaccount-in-region` | `sts.account` | アカウント/リージョンの STS 設定 | 1 |
| `rdscluster-snapshot` | `rds.clustersnapshot` | RDS クラスタースナップショット | 1 |
| `ec2vpc-endpoint` | `ec2.vpcendpoint` | VPC エンドポイント | 1 |
| `iampolicygroup` / `iampolicyrole` / `iampolicyuser` | `iam.policy*`(割当関係) | ポリシーとグループ/ロール/ユーザーの割当関係 | 各 1 |

注意点:

- この表は **評価 (assessment) が生成されている資産型**のみを反映します。資産は存在しても対象コントロールがなければ現れないため、`Identifiers.Type` の網羅リストとは一致しません。
- 2.10.6 の検証済み型のうち `rds.dbinstance`・`efs.filesystem`・`guardduty.detector`・`logs.loggroup`・`secretsmanager.secret`・`ec2.networkinterface` は、2026-07-30 時点の ZAVA 評価データには現れませんでした(該当資産が未作成か、対象評価が未生成)。カスタム推奨事項を作る際は 2.10.5 の探索クエリで実在を再確認してください。
- コンテナ/イメージ由来の検出(`ecs.container`、`K8s-container`、`.containerimage` など)は CSPM の設定資産型ではなく、コンテナ体制評価の結果のため上表から除外しています。
