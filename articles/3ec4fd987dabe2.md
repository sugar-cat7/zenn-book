---
title: "Agones Fleet Autoscaler編"
emoji: "🎮"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [agones, kubernetes, gameserver]
published: false
---

こんにちは、[@sugar235711](https://twitter.com/sugar235711)です。
この記事は「ひとりで気になるOSSのソースコード全部読んで何かする Advent Calendar 2025」5日目の記事です。
https://qiita.com/advent-calendar/2025/sugarcat

- [全体編](https://zenn.dev/king/articles/4e55f030aa01ee)
- [GameServer編](https://zenn.dev/king/articles/003f7b79409b4a)
- [Fleet編](https://zenn.dev/king/articles/e956ef8f3cd6bf)
- [GameServerAllocation編](https://zenn.dev/king/articles/a156a6243724df)
- FleetAutoscaler編 ←今ここ


## Fleet Autoscalerに関して
お待ちかねのFleet Autoscalerに関してです。
FleetAutoScalerはFleetを自動でスケーリングするためのコンポーネントです。

Fleetに関して
https://zenn.dev/king/articles/e956ef8f3cd6bf


FleetはCustom Resourceであるため通常のHPA等ではスケールができません。そのためFleetに対して様々な方法でスケールできる手段が用意されています。
- Ready Buffer Autoscaling: Replica数に応じてバッファを持たせてスケールする方法
- Counter and List Autoscaling: プレイヤー数/Room数に応じてスケールする方法
- Webhook Autoscaling: 複数の外部のシステムのシグナルに基づいてスケールする方法
- Schedule and Chain Autoscaling: 時間ベースでスケールする方法

https://github.com/googleforgames/agones/blob/3b94c58f3819456ba259195e9785d87cf3392d9b/pkg/fleetautoscalers/fleetautoscalers.go#L60-L95


### Ready Buffer Autoscaling
`BufferPolicy`ではAllocatedGameServer数に基づいてFleetのReplica数を調整します。

例えば絶対値/割合でそれぞれバッファを持たせられます。

**絶対値でのバッファ**
BufferSize: 5とした場合
- 計算式：`replicas = AllocatedReplicas + BufferSize`
- 例：8台がAllocated → 8 + 5 = 13台必要

**割合でのバッファ**
BufferSize: 30%とした場合
- 30%のReadyサーバーを維持する
- 計算式：`replicas = ceil(AllocatedReplicas × 100 / (100 - BufferPercent))`
- 例：8台がAllocated、30%のバッファを希望
  - Allocatedが70%になる必要がある
  - 8 × 100 / 70 = 11.43 → 切り上げて12台必要

https://github.com/googleforgames/agones/blob/3b94c58f3819456ba259195e9785d87cf3392d9b/pkg/fleetautoscalers/fleetautoscalers.go#L360-L397

### Counter and List Autoscaling
`CounterOrListPolicy`ではプレイヤー数やRoom数に基づいてFleetのReplica数を調整します。
Fleetごとに`FleetStatus`としてFleetに属するGameServerの集約値としてCounterやListを計算できるようになっています。
https://github.com/googleforgames/agones/blob/3b94c58f3819456ba259195e9785d87cf3392d9b/pkg/apis/agones/v1/fleet.go#L88-L112
https://github.com/googleforgames/agones/blob/3b94c58f3819456ba259195e9785d87cf3392d9b/pkg/apis/agones/v1/common.go#L57-L71


Controller側でこれらのCounterやListに基づいた計算ロジックを呼び出しています。

https://github.com/googleforgames/agones/blob/3b94c58f3819456ba259195e9785d87cf3392d9b/pkg/fleetautoscalers/controller.go#L363

- CounterPolicy
```yaml
policy:
  type: Counter
  counter:
    key: "players"
    bufferSize: 100  # 100プレイヤー分の空き容量を維持
    minCapacity: 1000
    maxCapacity: 10000
```

- ListPolicy
```yaml
policy:
  type: List
  list:
    key: "sessions"
    bufferSize: "20%"  # 20%の空きセッションスロットを維持
    minCapacity: 50
    maxCapacity: 500
```


https://github.com/googleforgames/agones/blob/3b94c58f3819456ba259195e9785d87cf3392d9b/pkg/fleetautoscalers/fleetautoscalers.go#L400-L421

### Webhook Autoscaling
FleetAutoscaler Controllerが外部または内部の`/scale`エンドポイントに対してHTTPリクエストを送信し、レスポンスに基づいてFleetのReplica数を調整します。

```json
{
  "response": {
    "uid": "abc-123",
    "scale": true,
    "replicas": 15
  }
}
```

Kubernetes内のServiceでも、外部URLでもどちらの指定でも問題なく動きます。

```yaml
  policy:
    # type of the policy - this example is Webhook
    type: Webhook
    # parameters for the webhook policy - this is a WebhookClientConfig, as per other K8s webhooks
    webhook:
      # use a service, or URL
      service:
        name: autoscaler-webhook-service
        namespace: default
        path: scale
```

https://github.com/googleforgames/agones/blob/3b94c58f3819456ba259195e9785d87cf3392d9b/pkg/fleetautoscalers/fleetautoscalers.go#L276-L358

### Schedule and Chain Autoscaling
SchedulePolicyでは時間ベースでFleetのReplica数を調整します。
Cron形式で時間を指定し、その時間にFleetのReplica数を指定した値にスケールします。
```yaml
  policy:
    # Schedule based policy for autoscaling.
    type: Schedule
    schedule:
      between:
        # The policy becomes eligible for application starting on October 31st, 2024 at 12:00 AM PST. If not set, the policy will immediately be eligible for application.
        start: "2024-10-31T00:00:00-07:00"
        # The policy is never ineligible for application. If not set, the policy will always be eligible for application (after the start time).
        end: ""
      activePeriod:
        # Use PST time for the startCron field. Defaults to UTC if not set.
        timezone: "America/Los_Angeles"
        # Start applying the policy everyday at 12:00 AM PST. If not set, the policy will always be applied in the .between window.
        # (Only eligible starting on October 31, 2024 at 12:00 AM PST).
        startCron: "0 0 * * *"
        # Apply this policy indefinitely. If not set, the duration will be defaulted to always/indefinite.
        duration: ""
```

https://github.com/googleforgames/agones/blob/3b94c58f3819456ba259195e9785d87cf3392d9b/pkg/fleetautoscalers/fleetautoscalers.go#L548-L563

Chain Autoscalingでは複数のAutoscalingポリシーを組み合わせて使用できるので、複数の条件でスケーリングが可能です。

https://github.com/googleforgames/agones/blob/3b94c58f3819456ba259195e9785d87cf3392d9b/pkg/fleetautoscalers/fleetautoscalers.go#L565-L608

### Wasm Autoscaling
最新バージョンではWasmを使用したAutoscalingもサポートされています。(v1.53開発中のため現時点では利用できません)
https://agones.dev/site/blog/2025/10/21/1.53.0-rust-counters-and-lists-sdkcontainer-startup-guarantees-and-more/

Extismを使用してマウントされたWasmファイルのバイナリをロードし、Wasm内の関数の計算結果に基づいてFleetのReplica数を調整します。


https://github.com/googleforgames/agones/blob/3b94c58f3819456ba259195e9785d87cf3392d9b/pkg/fleetautoscalers/fleetautoscalers.go#L136-L154

https://extism.org/
https://zenn.dev/k41531/articles/3e935bd04968d6

Autoscaler内で完結させたい計算処理を行う場合などに使えそうで、Webhook等の外部サービスに移譲していた処理をWasmに置き換えることでレイテンシーの削減などにもつながりそうです。

```yaml
apiVersion: autoscaling.agones.dev/v1
kind: FleetAutoscaler
metadata:
  name: webhook-fleet-autoscaler
spec:
  fleetName: simple-game-server
  policy:
    # type of the policy - this example is Webhook
    type: Wasm
    # parameters for the wasm policy
    wasm:
      # The exported function to call in the wasm module, defaults to 'scale'
      function: 'scale'
      # Config values to pass to the wasm program on startup
      config:
        buffer_size: "10"
      from:
        url:
          # use a service, or direct URL
          service:
            name: fileserver
            namespace: default
            path: /wasm/plugin.wasm
            # optionally can define a full URL if not hosted on cluster (or you just want to).
            # url: "https://my-bucket-storage.cloud/wasm/plugin.wasm"
            # caBundle:  optional, used for HTTPS paths with custom certs
      # optional hex encoded sha256 hash to match against wasm file (it's optional, but recommended)
      hash: "df7199d01a25bf34b3d650c7e6f685736b2c794e6a526d86b2e55bf074df3f36"
```

https://github.com/googleforgames/agones/blob/main/examples/wasmfleetautoscaler.yaml


## まとめ
FleetAutoscalerについて見ていきました。
まだMetricsやCore部分の実装はありますが、ブログには残しません。興味ある方はソースコード見てください。
明日からは別のOSSを書いていきます。