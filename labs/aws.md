# AWS 演習: RDS の公開アクセスを検査する

## 目標

AWS Security Hub の RDS.2 を題材に、公開アクセスが有効な RDS DB インスタンスを `UNHEALTHY` とするカスタム推奨事項を作成します。

## 手順

### 1. RDS の資産型を探す

```kusto
RawEntityMetadata
| where Environment =~ 'AWS'
| extend ResourceType = tostring(Identifiers.Type)
| where ResourceType contains 'rds'
| summarize Resources = count(), Samples = make_set(Name, 3) by ResourceType
```

結果から DB instance に対応する型を記録します。

### 2. プロパティを確認する

```kusto
RawEntityMetadata
| where Environment == 'AWS' and Identifiers.Type == 'REPLACE_WITH_DISCOVERED_TYPE'
| project Name,
          PubliclyAccessible = Record.PubliclyAccessible,
          RuntimeType = gettype(Record.PubliclyAccessible)
| take 20
```

値が boolean であり、AWS コンソールまたは API の設定と一致することを確認します。

### 3. 充足率を測る

```kusto
RawEntityMetadata
| where Environment == 'AWS' and Identifiers.Type == 'REPLACE_WITH_DISCOVERED_TYPE'
| summarize Total = count(), Present = countif(isnotnull(Record.PubliclyAccessible))
| extend CoveragePercent = round(100.0 * Present / Total, 1)
```

### 4. 製品内リファレンスと照合する

- [How to build a query](../HowToBuildaQuery.md) で、環境・型フィルター、`iff`、必須7列を確認します。
- [MDC が提供する AWS 参考 KQL](../MDC側で提供されている参考KQL%28AWS%29.md) で、`Record` 直下の参照方法と `join` などの参考ユースケースを確認します。
- 本演習は単一資産のフィールド判定です。履歴検索やライブ接続テストではないことを説明します。

### 5. 最終クエリを作る

[05-rds-no-public-access.kql](../queries/aws/05-rds-no-public-access.kql) の `Identifiers.Type` を Categories で確認します。公開アクセスが必要な承認済み DB がある場合は、タグなどの例外条件を追加します。

### 6. 正常・異常を検証する

検証用の2資産を準備するか、既存資産の設定を確認します。

| 資産 | AWS の設定 | 期待値 |
|---|---|---|
| private DB | Publicly accessible: No | `HEALTHY` |
| test public DB | Publicly accessible: Yes | `UNHEALTHY` |

本番データを公開状態へ変更しないでください。隔離した検証アカウントとセキュリティグループを使用します。

### 7. できること・できないことを確認する

| 要件 | 可否 | 理由 |
|---|---|---|
| 現在の `PubliclyAccessible` を判定 | 可能 | RDS レコードにフィールドが収集される場合 |
| 公開設定へ変更した日時を特定 | 不可 | `RawEntityMetadata` は変更履歴ではない |
| DB への実接続を試験 | 不可 | データプレーンの疎通機能ではない |
| AWS と GCP の DB を1つの推奨事項で評価 | 不可 | 複数環境の推奨事項は作成できない |

### 8. 登録して結果を確認する

重大度、説明、修復手順を設定し、小さいスコープへ登録します。評価後、[KQL の設計と検証](../docs/03-query-authoring.md) の ARG クエリで状態件数を確認します。

## 発展課題

- `StorageEncrypted` を追加し、公開と非暗号化を別々の推奨事項にする理由を説明する。
- 承認済み例外をタグで除外し、期限切れ例外を検出する運用を設計する。
- RDS.2 の Security Hub 結果とカスタム推奨事項の差分を調査する。
