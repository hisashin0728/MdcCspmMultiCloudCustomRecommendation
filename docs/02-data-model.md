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
