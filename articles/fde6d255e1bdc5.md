---
title: "Webサービス(+Discord Bot)のCloudflare移行記録(個人開発)"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

こんにちは、[sugar-cat](https://twitter.com/sugar235711)です。

この記事はとある個人開発のWeb/DiscordサービスをVercelからCloudflareへ移行した際の記録です。

:::message 
個人の趣味の範囲での検証のため、業務での利用を前提とした内容ではありません。
:::

個人開発の内容については下記のスライドもご参考ください。

https://speakerdeck.com/sugarcat7/cloudflare-workflows?slide=21
https://speakerdeck.com/sugarcat7/zui-jin-ge-ren-kai-fa-kare-i-duo-yan-yu-dui-ying-bian
https://speakerdeck.com/sugarcat7/zui-jin-ge-ren-kai-fa-gare-i-monitaringuqiang-hua-bian-v0-dot-1-0


## はじめに

### 開発したWebサービスについて

**サービスの内容**
配信者（Vtuber）の配信スケジュールを一覧で確認できるWebサイトと、Discord Botを組み合わせたキュレーションサービスです。

**利用状況**
- Webサイト：月間PV 20万〜30万程度
- Discord Bot：370サーバーで稼働中（2024年6月1日現在）

**システムの特徴**
- リアルタイムでのコンテンツ更新（約1分間隔）
- YouTube/Twitch/ツイキャスの外部APIを大量利用
- Discordの各サーバーに1分ごとに配信情報を通知

### なぜ移行することになったのか

最初は2022年ごろにVercel + Firestoreの構成で構築し、その後機能を継ぎ足しながら運用していました。

![移行前の構成](/images/vercel-to-cf/vercel.png)

しかし利用者や機能が増えるにつれて、コストと運用負荷が増大していました。

### 金銭的なコストの問題

具体的な額は伏せますが主にVercelとFirestoreのコストが増大していました。

**Vercelのコスト**
Pro Planの基本料金に加えて、以下の従量課金が発生していました。
- **Function Duration**：大量のCron実行による関数実行時間
- **Fast Origin Transfer**：海外からのアクセスによるデータ転送
- **Log Drain**：DatadogへのLog送信
- **ISR Writes**：Incremental Static Regenerationの書き込み
- **Web Analytics**：アクセス解析

https://vercel.com/pricing
https://zenn.dev/reiwatravel/articles/796bc3ad8be2fb

**Firestoreのコスト**
適切なチューニングを行わずに使用していたため、読み取り（Read）操作のコストが高くなっていました。

https://firebase.google.com/docs/firestore/pricing?hl=ja#no_free_quota_for_named_databases

### システム運用の負荷

**実行時間制限への対応**
当時Vercelには1つのCronあたり60秒の実行時間制限がありました。そのため、重い処理は一部をGoのServerless Functionで実装してパフォーマンスを向上させていました。
しかし、この対応により同じロジックが複数の言語で重複実装され、新機能追加時の工数が増大していました。

**外部サービスとの連携**
Vercel自体は優秀な実行環境でしたが、ちょっとしたキューイングやKV Storeが必要になると、外部プロバイダーとの組み合わせや[Vercel Storage](https://vercel.com/blog/vercel-storage)の利用が必要でした。
Vercel Storageは便利でしたが、当時は対応リージョンが限定的で、コールドスタートの問題もあったため採用を見送っていました。

**設計上の技術的負債**
初期の拡張性に欠ける設計により、継ぎ足しでの機能開発が続いていました。

特に[多言語対応](https://speakerdeck.com/sugarcat7/zui-jin-ge-ren-kai-fa-kare-i-duo-yan-yu-dui-ying-bian)では、一時的にWorkersをProxyとして挟み、Edgeで強引に翻訳処理を行うなどをしたりしてカオス状態になっていました。

![多言語対応の構成](/images/vercel-to-cf/translate.png)

参考
https://vercel.com/blog/vercel-storage

### Cloudflareへの移行を決めた理由

当初は以下のスライドで紹介されているような、Cloudflare Stackを前提とした非同期ジョブシステムの構築を検討していましたが、スライドにも記載がある通りStatus管理やフロー制御が複雑で実装が困難でした。

https://speakerdeck.com/aiji42/cloudflare-workersdegou-zhu-surufei-tong-qi-ziyobusisutemu


そんな中、2024年10月頃に**Cloudflare Workflows**がPreviewで公開されました。
これにより非同期ジョブシステムが格段に構築しやすくなったため、Cloudflareを中心としたシステムへの移行を決断しました。

https://blog.cloudflare.com/building-workflows-durable-execution-on-workers/
https://speakerdeck.com/sugarcat7/cloudflare-workflows

## 移行後のシステム構成

移行後は、Cloudflare Workersを以下の4つの役割に分離し、Workers間の通信は全てService Bindingsで接続しています（データベースのみ外部のDBaaSを利用）。
- **Cron Workers**：定期実行処理（WorkflowsをBindings）
- **API Workers**：外部連携が必要なAPI Gateway
- **Internal Workers**：ドメインロジック、DB・外部Provider連携（Hyperdrive/KV/QueueをBindings）
- **Webサイト用 Workers**：OpenNextによるホスティング

![移行後の構成](/images/vercel-to-cf/cf.png)



## 移行の実践：段階的なアプローチ

コードベースは全て1から作り直し、以下の段階で移行を進めました。

### ステップ1：非同期ジョブ部分の移行
既存システムを稼働させたまま、InternalロジックとDiscord Bot部分を並行稼働

### ステップ2：API Gatewayの構築
Web側で使用しているリクエストをCloudflare側に段階的に移行

### ステップ3：Webサイトの移行
OpenNextでWebサイトをCloudflareに移行し、Service BindingsでInternal Workersと接続

以下、各部分の具体的な設計と実装について簡単に説明します。

## サーバーサイドの設計と実装

### Internal Workers

各WorkersからService Bindingsで参照されるエントリーポイントとして、Internal Workersを設計しました。

**役割**
- 外部との接続処理
- データベースや外部API連携
- データの永続化

**設計方針**
各ドメインの`Usecase`を`RpcTarget`と1対1で対応させて分離しています。

https://developers.cloudflare.com/workers/runtime-apis/rpc/#class-instances

:::details 実装例
```ts:entrypoint.ts
import { RpcTarget, WorkerEntrypoint } from "cloudflare:workers";
// ...
export class HogeRPC extends RpcTarget {
  #usecase: IHogeInteractor;
  constructor(usecase: IHogeInteractor) {
    super();
    this.#usecase = usecase;
  }

  async list(params: ListHogesQuery) {
    return withTracerResult("HogeRPC", "list", async () => {
      return this.#usecase.list(params);
    });
  }
}

// 各WorkersからService Bindingsで参照されるEntry Point
export class AppWorker extends WorkerEntrypoint<AppWorkerEnv> {
  newHogeRPC() {
    const d = this.setup();
    return new HogeRPC(d.freechatInteractor);
  }

  private setup() {
    const e = zAppWorkerEnv.safeParse(this.env);
    if (!e.success) {
      throw new Error(e.error.message);
    }
    return new Container(e.data);
  }
}
```

利用側では環境変数経由で簡単にアクセスできます。
```ts:usage.ts
const hoge = await env.APP_WORKER.newHogeRPC();
const rt = await vu.list()
if (result.err) {
 // エラーハンドリング
}
// 処理続行
```
:::

### データベース接続：NeonとHyperdriveの活用

データベースにはNeonを利用し、Cloudflare WorkersからはHyperdriveを介して接続しています。
https://neon.com/
https://developers.cloudflare.com/hyperdrive/

**キャッシュ戦略**
HyperDriveのPollingおよびQuery Cachingは使用せず、KVを利用したカスタムキャッシュを実装しています。

移行当時、DatabaseをAsiaリージョンに配置した際にHyperDriveを経由したリクエストが遅くなる問題がありました。

https://discord.com/channels/595317990191398933/1150557986239021106/1304643699107434517
https://zenn.dev/okku000/articles/cb1f3d1a35bf2d

現在は解消されているようですが、念のため読み取りクエリはKVでキャッシュして様子を見ることにしています。

### Cloudflareサービスとの連携

#### AI Gateway
翻訳の自動化と一部AI AgentでAI Gatewayを利用しています。

現在AI GatewayにSemantic Cacheは未実装のため、入力の完全一致キャッシュのみを活用しています。(FastlyのAI AcceleratorのようなSemantic Cacheの利用を検討しています)

https://developers.cloudflare.com/ai-gateway/configuration/caching/

#### Queue
非同期ジョブで大量の書き込み処理を1度に行うため、Queueでバッファリングする構成にしています。

**制限への対応**
1回あたりのbatchサイズと数に制限があるため、アプリ側でバッチサイズとメッセージ数を計測し、超過時は分割送信しています。

>Maximum consumer batch size: 100 messages  
>Maximum messages per sendBatch call: 100 (or 256KB in total)

https://developers.cloudflare.com/queues/platform/limits/

#### KV
基本的な使い方に加えて、以下の機能を積極活用しています：
- シリアライズ高速化のtype指定
- bulk read機能

https://developers.cloudflare.com/kv/api/read-key-value-pairs/

#### Logs
アプリケーションコードにPerformance TimerとLogを組み込み、ダッシュボードで各Workerのパフォーマンスを監視しています。

エラートラッキングとTracingはSentryと併用しています。

https://developers.cloudflare.com/workers/runtime-apis/performance/

![開発環境のダッシュボード例](/images/vercel-to-cf/metrics.png)

## Discord Botの実装

WebSocket常駐型ではなく、Interaction Endpoint + 非同期にEmbedのテキストを配信する形で実装しています。

### ライブラリの選択
Cloudflareでは標準的なdiscord.jsが動作しないため、以下を組み合わせています：
- **Client部分**：DiscordHono
- **内部ロジック**：Discorddeno

https://speakerdeck.com/sugarcat7/discord-cloudflare

### レート制限への対応
Discord APIに下記制限がありますが、適切なハンドリングができていないのでリクエスト数の最適化や不正検知のシステムは鋭意実装中です。
- **リクエスト制限**：50 req/sec
- **無効アクセス制限**：10分あたり10,000件（ステータスコード401、403、429）

https://discord.com/developers/docs/topics/rate-limits#global-rate-limit


## Webフロントエンドの移行

### OpenNextを使ったNext.js移行
既存のNext.js（Pages Router）をOpenNextのCloudflare Adapterで移行しました。

https://opennext.js.org/cloudflare

ただし、単純にOpenNextを適用するだけでは動作せず、いくつかの修正が必要でした。

### 課題1：fs moduleの制約

i18n対応で使用していた`next-i18next`の`serverSideTranslations`が、内部でfsモジュールを使用してlocalesファイルを読み込んでいました。

https://github.com/i18next/next-i18next/blob/master/src/serverSideTranslations.ts#L27

そのため、Cloudflare WorkersランタイムではNode.jsのFile systemは使用できないため、Static Assetを使ってFetchする方式に変更しました。

https://developers.cloudflare.com/workers/runtime-apis/nodejs/

具体的には、`i18n`インスタンス初期化時に読み込ませる`BackendModule`をカスタム実装しました。

https://github.com/i18next/i18next/blob/master/index.d.ts#L96

:::details カスタムBackendModuleの実装例
```ts:example.ts
export class CloudflareAssetsBackend implements BackendModule {
  type = "backend" as const;
  private services: Services | undefined;
  private options: BackendOptions = {
    loadPath: "/locales/{{lng}}/{{ns}}.json",
  };

  static type = "backend";

  constructor(services?: Services, options: Partial<BackendOptions> = {}) {
    this.init(services, options);
  }

  init(services?: Services, options: Partial<BackendOptions> = {}): void {
    this.services = services;
    this.options = {
      ...this.options,
      ...options,
    };
  }

  async read(
    language: string,
    namespace: string,
    callback: ReadCallback,
  ): Promise<void> {
  // OpenNextを使ってCloudflareコンテキストを取得
    const cloudflareContextResult = await this.getCloudflareContext();
    if (cloudflareContextResult.err) {
      return Err(cloudflareContextResult.err);
    }

    const { env } = cloudflareContextResult.val;

    if (!env.ASSETS) {
      return Err(
        new AppError({
          message: "ASSETS binding not available",
          code: "INTERNAL_SERVER_ERROR",
          context: {},
        }),
      );
    }

    // localeファイルのパスを構築
    const loadPath = this.options.loadPath
      .replace("{{lng}}", language)
      .replace("{{ns}}", namespace);

    // ASSETSからlocaleファイルを取得
    const fetchResult = await wrap(
      env.ASSETS.fetch(loadPath),
      (error) =>
        new AppError({
          message: `Failed to fetch asset: ${loadPath}`,
          code: "INTERNAL_SERVER_ERROR",
          cause: error,
          context: {},
        }),
    );
    if (fetchResult.err) {
      return Err(fetchResult.err);
    }

    const response = fetchResult.val;
    if (!response.ok) {
      return Err(
        new AppError({
          message: `Failed to load ${loadPath} (${response.status})`,
          code: "INTERNAL_SERVER_ERROR",
          context: {},
        }),
      );
    }

    const jsonResult = await wrap(
      response.json(),
      (error) =>
        new AppError({
          message: "Failed to parse JSON response",
          code: "INTERNAL_SERVER_ERROR",
          cause: error,
          context: {},
        }),
    );

    if (jsonResult.err) {
      callback(jsonResult.err, false);
    } else {
      callback(null, jsonResult.val);
    }
  }
}
```
:::

### 課題2：パフォーマンスの問題
これは未解決ですが、CPU Timeが消費されTTFBが長くなる問題が発生しています。
メトリクスやIssueを見る限りではOpenNextのNextServer読み込み時に原因不明のブロッキングが発生しているようです。
https://github.com/opennextjs/opennextjs-cloudflare/issues/653

下記を試しましたが特に効果はなく依然として未解決です。
- Smart PlacementをON
- Tiered Cacheの利用
- その他の最適化施策（KVのBindingsやCache APIの利用など)


### 移行してみて
- Server側はCloudflareのStackに寄せたことで今までできていなかったQueingやWorkflowsによるフローの制御などを実装でき、かつコストは大幅に削減できました。
- 一方でWebフロントに関してはまだOpenNextが枯れていないこともあり、継続的にパフォーマンスの課題と向き合う必要があると考えています。

## まとめ

個人開発のWebサービスとDiscord BotをVercelからCloudflareに移行した記録をまとめました。
業務で使うには権限等セキュリティの観点でさまざまな課題がありますが、サーバー側に関しては個人開発や小規模なプロジェクトでは十分に実用的なレベルではあると感じました。
