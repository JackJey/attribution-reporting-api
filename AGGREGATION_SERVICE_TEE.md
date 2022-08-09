# Aggregation Service for the Attribution Reporting API

Authors:

- Martin Pál <mpal@google.com>
- Ruchi Lohani <rlohani@google.com>

## Introduction

The Attribution Reporting API is designed to provide two types of reports: an "event-level report," which corresponds to a single attributed event, and a summary report. Summary reports can be used to provide aggregated statistics from Attribution Reporting API events in the client software, such as a [Chrome browser](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md) or an [Android device](https://developer.android.com/design-for-safety/ads/attribution).

Attribution Reporting API は、1 つの帰属イベントに対応する「イベントレベルレポート」と「サマリーレポート」の 2 種類のレポートを提供するように設計されています。サマリーレポートは、Attribution Reporting API のイベントから、[Chrome ブラウザ](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md)や[Android デバイス](https://developer.android.com/design-for-safety/ads/attribution)などのクライアントソフトウェアで集計した統計を提供するために使用することができます。

In this document, you'll find the proposed server-side mechanisms for the aggregation service, which allows adtechs to create summary reports.

このドキュメントでは、アドテクがサマリーレポートを作成するためのアグリゲーションサービスのサーバーサイドの仕組みについて提案されています。

- [Proposed design principles](#proposed-design-principles)
- [Key terms](#key-terms)
- [Aggregation workflow](#aggregation-workflow)
- [Security considerations](#security-considerations)
- [Privacy considerations](#privacy-considerations)
- [Mechanics of aggregation](#mechanics-of-aggregation)
- [Initial experiment plans](#initial-experiment-plans)
- [Open questions](#open-questions)

_Note that this is an explainer, the first step in the standardization process. The aggregation service is not finalized. The service as described (including report metrics, privacy mechanisms, and method of execution) is subject to change as we incorporate ecosystem feedback and iterate._

_注意：これは標準化の最初のステップである explorer です。アグリゲーションサービスは最終的なものではありません。説明されているサービス（レポートメトリクス、プライバシーメカニズム、実行方法を含む）は、エコシステムのフィードバックを取り入れ、繰り返し行うため、変更される可能性があります_。

## Proposed design principles

The aggregation service for the Attribution Reporting API intends to meet a set of [privacy and security goals](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md#privacy-considerations) through technical measures. This includes:

Attribution Reporting API のアグリゲーションサービスは、技術的な手段によって一連の[プライバシーとセキュリティの目標](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md#privacy-considerations)を満たすことを意図しています。これには以下が含まれます。

1. Uphold The Privacy Sandbox design goals for privacy protections in Attribution Reporting. Specifically, we intend to provide appropriate infrastructure for noise addition and aggregation, aligned with the long-term goal of differential privacy.

1. アトリビューション・レポートにおけるプライバシー保護のための Privacy Sandbox の設計目標を維持すること。具体的には、差分プライバシーという長期的な目標に沿って、ノイズの追加と集計のための適切なインフラを提供する予定である。

1. Prevent inappropriate access to raw attributed conversion data or other intermediate data through technical enforcement.

1. 技術的な実施により、属性変換の生データまたはその他の中間データへの不適切なアクセスを防止する。

1. Allow adtechs to retain control over the data they've collected and access noisy aggregated data without sharing their data with any third party.

1. アドテクが収集したデータのコントロールを保持し、第三者とデータを共有することなく、ノイジーな集計データにアクセスできるようにする。

1. Support flexible, scalable, and extensible aggregation strategies and on-demand access to the aggregation infrastructure, so that adtechs can choose when and how often to generate summary reports.

1. アドテクがサマリーレポートを作成するタイミングや頻度を選択できるよう、柔軟でスケーラブル、かつ拡張性の高いアグリゲーション戦略とアグリゲーションインフラへのオンデマンドアクセスをサポートします。

1. Provide open and transparent implementations for any infrastructure outside of the client.

1. クライアント外のあらゆるインフラに対して、オープンで透過的な実装を提供すること。

## Key terms

Before reading this explainer, it will be helpful to familiarize yourself with key terms and concepts. These have been ordered non-alphabetically to build knowledge based on previous terms. All of these terms will be reintroduced and further described throughout this proposal.

この説明書を読む前に、重要な用語と概念に慣れておくと便利です。これらの用語は、以前の用語を基に知識を深めるために、アルファベット順ではありません。これらの用語はすべて、この提案の中で再度紹介され、さらに説明されます。

- _Adtech_: a company that provides services to deliver ads. This term is used to generally describe companies who are most likely to use the aggregation service.
- _Client software:_ shorthand for the implementation of the Attribution Reporting API on a browser or device (such as a [Chrome browser](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md) or an [Android device](https://developer.android.com/design-for-safety/ads/attribution)).
- _Aggregatable reports_: encrypted reports sent from individual user devices which contain data about individual conversions. Conversions (sometimes called _attribution trigger events_) and associated metrics are defined by the advertiser or adtech. [Learn more about aggregatable reports](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md).
- _Aggregation service (this proposal)_: an adtech-operated service that processes data from aggregatable reports to create a summary report.
- _Summary reports_: the result of noisy aggregation applied to a batch of aggregatable reports. Summary reports allow for greater flexibility and a richer data model than event-level reporting, particularly for some use-cases like conversion values.
- _Reporting origin_: the entity that receives aggregatable reports–in other words, the adtech that called the Attribution Reporting API. Aggregatable reports are sent from user devices to a [well-known URL](https://datatracker.ietf.org/doc/html/rfc5785) associated with the reporting [origin](https://developer.mozilla.org/en-US/docs/Glossary/Origin).
- _Contribution bounding_: aggregatable reports may contain an arbitrary number of counter increments. For example, a report may contain a count of products that a user has viewed on an advertiser's site. The sum of increments in all aggregatable reports related to a single source event must not exceed a given limit, `L1=2^16`. [Learn more in the aggregatable reports explainer](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md#contribution-bounding-and-budgeting).
- _Attestation_: a mechanism to authenticate software identity, usually with [cryptographic hashes](https://en.wikipedia.org/wiki/Cryptographic_hash_function) or signatures. In this proposal, attestation matches the code running in the adtech-operated aggregation service with the open source code.
- _Trusted Execution Environment (TEE)_: a dedicated, closed execution context that is isolated through hardware memory protection and cryptographic protection of storage. The TEE's contents are protected from observation and tampering by unauthorized parties, including the root user.
- _Key management service_: a centralized component tasked with provision of decryption keys to appropriately secured aggregation server instances. Provision of public encryption keys to end user devices and key rotation also fall under key management.
- _Coordinator:_ an entity responsible for key management and aggregatable report accounting. The coordinator maintains a list of hashes of approved aggregation service configurations and configures access to decryption keys.

## Aggregation workflow

Adtechs use the aggregation service to generate summary reports, which offer detailed conversion data (such as purchase values and cart contents) and flexibility for click and view data. The overall aggregation workflow is as follows:

アドテックでは、アグリゲーションサービスを利用して、サマリーレポートを作成し、コンバージョンデータ（購入金額やカート内容など）の詳細や、クリックやビューのデータに対して柔軟な対応ができるようになっています。全体の集計ワークフローは以下の通りです。

1.An aggregatable report is generated on client software (such as the [Chrome browser](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md#aggregatable-reports) or [Android device](https://developer.android.com/design-for-safety/ads/attribution)). The client software encrypts the contents of the report and sends it to the adtech's reporting origin.

1.クライアントソフト（[Chrome ブラウザ](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md#aggregatable-reports)や[Android 端末](https://developer.android.com/design-for-safety/ads/attribution)など）上で集計可能なレポートが生成されます。クライアントソフトウェアはレポートの内容を暗号化し、アドテックのレポートオリジンに送信する。

1.The adtech collects aggregatable reports and assembles them [into batches](#disjoint-batches) (for example, all the reports generated for a given advertiser on a given date).

1.アドテクノロジーは、集計可能なレポートを収集し、バッチ（#disjoint-batches）にまとめます（例えば、ある広告主に対してある日付に生成されたすべてのレポート）。

1.The adtech sends a batch of reports for aggregation to its aggregation service.

1.アドテックがアグリゲーションサービスにアグリゲーション用のレポートを一括して送信する。

1.To access decrypted versions of the reports, the adtech-operated aggregation service runs an approved version of the aggregation logic in a Trusted Execution Environment (TEE). Decryption keys will only be released to TEEs running an approved version of the aggregation logic.

1.レポートの解読されたバージョンにアクセスするために、adtech が運営するアグリゲーションサービスは、信頼できる実行環境（TEE）でアグリゲーションロジックの承認されたバージョンを実行します。復号化キーは、承認されたバージョンのアグリゲーションロジックを実行する TEE にのみリリースされる。

1.The aggregation service decrypts and aggregates the batched aggregatable reports to generate a summary report.

1.アグリゲーションサービスは、バッチされたアグリゲーション可能なレポートを復号化し、サマリーレポートを生成するためにアグリゲーションする。

1. The summary report is released from the aggregation service, to be used by the adtech.

1.アグリゲーションサービスから、アドテクで利用するためのサマリーレポートを公開します。

![Aggregation workflow diagram](aggregation-service.svg)

_Multiple clients encrypt aggregatable reports and send the reports to an adtech-owned reporting origin. The adtech's aggregation service transforms this data into a summary report._

複数のクライアントが集計可能なレポートを暗号化し、アドテック所有のレポートオリジンに送信します。アドテックのアグリゲーションサービスは、このデータをサマリーレポートに変換します。

## Security considerations

The aggregation service design outlined in this proposal uses a combination of TEEs and attestation to an independent coordinator to ensure that all data remains secure.

本提案で示されたアグリゲーションサービスデザインは、TEE の組み合わせと独立したコーディネーターへの認証により、すべてのデータの安全性を確保する。

- Decryption and aggregation of raw aggregatable report data happens within a secure and isolated Trusted Execution Environment. This prevents adtech from accessing the decrypted inputs, intermediate data, and the decryption key.

- 集計可能な生のレポートデータの復号化と集計は、安全で隔離された Trusted Execution Environment 内で行われます。これにより、アドテックは復号化された入力、中間データ、および復号化キーにアクセスすることができなくなります。

- Attestation ensures that an adtech-operated aggregation service runs an approved codebase and can access the decryption keys needed to decrypt and aggregate data.

- 認証は、アドテクノロジーが運営するアグリゲーションサービスが承認されたコードベースを実行し、データの復号化とアグリゲーションに必要な復号化キーにアクセスできることを保証します。

  - Splitting decryption keys among multiple coordinators increases the difficulty of compromising this trust model.

  - 復号化キーを複数のコーディネーターに分割することで、この信頼モデルを崩すことが難しくなる。

- Open source implementations of the aggregation service and coordinator logic ensure that these systems are publicly accessible and can be inspected by a broad set of stakeholders.

- アグリゲーションサービスとコーディネータロジックのオープンソース実装により、これらのシステムは一般に公開され、幅広いステークホルダーが検査することができます。

### Trusted Execution Environment

A Trusted Execution Environment (TEE) is a combination of hardware and software mechanisms that allows for code to execute in isolation, not observable by any other process regardless of the credentials used. The code running in TEE will be open sourced and audited to ensure proper operation. Only TEE instances running an approved version of the aggregation code can access decryption keys.

Trusted Execution Environment (TEE) は、ハードウェアとソフトウェアのメカニズムの組み合わせで、使用される認証情報に関係なく、他のプロセスから観察できないように、コードを分離して実行できるようにするものです。TEE で実行されるコードは、オープンソース化され、適切な動作を保証するために監査されます。アグリゲーションコードの承認されたバージョンを実行している TEE インスタンスのみが、復号化キーにアクセスすることができます。

The code running within a TEE performs the following tasks:

TEE 内で実行されるコードは、以下のタスクを実行します。

- Parse batches of aggregatable reports, decrypt, and aggregate them

- 集計可能なレポートのバッチを解析し、復号化し、集計を行う

- Add noise to the aggregates and output a summary report for each batch processed

- 集計にノイズを加え、処理されたバッチごとにサマリーレポートを出力

- Handle error reporting, logging, crashes and stack traces, access to external storage and network requests in a way that aids usability and troubleshooting while protecting raw and intermediate data at all times.

- エラーレポート、ログ、クラッシュ、スタックトレース、外部ストレージへのアクセス、ネットワークリクエストを、生データと中間データを常に保護しながら、ユーザビリティとトラブルシューティングを支援する方法で処理します。

Adtechs will operate their own TEE-based aggregation service deployment. Adtechs will also control the collection and storage of requested aggregatable reports, as well as storage and usage of summary reports produced via aggregation. The TEE ensures that the adtech cannot decrypt the aggregatable reports or access any intermediate data.

Adtech は、独自の TEE ベースのアグリゲーションサービス展開を運営します。また、アドテクノロジーは、要求されたアグリゲーションレポートの収集と保管、およびアグリゲーションによって作成されたサマリーレポートの保管と使用を管理します。TEE は、アドテクがアグリゲーションレポートを解読したり、中間データにアクセスできないことを保証します。

### Attestation and the coordinator

The code running within the TEE is the only place in the system where raw reports from client software can be decrypted. The code will be open sourced so it can be audited by security researchers, privacy advocates, and adtechs.

TEE 内で実行されるコードは、システム内で唯一、クライアントソフトウェアからの生のレポートを解読できる場所です。このコードは、セキュリティ研究者、プライバシー擁護者、広告技術者が監査できるように、オープンソース化される予定です。

Google will periodically release binary images of the aggregation server code for TEE deployment. A cryptographic hash of the build product (the image to be deployed on the TEE) is obtained as part of the build process. The build is reproducible so that anyone can build binaries from source and verify they are identical to the images released by Google.

Google は、TEE 展開用のアグリゲーションサーバコードのバイナリイメージを定期的にリリースします。ビルドプロセスの一部として、ビルドプロダクト（TEE にデプロイされるイメージ）の暗号化ハッシュが取得されます。ビルドは再現可能であるため、誰でもソースからバイナリをビルドし、Google がリリースしたイメージと同一であることを検証することができる。

The coordinator has several responsibilities, including: maintain and publish a list of hashes of the authorized binary images, operate shared key services, and operate a tracking system for aggregatable event metadata processing.

コーディネータは、許可されたバイナリ画像のハッシュ一覧の維持と公開、共有鍵サービスの運用、集約可能なイベントメタデータ処理の追跡システムの運用などの責任を持つ。

When a new approved version of the aggregation service binary is released, the coordinator will add the new version to its list of authorized images. If an image is found in retrospect to contain a critical security or functional flaw, it can be removed from the list. Images older than a certain age will also be periodically retired from the list.

アグリゲーション・サービス・バイナリの新しい承認済みバージョンがリリースされると、コーディネータはその新しいバージョンを承認済み画像のリストに追加します。もし、ある画像がセキュリティ上または機能上の重大な欠陥を含んでいることが後から判明した場合、その画像をリストから削除することができます。また、一定期間以上経過したイメージは、定期的にリストから抹消されます。

The coordinator operates the key management system. Encryption keys are necessary for the client software to fetch and encrypt aggregatable reports. The encryption keys are public. Decryption keys are required for the aggregation service to process the data from the aggregatable reports. The decryption keys are released only to TEE images whose cryptographic hash matches one of the images on the authorized images list.

コーディネーターは、鍵の管理システムを操作する。暗号化キーは、クライアントソフトウェアが集計可能なレポートを取得し、暗号化するために必要である。暗号化キーは公開される。復号化キーは、アグリゲーションサービスがアグリゲーションレポートからデータを処理するために必要である。復号化キーは、暗号化ハッシュが許可されたイメージリストのイメージの 1 つと一致する TEE イメージにのみ解放される。

The coordinator operates a system that records metadata about processed aggregatable events (these events encapsulate the user action which lead to aggregatable reports). This is instrumental in preventing aggregatable reports from being reused in multiple batches to craft summary reports which may reveal PII.

コーディネーターは、処理された集計可能なイベントに関するメタデータを記録するシステムを運用する（これらのイベントは、集計可能なレポートにつながるユーザーアクションをカプセル化する）。これは、集計可能なレポートが複数のバッチで再利用され、PII を明らかにする可能性のある要約レポートを作成することを防止するのに役立ちます。

## Privacy considerations

In the proposed architecture for the Attribution Reporting API, logic in the client software and server work together to keep data private and secure.

Attribution Reporting API の提案アーキテクチャでは、クライアントソフトウェアとサーバーのロジックが連携して、データの機密性と安全性を確保します。

When a summary report is released, the adtech learns some information about users whose contributions are included in the summary report. A goal for this system is to provide a framework that supports [differential privacy](https://en.wikipedia.org/wiki/Differential_privacy) (DP), as such we use tools and mechanisms to quantify and limit the amount of information revealed about any individual user.

要約レポートが公開されると、アドテクノロジーは要約レポートに含まれるユーザーに関するいくつかの情報を知ることになります。このシステムの目標は、[Differential Privacy](https://en.wikipedia.org/wiki/Differential_privacy) (DP) をサポートするフレームワークを提供することです。そのため、個々のユーザーについて明らかにする情報量を定量化し制限するためのツールやメカニズムを使用します。

### Added noise to summary reports

The aggregation service uses an [additive noise mechanism](https://en.wikipedia.org/wiki/Additive_noise_mechanisms). This means that a certain amount of statistical noise is added to each aggregate value before its release in a summary report.

集計サービスでは、[Additive noise mechanism](https://en.wikipedia.org/wiki/Additive_noise_mechanisms)を使用しています。これは、サマリーレポートとして公開する前に、各集計値に一定量の統計的ノイズを付加することを意味します。

We propose to add noise drawn from a discrete version of the [Laplace distribution](https://en.wikipedia.org/wiki/Laplace_distribution), scaled based on the `L1` sensitivity, which is enforced by the client (known as [contribution bounding](#contribution bounding)) (`2^16`) and a desired privacy parameter `epsilon`.
我々は、[ラプラス分布](https://en.wikipedia.org/wiki/Laplace_distribution)の離散版から引き出されたノイズを、クライアントによって強制される`L1`感度に基づいてスケーリングし、[寄与バウンディング](#contribution bounding) (`2^16`)と望ましいプライバシーパラメータ`epsilon`を加えることを提案します。

### Contribution bounding

The client API [limits the magnitude of contributions](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md#contribution-bounding-and-budgeting) from aggregatable reports linked to a single source event. This design provides a proper foundation to control the sensitivity of summary reports, which is critical for calibration of the amount of noise to add. Sensitivity is reflected in the amount a summary report could change if any given aggregatable report were removed from the process.

クライアント API は、単一のソースイベントにリンクされた集計可能なレポートからの[寄与の大きさを制限する](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md#contribution-bounding-and-budgeting)ものです。この設計は、追加するノイズの量の較正に重要な、要約レポートの感度を制御するための適切な基盤を提供します。感度は、任意の集計可能なレポートがプロセスから削除された場合に、サマリーレポートが変化し得る量に反映されます。

### "No duplicates" rule

To gain insight into the contents of a specific aggregatable report, an attacker might make multiple copies of it, and include the copies in a single aggregation batch (as duplicates) or in multiple batches. Because of this, the aggregation service enforces a "no duplicates" rule:

特定の集計可能なレポートの内容を把握するために、攻撃者はそのレポートの複数のコピーを作成し、そのコピーを単一の集計バッチに含める（複製として）か、複数のバッチに含める可能性があります。このため、集計サービスでは、「重複禁止」ルールを適用しています。

- No aggregatable report can appear more than once within a batch.

- 集計可能なレポートがバッチ内で複数回表示されることはありません。

- No aggregatable report can appear in more than one batch or contribute to more than one summary report.

- 集計可能なレポートは、複数のバッチに表示されたり、複数のサマリーレポートに貢献することはできません。

The no-duplicates rule is enforced during aggregation. If duplicates are found, these batches may be rejected or duplicates may be filtered out.

重複禁止ルールは、集計時に強制的に適用されます。重複が見つかった場合、これらのバッチは拒否されるか、重複がフィルタリングされるかもしれません。

#### Disjoint batches

It is not technically practical to keep track of every single aggregatable report submitted for aggregation to check for batch disjointness, that is, that batches are not overlapping. Instead, each aggregatable report will be assigned a shared ID. This ID is generated from the combined data points: reporting origin, target site, and scheduled report transmission time, rounded down to the hour.

バッチの不一致、つまりバッチが重複していないことを確認するために、集計のために提出されたすべての集計可能なレポートを追跡することは技術的に現実的ではありません。代わりに、各集約可能なレポートには共有 ID が割り当てられます。この ID は、報告元、対象サイト、およびレポート送信予定時刻（時間単位で切り捨て）を組み合わせたデータポイントから生成されます。

The aggregation service will enforce that all aggregatable reports with the same ID must be included in the same batch. Conversely, if more than one batch is submitted with the same ID, only one batch will be accepted for aggregation and the others will be rejected.

集計サービスでは、同じ ID を持つ集計可能なレポートはすべて同じバッチに含まれなければならないことが強制されます。逆に、同じ ID で複数のバッチが送信された場合、1 つのバッチのみがアグリゲーションに受け入れられ、他のバッチは拒否されます。

#### Pre-declaring aggregation buckets

The Attribution Reporting API client describes a [128 bit key space](https://en.wikipedia.org/wiki/Key_size) for assigning aggregation "buckets". This large space gives adtechs flexibility in how they define and use aggregated metrics. At this size, it's infeasible for the aggregation service to output a value for every possible bucket. Therefore, adtechs will need to declare a select set of buckets to be included in the collected data.

Attribution Reporting API クライアントは、アグリゲーション「バケット」を割り当てるための[128 ビットキースペース](https://en.wikipedia.org/wiki/Key_size)を記述しています。この大きなスペースは、アドテクがアグリゲーションメトリックスをどのように定義し、使用するかについて柔軟性を与えます。このサイズでは、アグリゲーションサービスがすべての可能なバケットに対して値を出力することは不可能です。したがって、広告主は、収集したデータに含めるバケットのセットを選択し、宣言する必要があります。

Only buckets which are pre-declared will be considered in aggregation. This ensures the aggregation service doesn't exceed reasonable storage capacity.

事前に宣言されたバケットのみがアグリゲーションで考慮されます。これにより、アグリゲーションサービスが合理的なストレージ容量を超えないようにすることができます。

The aggregated value for each bucket is output with addition of statistical noise. If a bucket is pre-declared but not present in any input aggregatable report, a value is still output, drawn from the same statistical noise distribution.

Other models, such as outputting buckets and values if the value exceeds a noisy threshold may be considered in future iterations.

その他のモデル、例えば値がノイズの多い閾値を超えたらバケットと値を出力するといったことも、今後の反復で検討されるかもしれない。

## Mechanics of aggregation

### Reporting origin

Before adtechs can generate summary reports with the aggregation service, they must first collect the aggregatable reports from user devices. The reporting origin (or collector server) is owned by the adtech.

アドテクノロジーがアグリゲーションサービスを使用してサマリーレポートを生成する前に、まずユーザーデバイスからアグリゲーション可能なレポートを収集する必要があります。レポート作成元（またはコレクターサーバー）は、adtech が所有します。

Adtechs must then batch the aggregatable reports on a reliable storage server. Once batched, the adtech sends a request to the aggregation service to process the batched reports.

アドテックは、その後、信頼できるストレージサーバー上で集計可能なレポートをバッチ処理する必要があります。バッチ処理されると、アドテックはアグリゲーション・サービスにリクエストを送信し、バッチ処理されたレポートを処理します。

### Aggregatable reports

The encrypted payload of an aggregatable report is a [list of key-value pairs](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md#encrypted-payload). The keys are 128 bit "bucket labels", and the values are positive integers.

集約可能なレポートの暗号化されたペイロードは、[キーと値のペアのリスト](https://github.com/WICG/conversion-measurement-api/blob/main/AGGREGATE.md#encrypted-payload)です。キーは 128 ビットの「バケットラベル」であり、値は正の整数である。

For example:

```
"report": [{"bucket": <bucket>, "value": <value> }, ...]
```

### Summary report

The summary report is a list JSON-formatted, dictionary-style set of key-value pairs. This means each key-value pair should appear only once.

要約レポートは、JSON 形式のリストで、キーと値のペアの辞書スタイルのセットです。これは、各キー・バリュー・ペアが一度だけ表示されることを意味します。

For each bucket, the summary report contains the sum of values from all aggregatable reports in the input. The core aggregation logic is represented by the following snippet of Python pseudo code.

各バケットについて、サマリーレポートには、入力のすべての集約可能なレポートからの値の合計が含まれます。コアとなる集計ロジックは、以下の Python 疑似コードで表現される。

```python
# Input: a list of aggregatable reports. Each aggregatable report is a dictionary.
# Output: a raw summary report, which is also a dictionary.
def aggregate(aggregatable_reports):
  raw_summary_report = {}
  for agg_report in aggregatable_reports:
    for bucket_label, value in agg_report:
      raw_summary_report[bucket_label] = \
          raw_summary_report.get(bucket_label, 0) + value
  return raw_summary_report
```

### Added noise

We will never return the raw summary report. Instead, we'll add noise to each value to preserve privacy.

私たちは、生のサマリーレポートを返すことはありません。その代わり、プライバシーを守るために各値にノイズを加えることにする。

However, if we added noise to every possible `bucket_label`, even if the vast majority of labels never appear in the input, the output would be astronomically large (2^128 entries!). To work around this problem, we require adtech to [pre-declare aggregation buckets](#pre-declaring-aggregation-buckets).

しかし、もしすべてのバケットラベルにノイズを加えてしまうと、たとえ大多数のラベルが入力に現れなかったとしても、出力は天文学的に大きくなってしまいます (2^128 entries!) 。この問題を回避するために、我々は adtech に[集約バケットを事前に宣言する](#pre-declaring-aggregation-buckets)ことを要求しています。

```python
# Input: a raw summary report (a dictionary)
# Output: a noised report (a very large dictionary with all 2^128 possible keys present)
def add_noise(raw_summary_report, declared_labels):
  noised_summary_report = {}
  for bucket_label in declared_labels:
    noised_summary_report[bucket_label] =  \
    raw_summary_report.get(bucket_label, 0) + generateNoise()
  return noised_summary_report
```

## Initial experiment plans

The initial implementation strategy for the aggregation service is as follows:

アグリゲーションサービスの初期実装方針は以下の通りである。

- The TEE based aggregation service implementation would be deployed on cloud service(s) which support needed security features. We envision the aggregation service being capable of being deployed with multiple cloud providers.

- TEE ベースのアグリゲーションサービスの実装は、必要なセキュリティ機能をサポートするクラウドサービス上に展開されます。このアグリゲーションサービスは、複数のクラウドプロバイダーで展開することが可能であると想定しています。

- In our current implementation, batches can be assembled on any reliable storage service. However, batches will need to be uploaded to the cloud provider to be processed by the aggregation service.

- 現在の実装では、バッチは任意の信頼できるストレージサービス上で組み立てることができます。しかし、バッチはアグリゲーションサービスによって処理されるために、クラウドプロバイダーにアップロードされる必要があります。

- To make testing of the aggregation service available in an origin trial, Google will play the role of coordinator. Longer term, we prefer that one or more independent entities can share this role.

- アグリゲーションサービスをオリジン試験で利用できるようにするため、Google はコーディネーターの役割を果たす。長期的には、1 つまたは複数の独立した事業者がこの役割を分担できることが望ましい。

- During the initial experiment, we expect that adtechs will be able to access decrypted payloads via a debugging mode tied to the availability of third party cookies.

- 最初の実験では、サードパーティのクッキーの可用性と結びついたデバッグ・モードを通じて、アドテクが解読されたペイロードにアクセスできるようになると予想されます。

- Our initial experiment will support a range of epsilon values for testing, up to `epsilon=64`. This allows adtechs to experiment with different aggregation strategies and provide feedback on the utility of the system, with different privacy parameters.

- 最初の実験では、テスト用にイプシロン値の範囲をサポートし、最大で`epsilon=64`までとする予定です。これにより、アドテク企業はさまざまな集計戦略を試し、さまざまなプライバシー・パラメーターで、システムの有用性についてフィードバックを提供することができます。

## How this proposal addresses our design goals

- 最初の実験では、テスト用にイプシロン値の範囲をサポートし、最大で`epsilon=64`までとする予定です。これにより、アドテク企業はさまざまな集計戦略を試し、さまざまなプライバシー・パラメーターで、システムの有用性についてフィードバックを提供することができます。

We started this proposal with a list of [proposed design principles](#proposed-design-principles). Here's a summary of how we addressed each goal in the proposal:

私たちはこの提案を、【デザイン原則の提案】(#proposed-design-principles)の一覧から始めました。ここでは、提案の中でそれぞれの目標にどのように取り組んだかをまとめています。

- Our proposal lays a foundation to meet our goal of differential privacy through a number of mechanisms (such as additive noise) (goal 1). There is more work to be done, and there are many [open questions](#open-questions) related to improved mechanisms and more nuanced privacy accounting.

- 私たちの提案は、多くのメカニズム（加法的ノイズなど）を通じて、差分プライバシーという私たちの目標を達成するための基礎を築くものである（目標 1）。やるべきことはもっとあり、改善されたメカニズムやよりニュアンスのあるプライバシー会計に関連する多くの[open questions](#open-questions)が存在します。

- By choosing Trusted Execution Environments for the initial infrastructure, we're balancing the need to prevent inappropriate access to raw or intermediate data (goal 2) with allowing adtechs to retain control and ownership of their data (goal 3). This also supports flexible development with scalable, on-demand access to the aggregation service, so that adtechs can process data when it is most convenient for them (goal 4).

- 初期インフラに Trusted Execution Environments を選択することで、生データや中間データへの不適切なアクセスを防ぐ必要性（目標 2）と、アドテク企業がデータのコントロールと所有権を保持できること（目標 3）のバランスを取っています。また、アグリゲーション・サービスへのスケーラブルでオンデマンドなアクセスにより、アドテック企業が都合のよいときにデータを処理できるよう、柔軟な開発をサポートしています（目標 4）。

- The aggregatable reports are encrypted on the user's device, so we must ensure these reports are never accessible in a decrypted format to any party (including the adtech which collected this data). The aggregation logic, along with the coordinator and attestation process, meet this need (goal 3).

- 集計可能なレポートはユーザーのデバイス上で暗号化されるため、これらのレポートがいかなる当事者（このデータを収集したアドテクノロジーを含む）からも復号化された形式で決してアクセスできないようにする必要があります。集計ロジックは、コーディネータと認証プロセスとともに、このニーズを満たします（目標 3）。

- We intend to open source all code for the aggregation service. This promotes transparency and openness both in implementation and generation of new ideas for all parts of this service (goal 5). Open source implementations also provide an opportunity for validation by outside parties.

- 私たちは、アグリゲーション・サービスのすべてのコードをオープンソース化するつもりです。これは、このサービスのすべての部分の実装と新しいアイデアの生成の両方において、透明性と開放性を促進するものです（目標 5）。また、オープンソースの実装は、外部の関係者による検証の機会を提供します。

## Open questions

This explainer is the first step in the standardization process followed by The Privacy Sandbox proposals. The aggregation service is not finalized, and we expect that it will benefit greatly from feedback from the community. There are many ways to provide feedback on this proposal and participate in ongoing discussions, which includes commenting on the Issues below, [opening new Issues in this repository](https://github.com/WICG/conversion-measurement-api/issues), or attending a [WICG Measurement meeting](https://github.com/WICG/conversion-measurement-api/tree/main/meetings). We intend to incorporate and iterate based on feedback.

この説明書は、The Privacy Sandbox の提案に従った標準化プロセスの最初のステップです。アグリゲーションサービスは最終的なものではなく、コミュニティからのフィードバックによって大きな恩恵を受けると期待しています。この提案に対するフィードバックを提供し、進行中の議論に参加するには、以下の Issues にコメントする、[このリポジトリで新しい Issues を開く](https://github.com/WICG/conversion-measurement-api/issues)、[WICG 測定会議](https://github.com/WICG/conversion-measurement-api/tree/main/meetings)に参加するなど、多くの方法があります。我々はフィードバックを取り入れ、反復するつもりである。

1. Who reviews and validates the open source code? What is the right contribution and approval model? [issues/322](https://github.com/WICG/conversion-measurement-api/issues/322)

1. 誰がオープンソースコードをレビューし、検証するのか？正しい貢献と承認モデルとは？[課題/322](https://github.com/WICG/conversion-measurement-api/issues/322)

1. Who are appropriate parties to serve as coordinators? [issues/323](https://github.com/WICG/conversion-measurement-api/issues/323)

1. コーディネータとして適切な人物は誰か？[課題/323](https://github.com/WICG/conversion-measurement-api/issues/323)

1. What is the right way to support debugging or error investigations without compromising the security of the system? [issues/324](https://github.com/WICG/conversion-measurement-api/issues/324)

1. システムの安全性を損なわずにデバッグやエラー調査を支援する正しい方法とは？[課題/324](https://github.com/WICG/conversion-measurement-api/issues/324)

1. How can adtechs recover from failures (such as a misconfigured batch)? [issues/325](https://github.com/WICG/conversion-measurement-api/issues/325)

1. アドテックは失敗（バッチの設定ミスなど）からどうリカバーすればいいのか？[課題/325](https://github.com/WICG/conversion-measurement-api/issues/325)

1. This proposal requires that the adtech works with a cloud provider with specific TEE capabilities. How do we balance the security needs of the system with the costs and implementation effort? [issues/326](https://github.com/WICG/conversion-measurement-api/issues/326)

1. この提案では、アドテックが特定の TEE 機能を持つクラウドプロバイダーと連携することが必要です。システムのセキュリティニーズとコストや導入の手間をどのようにバランスさせるか？[課題/326】(https://github.com/WICG/conversion-measurement-api/issues/326)

1. Our proposal includes one specific aggregation mechanism. There are many alternative, differentially private mechanisms that could support specific use cases or provide better quality results. How can we best build the system so it can support a wide range of use cases? [issues/327](https://github.com/WICG/conversion-measurement-api/issues/327)

1. 我々の提案は、1 つの特定の集計メカニズムを含んでいる。特定のユースケースをサポートしたり、より質の高い結果を提供できるような、多くの代替的な、異なるプライベートなメカニズムが存在する。どのようにすれば、幅広いユースケースをサポートできるようにシステムを構築することができるでしょうか。[課題/327](https://github.com/WICG/conversion-measurement-api/issues/327)

1. How should proposed extensions be evaluated for privacy, security and utility to be included in the aggregation service? [issues/328](https://github.com/WICG/conversion-measurement-api/issues/328)

1. 提案された拡張機能は、アグリゲーションサービスに含まれるために、プライバシー、セキュリティ、実用性についてどのように評価されるべきか？[課題/328](https://github.com/WICG/conversion-measurement-api/issues/328)

1. Are there any other approaches to the security architecture that could work here? For example, multi-party computation (MPC) is an interesting approach that can be complementary. [issues/330](https://github.com/WICG/conversion-measurement-api/issues/330)

1. セキュリティ・アーキテクチャについて、ここで使える他のアプローチはないのでしょうか？例えば、MPC(multi-party computation)は補完し合える興味深いアプローチです。[課題/330](https://github.com/WICG/conversion-measurement-api/issues/330)
