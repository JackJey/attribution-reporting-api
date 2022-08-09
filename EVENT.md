_This is an in-depth technical explainer. If you're looking for a high-level introduction to Attribution Reporting with event-level reports, head over to [Event-level reports in the Attribution Reporting API](https://developer.chrome.com/docs/privacy-sandbox/attribution-reporting-event-introduction/)._
_A list of all API guides and blog posts for this API is also available [here](https://developer.chrome.com/docs/privacy-sandbox/attribution-reporting/)._

# Attribution Reporting with event-level reports

This document is an explainer for a potential new web platform feature which allows for measuring and reporting ad click and view conversions.

See the explainer on [aggregate measurement](AGGREGATE.md) for a potential extension on top of this.

!-- START doctoc generated TOC please keep comment here to allow auto update --

!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE --

**Table of Contents**

- [Motivation](#motivation)
- [Related work](#related-work)
- [API Overview](#api-overview)
  - [Registering attribution sources](#registering-attribution-sources)
  - [Handling an attribution source]()
    event](#handling-an-attribution-source-event)
  - [Publisher-side Controls for Attribution Source]()
    Declaration](#publisher-side-controls-for-attribution-source-declaration)
  - [Triggering Attribution](#triggering-attribution)
  - [Registration requests](#registration-requests)
  - [Data limits and noise](#data-limits-and-noise)
  - [Trigger attribution algorithm](#trigger-attribution-algorithm)
  - [Multiple sources for the same trigger]()
    (Multi-touch)](#multiple-sources-for-the-same-trigger-multi-touch)
  - [Sending Scheduled Reports](#sending-scheduled-reports)
  - [Attribution Reports](#attribution-reports)
  - [Data Encoding](#data-encoding)
  - [Optional attribution filters](#optional-attribution-filters)
  - [Optional: extended debugging reports](#optional-extended-debugging-reports)
  - [Noisy fake conversion example](#noisy-fake-conversion-example)
  - [Storage limits](#storage-limits)
- [Privacy Considerations](#privacy-considerations)
  - [Trigger Data](#trigger-data)
  - [Reporting Delay](#reporting-delay)
  - [Reporting origin limits](#reporting-origin-limits)
  - [Clearing Site Data](#clearing-site-data)
  - [Reporting cooldown / rate limits](#reporting-cooldown--rate-limits)
  - [Less trigger-side data](#less-trigger-side-data)
  - [Browsing history reconstruction](#browsing-history-reconstruction)
    - [Adding noise to whether a trigger is]()
      genuine](#adding-noise-to-whether-a-trigger-is-genuine)
    - [Limiting the number of unique destinations covered by pending]()
      sources](#limiting-the-number-of-unique-destinations-covered-by-pending-sources)
  - [Differential privacy](#differential-privacy)
  - [Speculative: Limits based on first party]()
    storage](#speculative-limits-based-on-first-party-storage)
- [Security considerations](#security-considerations)
  - [Reporting origin control](#reporting-origin-control)
  - [Denial of service](#denial-of-service)
  - [Top site permission model](#top-site-permission-model)

!-- END doctoc generated TOC please keep comment here to allow auto update --


## Motivation

Currently, the web ad industry measures conversions via identifiers they can associate across sites. These identifiers tie information about which ads were clicked to information about activity on the advertiser's site (the conversion). This allows advertisers to measure ROI, and for the entire ads ecosystem to understand how well ads perform.

現在、ウェブ広告業界では、サイト間で関連付けることのできる識別子によってコンバージョンを測定しています。これらの識別子は、どの広告がクリックされたかという情報を、広告主のサイトでの活動(コンバージョン)に関する情報と結びつけています。これにより、広告主は ROI を測定し、広告のエコシステム全体は広告のパフォーマンスを理解することができます。

Since the ads industry today uses common identifiers across advertiser and publisher sites to track conversions, these common identifiers can be used to enable other forms of cross-site tracking.

現在、広告業界では、広告主と出版社のサイト間で共通の識別子を使用してコンバージョンを追跡しているため、この共通の識別子を使用して、他の形式のクロスサイトトラッキングを可能にすることができます。

This doesn't have to be the case, though, especially in cases where identifiers such as third-party cookies are either unavailable or undesirable. A new API surface can be added to the web platform to satisfy this use case without them, in a way that provides better privacy to users.

しかし、特にサードパーティークッキーのような識別子が利用できないか、望ましくない場合には、そうである必要はない。新しい API サーフェイスをウェブプラットフォームに追加することで、ユーザーにより良いプライバシーを提供する方法で、それらなしでこのユースケースを満たすことができます。

This API alone will not be able to support all conversion measurement use cases.

この API だけでは、すべてのコンバージョン測定のユースケースに対応することはできません。

We envision this API as one of potentially many new APIs that will seek to reproduce valid advertising use cases in the web platform in a privacy preserving way. In particular, we think this API could be extended by using server side aggregation to provide richer data, which we are continuing to explore.

私たちは、この API を、ウェブプラットフォームにおける有効な広告ユースケースをプライバシーを保護した形で再現しようとする、潜在的に多くの新しい API の一つとして想定しています。特に、この API はサーバーサイドのアグリゲーションを使ってよりリッチなデータを提供することで拡張できると考えており、現在も調査を続けている。


## Related work

There is an alternative [Private Click Measurement](https://github.com/privacycg/private-click-measurement) draft spec in the PrivacyCG. See this [WebKit blog post](https://webkit.org/blog/8943/privacy-preserving-ad-click-attribution-for-the-web/) for more details.

PrivacyCG には、代替となる [Private Click Measurement](https://github.com/privacycg/private-click-measurement) のドラフト仕様があります。詳しくは、こちらの[WebKit blog post](https://webkit.org/blog/8943/privacy-preserving-ad-click-attribution-for-the-web/)をご覧ください。

There are multiple aggregate designs in [wicg/privacy-preserving-ads](https://github.com/WICG/privacy-preserving-ads), including Bucketization and MaskedLARK.

[wicg/privacy-preserving-ads](https://github.com/WICG/privacy-preserving-ads) には、Bucketization や MaskedLARK など、複数のアグリゲートデザインが存在します。

Folks from Meta and Mozilla have published the [Interoperable Private Attribution(IPA)](https://docs.google.com/document/d/1KpdSKD8-Rn0bWPTu4UtK54ks0yv2j22pA5SrAD9av4s/edit#heading=h.f4x9f0nqv28x) proposal for doing aggregate attribution measurement.

Meta と Mozilla の人たちが、アトリビューションの集計を行うための[Interoperable Private Attribution(IPA)](https://docs.google.com/document/d/1KpdSKD8-Rn0bWPTu4UtK54ks0yv2j22pA5SrAD9av4s/edit#heading=h.f4x9f0nqv28x)という提案を発表しています。

Brave has published and implemented an [Ads Confirmation Protocol](https://github.com/brave/brave-browser/wiki/Security-and-privacy-model-for-ad-confirmations).

Brave では、[Ads Confirmation Protocol](https://github.com/brave/brave-browser/wiki/Security-and-privacy-model-for-ad-confirmations)を公開・実装しています。


## API Overview

### Registering attribution sources

Attribution sources are events which future triggers can be attributed to.

Attribution sources とは、将来のトリガーを Attribute させることができる事象のことである。

Sources are registered by returning a new HTTP response header on requests which are eligible for attribution. A request is eligible as long as it has the `Attribution-Reporting-Eligible` request header.

ソースは、Attribute の対象となるリクエストに対して新しい HTTP レスポンスヘッダを返すことで登録されます。リクエストは `Attribution-Reporting-Eligible` リクエストヘッダを持つ限り、適格とみなされます。

There are two types of attribution sources, `navigation` sources and `event` sources.

アトリビューションソースには、 `navigation` ソースと `event` ソースの 2 種類がある。

`navigation` sources are registered via clicks on anchor tags:

アンカータグをクリックすると、 `navigation` ソースが登録されます。

```html
<a
  href="https://advertiser.example/landing"
  attributionsrc="https://adtech.example/attribution_source?my_ad_id=123"
>
  click me
</a>
```

or via calls to `window.open` that occur with [transient activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation):

または、[transient activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation) で発生する `window.open` の呼び出しを経由しています。

```javascript
// Encode the attributionsrc URL in case it contains special characters, such as '=', that will
// cause the parameter to be improperly parsed const encoded = encodeURIComponent(
  "https://adtech.example/attribution_source?my_ad_id=123"
);
window.open(
  "https://advertiser.example/landing",
  "_blank",
   `attributionsrc=${encoded}`
);
```

`event` sources do not require any user interaction and can be registered via `<img>` or `<script>` tags with the new `attributionsrc` attribute:

`event` ソースはユーザーとのインタラクションを必要とせず、新しい `attributionsrc` 属性を持つ `<img>` や `<script>` タグを介して登録することができます。

```html
<img src="https://advertiser.example/pixel"
     attributionsrc="https://adtech.example/attribution_source?my_ad_id=123">

<script src="https://advertiser.example/register-view"
        attributionsrc="https://adtech.example/attribution_source?my_ad_id=123">
```

Specifying a URL value for `attributionsrc` within `<a>` , `<img>` , `<script>` or `window.open` will cause the browser to initiate a separate `keepalive` fetch request which includes the `Attribution-Reporting-Eligible` request header.

`attributionsrc` の URL を `<a>` , `<img>` , `<script>` , `window.open` 内に指定すると、ブラウザは `Attribution-Reporting-Eligible` リクエストヘッダーを含む別の `keepalive` フェッチリクエストを開始します。

When the `attributionsrc` attribute is present in these surfaces/APIs, both with and without a value, existing requests made via `src` / `href` attributes or `window.open` will now include the `Attribution-Reporting-Eligible` request header. Each of these requests will be able to register attribution sources.

これらの surfaces/API に `attributionsrc` 属性が存在する場合、値を指定してもしなくても、 `src` / `href` 属性や `window.open` を通じて行われた既存のリクエストには `Attribution-Reporting-Eligible` リクエストヘッダが含まれるようになりました。これらのリクエストはそれぞれアトリビューションソースを登録できるようになります。

`event` sources can also be registered using existing JavaScript request APIs by setting the `Attribution-Reporting-Eligible` header manually:

イベントソースは、既存の JavaScript リクエスト API を使って、 `Attribution-Reporting-Eligible` ヘッダーを手動で設定することによっても登録することができます。

```javascript
const headers = {
  "Attribution-Reporting-Eligible": "event-source",
};
// Optionally set keepalive to ensure the request outlives the page.
window.fetch("https://adtech.example/attribution_source?my_ad_id=123", {
  headers,
  keepalive: true,
});
```

Other requests APIs which allow specifying headers (e.g. `XMLHttpRequest` ) will also work.

ヘッダを指定できる他のリクエスト API (例: `XMLHttpRequest` ) も動作する。

The response to these requests will configure the API via a new JSON HTTP header `Attribution-Reporting-Register-Source` of the form:

これらのリクエストに対するレスポンスは、フォームの新しい JSON HTTP ヘッダー `Attribution-Reporting-Register-Source` を介して API を設定することになります。

```jsonc
{
  "source_event_id": "12340873456",
  "destination": "[eTLD+1]",
  "expiry": "[64-bit signed integer]",
  "priority": "[64-bit signed integer]"
}
```

- `destination` : an origin whose eTLD+1 is where attribution will be triggered for this source.

- `destination` : eTLD+1 がこのソースのアトリビューションがトリガーされるオリジンです。

- `source_event_id` : (optional) A string encoding a 64-bit unsigned integer which represents the event-level data associated with this source. This will be limited to 64 bits of information but the value can vary. Defaults to 0.

- `source_event_id` : (オプション)このソースに関連するイベントレベルのデータを表す、64 ビットの符号なし整数をエンコードした文字列。これは 64 ビットの情報に制限されるが、値は変化することができる。デフォルトは 0 です。

- `expiry` : (optional) expiry in seconds for when the source should be deleted. Default is 30 days, with a maximum value of 30 days. The maximum expiry can also vary between browsers. This will be rounded to the nearest day.

- `expiry` : (オプション) ソースを削除する際の有効期限を秒数で指定します。デフォルトは 30 日で、最大値は 30 日です。最大有効期限はブラウザによって異なる可能性もあります。これは直近の日数に丸められます。

- `priority` : (optional) a signed 64-bit integer used to prioritize this source with respect to other matching sources. When a trigger redirect is received, the browser will find the matching source with highest `priority` value and generate a report. The other sources will not generate reports.

- `priority` : (オプション) 他のマッチングソースに対してこのソースの優先順位を付けるために使用される符号付き 64 ビット整数です。トリガーのリダイレクトを受信すると、ブラウザは最も高い `priority` 値を持つ一致するソースを見つけ、レポートを生成します。他のソースはレポートを生成しません。

Once this header is received, the browser will proceed with [handling an attribution source event](#handling-an-attribution-source-event). Note that it is possible to register multiple sources for the same request using HTTP redirects (though these multiple sources may not set distinct destinations).

このヘッダーを受信すると、ブラウザは[Attribute 元イベントの処理](#handling-an-attribution-source-event)に進みます。HTTP リダイレクトを使用して、同じリクエストに対して複数のソースを登録することが可能であることに注意してください (ただし、これらの複数のソースは個別の宛先を設定しないかもしれません)。

Note that we sometimes call the `attributionsrc` 's origin the "reporting origin" since it is the origin that will end up receiving attribution reports.

なお、 `attributionsrc` のオリジンを「reporting origin」と呼ぶことがありますが、これは最終的にアトリビューションレポートを受け取ることになるオリジンであるためです。


### Handling an attribution source event

A `navigation` attribution source event will be logged to storage if the resulting document being navigated to ends up sharing an eTLD+1 with the `destination` origin. Additionally, the navigation needs occur with [transient user activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation).

ナビゲートされた結果のドキュメントが `destination` オリジンと eTLD+1 を共有することになった場合、 `navigation` 属性ソースイベントがストレージにログ記録されます。さらに、ナビゲーションは[transient user activation](https://html.spec.whatwg.org/multipage/interaction.html#transient-activation)と共に発生する必要があります。

`event` sources don't require any of the above constraints to be logged.

`event` ソースはログに記録されるために上記の制約を必要としません。

When a source is logged for [\`attributionsrc\` origin, \`destination\`](`attributionsrc` origin, `destination`) pair, existing sources matching this pair will be looked up in storage. If the matching sources have been triggered at least once (i.e. have scheduled a report), they will be removed from browser storage and will not be eligible for further reporting. Any pending reports for these sources will still be sent.

ソースが [\`attributionsrc\` origin, \`destination\`](`attributionsrc` origin, `destination`) のペアでログ記録されると、このペアに一致する既存のソースがストレージで検索されま す。一致するソースが少なくとも 1 回トリガされた (つまり、レポートがスケジュールされた) 場合、ブラウザのストレージから削除され、以降のレポート作成の対象外となります。これらのソースの保留中のレポートは、引き続き送信されます。

An attribution source will be eligible for reporting if any page on the `destination` eTLD+1 (advertiser site) triggers attribution for the associated reporting origin.

アトリビューションソースは、「宛先」eTLD+1(広告主サイト)上のいずれかのページが、関連するレポートオリジンのアトリビューションを引き起こす場合、レポートの対象となります。


### Publisher-side Controls for Attribution Source Declaration

In order to prevent arbitrary third parties from registering sources without the publisher's knowledge, the Attribution Reporting API will need to be enabled in child contexts by a new [Permissions Policy](https://w3c.github.io/webappsec-permissions-policy/):

出版社が知らないうちに任意の第三者がソースを登録することを防ぐため、Attribution Reporting API を子コンテキストで有効にするには、新しい [Permissions Policy](https://w3c.github.io/webappsec-permissions-policy/) が必要です。

```html
<iframe src="https://advertiser.example" allow="attribution-reporting 'src'">
  <a … attributionsrc="https://ad-tech.example?..."></a>
</iframe>
```

The API will be enabled by default in the top-level context and in same-origin children. Any script running in these contexts can declare a source with any reporting origin. Publishers who wish to explicitly disable the API for all parties can do so via an [HTTP header](https://w3c.github.io/webappsec-permissions-policy/#permissions-policy-http-header-field).

API は、最上位コンテキストおよび同一生成元の子コンテキストでデフォルトで有効になります。これらのコンテキストで実行されるスクリプトは、任意のレポート起源を持つソースを宣言することができます。すべての当事者に対して明示的に API を無効にしたいパブリッシャーは、[HTTP ヘッダー](https://w3c.github.io/webappsec-permissions-policy/#permissions-policy-http-header-field) を介してそうすることができます。

Without a Permissions Policy, a top-level document and cooperating iframe could recreate this functionality. This is possible by using [postMessage](https://html.spec.whatwg.org/multipage/web-messaging.html#dom-window-postmessage) to send the `source_event_id` , `attributionsrc` origin, `destination` values to the top level document who can then wrap the iframe in an anchor tag (with some additional complexities behind handling clicks on the iframe). Using Permissions Policy prevents the need for these hacks. This is inline with the classification of powerful features as discussed on [this issue](https://github.com/w3c/webappsec-permissions-policy/issues/252).

Permissions Policy がなければ、トップレベルドキュメントと協力する iframe がこの機能を再現することができます。これは [postMessage](https://html.spec.whatwg.org/multipage/web-messaging.html#dom-window-postmessage) を使って `source_event_id` 、 `attributionsrc` origin、 `destination` の値をトップレベルドキュメントに送ることで可能です。トップレベルドキュメントはその後 iframe をアンカータグでラップできます (iframe のクリックを扱う際にいくつかの追加の複雑な処理が発生します)。Permissions Policy を使用することで、このようなハッキングの必要性を防ぐことができます。これは [this issue](https://github.com/w3c/webappsec-permissions-policy/issues/252) で議論された強力な機能の分類と一致しています。


### Triggering Attribution

Attribution can only be triggered for a source on a page whose eTLD+1 matches the eTLD+1 of the site provided in `destination` . To trigger attribution, a similar mechanism is used as source event registration, via HTML:

アトリビューションは、eTLD+1 が `destination` で指定されたサイトの eTLD+1 と一致するページ上のソースに対してのみトリガーすることができます。アトリビューションのトリガーには、ソースイベントの登録と同様のメカニズムが、HTML を介して使用されます。

```html
<img
  src="https://ad-tech.example/conversionpixel"
  attributionsrc="https://adtech.example/attribution_trigger?purchase=13"
/>
```

or JavaScript:

```javascript
const headers = {
  "Attribution-Reporting-Eligible": "trigger",
};
// Optionally set keepalive to ensure the request outlives the page.
window.fetch("https://adtech.example/attribution_trigger?purchase=13", {
  headers,
  keepalive: true,
});
```

As a stop-gap to support pre-existing conversion tags which do not include the `attributionsrc` attribute, or use a different Fetch API, the browser will also process trigger registration headers for all subresource requests on the page where the `attribution-reporting` Permissions Policy is enabled.

`attributionsrc` 属性を含まない、あるいは異なる Fetch API を使用する既存の変換タグをサポートするための応急処置として、ブラウザは `attribution-reporting` Permissions Policy が有効なページ上のすべてのサブリソース要求に対してトリガー登録ヘッダーを処理することもします。

Like source event registrations, these requests should respond with a new HTTP header `Attribution-Reporting-Register-Trigger` which contains information about how to treat the trigger event:

ソースイベント登録と同様に、これらのリクエストはトリガーイベントをどのように 扱うかについての情報を含む新しい HTTP ヘッダー `Attribution-Reporting-Register-Trigger` で応答する必要があります。

```jsonc
{
  "event_trigger_data": [
    {
      "trigger_data": "[unsigned 64-bit integer]",
      "priority": "[signed 64-bit integer]",
      "deduplication_key": "[unsigned 64-bit integer]"
    }
  ]
}
```

- `trigger_data` : optional coarse-grained data to identify the triggering event. The value will be limited to either 3 bits or 1 bit [depending on the attributed source type](#data-limits-and-noise).

- `trigger_data` : トリガーとなるイベントを特定するための、オプションの粗視化されたデータ。この値は 3 ビットまたは 1 ビットに制限される[帰属するソースのタイプに依存](#data-limits-and-noise)。

- `priority` : optional signed 64-bit integer representing the priority of this trigger compared to other triggers for the same source.

- `priority` : オプションの符号付き 64 ビット整数で、同じソースに対する他のトリガーと比較したこのトリガーの優先度を表します。

- `deduplication_key` : optional unsigned 64-bit integer which will be used to deduplicate multiple triggers which contain the same `deduplication_key` for a single source.

- `deduplication_key` : オプションの符号なし 64 ビット整数で、一つのソースに対して同じ `deduplication_key` を含む複数のトリガーを重複排除するために使用されます。

When this header is received, the browser will schedule an attribution report as detailed in [Trigger attribution algorithm](#trigger-attribution-algorithm).

このヘッダーを受信すると、ブラウザは[トリガー属性アルゴリズム](#trigger-attribution-algorithm)の詳細に従って属性レポートをスケジューリングするようになります。

Note that the header can be present on redirect requests.

このヘッダーはリダイレクトリクエストに存在することができることに注意してください。

Triggering attribution requires the `attribution-reporting` Permissions Policy to be enabled in the context the request is made. As described in [Publisher Controls for Attribution Source Declaration](#publisher-side-controls-for-attribution-source-declaration), this Permissions Policy will be enabled by default in the top-level context and in same-origin children, but disabled in cross-origin children.

属性をトリガーするには、リクエストが行われたコンテキストで `attribution-reporting` Permissions Policy が有効であることが必要です。[Publishers Controls for Attribution Source Declaration](#publisher-side-controls-for-attribution-source-declaration) で説明されているように、この Permissions Policy はトップレベルのコンテキストと同一生成元の子コンテンツでデフォルトで有効になり、クロス生成元の子コンテンツで無効になります。

Navigation sources may be attributed up to 3 times. Event sources may be attributed up to 1 time.

ナビゲーションソースは最大 3 回まで帰属させることができます。イベントソースは 1 回まで帰属させることができます。


### Registration requests

Depending on the context in which it was made, a request is eligible to register sources, triggers, sources or triggers, or nothing, as indicated in the `Attribution-Reporting-Eligible` request header, which is a [structured dictionary](https://www.rfc-editor.org/rfc/rfc8941.html#name-dictionaries).

リクエストが行われたコンテキストに応じて、リクエストはソース、トリガー、 ソースまたはトリガー、あるいは何も登録する資格がある。これは、[structured dictionary](https://www.rfc-editor.org/rfc/rfc8941.html#name-dictionaries) である `Attribution-Reporting-Eligible` リクエストヘッダに示されている通りである。

The reporting origin may use the value of this header to determine which registrations, if any, to include in its response. The browser will likewise ignore invalid registrations:

報告元は、もしあれば、どの登録をその応答に含めるかを決定するために、このヘッダーの値を使用することができる。ブラウザは同様に、無効な登録を無視する。

1. `<a>` and `window.open` will have `navigation-source`.
2. `<a>` と `window.open` は `navigation-source` を持つことになります。
3. Other APIs that automatically set `Attribution-Reporting-Eligible` (like `<img>` ) will contain `event-source, trigger` .
4. その他の API で、自動的に `Attribution-Reporting-Eligible` を設定するもの(`<img>` など)には、 `event-source, trigger` .
5. Requests from JavaScript, e.g. `window.fetch` , can set this header manually, but it is an error for such requests to specify `navigation-source` .
6. JavaScript からのリクエスト、例えば `window.fetch` はこのヘッダーを手動で設定できますが、このようなリクエストで `navigation-source` を指定するのはエラーになります。
7. All other requests will not have the `Attribution-Reporting-Eligible` header. For those requests the browser will permit trigger registration only.
8. その他のすべてのリクエストは `Attribution-Reporting-Eligible` ヘッダーを持ち ません。これらのリクエストに対して、ブラウザはトリガー登録のみを許可します。


### Data limits and noise

The `source_event_id` will be limited to 64 bits of information to enable uniquely identifying an ad click.

広告のクリックを一意に識別できるように、`source_event_id`は 64 ビットの情報に制限されます。

The advertiser-side data must therefore be limited quite strictly, by limiting the amount of data and by applying noise to the data. `navigation` sources will be limited to only 3 bits of `trigger_data` , while `event` sources will be limited to only 1 bit.

そのため、広告主側のデータは、データ量を制限したり、データにノイズを加えたりして、かなり厳密に制限する必要があります。ナビゲーションのソースは 3 ビットの `trigger_data` に制限され、イベントのソースは 1 ビットに制限されます。

Noise will be applied to whether a source event will be reported truthfully.

ノイズは、ソースイベントが正直に報告されるかどうかに適用されます。

When an attribution source is registered, the browser will perform one of the following steps given a probability `p` :

アトリビューションソースが登録されると、ブラウザは確率 `p` が与えられると、次のいずれかのステップを実行します。

- With probability `1 - p` , the browser logs the source as normal

- 確率 `1 - p` で、ブラウザは通常通りソースをログに記録します。

- With probability `p` , the browser chooses randomly among all the possible output states of the API. This includes the choice of not reporting anything at all, or potentially reporting multiple fake reports for the event.

- 確率 `p` で、ブラウザは API のすべての可能な出力状態の中からランダムに選択する。これには、全く何も報告しない、あるいはイベントに対して複数の偽の報告をする可能性がある、という選択肢が含まれる。

Note that this scheme is an instantiation of k-randomized response, see [Differential privacy](#differential-privacy).

この方式は、[Differential privacy](#differential-privacy)を参照した k-randomized response のインスタンス化であることに注意してください。

Strawman: we can set `p` such that each source is protected with randomized response that satisfies an epsilon value of 14. This would entail:

ストローマン:各ソースが 14 のイプシロン値を満たすランダム化されたレスポンスで保護されるように `p` を設定することができます。これは必然的にそうなる。

- `p = .24%` for `navigation` sources
- `p = .00025%` for `event` sources

Note that correcting for this noise addition is straightforward in most cases, please see TODO link to de-biasing advice/code snippet here. Reports will be updated to include `p` so that noise correction can work correctly in the event that `p` changes over time, or if different browsers apply different probabilities:

このノイズの追加を修正することは、ほとんどの場合、簡単なことであることに注意してください。レポートは `p` を含むように更新され、`p` が時間とともに変化したり、ブラウザによって異なる確率が適用された場合、ノイズ補正が正しく動作するようになります。

```jsonc
{
  "randomized_trigger_rate": 0.0024,
  ...
}
```

Note that these initial strawman parameters were chosen as a way to ease adoption of the API without negatively impacting utility substantially. They are subject to change in the future with additional feedback and do not necessarily reflect a final set of parameters.

これらの初期のストローマンパラメータは、実用性に大きな影響を与えることなく、API の採用を容易にする方法として選択されたことに留意されたい。これらは将来、追加的なフィードバックによって変更される可能性があり、必ずしも最終的なパラメータセットを反映するものではありません。


### Trigger attribution algorithm

When the browser receives an attribution trigger redirect on a URL matching the `destination` eTLD+1, it looks up all sources in storage that match [\`attributionsrc\` origin, \`destination\`](`attributionsrc` origin, `destination`) and picks the one with the greatest `priority` . If multiple sources have the greatest `priority` , the browser picks the one that was stored most recently.

ブラウザは、`destination` eTLD+1 に一致する URL で属性トリガーリダイレクトを受信すると、[\`attributionsrc\` origin, \`destination\`](`attributionsrc` origin, `destination`) に一致するストレージ内のすべてのソースを検索して、最大 の `priority` を持つ 1 つを選択します。複数のソースが最大の `priority` を持つ場合、ブラウザは最も新しく保存されたソースを選択する。

The browser then schedules a report for the source that was picked by storing { `attributionsrc` origin, `destination` eTLD+1, `source_event_id` , [decoded](#data-encoding) `trigger_data` , `priority` , `deduplication_key` } for the source. Scheduled reports will be sent as detailed in [Sending scheduled reports](#sending-scheduled-reports).

ブラウザは次に、{ `attributionsrc` origin, `destination` eTLD+1, `source_event_id` , [decoded](#data-encoding) `trigger_data` , `priority` , `deduplication_key` } を保存して選択したソースに対してレポートをスケジューリングします。スケジュールレポートは[スケジュールレポートの送信](#sending-scheduled-reports)で説明されている方法で送信されます。

The browser will create reports for a source only if the trigger's `deduplication_key` has not already been associated with a report for that source.

ブラウザは、トリガーの `deduplication_key` がそのソースのレポートにまだ関連付けられていない場合にのみ、そのソースのレポートを作成します。

Each `navigation` source is allowed to schedule only a maximum of three reports, while each `event` source is only allowed to schedule a maximum of one.

各 `navigation` ソースは最大 3 つのレポートしかスケジュールできません。一方、各 `event` ソースは最大 1 つしかスケジュールできません。

If a source has already scheduled the maximum number of reports when a new report is being scheduled, the browser will compare the priority of the new report with the priorities of the scheduled reports for that source. If the new report has the lowest priority, it will be ignored. Otherwise, the browser will delete the scheduled report with the lowest priority and schedule the new report.

新しいレポートがスケジュールされているときに、ソースがすでに最大数のレポートをスケジュールしている場合、ブラウザは新しいレポートの優先度をそのソースのスケジュールされたレポートの優先度と比較します。新しいレポートの優先順位が最も低い場合、それは無視されます。そうでない場合、ブラウザは最も低い優先度を持つスケジュールされたレポートを削除し、新しいレポートをスケジュールします。


### Multiple sources for the same trigger (Multi-touch)

If multiple sources were registered and associated with a single attribution trigger, send reports for the one with the highest priority. If no priority is specified, the browser performs last-touch.

複数のソースが登録され、1 つのアトリビューション・トリガーに関連付けられた場合、最も高い優先度を持つもののレポートを送信します。優先順位が指定されていない場合、ブラウザはラストタッチを実行します。

There are many possible alternatives to this, like providing a choice of rules-based attribution models. However, it isn't clear the benefits outweigh the additional complexity. Additionally, models other than last-click potentially leak more cross-site information if sources are clicked across different sites.

これには、ルールベースのアトリビューションモデルの選択肢を提供するなど、多くの代替案が考えられます。しかし、追加される複雑さに見合うだけのメリットがあるかどうかは明らかではありません。さらに、Last-Click 以外のモデルは、ソースが異なるサイトでクリックされた場合、より多くのクロスサイト情報を漏らす可能性があります。


### Sending Scheduled Reports

Reports for `event` sources will be sent 1 hour after the source expires at its `expiry` .

`event` ソースのレポートは、そのソースの有効期限が切れた 1 時間後に `expiry` として送信されます。

Reports for `navigation` sources may be reported earlier than the source's `expiry` , at specified points in time relative to when the source event was registered. See [here](https://wicg.github.io/conversion-measurement-api/#delivery-time) for the precise algorithm.

`navigation` のレポートは、ソース・イベントが登録された時点から相対的に指定された時点で、ソースの `expiry` よりも早くレポートされることがあります。正確なアルゴリズムについては、[here](https://wicg.github.io/conversion-measurement-api/#delivery-time)を参照してください。

Note that the report may be sent at a later date if the browser was not running when the window finished. In this case, reports will be sent on startup. The browser may also decide to delay some of these reports for a short random time on startup, so that they cannot be joined together easily by a given reporting origin.

なお、ウィンドウ終了時にブラウザが起動していなかった場合、レポートが後日送信されることがあります。この場合、レポートは起動時に送信されます。また、ブラウザは起動時にこれらのレポートのいくつかを短いランダムな時間だけ遅延させることにして、所定のレポート作成元で簡単に結合できないようにすることもできる。


### Attribution Reports

To send a report, the browser will make a non-credentialed (i.e. without session cookies) secure HTTP POST request to:

レポートを送信するには、ブラウザは、クレデンシャルを持たない(すなわち、セッションクッキーを持たない)安全な HTTP POST リクエストを次のように行います。

```
https://<reporting origin>/.well-known/attribution-reporting/report-event-attribution
```

The report data is included in the request body as a JSON object with the following keys:

レポートデータは、以下のキーを持つ JSON オブジェクトとしてリクエストボディに含まれます。

- `attribution_destination` : the attribution destination set on the source
- `source_event_id` : 64-bit event id set on the attribution source
- `trigger_data` : Coarse data set in the attribution trigger registration
- `report_id` : A UUID string for this report which can be used to prevent double counting
- `source_type` : Either "navigation" or "event", indicates whether this source was associated with a navigation.
- `randomized_trigger_rate` : Decimal number between 0 and 1 indicating how often noise is applied.


### Data Encoding

The source event id and trigger data should be specified in a way that is amenable to the privacy assurances a browser wants to provide (i.e. the number of distinct data states supported).

ソースイベント ID とトリガーデータは、ブラウザが提供したいプライバシー保証に従順な方法で指定されるべきである(すなわち、サポートされる別個のデータステートの数)。

The input values will be 64-bit integers which the browser will interpret modulo its maximum data value chosen by the browser. The browser will take the input and performs the equivalent of:

入力値は 64 ビット整数で、ブラウザが選択した最大データ値に変調を加えて解釈する。ブラウザは入力を受けて、それに相当する処理を行う。

```javascript
function getData(input, max_value) {
  return input % max_value;
}
```

The benefit of this method over using a fixed bit mask is that it allows browsers to implement max_values that aren't multiples of 2. That is, browers can choose a "fractional" bit limit if they want to.

この方法の利点は、ブラウザが 2 の倍数でない max_value を実装できることです。つまり、ブラウザが望むなら、「小数」のビット制限を選択できます。


### Optional attribution filters

Source and trigger registration has additional optional functionality to both:

ソースとトリガーの登録は、両者にオプション機能を追加しています。

1. Selectively filter some triggers (effectively ignoring them)

1. 一部のトリガーを選択的にフィルタリングする(効果的に無視する)。

1. Choose trigger data based on source event data

1. ソースイベントデータを元にトリガーデータを選択する

This can be done via simple extensions to the registration configuration.

これは、登録設定の簡単な拡張機能によって行うことができます。

Source registration:

```jsonc
{
  "source_event_id": "12345678",
  "destination": "https://toasters.example",
  "expiry": "604800000",
  "filter_data": {
    "conversion_subdomain": ["electronics.megastore", "electronics2.megastore"],
    "product": ["1234"]
    // Note that "source_type" will be automatically generated as
    // one of {"navigation", "event"}
  }
}
```

Trigger registration:

```jsonc
{
  ... // existing fields, such as  `event_trigger_data`

  // Note that "not_filters", which filters with a negation, is also supported.
  "filters": {
    "conversion_subdomain": ["electronics.megastore"],
    // Not set on the source side, so this key is ignored
    "directory": ["/store/electronics]"
  }
}
```

If keys in the filters JSON match keys in `filter_data` , the trigger is completely ignored if the intersection is empty.

filters JSON のキーが `filter_data` のキーにマッチする場合、交差点が空であれば、トリガーは完全に無視されます。

Note: A key which is present in one JSON and not the other will not be included in the matching logic.

注:一方の JSON に存在し、他方に存在しないキーは、マッチングロジックに含まれません。

Note: The filter JSON does not support nested dictionaries or lists. `filter_data` and `filters` are only allowed to have a list of values with string type.

Note: フィルター JSON は、ネストされた辞書やリストをサポートしない。`filter_data` と `filters` は、文字列型の値のリストのみを持つことができる。

The `event_trigger_data` field can also be extended to do selective filtering to set `trigger_data` based on `filter_data` :

`event_trigger_data` フィールドを拡張して、 `filter_data` に基づいて `trigger_data` を設定する選択的なフィルタリングを行うことも可能である。

```jsonc
// Filter by the source type to handle different bit limits.
{
  "event_trigger_data": [
    {
      "trigger_data": "2",
      // Note that "not_filters", which filters with a negation, is also supported.
      "filters": { "source_type": ["navigation"] }
    },
    {
      "trigger_data": "1",
      "filters": { "source_type": ["event"] }
    }
  ]
}
```

If the filters do not match for any of the event triggers, no event-level report will be created.

イベントトリガーのいずれにもフィルタが一致しない場合、イベントレベルのレポートは作成されません。

If the filters match for multiple event triggers, the first matching event trigger is used.

複数のイベントトリガーのフィルターが一致した場合、最初に一致したイベントトリガーが使用されます。


### Optional: extended debugging reports

The Attribution Reporting API is a new and fairly complex way to do attribution measurement without third-party cookies. As such, we are open to introducing a transitional mechanism to learn more information about attribution reports _while third-party cookies are available_. This ensures that the API can be fully understood during roll-out and help flush out any bugs (either in browser or caller code), and more easily compare the performance to cookie-based alternatives.

Attribution Reporting API は、サードパーティの Cookie を使わずにアトリビューション測定を行うための新しく、かなり複雑な方法です。そのため、サードパーティのクッキーが利用できる間は、アトリビューションレポートの詳細情報を知るための過渡的なメカニズムを導入することにオープンである\_。これにより、ロールアウト中に API を完全に理解することができ、(ブラウザまたは呼び出し元のコードの)バグを洗い出し、より簡単にクッキーベースの代替品とパフォーマンスを比較することができます。

Source and trigger registrations will both accept a new field `debug_key` :

ソース登録とトリガー登録は、両方とも新しいフィールド `debug_key` を受け入れます。

```jsonc
{
  ...
  "debug_key": "[64-bit unsigned integer]"
}
```

Reports will include up to two new parameters which pass any specified debug keys from source and trigger events unaltered:

レポートには、ソースおよびトリガーイベントから指定された任意のデバッグキーをそのまま渡す、最大 2 つの新しいパラメータが含まれます。

```jsonc
{
  // normal report fields...
  "source_debug_key": "[64-bit unsigned integer]",
  "trigger_debug_key": "[64-bit unsigned integer]"
}
```

If a report is created with both source and trigger debug keys, a duplicate debug report will be sent immediately to a `.well-known/attribution-reporting/debug/report-event-attribution` endpoint. The debug reports will be identical to normal reports, including the two debug key fields. Including these keys in both allows tying normal reports to the separate stream of debug reports.

もしレポートがソースとトリガーの両方のデバッグキーで作成された場合、デバッグレポートの複製が `.well-known/attribution-reporting/debug/report-event-attribution` エンドポイントに直ちに送信されます。デバッグレポートは、2 つのデバッグキーフィールドを含む、通常のレポートと同じものになります。これらのキーを両方に含めることで、通常のレポートをデバッグレポートの個別のストリームに関連付けることができます。

Note that event-level reports associated with false trigger events will not have `trigger_debug_key` s. This allows developers to more closely understand how noise is applied in the API.

偽のトリガーイベントに関連するイベントレベルのレポートは `trigger_debug_key` を持たないことに注意してください。

To ensure that this data (which could enable cross-site tracking) is only available in a transitional phase while third-party cookies are available and are already capable of user tracking, the browser will check (at both source and trigger registration) for the presence of a special `SameSite=None` cookie set by the reporting origin:

このデータ(クロスサイト・トラッキングを可能にする)が、サードパーティ・クッキーが利用可能で、すでにユーザー・トラッキングが可能である間の過渡期にのみ利用できるように、ブラウザは、報告元の設定による特別な`SameSite=None`クッキーの存在を(報告とトリガーの両方の登録で)確認します。

```http
Set-Cookie: ar_debug=1; SameSite=None; Secure; HttpOnly
```

If a cookie of this form is not present, debugging information will be ignored.

この形式のクッキーが存在しない場合、デバッグ情報は無視されます。


## Sample Usage

`publisher.example` wants to show ads on their site, so they contract out to `ad-tech.example` . `ad-tech.example` 's script in the main document creates a cross-origin iframe to host the third party advertisement for `toasters.example` .

`publisher.example` は自分のサイトに広告を表示したいので、 `ad-tech.example` と契約しています。メインドキュメントの `ad-tech.example` のスクリプトがクロスオリジン iframe を作成し、 `toasters.example` のサードパーティ広告をホスティングします。

Within the iframe, `ad-tech-3p.example` code annotates their anchor tags to use the `ad-tech.example` reporting origin, and sets the `attributionsrc` attribute based on the ad that was served (e.g. some ad with id 123456).

iframe 内の `ad-tech-3p.example` コードは、アンカー タグに `ad-tech.example` のレポート オリジンを使用するように注釈を付け、配信された広告(例:ID 123456 の広告)に基づいて `attributionsrc` 属性を設定します。

```html
<iframe
  src="https://ad-tech-3p.example/show-some-ad"
  allow="attribution-reporting"
>
  ...
  <a
    href="https://toasters.example/purchase"
    attributionsrc="https://ad-tech.example?adid=123456"
  >
    click me!
  </a>
  ...
</iframe>
```

A user clicks on the ad and this opens a window that lands on a URL to `toasters.example/purchase` . In the background, the browser issues an HTTP request to `https://ad-tech.example?adid=123456` . The ad-tech responds with a `Attribution-Reporting-Register-Source` JSON header:

ユーザーが広告をクリックすると、ウィンドウが開いて `toasters.example/purchase` への URL が表示されます。バックグラウンドで、ブラウザは`https://ad-tech.example?adid=123456`に HTTP リクエストを発行します。アドテクノロジーは `Attribution-Reporting-Register-Source` という JSON ヘッダーで応答する。

```jsonc
{
  "source_event_id": "12345678",
  "destination": "https://toasters.example",
  "expiry": "604800000"
}
```

2 days later, the user buys something on `toasters.example` . `toasters.example` triggers attribution on the few different ad-tech companies it buys ads on, including `ad-tech.example` , by adding conversion pixels:

2 日後、そのユーザーは `toasters.example` で何かを購入しました。`toasters.example` はコンバージョンピクセルを追加することで、 `ad-tech.example` を含む、広告を購入しているいくつかの異なるアドテク企業でアトリビューションを引き起こします。

```html
<img
  src="..."
  attributionsrc="https://ad-tech.example/trigger-attribution?model=toastmaster3000&price=$49.99&..."
/>
```

`ad-tech.example` receives this request, and decides to trigger attribution on `toasters.example` . They must compress all of the data into 3 bits, so `ad-tech.example` chooses to encode the value as "2" (e.g. some bucketed version of the purchase value). They respond to the request with an `Attribution-Reporting-Register-Trigger` header:

このリクエストを受け取った `ad-tech.example` は、`toasters.example` にアトリビューションをかけることを決定しました。彼らはすべてのデータを 3 ビットに圧縮しなければならないので、`ad-tech.example`は値を「2」としてエンコードすることを選択します(例えば、購入値のバケット化されたバージョンなど)。彼らは `Attribution-Reporting-Register-Trigger` ヘッダでリクエストに応答します。

```jsonc
{
  "event_trigger_data": [
    {
      "trigger_data": "2"
    }
  ]
}
```

The browser sees this response, and schedules a report to be sent. The report is associated with the 7-day deadline as the 2-day deadline has passed. Roughly 5 days later, `ad-tech.example` receives the following HTTP POST to `https://ad-tech.example/.well-known/attribution-reporting/report-event-attribution` with the following body:

ブラウザはこの応答を見て、レポートの送信を予約する。このレポートは、2 日間の期限が過ぎたので、7 日間の期限に関連付けられます。およそ 5 日後、 `ad-tech.example` は `https://ad-tech.example/.well-known/attribution-reporting/report-event-attribution` に次のボディを持つ HTTP POST を受信します。

```jsonc
{
  "attribution_destination": "https://toasters.example",
  "source_event_id": "12345678",
  "trigger_data": "2"
}
```


### Noisy fake conversion example

Assume the caller uses the same inputs as in the above example, however, the [noise mechanism](#data-limits-and-noise) in the browser chooses to generate completely fake data for the source event. This occurs with some probability `p` .

呼び出し側が上記の例と同じ入力を使用し、しかしブラウザの [noise mechanism](#data-limits-and-noise) がソースイベントに対して完全に偽のデータを生成することを選択したと仮定します。これはある確率 `p` で発生します。

To generate fake events, the browser considers all possible outputs for a given source event:

偽イベントを生成するために、ブラウザは与えられたソースイベントに対して可能なすべての出力を考慮します。

- No reports at all
- One report with metadata "0" at the first reporting window
- One report with metadata "1" at the first reporting window and one report with metadata "3" at the second reporting window
- etc. etc. etc.

After enumerating all possible outputs of the API for a given source event, the browser simply selects one of them at random uniformly. Any subsequent true trigger events that would be attributed to the source event are completely ignored.

与えられたソースイベントに対する API のすべての可能な出力を列挙した後、ブラウザは単にそれらのうちの 1 つを一様にランダムに選択します。そのソースイベントに起因する、それ以降の真のトリガーイベントは完全に無視されます。

In the above example, the browser could have chosen to generate three reports:

上記の例では、ブラウザは 3 つのレポートを生成することを選択することができました。

- One report with metadata "7", sent 2 days after the click
- One report with metadata "3", sent 7 days after the click
- One report with metadata "0", also sent 7 days after the click


### Storage limits

The browser may apply storage limits in order to prevent excessive resource usage.

Strawman: There should be a limit of 1024 pending sources per source origin.

Strawman: There should be a limit of 1024 pending event-level reports per destination site.


## Privacy Considerations

A primary privacy goal of the API is to make _linking identity_ between two different top-level sites difficult. This happens when either a request or a JavaScript environment has two user IDs from two different sites simultaneously.

API の主なプライバシー目標は、2 つの異なるトップレベルサイト間の*linking identity*を困難にすることである。これは、リクエストまたは JavaScript 環境のいずれかが、2 つの異なるサイトからの 2 つのユーザー ID を同時に持っている場合に起こります。

Secondary goals of the API are to:

API の二次的な目標は以下の通りです。

- give some level of _plausible deniability_ to cross-site data leakage associated with source events.
- limit the raw amount of cross-site information a site can learn relative to a source event

In this API, the 64-bit source ID can encode a user ID from the publisher's top- level site, but the low-entropy, noisy trigger data could only encode a small part of a user ID from the advertiser's top-level site. The source ID and the trigger data are never exposed to a JavaScript environment together, and the request that includes both of them is sent without credentials and at a different time from either event, so the request adds little new information linkable to these events. This allows us to limit the information gained by the ad-tech relative to a source event.

この API では、64 ビットのソース ID はパブリッシャーのトップレベルサイトからのユーザー ID をエンコードできますが、低エントロピーでノイズの多いトリガーデータは、広告主のトップレベルサイトからのユーザー ID のほんの一部をエンコードするだけでした。ソース ID とトリガーデータが一緒に JavaScript 環境にさらされることはなく、両者を含むリクエストはクレデンシャルなしで、どちらのイベントとも異なる時間に送信されるため、リクエストはこれらのイベントにリンクできる新しい情報をほとんど追加しない。これにより、ソースイベントに対してアドテクが得る情報を制限することができます。

Additionally, there is a small chance that all the output for a given source event is completely fabricated by the browser, giving the user plausible deniability whether subsequent trigger events actually occurred the way they were reported.

さらに、あるソースイベントに対する全ての出力が、ブラウザによって完全に捏造される可能性もわずかながらあり、その後のトリガーイベントが実際に報告されたとおりに発生したかどうか、ユーザにもっともらしい否認権を与えることになります。


### Trigger Data

Trigger data, e.g. advertiser-side data, is extremely important for critical use cases like reporting the _purchase value_ of a conversion. However, too much advertiser-side data could be used to link advertiser identity with publisher identity.

トリガーデータ、例えば広告主側のデータは、コンバージョンの*購入価値*を報告するような重要なユースケースには非常に重要です。しかし、広告主側のデータが多すぎると、広告主のアイデンティティとパブリッシャーのアイデンティティをリンクするために使用される可能性があります。

Mitigations against this are to provide only coarse information (only a few bits at a time), and introduce some noise to the API output. Even sophisticated attackers will therefore need to invoke the API multiple times (through multiple clicks/views) to join identity between sites with high confidence.

これに対する緩和策は、粗い情報(一度に数ビットのみ)しか提供しないことと、API 出力に多少のノイズを導入することである。したがって、洗練された攻撃者であっても、サイト間のアイデンティティを高い信頼性で結合するためには、(複数のクリックやビューを通じて)API を何度も呼び出す必要があります。

Note that this noise still allows for aggregate measurement of bucket sizes with an unbiased estimator (assuming rate-limits are not hit) See generic approaches of dealing with [Randomized response](https://en.wikipedia.org/wiki/Randomized_response) for a starting point.

このノイズでも、不偏推定量によるバケットサイズの集計が可能であることに注意してください(レートリミットにヒットしないことを前提としています)。出発点として、[Randomized response](https://en.wikipedia.org/wiki/Randomized_response) を扱う一般的なアプローチを参照してください。

TODO: Update this script to account for the more complicated randomized response approach.

TODO: このスクリプトを更新して、より複雑なランダム化された応答方法を考慮するようにした。


### Reporting Delay

By bucketing reports within a small number reporting deadlines, it becomes harder to associate a report with the identity of the user on the advertiser's site via timing side channels.

レポートを少数のレポート期限内にバケット化することで、タイミング・サイド・チャネルを介して広告主サイトのユーザーのアイデンティティとレポートを関連付けることが難しくなります。

Reports within the same reporting window occur within an anonymity set with all others during that time period. For example, if we didn't bucket reports with a delay (and instead sent them immediately after a trigger event), the reports (which contain publisher IDs) could be easily joined up with the advertiser's first-party information via correlating timestamps.

同じレポートウィンドウ内のレポートは、その時間帯の他のすべてのレポートと一緒に匿名セットで表示されます。例えば、レポートをバケットに入れず(トリガーイベントの直後に送信)、レポート(パブリッシャー ID を含む)は、相関するタイムスタンプによって広告主のファーストパーティ情報と容易に結合することができます。

Note that the delay windows / deadlines chosen represent a trade-off with utility, since it becomes harder to properly assign credit to a click if the time from click to conversion is not known. That is, time-to-conversion is an important signal for proper attribution. Browsers should make sure that this trade-off is concretely evaluated for both privacy and utility before deciding on a delay.

クリックからコンバージョンまでの時間がわからないと、クリックのクレジットを適切に割り当てることが難しくなるため、選択した遅延ウィンドウ/デッドラインは、効用とのトレードオフであることに注意してください。つまり、コンバージョンまでの時間は、適切なアトリビューションのための重要な信号です。ブラウザは、遅延を決定する前に、このトレードオフがプライバシーと実用性の両方について具体的に評価されていることを確認する必要があります。


### Reporting origin limits

If the advertiser is allowed to cycle through many possible reporting origins, then the publisher and advertiser don't necessarily have to agree apriori on what origin to use, and which origin actually ends up getting used reveals some extra information.

もし広告主が多くの可能な報告元を循環させることができれば、出版社と広告主はどの報告元を使うかについて必ずしも事前に合意する必要はなく、実際に使われることになる報告元はいくつかの追加情報を明らかにすることができます。

To prevent this kind of abuse, the browser should limit the number of reporting origins per source site, destination site pair, counted per source registration. This should be limited to 100 origins per 30 days.

この種の不正使用を防止するために、ブラウザは、送信元サイトと送信先サイトのペアごとに、送信元登録ごとにカウントされるレポート送信元の数を制限する必要があります。これは、30 日あたり 100 オリジンまでに制限されるべきです。

Additionally, there should be a limit of 10 reporting origins per source site, destination site, 30 days, counted for every attribution that is generated.

さらに、アトリビューションが発生するごとにカウントされるレポートオリジンは、ソースサイト、デスティネーションサイト、30 日あたり 10 件までとする。


### Clearing Site Data

Attribution source data and attribution reports in browser storage should be clearable using existing "clear browsing data" functionality offered by browsers.

ブラウザストレージ内のアトリビューションソースデータとアトリビューションレポートは、ブラウザが提供する既存の「閲覧データのクリア」機能を使ってクリアできるようにする必要があります。


### Reporting cooldown / rate limits

To limit the amount of user identity leakage between a source site, destination pair, the browser should throttle the amount of total information sent through this API in a given time period for a user. The browser should set a maximum number of attributions per

送信元サイトと送信先ペアの間でユーザーの身元が漏れる量を制限するために、ブラウザは、ユーザーの一定時間内にこの API を通じて送信される情報の総量を調整する必要があります。ブラウザは、1 回の送信で送信できる属性数の上限を設定する必要があります。

source site, destination, reporting origin, user tuple per time period. If this threshold is hit, the browser will stop scheduling reports the API for the rest of the time period for attributions matching that tuple.

ソースサイト、宛先、レポート元、期間ごとのユーザータプル。この閾値に達した場合、ブラウザはそのタプルに一致する属性について、残りの期間の API レポートのスケジューリングを停止します。

The longer the cooldown windows are, the harder it is to abuse the API and join identity. Ideally attribution thresholds should be low enough to avoid leaking too much information, with cooldown windows as long as practically possible.

クールダウンウィンドウが長ければ長いほど、API を悪用して ID に参加することが難しくなります。理想的には、アトリビューションのしきい値は、多くの情報を漏らさないように十分に低く、クールダウンウィンドウは現実的に可能な限り長くする必要があります。

Note that splitting these limits by the reporting origin introduces a possible leak when multiple origins collude with each other. However, the alternative makes it very difficult to adopt the API if all reporting origins had to share a budget.

これらの制限を報告元で分割すると、複数の報告元が互いに共謀した場合にリークする可能性があることに注意してください。しかし、すべての報告元が予算を共有しなければならない場合、この代替案では API を採用することが非常に難しくなる。

Strawman rate limit: 100 attributions per {source site, destination, reporting origin, 30 days}

ストローマン率制限:100 アトリビュート/{ソースサイト、デスティネーション、レポートオリジン、30 日}。


### Less trigger-side data

Registering event attribution sources is not gated on a user interaction or top-level navigation, allowing them to be registered more frequently and with greater ease. For example, by restricting to 1 bit of data and 1 report per event source, a `reportingorigin` would need to register many more sources in order to link cross-site identity relative to the Click Through API.

イベントアトリビューションソースの登録は、ユーザーインタラクションやトップレベルナビゲーションで制限されないため、より頻繁に、より簡単に登録することができます。例えば、イベントソースごとに 1 ビットのデータと 1 レポートに制限することで、`reportingorigin` はクリックスルー API と比較して、クロスサイトのアイデンティティをリンクするために、より多くのソースを登録する必要が生じます。

This is further restricted by rate-limiting the usage of the API between two sites, using [reporting cooldowns](event_attribution_reporting_clicks.md#reporting-cooldown--rate-limits).

さらに、 [reporting cooldowns](event_attribution_reporting_clicks.md#reporting-cooldown--rate-limits) を使って、2 つのサイト間で API の使用率を制限することで制限をかけることが可能です。

Due to the different characteristics between classes of sources, these cooldowns should have independent limits on the number of reports of each type.

ソースのクラスによって特性が異なるため、これらのクールダウンは、各タイプのレポート数に独立した制限を設ける必要があります。

The number of reporting windows is another vector which can contain trigger-side information. By restricting to a [single window](#single-reporting-window), a `reportingorigin` does not receive any additional information on when in the attribution window a source was triggered.

レポートウィンドウの数は、トリガー側の情報を含むことができる別のベクトルです。[シングルウィンドウ](#single-reporting-window)に制限することで、`reportingorigin`はアトリビューションウィンドウ内のいつソースがトリガーされたかという追加情報を受け取ることがありません。


### Browsing history reconstruction

Reporting attribution without a pre-existing navigation allows the `reportingorigin` to learn whether a given user on the source site visited the `destination` site at all. For click-through reports, this is not an issue because the `reportingorigin` knows a priori the user was navigating to `destination` .

既存のナビゲーションを使用せずにアトリビューションをレポートすると、`reportingorigin`はソースサイトの特定のユーザーが`destination`サイトを訪れたかどうかをまったく知ることができません。クリックスルー・レポートでは、`reportingorigin`はユーザーが`destination`に移動したことを事前に知っているので、これは問題ではありません。

This new threat is be mitigated in a number of ways:

この新たな脅威は、様々な方法で軽減することができます。


#### Adding noise to whether a trigger is genuine

By adding noise to whether an attribute source gets [triggered](#data-limits-and-noise), a reporting origin will not know with absolute certainty whether a particular ad view led to a site visit. See [Differential privacy](#differential-privacy).

属性ソースが[トリガー](#data-limits-and-noise)を取得するかどうかにノイズを加えることで、報告元は特定の広告ビューがサイト訪問につながったかどうかを絶対確実には知ることができなくなる。[差分プライバシー](#differential-privacy)を参照してください。


#### Limiting the number of unique destinations covered by pending sources

To limit the breadth of `destination` sites that a reporting origin may be trying to measure user visits on, the browser can limit the number `destination` eTLD+1s represented by pending sources for a source-site.

レポート作成元がユーザーの訪問を測定しようとする「宛先」サイトの幅を制限するために、ブラウザはソースサイトの保留中のソースによって表される「宛先」eTLD+1 の数を制限することができます。

The browser can place a limit on the number of a source site's pending source's unique `destination` s. When an attribution source is registered for an eTLD+1 that is not already in the pending sources and a source site is at its limit, the browser will drop the new source.

ブラウザは、ソースサイトの保留中のソースのユニークな `destination` の数に制限を設けることができます。保留中のソースにまだない eTLD+1 に対してアトリビューションソースが登録され、ソースサイトがその制限に達した場合、ブラウザは新しいソースを取り下げます。

The lower this value, the harder it is for a reporting origin to use the API to try and measure user browsing activity not associated with ads being shown. Browsers may choose parameters on the number of `destination` s to make their own tradeoffs for privacy and utility.

この値が低ければ低いほど、広告が表示されていないユーザーのブラウジング活動を測定するために、レポート作成者が API を使用することが難しくなります。ブラウザは、プライバシーと実用性のトレードオフを行うために、`destination`の数に関するパラメータを選択することができます。

Because this limit is per source site, it is possible for different reporting origin on a site to push the other attribution sources out of the browser. See the [denial of service](#denial-of-service) for more details. To prevent this attack, the browser should maintain these limits per reporting origin. This effectively limits the number of unique sites covered by pending sources from any one reporting origin.

この制限はソースサイトごとのため、サイト内の異なる報告元が他のアトリビューションソースをブラウザから追い出すことが可能です。詳しくは、[サービス拒否](#denial-of-service)を参照してください。この攻撃を防ぐために、ブラウザは報告元ごとにこの制限を維持する必要があります。これは、一つの報告元からの保留中のソースによってカバーされるユニークなサイトの数を効果的に制限します。

Strawman: 100 distinct destination sites per-{source site, reporting origin}, applied to all pending sources regardless of type.

ストローマン:{ソースサイト、報告元}ごとに 100 の異なるデスティネーションサイト、タイプに関係なくすべての保留中のソースに適用されます。

TODO: should this be applied over a time period?

TODO: これは一定期間適用されるべきか?


### Differential privacy

A goal of this work is to create a framework that would support making event-level measurement that satisfies local differential privacy. This follows from our use of k-randomized response to generate noisy output for each source event. For a given output space O with cardinality k, true value v in the output space, and flip probability p, the k-randomized response algorithm:

この研究の目的は、局所的な差分プライバシーを満たすイベントレベルの計測をサポートするフレームワークを作成することである。これは、各ソースイベントに対してノイズの多い出力を生成するために k-ランダム応答を用いることから導かれる。k-ランダム応答アルゴリズムは、出力空間 O と出力空間における真値 v、そしてフリップ確率 p が与えられた場合、以下のようになる。

- Flips a biased coin with heads probability p
- If heads, return a random value in O
- Otherwise return v

k-randomized response is an algorithm that is epsilon differentially private if `p = k / (k + exp(epsilon) - 1)` . For low enough values of epsilon, a given source's true output should be well protected by the randomized response mechanism.

k-ランダム化応答は、`p = k / (k + exp(epsilon) - 1)` のとき、ε-差分プライベートであるアルゴリズムである。epsilon の値が十分に小さい場合、与えられたソースの真の出力はランダム化応答のメカニズムによって十分に保護されるはずである。

Note that the number of all possible output states k in the above design is:

なお、上記の設計で可能なすべての出力状態の数 k は

- 2925 for click sources. This results from a particular "stars and bars" counting method which derives `k = (num_reporting_windows * num_trigger_data + num_reports choose num_reports) = (3 * 8 + 3 choose 3)` . TODO: outline why this is the case.
- 3 for event-sources (no attribution, attribution with trigger data 1, attribution with trigger data 0). This also follows from `(1 * 2 + 1 choose 1)` .

Note that the scope of privacy in the current design is not user-level, but rather source-level. This follows from the fact that noise is added independently per source-event, and source events are not strictly rate-limited. Exact noise parameters are subject to change with feedback. The primary goal with this proposal as written is to ensure that the browser has the appropriate infrastructure foundation to develop locally differentially private methods in the future. Tightening the privacy scope will also be considered in future work.

現在の設計におけるプライバシーの範囲は、ユーザーレベルではなく、ソースレベルであることに注意してください。これは、ノイズはソースイベントごとに独立して追加され、ソースイベントは厳密にレート制限されないという事実に起因する。ノイズの正確なパラメータは、フィードバックによって変更される可能性がある。この提案の主な目標は、将来的にブラウザがローカルに差分化されたプライベートメソッドを開発するための適切なインフラ基盤を持つことを確実にすることです。プライバシーの範囲を狭めることも今後の課題として検討します。

Our plan is to adjust the level of noise added based on feedback during the origin trial period, and our goal with this initial version is to create a foundation for further exploration into formally private methods for measurement.

今後、原点回帰の試行期間中のフィードバックにより、ノイズの付加レベルを調整していく予定ですが、この初期バージョンでは、さらに正式な非公開測定方法を検討するための土台を作ることを目的としています。


### Speculative: Limits based on first party storage

Another mitigation on joining identity across publisher and advertiser sites is to limit the number of reports for any given publisher, advertiser pair until the advertiser clears their site data. This could occur via the [Clear-Site-Data](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Clear-Site-Data) header or by explicit user action.

パブリッシャーと広告主のサイト間で ID を結合する際のもう 1 つの緩和策は、広告主がサイトデータをクリアするまで、任意のパブリッシャーと広告主のペアに対するレポート数を制限することです。これは、[Clear-Site-Data](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Clear-Site-Data)ヘッダーまたは明示的なユーザーアクションによって行われる可能性があります。

To prevent linking across deletions, we might need to introduce new options to the Clear-Site-Data header to only clear data after the page has unloaded.

削除をまたいだリンクを防ぐために、Clear-Site-Data ヘッダーに新しいオプションを導入して、ページがアンロードされた後にのみデータをクリアする必要があるかもしれません。


## Security considerations

### Reporting origin control

Reports can only be generated if the same origin responds with headers that register source events and trigger events. There is no way for an origin to register events on behalf of another origin, which is an important restriction to prevent fraudulent reports.


### Denial of service

Rate limits and other restrictions to the API can cause reports to no longer be sent in some cases. It is important to consider all the cases where an origin could utilize the API in some way to lock out other origins, and minimize that risk if possible.

Currently, the only known limit in this proposal that could risk denial of service is the [reporting origin limits](#reporting-origin-limits). This is an explicit trade-off for privacy that should be monitored for abuse.


### Top site permission model

The Permissions Policy is used to globally enable or disable the API. Additionally, network requests need fine grained permission in the form of requiring an `attributionsrc` attribute.