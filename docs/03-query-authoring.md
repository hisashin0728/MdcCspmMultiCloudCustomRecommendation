# 3. KQL の設計と検証

## 3.1 基本テンプレート

```kusto
RawEntityMetadata
| where Environment == 'AWS' and Identifiers.Type == 'REPLACE_WITH_DISCOVERED_TYPE'
| extend Setting = Record.REPLACE_WITH_PROPERTY
| extend IsUnhealthy = REPLACE_WITH_BOOLEAN_EXPRESSION
| extend HealthStatus = iff(IsUnhealthy, 'UNHEALTHY', 'HEALTHY')
| project Id, Name, Environment, Identifiers, AdditionalData, Record, HealthStatus
```

このテンプレートの変更箇所は、環境、資産型、設定値の JSON パス、違反条件です。

## 3.2 ベストプラクティスを真偽条件にする

自然言語の要件を、次の順に分解します。

1. 対象: どのクラウド資産か。
2. 証拠: どのフィールドまたは配列が設定を表すか。
3. 期待値: 有効、無効、空でない、許可リスト内など。
4. 違反条件: `IsUnhealthy` が `true` になる条件。
5. 適用除外: サービス管理資産、意図した公開資産など。

例として「RDS DB インスタンスは公開しない」は次のように分解できます。

- 対象: RDS DB instance
- 証拠: `PubliclyAccessible`
- 期待値: `false`
- 違反条件: `PubliclyAccessible == true`
- 適用除外: 承認済みの公開 DB があれば、別途タグやアカウントで除外

## 3.3 配列を評価する

ファイアウォール規則などは配列です。`mv-apply` で資産ごとに違反の有無を集約し、元の1行を維持します。

```kusto
RawEntityMetadata
| where Environment == 'AWS' and Identifiers.Type == 'ec2.securitygroup'
| mv-apply Permission = Record.IpPermissions on (
    summarize IsUnhealthy = countif(
        tostring(Permission.IpProtocol) in ('-1', 'tcp')
        and toint(Permission.FromPort) <= 22
        and toint(Permission.ToPort) >= 22
        and tostring(Permission.IpRanges) has '0.0.0.0/0'
    ) > 0
)
| extend HealthStatus = iff(IsUnhealthy, 'UNHEALTHY', 'HEALTHY')
| project Id, Name, Environment, Identifiers, AdditionalData, Record, HealthStatus
```

配列内の実際の CIDR 表現が文字列配列かオブジェクト配列かを確認し、必要に応じて内側でも `mv-expand` してください。

## 3.4 製品内 KQL から確認できる設計パターン

| パターン | 製品内の参考ユースケース | 設計上の意味 |
|---|---|---|
| 単一資産の値を判定 | KMS キーの `PendingDeletion` | `Record` 直下の API フィールドを `iff` で分類できる |
| 現在レコード内の日時を比較 | 停止から30日超の EC2 | `Record` に日時があれば `ago()` と比較できる。履歴検索ではない |
| 配列を展開 | EC2 セキュリティグループの関連付け、ルートテーブル | `mv-expand` で API 配列の要素を評価できる |
| 関連型を結合 | CloudTrail と Trail Status | 同じ割り当て範囲に両方のレコードがあれば `join` できる |
| 任意の関連レコードを結合 | EBS Snapshot と CreateVolumePermission | `leftouter` と null を使い、関連レコードがある資産だけを不健全にできる |
| 関連レコードの不在を検査 | VPC と Active Flow Log | 親資産を起点に `leftanti` で関連資産不在を判定できる |
| 健全・不健全集合を統合 | 未使用セキュリティグループ、VPC Flow Log | `union` 後も資産ごとの `HealthStatus` を返せる |
| 集合への所属を判定 | VPC Endpoint、KMS Rotation、IAM Policy | 補助クエリを `summarize` し、`in` / `!in` で親資産を分類できる |
| JSON 文字列を再解析 | ECR Policy、S3 Bucket Policy、IAM Policy Version | `parse_json(tostring(...))` 後に Statement 配列を展開できる |

### 関連資産を安全に照合する

製品内例では資産 ID や名前だけで関連付けるクエリもあります。ただし、推奨事項が複数アカウントまたは複数リージョンへ割り当てられる場合、同名資産を誤って結合する可能性があります。API 上で識別子が全スコープ一意であると確認できない限り、次のようにコンテキストを含む複合キーを作ります。

```kusto
| extend AccountId = tostring(Identifiers.AmazonAccountId),
         Region = tostring(AdditionalData.Region),
         ResourceKey = tostring(Record.REPLACE_WITH_RESOURCE_ID)
```

`join`、`in` 用の集合、`leftanti` の両側で同じキーを使用します。グローバル資産やアカウント単位資産では、サービスの識別規則に合わせて不要なリージョンを除きます。

### `Record` の形と実行時型を確認する

同じ `Identifiers.Type` に本体行と属性行が含まれる製品例があります。必要なフィールドの存在、`gettype()`、充足率を確認し、属性名などで対象行を識別してから判定します。日時、真偽値、ポリシー JSON は、参考例の変換式をそのまま信頼せず、実データで変換結果を確認してください。

製品内参考例には最後の `project` が省略されたものや、`iff` ではなく健全・不健全集合へ直接状態値を設定するものがあります。一方、HELP は `iff` と必須7列を要求しています。参考例は構造を理解する資料として扱い、登録用クエリでは HELP の契約を優先します。本教材では最終出力に必ず次を付けます。

```kusto
| project Id, Name, Environment, Identifiers, AdditionalData, Record, HealthStatus
```

また、HELP に従い Timespan は指定しません。評価サービスが各実行で最新レコードを選びます。ARN やアカウント ID の固定フィルターも、意図的に対象を限定するとき以外は追加しません。

## 3.5 検証手順

1. `where` までを実行し、対象件数が期待どおりか確認する。
2. 設定値だけを `project` し、値、型、null を確認する。
3. `Name`、設定値、`IsUnhealthy` を返し、正常・異常の既知資産と比較する。
4. 必須7列を返す最終形にする。
5. 小さいスコープで登録し、評価完了後にポータルと ARG の結果を確認する。

## 3.6 公開前チェックリスト

- [ ] `Identifiers.Type` を実環境から取得した
- [ ] `microsoft.awsconnector/*` ではなく Categories の型を使用した
- [ ] API フィールドを `Record` 直下から参照した
- [ ] JSON パスの大文字小文字を確認した
- [ ] 候補フィールドの充足率を確認した
- [ ] 同じ型に異なる `Record` 形状がないか確認した
- [ ] JSON文字列を必要に応じて再解析した
- [ ] 正常資産と異常資産の両方でテストした
- [ ] 対象資産を不健全な行だけに絞っていない
- [ ] 必須7列を返している
- [ ] `HealthStatus` は2つの許可値だけを返す
- [ ] 例外条件が文書化されている
- [ ] 結合・集合キーに必要なアカウントとリージョンを含めた
- [ ] 修復手順が資産所有者に実行可能である
- [ ] ベンダー資料のコントロール ID と確認日を記録した
- [ ] Timespan を追加していない
- [ ] 1つの推奨事項に複数環境を含めていない

## 3.7 登録後の結果確認

カスタム推奨事項が評価された後は、Azure Resource Graph で件数を確認できます。評価キーや表示名は環境の結果から特定してください。

```kusto
securityresources
| where type =~ 'microsoft.security/assessments'
| extend AssessmentName = tostring(properties.displayName)
| where AssessmentName =~ 'REPLACE_WITH_RECOMMENDATION_NAME'
| summarize Resources = count() by Status = tostring(properties.status.code)
```
