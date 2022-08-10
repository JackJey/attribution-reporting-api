# Attribution Reporting API with Aggregatable Reports

<!-- START doctoc generated TOC please keep comment here to allow auto update -->

<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

- [Authors](#authors)
- [Introduction](#introduction)
- [Goals](#goals)
- [API changes](#api-changes)
  - [Attribution source registration](#attribution-source-registration)
  - [Attribution trigger registration](#attribution-trigger-registration)
  - [Aggregatable reports](#aggregatable-reports)
  - [Contribution bounding and budgeting](#contribution-bounding-and-budgeting)
  - [Storage limits](#storage-limits)
- [Data processing through a Secure Aggregation Service](#data-processing-through-a-secure-aggregation-service)
- [Privacy considerations](#privacy-considerations)
  - [Differential Privacy](#differential-privacy)
- [Ideas for future iteration](#ideas-for-future-iteration)
  - [Worklet-based aggregation key generation](#worklet-based-aggregation-key-generation)
  - [Custom attribution models](#custom-attribution-models)
  - [Hide the true number of attribution reports](#hide-the-true-number-of-attribution-reports)
  - [More advanced contribution bounding](#more-advanced-contribution-bounding)
  - [Choosing among aggregation services](#choosing-among-aggregation-services)
- [Considered alternatives](#considered-alternatives)
  - ["Count" vs. "value" histograms](#count-vs-value-histograms)
  - [Binary report format](#binary-report-format)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Authors

- csharrison@chromium.org
- johnidel@chromium.org
- marianar@google.com

## Introduction

This document proposes extensions to our existing [Attribution Reporting API](https://github.com/csharrison/conversion-measurement-api) that reports event-level data. The intention is to provide a mechanism for rich metadata to be reported in aggregate, to better support use-cases such as campaign-level performance reporting or conversion values.

この文書では、イベントレベルのデータを報告する既存の [Attribution Reporting API](https://github.com/csharrison/conversion-measurement-api) の拡張を提案する。この意図は、キャンペーンレベルのパフォーマンスレポートやコンバージョン値のようなユースケースをより良くサポートするために、リッチなメタデータを集約してレポートするためのメカニズムを提供することです。

## Goals

Aggregatable attribution reports should support legitimate measurement use cases not already supported by the event-level reports. These include:

集計可能なアトリビューションレポートは、イベントレベルレポートで既にサポートされていない正当な測定ユースケースをサポートする必要があります。これには以下が含まれる。

- Higher fidelity measurement of attribution-trigger (conversion-side) data, which is very limited in event-level attribution reports, including the ability to sum _values_ rather than just counts

- イベントレベルのアトリビューションレポートでは非常に限られている、アトリビューショントリガー(コンバージョン側)のデータをより忠実に測定し、カウントだけでなく\_値を合計することが可能です。

- A system which enables the most robust privacy protections

- 最も強固なプライバシー保護を可能にするシステム

- Ability to receive aggregatable reports alongside event-level reports

- イベントレベルのレポートと同時に、集計可能なレポートを受け取ることができます。

- Ability to receive data at a faster rate than with event-level reports

- イベントレベルのレポートに比べ、高速にデータを受信することが可能

- Greater flexibility to trade off attribution-trigger (conversion-side) data, reporting rate and accuracy.

- アトリビューション・トリガー(コンバージョン側)データ、レポート作成率、精度をトレードオフできる柔軟性を向上。

Note: fraud detection (enabling the filtration of reports you are not expecting) is a goal but it is left out of scope for this document for now.

注:不正検知(想定していないレポートのフィルタリングを可能にする)は目標ではあるが、今のところこのドキュメントの範囲外としている。

## API changes

The client API for creating contributions to an aggregate report uses the same API base as for event level reports with a few extensions. In the following example an ad-tech will use the API to collect:

集計レポートへの貢献を作成するためのクライアント API は、イベントレベルレポートと同じ API ベースに、いくつかの拡張機能を使用しています。次の例では、広告技術者が API を使用して収集します。

- Aggregate conversion counts at a per-campaign level
- Aggregate purchase values at a per geo level

### Attribution source registration

Registering sources eligible for aggregate reporting entails adding a new `aggregation_keys` dictionary field to the JSON dictionary of the [`Attribution-Reporting-Register-Source` header](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#registering-attribution-sources):

集約報告の対象となるソースを登録するには、[`Attribution-Reporting-Register-Source` ヘッダー](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#registering-attribution-sources) の JSON 辞書に新しい `aggregation_keys` 辞書フィールドを追加する必要があります。

```jsonc
{
  ... // existing fields, such as `source_event_id` and `destination`

  "aggregation_keys": {
    // Generates a "0x159" key piece (low order bits of the key) for the key named
    // "campaignCounts".
    "campaignCounts": "0x159", // User saw ad from campaign 345 (out of 511)

    // Generates a "0x5" key piece (low order bits of the key) for the key named "geoValue".
    "geoValue": "0x5" // Source-side geo region = 5 (US), out of a possible ~100 regions
  }
}
```

This defines a dictionary of named aggregation keys, each with a piece of the aggregation key defined as a hex-string. The final histogram bucket key will be fully defined at trigger time using a combination (binary OR) of this piece and trigger-side pieces.

これは、16 進文字列として定義された集約キーの断片を持つ、名前付き集約キーの辞書を定義するものである。最終的なヒストグラムのバケットキーは、このピースとトリガー側のピースの組み合わせ(バイナリ OR)を用いて、トリガー時に完全に定義されます。

Final keys will be restricted to a maximum of 128 bits. This means that hex strings in the JSON must be limited to at most 32 digits.

最終的な鍵は最大 128 ビットに制限されます。つまり、JSON 内の 16 進文字列は最大 32 桁に制限される。

### Attribution trigger registration

Trigger registration will also add two new fields to the JSON dictionary of the [`Attribution-Reporting-Register-Trigger` header](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#triggering-attribution):

また、トリガー登録により、[`Attribution-Reporting-Register-Trigger` ヘッダー](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#triggering-attribution) の JSON 辞書に 2 つの新しいフィールドが追加されます。

```jsonc
{
  ... // existing fields, such as `event_trigger_data`

  "aggregatable_trigger_data": [
    // Each dict independently adds pieces to multiple source keys.
    {
      // Conversion type purchase = 2 at a 9 bit offset, i.e. 2 << 9.
      // A 9 bit offset is needed because there are 511 possible campaigns, which
      // will take up 9 bits in the resulting key.
      "key_piece": "0x400",
      // Apply this key piece to:
      "source_keys": ["campaignCounts"]
    },
    {
      // Purchase category shirts = 21 at a 7 bit offset, i.e. 21 << 7.
      // A 7 bit offset is needed because there are ~100 regions for the geo key,
      // which will take up 7 bits of space in the resulting key.
      "key_piece": "0xA80",
      // Apply this key piece to:
      "source_keys": ["geoValue", "nonMatchingKeyIdsAreIgnored"]
    }
  ],
  "aggregatable_values": {
    // Each source event can contribute a maximum of L1 = 2^16 to the aggregate
    // histogram. In this example, use this whole budget on a single trigger,
    // evenly allocating this "budget" across two measurements. Note that this
    // will require rescaling when post-processing aggregates!

    // 1 count =  L1 / 2 = 2^15
    "campaignCounts": 32768,

    // Purchase was for $52. The site's max value is $1024.
    // $1 = (L1 / 2) / 1024.
    // $52 = 52 * (L1 / 2) / 1024 = 1664
    "geoValue": 1664
  }
}
```

The `aggregatable_trigger_data` field is a list of dict which generates aggregation keys.

`aggregatable_trigger_data` フィールドは、集約キーを生成する dict のリストである。

The `aggregatable_values` field lists an amount of an abstract "value" to contribute to each key, which can be integers in [1, 2^16). These are attached to aggregation keys in the order they are generated. See the [contribution budgeting](#contribution-bounding-and-budgeting) section for more details on how to allocate these contribution values.]()

`集約可能な値`フィールドは、各キーに寄与する抽象的な「値」の量をリストアップします。これは [1, 2^16) の整数値です。これらは生成された順番に集約キーに付けられます。これらの貢献度の割り当て方法の詳細については、[貢献度予算](#contribution-bounding and-budgeting) のセクションを参照してください。]()

The scheme above will generate the following abstract histogram contributions:

上記の方式では、以下のような抽象的なヒストグラムの投稿が生成されます。

```jsonc
[
  // campaignCounts
  {
    "key": 0x559, // = 0x159 | 0x400
    "value": 32768
  },
  // geoValue:
  {
    "key": 0xa85, // = 0x5 | 0xA80
    "value": 1664
  }
]
```

Note: The `filters` field will still apply to aggregatable reports, and each dict in `aggregatable_trigger_data` can still optionally have filters applied to it just like for event-level reports.

注意: `filters` フィールドは、集約可能なレポートにも適用されます。また、 `aggregatable_trigger_data` の各 dict には、イベントレベルのレポートと同様に、オプションでフィルタを適用することができます。

Note: the above scheme was used to maximize the [contribution budget](#contribution-bounding-and-budgeting) and optimize utility in the face of constant noise. To rescale, simply inverse the scaling factors used above:

注:上記のスキームは、[貢献予算](#contribution-bounding and-budgeting) を最大化し、一定のノイズに直面した場合の効用を最適化するために使用されたものです。再スケールするには、上記で使用したスケーリングファクターを単純に反転させる。

```python
L1 = 1 << 16
true_agg_campaign_counts = raw_agg_campaign_counts / (L1 / 2)
true_agg_geo_value = 1024 * raw_agg_geo_value / (L1 / 2)
```

Note that aggregatable trigger registration is independent of event-level trigger registration.

集約可能なトリガ登録は、イベントレベルのトリガ登録と独立していることに注意。

### Aggregatable reports

Aggregatable reports will look very similar to event-level reports. They will be reported to the reporting origin at the path `.well-known/attribution-reporting/report-aggregate-attribution`.

集計可能なレポートはイベントレベルのレポートと非常によく似ています。これらは、`.well-known/attribution-report/report-aggregate-attribution`というパスにあるレポート作成元に報告されます。

The report itself does not contain histogram contributions in the clear. Rather, the report embeds them in an _encrypted payload_ that can only be read by a trusted aggregation service known by the browser.

レポート自体には、ヒストグラムの寄与は含まれていません。むしろ、レポートは、ブラウザが知っている信頼できるアグリゲーション・サービスによってのみ読むことができる「暗号化されたペイロード」にそれらを埋め込むのです。

The report will be JSON encoded with the following scheme:

報告書は以下のスキームで JSON エンコードされたものになります。

```jsonc
{
  // Info that the aggregation services also need encoded in JSON
  // for use with AEAD.
  "shared_info": "{\"api\":\"attribution-reporting\",\"attribution_destination\":\"https://advertiser.example\",\"report_id\":\"[UUID]\",\"reporting_origin\":\"https://reporter.example\",\"scheduled_report_time\":\"[timestamp in seconds]\",\"source_registration_time\":\"[timestamp in seconds]\",\"version\":\"[api version]\"}",

  // Support a list of payloads for future extensibility if multiple helpers
  // are necessary. Currently only supports a single helper configured
  // by the browser.
  "aggregation_service_payloads": [
    {
      "payload": "[base64-encoded HPKE encrypted data readable only by the aggregation service]",
      "key_id": "[string identifying public key used to encrypt payload]",

      // Optional debugging information, if the cookie `ar_debug` is present.
      "debug_cleartext_payload": "[base64-encoded unencrypted payload]"
    }
  ],

  // Optional debugging information (also present in event-level reports),
  // if the cookie `ar_debug` is present.
  "source_debug_key": "[64 bit unsigned integer]",
  "trigger_debug_key": "[64 bit unsigned integer]"
}
```

Reports will not be delayed to the same extent as they are for event level reports. The browser will delay them with a random delay between 10 minutes to 1 hour, or with a small delay after the browser next starts up. The minimum 10 minutes delay allows regretful users to have a chance to delete the reports. The browser is free to utilize techniques like retries to minimize data loss.

イベントレベルのレポートと同じ程度には、レポートは遅延されません。ブラウザは、10 分から 1 時間の間のランダムな遅延、またはブラウザが次に起動した後の小さな遅延で、それらを遅らせます。最低 10 分の遅延は、後悔しているユーザーにレポートを削除する機会を与えることができます。ブラウザは、データ損失を最小限に抑えるために再試行などの技術を自由に利用することができます。

- The `shared_info` will be a serialized JSON object. This exact string is used as authenticated data for decryption, see [below](#encrypted-payload). The string therefore must be forwarded to the aggregation service unmodified. The reporting origin can parse the string to access the encoded fields.

- `shared_info`はシリアライズされた JSON オブジェクトになります。この正確な文字列は、復号化のための認証データとして使用されます([下記](#encrypted-payload)を参照)。したがって、この文字列は修正されないままアグリゲーションサービスに転送される必要があります。報告元は、文字列を解析して、エンコードされたフィールドにアクセスすることができます。

- The `api` field is a string enum identifying the API that triggered the report. This allows the aggregation service to extend support to other APIs in the future.

- `api` フィールドは、レポートのトリガーとなった API を特定するための文字列列列挙型である。これにより、将来的にアグリゲーションサービスが他の API にサポートを拡張することができる。

- The `scheduled_report_time` will be the number of seconds since the Unix Epoch (1970-01-01T00:00:00Z, ignoring leap seconds) to align with [DOMTimestamp](https://heycam.github.io/webidl/#DOMTimeStamp) until the browser initially scheduled the report to be sent (to avoid noise around offline devices reporting late).

- `scheduled_report_time` は、ブラウザが最初にレポートを送信するようにスケジュールするまでの、Unix Epoch (1970-01-01T00:00:00Z, うるう秒は無視) から [DOMTimestamp](https://heycam.github.io/webidl/#DOMTimeStamp) に合わせた秒数です (オフラインデバイスからの遅いレポートによるノイズを回避するため).

- The `source_registration_time` will represent (in seconds since the Unix Epoch) the time the source event was registered, rounded down to a whole day.

- `source_registration_time` はソースイベントが登録された時刻を (Unix Epoch からの秒数で) 表し、丸一日に切り捨てた値となります。

- The `payload` will contain the actual histogram contributions. It should be be encrypted and then base64 encoded, see [below](#encrypted-payload).

- この `payload` には，実際のヒストグラムの投稿が含まれます．これは暗号化され，さらに base64 でエンコードされている必要があります．

Optional debugging fields are discussed [below](#optional-extended-debugging-reports).

オプションのデバッグフィールドについては、[後述](#optional-extended-debugging-reports)で説明します。

#### Encrypted payload

The `payload` should be a [CBOR](https://cbor.io) map encrypted via [HPKE](https://datatracker.ietf.org/doc/draft-irtf-cfrg-hpke/) and then base64 encoded. The map will have the following structure:

ペイロードは [CBOR](https://cbor.io) マップを [HPKE](https://datatracker.ietf.org/doc/draft-irtf-cfrg-hpke/) で暗号化し、base64 エンコードしたものです。このマップは以下のような構造をしています。

```jsonc
// CBOR
{
  "operation": "histogram",  // Allows for the service to support other operations in the future
  "data": [{
    "bucket": <bucket, encoded as a 16-byte (i.e. 128-bit) big-endian bytestring>,
    "value": <value, encoded as a 4-byte (i.e. 32-bit) big-endian bytestring>
  }, ...]
}
```

Optionally, the browser may encode multiple contributions in the same payload; this is only possible if all other fields in the report/payload are identical for the contributions.

オプションとして、ブラウザは複数の投稿を同じペイロードにエンコードすることができる。これは、レポート/ペイロードの他のすべてのフィールドが投稿に対して同一である場合にのみ可能である。

This encryption should use [AEAD](https://en.wikipedia.org/wiki/Authenticated_encryption) to ensure that the information in `shared_info` is not tampered with, since the aggregation service will need that information to do proper replay protection. The authenticated data will consist of the `shared_info` string (encoded as UTF-8) with a constant prefix added for domain separation, i.e. to avoid ciphertexts being reused for different protocols, even if public keys are shared.

この暗号化には [AEAD](https://en.wikipedia.org/wiki/Authenticated_encryption) を使用し、`shared_info` の情報が改竄されないようにする必要があります。なぜなら、アグリゲーションサービスは適切なリプレイ保護を行うために、その情報を必要とするためです。認証されたデータは `shared_info` の文字列 (UTF-8 でエンコードされる) に、ドメイン分離のために一定のプレフィックスを付加したもので、公開鍵を共有していても異なるプロトコルで暗号文が再利用されないようにする。

The encryption will use public keys specified by the aggregation service. The browser will encrypt payloads just before the report is sent by fetching the public key endpoint with an un-credentialed request. The processing origin will respond with a set of keys which will be stored according to standard HTTP caching rules, i.e. using Cache-Control headers to dictate how long to store the keys for (e.g. following the [freshness lifetime](https://datatracker.ietf.org/doc/html/)

この暗号化には [AEAD](https://en.wikipedia.org/wiki/Authenticated_encryption) を使用し、`shared_info` の情報が改竄されないようにする必要があります。なぜなら、アグリゲーションサービスは適切なリプレイ保護を行うために、その情報を必要とするためです。認証されたデータは `shared_info` の文字列 (UTF-8 でエンコードされる) に、ドメイン分離のために一定のプレフィックスを付加したもので、公開鍵を共有していても異なるプロトコルで暗号文が再利用されないようにする。

rfc7234#section-4.2)). The browser could enforce maximum/minimum lifetimes of stored keys to encourage faster key rotation and/or mitigate bandwidth usage. The scheme of the JSON encoded public keys is as follows:

暗号化には、アグリゲーションサービスによって指定された公開鍵が使用されます。ブラウザは、レポートが送信される直前に、認証されていないリクエストで公開鍵のエンドポイントを取得することにより、ペイロードを暗号化します。つまり、Cache-Control ヘッダを使用して、鍵をどれくらいの期間保存するかを指定します(たとえば [freshness lifetime](https://datatracker.ietf.org/doc/html/rfc7234#section-4.2) に従います)。ブラウザは、保存される鍵の最大/最小寿命を強制することで、鍵のローテーションの 高速化を促進したり、帯域幅の使用を軽減したりすることができる。JSON 暗号化された公開鍵のスキームは以下のとおりである。

```jsonc
{
  "keys": [
    {
      "id": "[arbitrary string identifying the key (up to 128 characters)]",
      "key": "[base64 encoded public key]"
    }
    // Optionally, more keys.
  ]
}
```

To limit the impact of a single compromised key, multiple keys (up to a small limit) can be provided. The browser should independently pick a key uniformly at random for each payload it encrypts to avoid associating different reports. Additionally, a public key endpoint should not reuse an ID string for a different key. In particular, IDs must be unique within a single response to be valid. In the case of backwards incompatible changes to this scheme (e.g. in future versions of the API), the endpoint URL should also change.

単一の漏洩鍵の影響を制限するために、複数の鍵を提供することができる(小さな制限まで)。ブラウザは、異なるレポートを関連付けることを避けるために、暗号化する各ペイロードに対して、独自に一様にランダムに鍵を選ぶべきです。さらに、公開鍵のエンドポイントは、異なる鍵のために ID 文字列を再利用してはならない。特に、ID は 1 つのレスポンス内で一意でなければ有効とはならない。このスキームに対して後方互換性のない変更が行われた場合 (たとえば、API の将来のバージョン)、エンドポイント URL も変更される必要があります。

**Note:** The browser may need some mechanism to ensure that the same set of keys are delivered to different users.

**注:**ブラウザは、同じ鍵のセットが異なるユーザーに配信されることを保証するために、何らかのメカニズムが必要かもしれません。

#### Optional: extended debugging reports

If [debugging](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#optional-extended-debugging-reports) is enabled, additional debug fields will be present in aggregatable reports. The `source_debug_key` and `trigger_debug_key` fields match those in the event-level reports. If both the source and trigger debug keys are set, there will be a `debug_cleartext_payload` field included in the report. It will contain the base64-encoded cleartext of the encrypted payload to allow downstream systems to verify that reports are constructed correctly. If both debug keys are set, the `shared_info` will also include the flag `"debug_mode": "enabled"` to allow the aggregation service to support debugging functionality on these reports.

[debugging](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#optional-extended-debugging-reports) が有効な場合、集約可能なレポートに追加のデバッグフィールドが表示されます。`source_debug_key`と`trigger_debug_key`フィールドはイベントレベルのレポートにあるものと同じです。ソースデバッグキーとトリガーデバッグキーの両方が設定されている場合、レポートに`debug_cleartext_payload` フィールドが含まれます。これは、レポートが正しく構築されていることを下流のシステムが検証できるように、暗号化されたペイロードの base64 エンコードされた平文が含まれます。両方のデバッグキーが設定されている場合、`shared_info`にはフラグ`"debug_mode":"enabled"`これは、アグリゲーションサービスがこれらのレポートのデバッグ機能をサポートできるようにするためである。

Additionally, a duplicate debug report will be sent immediately (i.e. without the random delay) to a `.well-known/attribution-reporting/debug/report-aggregate-attribution` endpoint. The debug reports should be almost identical to the normal reports, including the additional debug fields. However, the `payload` ciphertext will differ due to repeating the encryption operation and the `key_id` may differ if the previous key had since expired or the browser randomly chose a different valid public key.

さらに、重複したデバッグレポートが `.well-known/attribution-reporting/debug/report-aggregate-attribution` エンドポイントに (つまりランダムな遅延なしに) すぐに送信されるようになります。デバッグレポートは、追加のデバッグフィールドを含めて、通常のレポートとほとんど同じであるべきです。しかし、暗号化処理を繰り返すため `payload` の暗号文は異なり、`key_id` は以前の鍵が期限切れになっていたり、ブラウザがランダムに別の有効な公開鍵を選んだりすると異なる可能性があります。

### Contribution bounding and budgeting

Each attribution can make multiple contributions to an underlying aggregate histogram, and a given user can trigger multiple attributions for a particular source / trigger site pair. Our goal in this section is to _bound_ the contributions any source event can make to a histogram.

各属性は、基礎となる集約ヒストグラムに複数の寄与をすることができ、特定のユーザは、特定のソース/トリガ サイトのペアで複数の属性をトリガすることができます。このセクションの目標は、任意のソース・イベントがヒストグラムに与える貢献度を「制限」することです。

This bound is characterized by a single parameters: `L1`, the maximum sum of the contributions (values) across all buckets for a given source event. L1 refers to the L1 sensitivity / norm of the histogram contributions per source event.

この境界は一つのパラメータで特徴付けられる。 `L1` は、任意のソース・イベントに対するすべてのバケット間の寄与(値)の最大合計です。L1 とは、ソース・イベントごとのヒストグラム寄与の L1 感度/ノルムを指します。

Exceeding these limits will cause future contributions to silently drop. While exposing failure in any kind of error interface can be used to leak sensitive information, we might be able to reveal aggregate failure results via some other monitoring side channel in the future.

For the initial proposal, set `L1 = 65536`. Note that for privacy, this parameter can be arbitrary, as noise in the aggregation service will be scaled in proportion to this parameter. In the example above, the budget is split equally between two keys, one for the number of conversions per campaign and the other representing the conversion dollar value per geography. This budgeting mechanism is highly flexible and can support many different aggregation strategies as long as the appropriate scaling is performed on the outputs.

最初の提案では、`L1 = 65536`とする。アグリゲーションサービスのノイズはこのパラメータに比例してスケールされるので、プライバシーのために、このパラメータは任意であることに注意してください。上記の例では、予算は 2 つのキーに均等に分配されます。1 つはキャンペーンごとのコンバージョン数、もう 1 つは地域ごとのコンバージョンドル値を表しています。このバジェットメカニズムは非常に柔軟で、出力に対して適切なスケーリングが行われる限り、多くの異なるアグリゲーション戦略をサポートすることができます。

### Storage limits

The browser may apply storage limits in order to prevent excessive resource usage.

ブラウザは、リソースの過剰な使用を防ぐために、ストレージの制限を適用することがあります。

Strawman: There should be a limit of 1024 pending aggregatable reports per destination site.

ストローマン送信先サイトごとに、保留中の集計可能レポート数を 1024 件に制限する必要があります。

Note: The storage limits for event-level and aggregatable reports are enforced independently of each other.

注:イベントレベルおよびアグリゲーションレポートの保存制限は、互いに独立して実施されます。

## Data processing through a Secure Aggregation Service

The exact design of the service is not specified here. We expect to have more information on the data flow from reporter → processing origins shortly, but what follows is a high-level summary.

サービスの正確な設計はここでは特定しない。報告者 → 処理起点のデータの流れについては、近日中に詳細をお知らせする予定ですが、以下はハイレベルな要約です。

As the browser sends individual aggregatable reports to the reporting origin, the reporting origin organizes them into [batches](AGGREGATION_SERVICE_TEE.md#disjoint-batches). They can send these batches to the aggregation service `origin` specified in the report.

The aggregation service will aggregate reports within a certain batch, and respond back with an aggregate histogram, i.e. a list of keys with associated _aggregate_ values. It is expected that as a privacy protection mechanism, a certain amount of noise will be a

ブラウザが個々の集約可能なレポートをレポート元に送信すると、レポート元はそれらを[バッチ](AGGREGATION_SERVICE_TEE.md#disjoint-batches)に整理する。これらのバッチは、レポートに指定されたアグリゲーションサービス `origin` に送信することができます。

dded to each output key's aggregate value.

集約サービスは、特定のバッチ内のレポートを集約し、集約ヒストグラム、 すなわち、関連する「集約値」を持つキーのリストで応答する。プライバシー保護の仕組みとして、各出力キーの集計値にある程度のノイズを加えることが期待される。

## Privacy considerations

This proposal introduces a new set of reports to the API. Alone they do not add much meaningful cross-site information, so they are fairly benign. However, they contain encrypted payloads which allow aggregate histograms to be computed.

この提案は、API に新しいレポートのセットを導入するものです。単独では意味のあるクロスサイト情報をあまり追加しないので、かなり良質のものです。しかし、それらは集約されたヒストグラムを計算することを可能にする暗号化されたペイロードを含んでいます。

These histograms should be protected with various techniques with a trusted server system. For example, it is expected that the histograms will be subject to noise proportional to the `L1` budget. Additionally, most rate-limits (except for the maximum number of reports per source) used for event-level reports will also be enforced for aggregatable reports, which limit the total amount of information that can be sent out for any one user.

これらのヒストグラムは、信頼できるサーバシステムで様々な手法で保護されるべきである。例えば、ヒストグラムは `L1` 予算に比例したノイズにさらされることが予想されます。さらに、イベントレベルのレポートに使用されるほとんどのレート制限(ソースごとのレポートの最大数を除く)は、任意の 1 ユーザーについて送信できる情報の総量を制限する集約可能なレポートに対しても適用されます。

Servers will need to be implemented such that browsers can trust them with sensitive cross-site data. There are various technologies and techniques that could be employed (e.g. [Trusted Execution Environments](https://en.wikipedia.org/wiki/Trusted_execution_environment), [Multi-party-computation](https://en.wikipedia.org/wiki/Secure_multi-party_computation), audits, etc) that could satisfy browsers that data is safely aggregated and the output maintains proper privacy.

サーバーは、ブラウザが機密性の高いクロスサイトデータを扱う際に信頼できるような実装が必要です。データが安全に集約され、出力が適切なプライバシーを維持することをブラウザに納得させるために、様々な技術やテクニック([Trusted Execution Environments](https://en.wikipedia.org/wiki/Trusted_execution_environment), [Multi-party-computation](https://en.wikipedia.org/wiki/Secure_multi-party_computation), 監査など)が採用される可能性がある。

### Differential Privacy

A goal of this work is to have a framework which can support differentially private aggregate measurement. In principle this can be achieved if the aggregation service adds noise proportional to the `L1` budget in principle, e.g. noise distributed according to Laplace(epsilon / L1) should achieve epsilon differential privacy. With small enough values of epsilon, reports for a given source will be well-protected in an aggregate release.

この研究のゴールは、差動的にプライベートな集約測定をサポートできるフレームワークを持つことである。例えば、Laplace(ε / L1)に従って分布するノイズは、ε の差分プライバシーを達成するはずです。十分小さい ε の値で、与えられたソースのレポートは、アグリゲートリリースで十分に保護されるでしょう。

Note: there are a few caveats about a formal differential privacy claim:

注:形式的な差分プライバシー要求については、いくつかの注意点があります。

- In the current design, the number of encrypted reports is revealed to the reporting origin in the clear without any noise. See [Hide the true number of attribution reports](#Hide the-true-number-of-attribution-reports).

- 現在の設計では、暗号化された報告書の数は、ノイズのないクリアな状態で報告元に明らかにされます。帰属報告書の本当の数を隠す】(#Hide the-true-number-of-attribution-reports) を参照してください。

- The scope of privacy in the current design is not user-level, but per-source. See [More advanced contribution bounding](#more-advanced-contribution-bounding) for follow-up work exploring tightening this.

- 現在の設計では、プライバシーの範囲はユーザーレベルではなく、ソース単位です。これを強化するためのフォローアップ作業については、[More advanced contribution bounding](#more-advanced-contribution-bounding) を参照してください。

- Our plan is to adjust the level of noise added based on feedback during the origin trial period, and our goal with this initial version is to create a foundation for further exploration into formally pri

- 現在の設計では、プライバシーの範囲はユーザーレベルではなく、ソース単位です。これを強化するためのフォローアップ作業については、[More advanced contribution bounding](#more-advanced-contribution-bounding) を参照してください。
  vate methods for aggregation.

- 今後は、オリジントライアル期間中のフィードバックをもとに、ノイズの付加レベルを調整する予定です。この初期バージョンでは、集計のための正式な非公開手法をさらに探求するための基盤を作ることを目的としています。

### Rate limits

Various rate limits outlined in the [event-level explainer](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#reporting-cooldown--rate-limits) should also apply to aggregatable reports. The limits should be shared across all types of reports.

イベントレベル説明書](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#reporting-cooldown--rate-limits)で説明されている様々なレート制限は、集計可能なレポートにも適用されるべきです。この制限は、すべての種類のレポートに共通するものでなければなりません。

## Ideas for future iteration

### Worklet-based aggregation key generation

At trigger time, we could have a [worklet-style](https://developer.mozilla.org/en-US/docs/Web/API/Worklet) API that allows passing in an arbitrary string of "trigger context" that specifies information about the trigger event (e.g. high fidelity data about a conversion). From within the worklet, code can access both the source and trigger context in the same function to generate an aggregate report. This allows for more dynamic keys than a declarative API (like the existing [HTTP-based triggering](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#triggering-attribution)), but disallows exfiltrating sensitive cross-site data out of the worklet.

トリガー時に、トリガーイベントに関する情報(例えば、変換に関する高忠実度データ)を指定する 「トリガーコンテキスト」の任意の文字列を渡すことができる[ワークレットスタイル](https://developer.mozilla.org/en-US/docs/Web/API/Worklet) API を持つことができる。小作業負荷の中から、コードは同じ関数でソースとトリガー コンテキストの両方にアクセスし、集約レポートを生成することができます。これは、既存の [HTTP-based triggering](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#triggering-attribution)のような宣言的な API よりも動的なキーを可能にしますが、機密性の高いクロスサイトデータを小作品から流出させることを禁止しています。

The worklet is used to generate histogram contributions, which are key-value pairs of integers. Note that there will be some maximum number of keys (e.g. 2^128 keys).

このワークレットは、ヒストグラムの寄与を生成するために使用され、整数のキーと値のペアになります。キーの最大数があることに注意してください(例えば 2^128 キー)。

The following code triggers attribution by invoking a worklet.

次のコードは、ワークレットを起動することでアトリビューションをトリガーします。

```javascript
await window.attributionReporting.worklet.addModule( "https://reporter.example/convert.js");

// The first argument should match the origin of the module we are invoking, and determines the scope of attribution similar to the existing HTTP-based API, i.e. it should match the "attributionreportto" attribute. The last argument needs to match what AggregateAttributionReporter uses upon calling registerAggregateReporter
// 最初の引数は、呼び出すモジュールの起源と一致する必要があり、既存の HTTP ベースの API と同様に属性のスコープを決定する、つまり "attributionreportto" 属性と一致する必要があります。最後の引数は、AggregateAttributionReporter が registerAggregateReporter を呼び出す際に使用するものと一致する必要があります。
window.attributionReporting.triggerAttribution("https://reporter.example", <triggerContextStr>, "my-aggregate-reporter");
```

Internally, the browser will look up to see which source should be attributed, similar to how [attribution](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#trigger-attribution-algorithm) works in the HTTP-based API. Note here that only a single source will be matched.

内部的には、HTTP ベースの API で[attribution](https://github.com/WICG/conversion-measurement-api/blob/main/EVENT.md#trigger-attribution-algorithm)がどのように動作するかと同様に、ブラウザはどのソースに帰属させるべきかを調べることになります。ここでは、単一のソースにのみマッチすることに注意してください。

Here is `convert.js` which crafts an aggregate report.

```javascript
class AggregateAttributionReporter {
  // attributionSourceContext set as "<campaignid>,<geoid>"
  processAggregate(triggerContext, attributionSourceContext, sourceType) {
    let [campaign, geo] = attributionSourceContext
      .split(",")
      .map((x) => parseInt(x, 10));

    let purchaseValue = parseInt(triggerContext, 10);

    histogramContributions = [
      { key: campaign, value: purchaseValue },
      { key: geo, value: purchaseValue },
    ];
    return {
      histogramContributions: histogramContributions,
    };
  }
}

// Bound classes will be invoked when an attribution triggered on this document is successfully attributed to a source whose reporting origin matches the worklet origin.

このドキュメントでトリガーされたアトリビューションが、レポートの起源がワークレットの起源と一致するソースに正常にアトリビュートされると、 // バインドクラスが呼び出されます。

registerAggregateReporter(
  "my-aggregate-reporter",
  AggregateAttributionReporter
);
```

This worklet approach provides greatly enhanced flexibility at the cost of complexity. It introduces a new security / privacy boundary, and there are several edge cases that must be handled carefully to avoid data loss (e.g. the document being destroyed while the worklet is processing, unless a `keepalive`-style mode for worklets is introduced). These issues must be solved before this design could be considered.

この小作業負荷のアプローチは、複雑さの代償として、非常に強化された柔軟性を提供する。また、新しいセキュリティとプライバシーの境界が導入され、データ損失を避けるために注意深く扱わなければならないいくつかのエッジケースがあります(例えば、小作品用の「keepalive」スタイルのモードが導入されない限り、小作品の処理中に文書が破壊される)。これらの問題は、この設計が検討される前に解決されなければならない。

### Custom attribution models

The worklet based scheme possibly allows for more flexible attribution options, including specifying partial "credit" for multiple previous attribution sources that would provide value to advertisers that are interested in attribution models other than last-touch.

ワークレットベースのスキームでは、より柔軟なアトリビューションオプションが可能で、複数の過去のアトリビューションソースに対する部分的な「クレジット」を指定することもでき、ラストタッチ以外のアトリビューションモデルに関心を持つ広告主にとって価値を提供できる可能性があります。

We should be careful in allowing reports to include cross site information from multiple sites, as it could increase the risk of cross site tracking.

### Hide the true number of attribution reports

The presence or absence of an attribution report leaks some potentially sensitive cross-site data in the current design. Therefore, revealing the total count of reports to the reporting origin could leak something sensitive as well (imagine if the reporting origin only ever registered a conversion or impression for a single user).

現在のデザインでは、アトリビューション レポートの有無によって、潜在的に機密性の高いクロスサイトデータが漏れる可能性があります。したがって、レポートの総数をレポート元に公開すると、何か機密情報が漏れる可能性があります(レポート元が 1 人のユーザーのコンバージョンまたはインプレッションしか登録しなかった場合を想像してください)。

To hide the true number of reports, we could:

本当の報道件数を隠すには

- Unconditionally send a null report for every registered attribution trigger (thus making the count a function of only destination-side information)

- 登録されたアトリビューショントリガーごとに無条件に NULL レポートを送信する(したがって、カウントを送信先側の情報のみの関数にする)。

- Add noise to the number of reports by having some clients randomly add noisy null reports. This technique would have to assume some threshold number of unattributed triggers to maintain privacy.

- いくつかのクライアントがランダムにノイズの多い NULL レポートを追加することで、レポート数にノイズを追加します。このテクニックはプライバシーを維持するために、無属性のトリガーをある閾値の数だけ想定する必要があります。

### More advanced contribution bounding

We might want to consider bounding contribution at levels other than the source event level in the future for more robust privacy protection. For instance, we could bound user contributions per trigger event, or even by {source site, destination site, day} for a more user-level bound.

将来的には、より強固なプライバシー保護のために、ソースイベントレベル以外のレベルで貢献度を制限することを検討したいかもしれません。例えば、トリガーイベントごとにユーザーの貢献を制限したり、よりユーザーレベルの制限のために{ソースサイト、デスティネーションサイト、日}で制限することも可能です。

This would likely come at the cost of some utility and complexity, as budgeting across multiple source events may not be straightforward.

この場合、複数のソースイベントをまたいで予算を立てることは容易ではないため、ある程度の実用性と複雑さを犠牲にしている可能性があります。

Additionally, there are more sophisticated techniques that can optimize utility and privacy if we bound more than just the L1 norm of the aggregate histogram. For instance, we could impose a stricter Linf bound (i.e. bounding the contribution to any one bucket). Care should be taken to ensure that either:

さらに、集約されたヒストグラムの L1 ノルム以上のものを束縛すれば、実用性とプライバシーを最適化できる、より洗練された技術もある。例えば、より厳格な Linf 境界(すなわち、任意の 1 つのバケットへの寄与を制限する)を課すことができる。のいずれかを確保するように注意する必要がある。

- A proper compromise is met across various use-cases

- 様々なユースケースで適切な妥協点を見出すことができます。

- We can support multiple types of contribution bounding for different reporting origins without introducing privacy leaks

- プライバシーリークを発生させることなく、異なる報告元に対して複数のタイプの貢献度バウンディングをサポートすることができます。

See [issue 249](https://github.com/WICG/conversion-measurement-api/issues/249) for more details.

詳しくは[249 号](https://github.com/WICG/conversion-measurement-api/issues/249)をご覧ください。

### Choosing among aggregation services

The server can add an optional `alternative_aggregation_mode` string field:

サーバーはオプションで `alternative_aggregation_mode` 文字列フィールドを追加することができる。

```http
Attribution-Reporting-Register-Source: {..., "aggregation_keys": ..., "alternative_aggregation_mode": "experimental-poplar"}
```

The optional field will allow developers to choose among different options for aggregation infrastructure supported by the user agent. This value will allow experimentation with new technologies, and allows us to try out new approaches without interfering with core functionality provided by the default option. The `"experimental-poplar"` option will implement a protocol similar to [poplar VDAF](https://github.com/cfrg/draft-irtf-cfrg-vdaf/blob/main/draft-irtf-cfrg-vdaf.md#poplar1-poplar1) in the [PPM Framework](https://datatracker.ietf.org/doc/draft-gpew-priv-ppm/).

このオプションフィールドは，開発者が，ユーザエージェントがサポートする集約基盤のための異なるオプションの中から選択することを可能にする。この値によって新しい技術の実験が可能になり、デフォルトのオプションで提供されるコア機能に干渉することなく、新しいアプローチを試すことができるようになります。 `"experimental-poplar"` オプションは [PPM Framework](https://datatracker.ietf.org/doc/draft-gpew-priv-ppm/) の [poplar VDAF](https://github.com/cfrg/draft-irtf-cfrg-vdaf/blob/main/draft-irtf-cfrg-vdaf.md#poplar1-poplar1) に似たプロトコルを実装する予定です。

## Considered alternatives

### "Count" vs. "value" histograms

There are some use-cases which require something close to binary input (i.e. counting conversions), and other conversions which require summing in some discretized domain (e.g. summing conversion value).

バイナリ入力に近いものを必要とするユースケース(すなわち計数変換)と，離散化された領域での合計を必要とする変換(例えば変換値の合計)がある。

For simplicity in this API we are treating these exactly the same. Count-based approaches could do something like submitting two possible values, 0 for 0 and MAX_VALUE for 1, and consider the large space to be just a discretized domain of fractions between 0 and 1.

この API では簡略化のため、これらを全く同じに扱っている。カウントベースのアプローチでは、0 には 0、1 には MAX_VALUE という 2 つの可能な値を提出し、大きな空間は 0 と 1 の間の端数の離散化されたドメインだけであると考えるようなことができる。

This has the benefit of keeping the aggregation infrastructure generic and avoids the need to "tag" different reports with whether they represent a coarse-grained or fine-grained value.

これは、アグリゲーションインフラストラクチャを汎用的に維持し、異なるレポートが粗視化された値か細粒化された値かを示す「タグ」の必要性を回避する利点がある。

In the end, we will use this MAX_VALUE to scale noise via computing the sensitivity of the computation, so submitting "1" for counts will yield more noise than otherwise expected.

最終的には、この MAX_VALUE を使って、計算の感度を計算してノイズをスケーリングすることになるので、counts に "1" を指定すると、他の予想よりもノイズが多くなる。

### Binary report format

A binary report format like CBOR could streamline AEAD authentication by passing raw bytes directly to the reporting origin, which could be passed through directly to an aggregation service. This design would avoid parsing / serialization errors in constructing authenticated data necessary to decrypt the payload.

CBOR のようなバイナリレポート形式は、生のバイトを直接レポート元に渡すことで AEAD 認証を合理化し、それを直接アグリゲーションサービスに渡すことができる。この設計により、ペイロードの復号化に必要な認証データの構築における構文解析／シリアライゼーションエラーを回避することができる。

However, binary formats are not as familiar to developers, so there is an ergonomics tradeoff to be made here.

しかし、バイナリ形式は開発者にとって馴染みが薄いため、ここでは人間工学的なトレードオフが存在する。
