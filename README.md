# Defender for Cloud: AWS/GCP カスタム推奨事項ワークショップ

Microsoft Defender for Cloud の Defender CSPM で、AWS/GCP 資産を対象とする KQL ベースのカスタム推奨事項を設計するための教材です。クラウド標準のベストプラクティスを、Defender for Cloud が収集した資産メタデータに照合する手順を扱います。

## AI利用に関する注記

本コンテンツは、生成AIの支援を利用して作成しています。内容には誤り、不完全な記述、製品やクラウドサービスの最新仕様との差異が含まれる可能性があります。利用者はMicrosoft、AWS、Google Cloudの公式資料および実環境のデータと照合し、人によるレビューと検証を行ったうえで使用してください。

## 重要な前提

- カスタム推奨事項の入力は `RawEntityMetadata` です。Azure Resource Graph の `securityresources` は、作成済み推奨事項の結果を分析する用途で使用します。
- クエリは対象資産をすべて返し、`Id`、`Name`、`Environment`、`Identifiers`、`AdditionalData`、`Record`、`HealthStatus` の7列を出力します。
- `HealthStatus` は `HEALTHY` または `UNHEALTHY` にします。
- AWS/GCP コネクタが返す `Identifiers.Type` と `Record` は、コネクタや資産種別によって異なります。`Record` はクラウド API のレスポンス構造を保持するため、サンプルの型名やプロパティを実環境の探索結果と照合してから公開してください。
- 同じ `Identifiers.Type` に異なる形の `Record` が含まれる場合があります。関連資産の結合では、資産名だけでなくアカウント・リージョンを含む一意なキーを検証してください。
- プロパティの欠損は設定不良を意味しません。欠損を `UNHEALTHY` に変換する前に、コネクタがその設定を収集していることを確認してください。

## [必須] `RawEntityMetadata` のフィールド情報から仕様上できないこと

| できないこと | 理由・代替策 |
|---|---|
| 1つの推奨事項で AWS と GCP を同時評価する | 製品仕様で複数環境を対象にできません。環境ごとに推奨事項を分けます。 |
| 過去の全構成や変更履歴を時系列分析する | 評価サービスは実行時に各資産の最新レコードを選びます。履歴分析にはクラウド監査ログや別のログ基盤を使用します。 |
| `Record` に収集されない設定を判定する | 判定材料はクラウド API から返されて `Record` に格納された情報に限られます。別 API、データプレーン、ライブ疎通が必要な検査は組み込み評価や外部処理と併用します。 |
| 存在しない資産を、比較対象なしで直接返す | KQL の出力行に対応する資産が必要です。親資産を起点に関連資産の不在を `leftanti` などで判定できる場合だけ検査できます。 |
| 割り当て範囲外の資産を評価する | 評価サービスは推奨事項が割り当てられた資産に対して実行します。アカウント ID や ARN の固定フィルターは意図した限定時だけ使用します。 |
| 欠損値だけから設定不良と断定する | API の既定値、権限不足、未収集を区別できない場合があります。フィールド充足率と既知資産で検証します。 |
| KQL から設定を修復する | クエリは読み取りと健全性分類のみです。修復はクラウド側の手順または自動化を別途用意します。 |

### 仕様上実現できないカスタム推奨事項の例10個

次の例は、通常の構成メタデータだけが `RawEntityMetadata` に収集される前提では、カスタム推奨事項単独で実現できません。必要な事実が別の `Record` として収集される場合は、その収集範囲に限って再評価できます。

| # | 実現できない推奨事項例 | 実現できない理由 | 代替手段 |
|---:|---|---|---|
| 1 | 過去90日間にセキュリティグループが何回変更されたか評価する | 最新の構成レコードは変更イベントの履歴を保持しません | AWS CloudTrail、Google Cloud Audit Logs、SIEMで監査ログを集計する |
| 2 | インターネットからEC2、RDS、GCE VMへ実際に接続できるか検査する | 構成上の公開設定は判定できても、DNS、経路、ファイアウォール、サービス待受を含む実通信は試行できません | 外部Attack Surface Management、脆弱性スキャナー、接続テストを使用する |
| 3 | S3バケットやCloud Storage内のファイルに個人情報が含まれていないか検査する | `RawEntityMetadata` は通常、オブジェクト本文やファイル内容を保持しません | DSPM、DLP、Macie、Sensitive Data Protectionを使用する |
| 4 | IAMユーザーが特定ロールを実際に引き受けられるか完全判定する | ポリシーのAllowだけでなく、信頼ポリシー、明示的Deny、SCP、Permission Boundary、Condition、実行時コンテキストの総合評価が必要です | IAM Access Analyzer、Policy Simulator、CloudTrailの `AssumeRole` イベントを使用する |
| 5 | EC2やGCE VM内の全パッケージに既知の脆弱性がないことを証明する | 構成メタデータだけではゲストOS内のパッケージ、バージョン、実行状態を網羅できません | Defender for Servers、Amazon Inspector、VM Manager等の脆弱性管理を使用する |
| 6 | Secrets ManagerやSecret Managerの秘密値が十分に強く、漏えいしていないか検査する | 秘密値そのものは構成メタデータとして公開されず、値の強度や漏えい状況を評価できません | シークレットスキャン、漏えい検知、ローテーション管理を使用する |
| 7 | AWSアカウントにCloudTrailが1つも存在しないことを、返却対象となる親資産なしで検出する | 存在しないTrailに対応する出力行を生成できません | アカウント等の収集済み親資産を起点にするか、AWS Config／Security Hubのアカウント単位評価を使用する |
| 8 | 1つの推奨事項でAWS EC2とGCP Compute Engineを横断比較して状態を返す | 1つのカスタム推奨事項で複数環境を対象にできません | AWS用とGCP用に推奨事項を分け、結果側で集約する |
| 9 | AWS OrganizationsまたはGCP組織内の全アカウント／プロジェクトがMDCへ接続済みであることを証明する | 割り当て外または未接続の環境は、評価入力に現れないため存在を把握できません | Organizations／Cloud Resource ManagerのインベントリとMDC接続一覧を外部で突合する |
| 10 | 不健全と判定したS3公開設定、IAMポリシー、ファイアウォール規則をKQLで自動修復する | カスタム推奨事項のKQLは読み取りと分類のみで、クラウドAPIへの変更操作を実行しません | Workflow Automation、Logic Apps、Lambda、Cloud Functions等で承認付き修復を構成する |

詳細なフィールド境界、可能な判定、代替策は [`RawEntityMetadata` とフィールド](docs/02-data-model.md) を参照してください。

## 対象読者

- Defender CSPM を有効化済みの環境を管理するクラウドセキュリティ担当者
- AWS Security Hub CSPM または Google Security Command Center の知識を Defender for Cloud に展開したい担当者
- KQL の基本的な `where`、`extend`、`project` を理解している利用者

## 前提条件

- AWS または GCP を Defender for Cloud に接続済みであること
- Defender CSPM プランが有効であること
- カスタム標準と推奨事項を作成できる Owner または Security Admin 相当の権限
- 対象スコープに、検証する資産が少なくとも1つ存在すること

## 学習の流れ

1. [カスタム推奨事項の概要](docs/01-overview.md)
2. [`RawEntityMetadata` とフィールド](docs/02-data-model.md)
3. [KQL の設計と検証](docs/03-query-authoring.md)
4. [AWS のクエリ例20個](docs/04-aws-examples.md)
5. [GCP のクエリ例20個](docs/05-gcp-examples.md)
6. [できること・できないこと](docs/06-capabilities-limitations.md)
7. [演習](labs/README.md)
8. [情報源](docs/references.md)
9. [製品内 HELP: How to build a query](HowToBuildaQuery.md)
10. [MDC が提供する AWS 参考 KQL](MDC側で提供されている参考KQL%28AWS%29.md)

## サンプルの成熟度

| 表記 | 意味 |
|---|---|
| 実行テンプレート | 必須7列と判定構造をそのまま利用できる |
| 検証必須 | ベストプラクティスの判定ロジック例。型名とプロパティを実環境で確認してから利用する |

本リポジトリの40例はベンダーのベストプラクティスを KQL に変換する教材です。Defender for Cloud コネクタの公開スキーマ契約では個々の `Record` 内部フィールドが固定されていないため、各例を本番環境へ無検証で登録することは想定していません。

## 免責

本ドキュメントはMicrosoftの公式ガイド、製品仕様書、サポート声明ではありません。作成時点でMicrosoft Defender for Cloudの製品画面に掲載されているHELP、参考KQL、公開ドキュメント、およびAWS／Google Cloudの公開情報を読み解いて得られた情報を基に作成した参考資料です。

本ドキュメントに記載した `Identifiers.Type`、`Record` のフィールド、関連レコード、KQL例が、すべてのお客様環境で取得できること、同じ構造で提供されること、完全または正確であること、期待どおりに動作することを保証するものではありません。取得できるデータは、接続方式、コネクタのバージョン、プラン、権限、リージョン、クラウド側API、対象資産、割り当てスコープなどによって異なる可能性があります。

サンプルは学習と検証を目的としています。本番利用前に、最新の公式資料、製品画面のCategories、対象環境の `RawEntityMetadata`、正常・異常の既知資産を使用して、人によるレビューと検証を行ってください。また、組織の例外、脅威モデル、データ分類、可用性要件を反映してから運用してください。