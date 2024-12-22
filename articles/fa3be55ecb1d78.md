---
title: "Datadog×Sentryで実現するエラートラッキング"
emoji: "📕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [datadog, sentry, observability]
published: false
---

## はじめに

最近、ソフトウェアやシステムの複雑化が進む中で、エラーの一元管理はシステムの健全な運用において必要不可欠です。
この記事では、クライアントおよびサーバーサイドのエラーを一元的に管理し、効果的にエラートラッキングを行うための設計や便利なツール群を紹介します。

## エラートラッキングの重要性

現代のシステム開発では、バックエンドとフロントエンドの両方でエラートラッキングが不可欠です。これにより、複雑な分散システムや多様なユーザーインターフェースを効果的に管理できます。
マルチクラウド環境や分散システムが一般化し、Web、モバイルアプリ、PCアプリケーションなど多様なユーザーインターフェースでのやり取りが増えています。
そのような中、以下の理由からバックエンドとフロントエンドの両方でエラートラッキングが不可欠です。

- エラー発生時に迅速に適切な対応チームへ通知
- ユーザーの報告を待たずに問題を検知し修正
- クライアント・サーバー双方で統一的なエラー監視を行い、運用コストを最適化

## クライアントにおけるエラートラッキングツールについて

クライアントにおけるエラートラッキングは、主にユーザーの端末、OS、操作といったユーザーごとのデータと、その環境で発生したエラーを関連付けることができます。これにより、障害の早期検知や機能改善に役立てることが可能です。

Unityでは、モバイル版（Android/iOS）とデスクトップ版（Windows/Mac/Linux）でビルドが異なるため、単に「Unity対応」とされるSDKであっても、モバイル版のUnityでは利用できる一方で、デスクトップ版のUnityでは利用できない場合があります。特に、プラットフォームごとに異なる特性を考慮する必要があります。
Webフロントエンドでは、各種フレームワークやサーバーサイドのスクリプトの実行、デプロイ先のランタイムに依存する場合があるため、各種フレームワークに対応したSDKが開発されていることなどを考慮する必要があります。

各ツールごとにトレース、セッションリプレイ、エラートラッキングなどカバーされている範囲は似ているため、現状使用しているツールや組織の規模感に応じて選定してください。

### Backtrace

**Backtrace** は、Unityにおける代表的なエラートラッキングツールの一つです。  
https://backtrace.io/

- モバイル版（Android/iOS）とデスクトップ版（Windows/Mac/Linux）の両方のUnityで利用可能
- 自動エラーレポートやグルーピング機能により、きめ細やかなエラー管理が可能
- **ユーザーあたりのライセンス課金**モデルのため、大規模なアプリケーションではコストが高くなる場合があります。

https://backtrace.io/pricing

利用事例については、以下をご参照ください。  
https://unity3d.jp/game/dena-backtrace/

### Datadog/New Relic

サーバーサイドでは定番の **Datadog** や **New Relic** ですが、どちらもモバイル版（Android/iOS）のUnityには対応している一方、デスクトップ版（Windows/Mac/Linux）のUnityには対応していません。

どちらもモバイル版のUnityであればエージェントを導入し、メトリクスやトレースの収集が可能です。  
また、Webに関してもSDKが対応しているため、トレース、セッションリプレイ、エラートラッキングなどの機能を一通り扱うことができます。

- **New Relic**

  モバイル版Unity向けのSDKを提供しており、リアルタイムでアプリのパフォーマンスを監視可能です。

  https://docs.newrelic.com/jp/docs/mobile-monitoring/new-relic-mobile-unity/monitor-your-unity-application/  
  https://qiita.com/ume67026265/items/72d5068edd9a77baef2e

  ブラウザであれば、ブラウザモニタリングを使用して、エラートラッキングやCore Web Vitalsの指標を収集することが可能です。  
  https://docs.newrelic.com/jp/docs/browser/browser-monitoring/getting-started/introduction-browser-monitoring/  
  https://qiita.com/MarthaS/items/c136a4f6d58845a7fa1e

- **Datadog**

  モバイル版Unity向けのRUM（Real User Monitoring）機能を提供しています。ただし、デスクトップ版やコンソール、Web向けのUnityビルドには対応していません。  
  https://docs.datadoghq.com/ja/real_user_monitoring/mobile_and_tv_monitoring/setup/unity/

  > Datadog does not support Desktop (Windows, Mac, or Linux), console, or web deployments from Unity. If you have a game or application and want to use Datadog RUM to monitor its performance, create a ticket with [Datadog support](https://docs.datadoghq.com/ja/help/).

  ブラウザであれば、RUMを使用することでエラートラッキングやCore Web Vitalsの指標を収集することが可能です。  
  https://docs.datadoghq.com/ja/real_user_monitoring/browser/monitoring_page_performance/

### Embrace

Embrace は、モバイルクライアントのUnity向けにテレメトリデータを収集できるツールです。  
OpenTelemetryをベースとしており、OTLPを扱える任意のバックエンドにテレメトリーデータを送信できます。

詳しくは以下の記事をご覧ください。  
https://qiita.com/ys-cover/items/687512e4cccc50a6781b

### Sentry

一般的にエラー監視で非常によく使われているツールですが、ブラウザはもちろん、Unityの文脈ではモバイルおよびデスクトップのどちらのUnity SDKでもエラートラッキングに対応しています。

https://docs.sentry.io/platforms/unity/native-support/

また、WebGLにも対応しているため、Webアプリ上でUnityのWebGLビルドを読み込んだ際のエラートラッキングも可能です。

> Supports C# errors on multiple platforms, including: Android, iOS, macOS, Linux, Windows, and WebGL  
> https://docs.sentry.io/platforms/unity/

#### Webフロントエンドにおけるエラートラッキング

近年、Webフロントエンド開発では、Next.js（App Router）やRemix（No SPAモード）など、サーバーサイドレンダリング（SSR）をサポートするフレームワークが主流となっています。これに伴い、エラートラッキングにおいても、クライアントサイドのJavaScriptエラー（Error Boundary）とサーバーサイドのJavaScriptエラーを統合的に扱えるツールが求められています。
例えば、Remixであれば`loader`や`action`のサーバーサイドで実行される機能の例外もキャッチしたいところです。SentryであればRemixのプラグインがサポートされています。
https://docs.sentry.io/platforms/javascript/guides/remix/manual-setup/

しかし、Cloudflareなどのエッジコンピューティングサービスを利用する場合、環境によってはツール自体の導入が難しくなる場合があります。
例えば、RemixのSSRをCloudflare Workers（Pages Functions）上で実行しようとすると、Sentryのインスタンスを正しく初期化できず、エラーをSentryに送信できない問題が報告されていました。
https://github.com/getsentry/sentry-javascript/issues/5610

最近では、SentryのCloudflare Workers SDKのサポートが強化され、Cloudflare Pagesの場合、ミドルウェアに処理を挟むだけで、Remixのクライアントおよびサーバーのエラーを統合的に扱えるようになりました。 
https://docs.sentry.io/platforms/javascript/guides/cloudflare/
https://github.com/getsentry/sentry-javascript/issues/12620

具体的な実装例として、以下のスライドが参考になります。
https://speakerdeck.com/nkzn/remix-pages-sentry

以上のような理由から、エラートラッキングを行うために使用できるツールにはさまざまな選択肢がありますが、昨今のユーザーインターフェースの多様化（Web/Mobile/Desktop）に対応しつつ、技術的な課題（Server Side JS/Unity WebGL）を克服するために、エラートラッキングではSentryの利用を個人的にはおすすめします。  
SentryはTeam Plan以上であればユーザー招待も無制限なため、大規模な組織でも扱いやすいです。（もちろん各プランの無料枠を超えると追加料金が発生するので、導入する際には現状のエラー数やカバーする範囲から料金を見積もりましょう。）  
https://sentry.io/pricing/

## サーバーサイドにおけるエラートラッキングツールについて

ここまででクライアントにおけるエラートラッキングを考察してきましたが、サーバーサイド側のエラートラッキングはどうでしょうか。

まず、サーバーサイドにおけるオブザーバビリティでは、Datadog、New Relic、Grafanaなどの監視ツールを利用することが一般的です。特に近年では、OTLP（OpenTelemetry Protocol）ベースのテレメトリデータに対応するベンダーが増えており、OpenTelemetryを活用した計測により、比較的容易に分散トレーシングやメトリクスの収集を実現できるようになっています。これにより、ベンダーロックインの懸念を軽減しつつ、高度な観測性を得ることが可能です。

これら収集したテレメトリデータからエラーを検知することで、バックエンドはエラートラッキングを実現できます。

### Q.Sentryはテレメトリデータの収集に使えないのか

利用することは可能ですが、主にサーバーレスや小規模なアプリケーションに限定されるケースが多いと考えられます。  
さらに、Sentryのトレーシング機能は現時点ではまだベータ版であるため、対応状況としては今後の改善に期待する段階と言えます。  
https://sentry.io/for/tracing/

Sentryはエージェント型でのテレメトリデータ収集方法を公式にはサポートしていません。特にKubernetesを使用している場合、インフラ属性のタグ付けやPod単位でのリソース使用量、細かなメトリクスを収集・可視化する際に困難が伴います。
ただし、Sentryもベータ版ながらOpenTelemetry（OTel）のサポートを開始しており、自前でOTel Collectorを設置すればこの制約を回避することは可能です。

一方、Datadogを利用する場合は、Datadog Operatorを活用することで、各ノードにエージェントのPodを作成できます。これにより、ホストメトリクスの収集が容易になり、Datadog監視バックエンドとのシームレスな統合も可能です。
https://docs.datadoghq.com/ja/getting_started/containers/datadog_operator/

### Datadogにおけるエラートラッキングについて

Datadogは、バックエンドサービスのエラートラッキングにおいて、主にAPMベースとログベースの2つのアプローチを提供しています。

[**Error Tracking for Backend Services**](https://docs.datadoghq.com/tracing/error_tracking/)
[**Error Tracking for Logs**](https://docs.datadoghq.com/logs/error_tracking/)

ここでは、リクエスト単位でコンテキストが付与可能かつ、トレースやパフォーマンスメトリクスと連携しコードレベルで問題を特定可能なAPMベースでのエラートラッキングを調査します。

Goなどのアプリケーションであれば、トレーシング用のライブラリを使用し、スパンに対してエラーを記録するだけで、各APIとAPI内部でのエラー箇所を紐づけることが可能になります。

ここでは、Goを使用して計装を行う例を示します。

Datadog AgentはOTLPベースのテレメトリデータの収集をサポートしているため、`ddtrace/otel`のライブラリを使用するだけで、ビジネスロジックの計装に関してはOpenTelemetryベースのものを使用することが可能です。

https://github.com/DataDog/dd-trace-go/tree/main/ddtrace/opentelemetry

```go
import (
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/propagation"
	oteltrace "go.opentelemetry.io/otel/trace"
	ddotel "gopkg.in/DataDog/dd-trace-go.v1/ddtrace/opentelemetry"
)

func setup(){
    // Providerの定義
    traceProvider := ddotel.NewTracerProvider(traceProviderOptions...)
    otel.SetTracerProvider(traceProvider)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(propagation.TraceContext{}, propagation.Baggage{}))
}

// ビジネスロジックの計装
func fugaService(ctx context.Context) {
	_, span := tracer.Start(ctx, "fugaService")
	defer span.End()
    // ...
}
```

また、エラー時にスパンにエラーを記録するようにしましょう。
https://opentelemetry.io/docs/languages/go/instrumentation/#record-errors
https://opentelemetry.io/blog/2024/otel-errors/

```go
import (
	// ...
	"go.opentelemetry.io/otel/codes"
	// ...
)

// ...

result, err := operationThatCouldFail()
if err != nil {
	span.SetStatus(codes.Error, "operationThatCouldFail failed")
	span.RecordError(err)
}
```

上記のような可視化を行うことで、エラー時に特定のスパンのエラーとトレースとしてのエラーが可視化され、このエラー自体がDatadogのエラートラッキングモニターとして使用可能になります。

Trace  
![alt text](/images/dd-sentry/datadog-apm.png)

Error Tracking  
![alt text](/images/dd-sentry/error-tracking.png)

## クライアントサイド・サーバーサイドを統合したエラートラッキングについて

ここまででクライアントおよびサーバーサイドのエラートラッキングをそれぞれ行うことができました。しかし、実際にはエラーを収集するだけではなく、一元管理しつつ、効果的にアラートを行える仕組みを作る必要があります。

ここでは、SentryとDatadogを組み合わせて効率的にエラートラッキングのアラート通知の構成を実現する方法を考えます。

最終的なイメージは以下の通りです。  
![alt text](/images/dd-sentry/dd-client-server.png)

1. API側はDatadog Agentを使用しOTLPでテレメトリデータを収集し、Error Tracking Monitorで通知
2. クライアント側はSentryでエラーを収集し、Datadog Integrationを使用してDatadogにイベントとして転送、Event Monitorで通知

上記それぞれで収集したデータソースごとのモニターを作成し、指定の通知先（Slackなど）に流せるようにします。
https://docs.datadoghq.com/ja/monitors/types/

1に関してはError Tracking用のモニターを使用します。  
https://docs.datadoghq.com/ja/monitors/types/error_tracking/

Error Tracking用のモニターは、アラートの条件で`High Impact`と`New Issue`を指定可能です。  
それぞれ、影響を受けたユーザー数や今まで起きたことのないエラー（スタックトレース等に基づいた）を元に通知することができます。そのため、単純にエラーが発生したらすべて通知するのではなく、未知のエラーや特定タイミングにおけるエラーを検知しやすくすることができます。（フィルタリングのルールもカスタムで定義可能です）

![alt text](/images/dd-sentry/threshold.png)

2では、DatadogのIntegrationとイベントモニターを使用します。  
https://docs.datadoghq.com/ja/service_management/events/

イベントモニターでは、前述のError Trackingモニターのようにエラー内容に基づいてDatadog側でエラーをグルーピングしてくれません。  
そこで、あらかじめSentry側で通知するエラーをフィルタリングし、通知が必要なものだけをイベントとしてDatadogに流すことでエラー通知を実現します。

Sentryにもスタックトレース等でグルーピングを行い、新しいエラーのみをアラートしたり、頻度やユーザーへの影響度によってアラートをカスタマイズ可能です。  
https://docs.sentry.io/product/issues/grouping-and-fingerprints/  
https://docs.sentry.io/product/issues/

Sentry側でIssue Alertを作成したら、IntegrationでDatadogのWebhookエンドポイントを指定することで、条件にマッチしたIssueが検知された時だけDatadogにイベントとして転送されます。
https://docs.datadoghq.com/integrations/sentry/  
![alt text](/images/dd-sentry/sentry-issue.png)

正しく送信ができると`source:sentry`のイベントを取得できるので、イベントモニターでアラートを送信できます。  
![alt text](/images/dd-sentry/sentry-error.png)

### エラーを扱いやすくするための工夫

ここまででDatadogにエラーを集約させ、モニターからクライアント・サーバーのエラーを通知することができました。  
このモニターを扱う上で、モニター数が増えたり、サービスチームが増えるとアラートの条件が複雑になりがちで、適切なハンドリングが難しくなります。  
そんな時に少し役立つかもしれないTipsを紹介します。

#### Datadog Teamの利用

Datadogにはチーム機能があります。  
これはUI上でチームごとにダッシュボードやモニターをまとめることができたり、クエリに対して使用可能なので、チーム単位でアラート先を振り分けたい場合に非常に使いやすいです。  
https://docs.datadoghq.com/ja/account_management/teams/

#### Sentry側でDatadogのタグを付与する

Integrationを使用してDatadogイベントとして取り込むと、デフォルトではDatadog側の予約済みタグ（例えば`env`や`service`）が付与されていません。  
https://docs.datadoghq.com/ja/getting_started/tagging/

そのため、Sentry SDK側でタグを付与する必要があります。  
例えば、JavaScriptであればSentry用のインスタンスに対して`beforeSend`メソッドがあり、この中でSentry送信前にイベントのタグを編集することが可能です（エラーのフィルタリング用途にも使えます）。  
https://docs.sentry.io/platforms/javascript/configuration/filtering/

```ts
Sentry.init({
  dsn: 'xxx',
  beforeSend(event) {
    event.tags = {
      ...event.tags,
      env: "dev",
      team: "dd-team",
      service: "dd-service",
    };
    return event;
  }
});
```

#### ダウンタイムの利用

ダウンタイムを使用して、深夜帯などの開発環境のモニターを停止します。  
https://docs.datadoghq.com/ja/monitors/downtimes

コスト削減などでインスタンス等の設定値を変更している場合、設定の変更に伴ってエラーが発生しアラートが飛んでしまうことを防ぐため、指定期間中は通知を行わないようにします。

その際に、モニタータグで大まかにチームを指定し、ダウンタイムスコープを利用してサービスや環境を指定してモニターを管理することで、アラート実行時に動的に必要なサービスや環境のグループで一致するアラートのみを無視することが可能になります（ダウンタイムのグループスコープは、モニター固有の対象の後にマッチするため）。  
https://docs.datadoghq.com/ja/monitors/downtimes/?tab=%E3%83%A2%E3%83%8B%E3%82%BF%E3%83%BC%E3%82%BF%E3%82%B0%E3%81%A7%E6%8C%87%E5%AE%9A#%E3%83%80%E3%82%A6%E3%83%B3%E3%82%BF%E3%82%A4%E3%83%A0%E3%82%B9%E3%82%B3%E3%83%BC%E3%83%97  
![alt text](/images/dd-sentry/downtime.png)
