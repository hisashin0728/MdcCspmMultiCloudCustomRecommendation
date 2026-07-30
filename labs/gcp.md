# GCP 演習: Compute Engine の IP forwarding を検査する

## 目標

Google Security Health Analytics の `IP_FORWARDING_ENABLED` を題材に、IP forwarding が有効な Compute Engine インスタンスを `UNHEALTHY` とするカスタム推奨事項を作成します。

## 手順

### 1. Compute Engine の資産型を探す

```kusto
RawEntityMetadata
| where Environment =~ 'GCP'
| extend ResourceType = tostring(Identifiers.Type)
| summarize Resources = count(), Samples = make_set(Name, 3) by ResourceType
| order by Resources desc
```

結果と GCP コンソールの資産名を比較し、Compute Instance の型を記録します。

### 2. フィールドを確認する

```kusto
RawEntityMetadata
| where Environment == 'GCP' and Identifiers.Type == 'REPLACE_WITH_DISCOVERED_TYPE'
| project Name,
          CanIpForward = Record.canIpForward,
          RuntimeType = gettype(Record.canIpForward)
| take 20
```

フィールドがない場合は `bag_keys(Record)` で実際の名前を確認します。収集されていない場合、このコントロールをカスタム推奨事項として公開しません。

### 3. 既定値を検証する

Google Compute Engine API では `canIpForward` の既定値は false です。一方、Defender のレコード上で null が API の既定値を表すのか、収集漏れなのかを複数資産で確認します。

### 4. 製品内リファレンスと照合する

[How to build a query](../HowToBuildaQuery.md) で、1つの推奨事項に複数環境を含められないこと、Timespan が不要であること、`Record` が GCP API のレスポンス構造であることを確認します。AWS 参考 KQL の構造パターンは GCP にも応用できますが、型名と API フィールドは GCP の Categories と実レコードから取得します。

### 5. 最終クエリを作る

[02-compute-ip-forwarding.kql](../queries/gcp/02-compute-ip-forwarding.kql) の `REPLACE_WITH_GCP_COMPUTE_INSTANCE_TYPE` を Categories で確認した `Identifiers.Type` に置換します。ルーターやネットワーク仮想アプライアンスとして承認された VM は、ラベルやプロジェクトで適用除外にします。

### 6. 正常・異常を検証する

| 資産 | GCP の設定 | 期待値 |
|---|---|---|
| standard VM | IP forwarding: Off | `HEALTHY` |
| isolated router VM | IP forwarding: On | `UNHEALTHY` または承認済み例外 |

### 7. できること・できないことを確認する

| 要件 | 可否 | 理由 |
|---|---|---|
| 現在の `canIpForward` を判定 | 可能 | Compute Instance レコードにフィールドが収集される場合 |
| 過去30日間の設定変更回数を算出 | 不可 | 最新レコードの評価であり履歴テーブルではない |
| 実際のパケット転送を試験 | 不可 | データプレーンの通信試験ではない |
| AWS EC2 と GCP VM を同じ推奨事項で比較 | 不可 | 複数環境の推奨事項は作成できない |

### 8. 登録して結果を確認する

小さいスコープで登録し、評価結果と GCP コンソールの設定を比較します。プロパティが null の資産は個別に調べ、誤って正常扱いまたは不健全扱いしていないことを確認します。

## 発展課題

- `PUBLIC_IP_ADDRESS` と組み合わせず、別の推奨事項に分ける利点を説明する。
- インスタンスラベルによるネットワークアプライアンスの例外条件を設計する。
- Security Health Analytics の結果と Defender の評価時刻の差を測定する。
