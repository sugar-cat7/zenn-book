---
title: "Honoを使い倒したい2024"
emoji: "❤️‍🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [hono,typescript,cloudflareworkers]
published: true
publication_name: aishift
---

# はじめに

こんにちは、AI Shift バックエンドエンジニアの[@sugar235711](https://twitter.com/sugar235711)です。
この記事では、Honoの使い方をおさらいし、API開発を通じてHonoの実際の開発で役立つTipsを紹介します。

Honoの基本的なコンセプトや網羅的な実装例については、公式ドキュメントを参照してください。
https://hono.dev/concepts/motivation


### 更新情報
:::details 2024/7/29更新
- [App/Contextオブジェクトの使い方を追記](#appcontextオブジェクトの使い方)
- [Combine Middleware](#combine-middleware)
- [Request ID Middleware](#request-id-middleware)
- [IP Restrict Middleware](#ip-restrict-middleware)
- [Cloudflare Pages Middleware](#cloudflare-pages-middleware)
- [Service Worker Adapter](#service-worker-adapter)
:::


:::message
Hono Conference 2024にて発表した内容についても追記しました！
[Honoの活用事例](#Honoの活用事例)にて、AI Shift内でHonoを使用した実際の開発事例を紹介しています。
:::
https://speakerdeck.com/sugarcat7/using-hono-in-b2b-saas-application


# 基本編

この章では、Honoの基本的な使い方を紹介します。
※Hono: v4.3系

## App/Contextオブジェクトの使い方

Honoでは、プライマリオブジェクトであるHonoインスタンスを生成し、そのインスタンスをもとにAPIのエンドポイントを定義します。
```ts:example.ts
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.text('Hono!'))
export default app
```

https://hono.dev/api/hono#app-hono

Honoにはリクエスト/レスポンスを扱いやすくする**Context**というオブジェクトがあります。上記の例で言えば、`c`がContextオブジェクトです。

https://hono.dev/api/context#context

この**Context**オブジェクトは、ヘッダーやリクエストボディ、レスポンスボディなどの操作を行うためのメソッドを提供します。さらに、環境変数やカスタムミドルウェアを設定することもできます。

```ts:example.ts
type Env = {
  Variables: {
    hoge: () => void
  }
}

const app = new Hono<Env>()

const middleware = createMiddleware<Env>(async (c, next) => {
  c.set('hoge', () => {console.log("fuga")}) // Contextにhoge関数を追加
  await next()
})

app.use(middleware)

app.get('/hello', (c) => {
    const { hoge } = c.var
    hoge() // output: fuga
  return c.text('Hello, Hono!')
})
```

上記のように、`Hono<Env>`とすることで、関数等をContext経由で型安全に別メソッドに伝播することができます。

さて、このままでも十分便利ですが、実際にAPIを作成する際は**カスタムのHonoインスタンス**（Factory）を作成して使い回すと、ファイル分割等を行った際に便利です。

- ReturnTypeを使用し、型付けがされたHonoインスタンスを返す関数を作成します。

```ts:customHono.ts
export const newApp = () => {
    const app = new Hono<HonoEnv>();
    app.use(prettyJSON());
    app.onError(handleError);
    // ....
    return app;
}

export type App = ReturnType<typeof newApp>;
```

- 作成したHonoインスタンスを使ってAPIを作成します。
```ts:routes/hoge.ts
export const hogeApi = (app: App) => {
    app.get('/hoge', (c) => c.text('hoge'))
}
```

- Honoインスタンスをシングルトンで使い回します。
```ts:entrypoint.ts
import { newApp } from './customHono';
import { hogeApi } from './routes/hoge';

const app = newApp();
hogeApi(app)
//...
```

上記のように自分で定義することも可能ですが、Honoの公式ヘルパーとしてFactoryメソッドが用意されているので、それらを使用するのも良いでしょう。
https://hono.dev/helpers/factory

また、Hono作者の[＠yusukebeさん](https://x.com/yusukebe)も型付きのAppを返す方法を紹介しています。

@[tweet](https://x.com/yusukebe/status/1780538942032650337)

Hono v4.5.0以降では`type`だけでなく`interface`もBindingやVariablesに直接使用できるようになりました。
この変更により、wrangler cliにより生成した型をそのまま使用できるなど、Cloudflareとの親和性が向上しています。


## ミドルウェアの設計

APIにおけるミドルウェアの責務は、ユーザー認証・認可、ロギング、キャッシュ等の共通処理を行うことです。
Honoでは、ビルトインのミドルウェアや、自作のカスタムミドルウェアを使用することが可能です。

```ts:example.ts
const example = (): MiddlewareHandler => {
    return async (c, next) => {
        console.log('middleware start')
        await next()
    }
}

// ---or---
import { createMiddleware } from 'hono/factory'

const example = createMiddleware(async (c, next) => {
    console.log('middleware start')
    await next()
})

// usage entrypoint.ts
const app = new Hono()
app.use(example()) // custom middleware
app.use('/posts/*', cors()) // built-in middleware
app.post('/posts', (c) => c.text('Created!', 201))
```

ミドルウェアは定義順に実行されるため、以下のような実行順序になります。
```
example() -> cors() -> post handler
```

`next`の後に処理を追加し、スタックのような処理を実現することも可能です。詳細については、公式ドキュメントを参照してください。
https://hono.dev/guides/middleware#execution-order

このミドルウェアの中では任意の処理を行うことができますが、基本的には**1リクエストのスコープに閉じた**動作を行うべきです。例えば、ユーザーID等の**リクエストごとに一意に扱いたい情報をContextに詰めて伝播させる**などです。

下記はリクエスト時に初期化を行うミドルウェアの例です。
リクエストごとに一意なIDとロガーを生成してContextに詰めています。
```ts:example.ts
export const init = (): MiddlewareHandler<HonoEnv> => {
    return async (c, next) => {
        const logger = new AppLogger({
            requestId: uuidv4(),
        });
        c.set("services", {
            logger: logger
        });
        logger.info("[Request Started]");
        await next();
    };
}
```

上記のように任意のオブジェクトをContextを通してhandlerから安全に利用することが可能になります。

### Combine Middleware
Hono v4.5.0以降では`Combine Middleware`を使用することで、複数のミドルウェアを組み合わせて扱えるようになります。

https://hono.dev/docs/middleware/builtin/combine#combine-middleware

下記を組み合わせることでミドルウェアの実行を制限できます。実行はミドルウェアの定義順です。

`some` - 複数のミドルウェアのうち一つ実行
`every` - 複数のミドルウェアを全て実行
`except` - 指定したミドルウェアを除外して実行

例えばあるエンドポイントを保護する場合に、IP制限またはBearer認証のどちらかを通過するといった処理を行うことができます。

例えばメンテナンスページ以外では、ミドルウェアを実行したり、
```ts:example.ts
app.use('*', except(['/maintenance'], middleware1))
```

IP制限に引っかかった場合は、Bearer認証を行うといった制御も簡単に記述することが可能です。
```ts:example.ts
app.use(
  '*',
  some(
    ipRestriction(getConnInfo, { allowList: ['192.168.0.2'] }),
    bearerAuth({ token })
  )
)
```


## エラーハンドリングについて

Honoでは、`onError`メソッドによりスローされたエラーをキャッチし、トップレベルでエラーハンドリングを行うことができます。

https://hono.dev/api/hono#error-handling

```ts:example.ts
import { HTTPException } from 'hono/http-exception'

// ...
app.onError((err, c) => {
  console.error(`${err}`)
  return c.text('Custom Error Message', 500)
})

app.post('/auth', async (c, next) => {
  // authentication
  if (authorized === false) {
    throw new HTTPException(401, { message: 'Custom error message' })
  }
  await next()
})
```

上記のようにHonoには`HTTPException`クラスが用意されており、認証時のエラーなどのHTTPステータスコードを指定してエラーをスローすることが推奨されています。

https://hono.dev/api/exception#exception

### エラーハンドリングの例

より実践的な例として、Honoのカスタムインスタンス内にonErrorメソッドを定義し、各エラータイプごとに処理を分けるハンドラーを定義してエラーハンドリングを行います。

```ts:example.ts
export const newApp = () => {
    const app = new Hono();
    app.onError(handleError); // エラーハンドリング
    // ....
    return app;
}

export const handleError = (err: Error, c: Context<HonoEnv>): Response => {
    const { logger } = c.get("services");
    if (err instanceof HTTPException) {
        if (err.status >= 500) {
            logger.error("HTTPException", {
                message: err.message,
                status: err.status,
                requestId: c.get("requestId"),
            });
        }
        const code = statusToCode(err.status);
        return c.json<ErrorResponse, StatusCode>(
            {
                error: {
                    code,
                    message: err.message,
                    requestId: c.get("requestId"),
                },
            },
            { status: err.status },
        );
    }

    logger.error("unhandled exception", {
        name: err.name,
        message: err.message,
        cause: err.cause,
        stack: err.stack,
        requestId: c.get("requestId"),
    });
    return c.json<ErrorResponse, StatusCode>(
        {
            error: {
                code: "INTERNAL_SERVER_ERROR",
                message: err.message ?? "something unexpected happened",
                requestId: c.get("requestId"),
            },
        },
        { status: 500 },
    );
}
```

### バリデーションエラーについて

致命的なエラー以外にも、**バリデーションエラー**などのエラーをハンドリングすることがあります。
HonoではZodやVailbotなどのライブラリを組み合わせ、**リクエストのスキーマに対するバリデーション**を簡単に実装することができます。
https://hono.dev/snippets/validator-error-handling#error-handling-in-validator

```ts:example.ts
import { z } from 'zod'
import { zValidator } from '@hono/zod-validator'

const app = new Hono()

const userSchema = z.object({
  name: z.string(),
  age: z.number(),
})

app.post(
  '/users/new',
  zValidator('json', userSchema, (result, c) => {
    if (!result.success) {
      return c.text('Invalid!', 400)
    }
  }),
  async (c) => {
    const user = c.req.valid('json')
    console.log(user.name) // string
    console.log(user.age) // number
  }
)
```

後述する**OpenAPIHonoとdefaultHooks**を組み合わせることで、zodエラーメッセージを加工する例を紹介します。

## 環境変数について

昨今ではJavaScriptを取り巻く環境が多様化しており、環境変数の読み込み方も多様化しています。
例えば、下記のような主要なランタイムでは、それぞれ異なる環境変数の読み込み方法があります。

- **Workerd**
  - wrangler.toml/.dev.vars
- **Deno**
  - Deno.env
  - .envファイル
- **Bun**
  - Bun.env
  - process.env
- **Node**
  - process.env

例えば、仮にCloudflare Workersで開発していたアプリケーションをコンテナ化して別のランタイムで動かす場合、環境変数の読み込み方法が異なるためコードを書き換える必要があります。
このような環境ごとに際を減らすため、Honoでは**ランタイムによらず環境変数を読み込む**ためのヘルパーが用意されています。

https://hono.dev/helpers/adapter#env

```ts:example.ts
import { env } from 'hono/adapter'

app.get('/env', (c) => {
  const { NAME } = env<{ NAME: string }>(c)
  return c.text(NAME)
})
```

Hono Adapterを通すことで**ランタイムによらない環境変数の読み込み**を行うことが可能となります。
さらに、下記のように**zodと組み合わせることで型安全に環境変数の値を取り出す**ことができます。

```ts:example.ts
import { env } from 'hono/adapter'
import { z } from 'zod'

const zEnv = z.object({
    HOGE_API_KEY: z.string()
});

type Env = z.infer<typeof zEnv>;

// middleware
const init = (): MiddlewareHandler<HonoEnv> => {
    return async (c, next) => {
        const honoEnv = env<Env, AppContext>(c)
        const envResult = zEnv.safeParse(honoEnv)
        if (!envResult.success) {
            console.error('Failed to parse environment variables', envResult.error)
            return
        }
        // ....
    }
}
```

## 構造化ロギングについて

一般的に**構造化ロギング**を行う際に、**RequestIDを付与したLogger**を作成することが多いです。
しかし、Node.js等の環境では**シングルスレッド**で動作するため、非同期処理を行う際にグローバルにRequestIDを保持することが難しいです。そのため、**AsyncLocalStorage**を組み合わせることでこれを回避することができます。

```ts:example.ts
const asyncLocalStorage = new AsyncLocalStorage<Map<string, any>>();

export function runWithRequestId(requestId: string, fn: () => void) {
  const store = new Map<string, any>();
  store.set('requestId', requestId);
  asyncLocalStorage.run(store, fn);
}

export function getRequestId(): string | undefined {
  const store = asyncLocalStorage.getStore();
  return store?.get('requestId');
}
```
`getRequestId()`を任意の場所で呼び出すことで、リクエストごとに一意なRequestIDを保持したLoggerを作成できます。

上記が一般的なロギングのプラクティスですが、HonoではContextを通して伝播させることでこの問題を解決することができます。
例えば、[ミドルウェアの設計](#ミドルウェアの設計)で紹介した初期化処理で**LoggerをContextに詰める**ことで、**リクエストごとのLoggerを保持する**ことができます。
```ts:example.ts
// middleware
export const init = (): MiddlewareHandler<HonoEnv> => {
    return async (c, next) => {
        const logger = new AppLogger({
            requestId: uuidv4(), // 一意なRequestIDを生成
        });
        c.set("services", {
            logger: logger
        });
        logger.info("[Request Started]");
        await next();
    };
}

// handler
app.get('/hello', (c) => {
    const { logger } = c.get("services");
    logger.info("Hello, Hono!");  // リクエストごとのLoggerを使用
    return c.text('Hello, Hono!')
})
```

ただし、実務ではクリーンアーキテクチャに則り、レイヤー分けがされている場合が多く、レイヤー間でContextを伝播させない選択をする場合は依然としてAsyncLocalStorageを使用することになると思います。

## オブザーバビリティについて

オブザーバビリティは**ログ、トレース、メトリクス**の3つの観点からシステムの内部状態を可視化し、観測性を向上させることです。
OpenTelemetry[^1]のSDKと組み合わせることでHonoでもトレースを取ることができます。ロギング同様に、**Context経由でTracerを伝播させる**ことで、リクエストごとのトレースを取ることができます。

[^1]:**OpenTelemetry**はログ、トレース、メトリクスを統一的に扱うためのライブラリです。
https://opentelemetry.io/

```ts:example.ts
import { Tracer } from "@opentelemetry/api";

type ServiceContext = {
    tracer: Tracer
    // ...
};

type HonoEnv = {
    Variables: {
        services: ServiceContext;
    };
};

app.get('/hoge', (c) => {
    const { tracer } = c.get("services");
    const result = await tracer.startActiveSpan(`business-logic`, async (span) => {
        const result = await businessLogic()
        span.setAttributes(result)
        return result
    });
    return c.json(result, 200)
})
```

より実践的な例として、Cloudflare Workersで計装を行えるOpenTelemetryライブラリを紹介します。
https://github.com/evanderkoogh/otel-cf-workers
このライブラリと**Honoを組み合わせ、Cloudflare Workers上で簡単に計装を行う**ことが可能です。下記ではランダムに複数ユーザーを登録するAPIを計装しています。(わざとBulk Insertをしていません)
```ts
export const registerUserRandomPostApi = (app: App) =>
    app.openapi(postUsersRoute, async (c: AppContext) => {
        //　...
        const { db, tracer } = c.get("services");
        return tracer.startActiveSpan('registerUserRandomPostApi', async (span) => {
            const dbSpan = tracer.startSpan('DB Transaction'); // Transaction内でSpanを作成
            const res = await db.query.transaction(async (tx) => {
                let insertedUsers: InsertUserTable[] = [];
                let i = 0;
                for (const user of users) {
                    i++;
                    const s = tracer.startSpan(`Insert User Count: ${i}`); // 1roopごとにSpanを作成
                    const u = await tx.insert(UserTableSchema).values(user).returning().execute();
                    if (u.length < 1) {
                        await tx.rollback();
                        return null;
                    }
                    insertedUsers.push(u[0]);
                    s.end(); // 1roopごとのSpanを終了
                }
                return insertedUsers;
            });
            dbSpan.end(); // Transaction内のSpanを終了
            //...
            return c.json(res, 200);
        });
    });
```
上記のように**ビジネスロジックに計装**を行い、リクエストごとのトレースを取ることで、**APIのパフォーマンスを可視化**することができます。

下記は**Exporter**を**Baselime**[^2]に設定し、Cloudflare Workers上でのTraceを可視化する例です。
![alt text](/images/hono/instrument.png)
[^2]: https://baselime.io/
# 応用編

ここからはサードパーティライブラリとの組み合わせや、より実践的な開発Tipsについて紹介します。

## 様々なRequest/Responseの扱い

### ファイルアップロード

ファイルアップロードといえば`multipart/form-data`ですが、Honoでは**HonoRequest**に`parseBody`が実装されており、これにより簡単に`multipart/form-data`を扱うことができます。
https://hono.dev/docs/api/request

```ts:example.ts
const body = await c.req.parseBody({ all: true })
```

`@hono/zod-openapi`との組み合わせで、複数ファイルアップロードのスキーマを定義し、型安全にファイルを扱うことができます。
```ts:schema.ts
const fileUploadRequestBodySchema = z.object({
  files: z
    .preprocess(
      (input) => {
        if (!Array.isArray(input)) {
          return [input]
        }
        return input
      },
      z.array(z.custom<File>((v) => v instanceof File))
    )
    .openapi({
      type: 'array',
      items: {
        type: 'string',
        format: 'binary',
        description: 'File',
      },
    }),
})
```
```ts:example.ts
const reqValidationResult = fileUploadRequestBodySchema.safeParse(
      await c.req.parseBody({ all: true })
    )
//reqValidationResult.data.files: File[]
```

また**Body Limit Middleware**と組み合わせることで、アップロードされたファイルのサイズに対して制限をかけることも可能です。

https://hono.dev/docs/middleware/builtin/body-limit#body-limit-middleware

### ストリーミング

Honoではストリーミングを扱いやすくするヘルパーが用意されています。
https://hono.dev/docs/helpers/streaming

最近ではOpenAIのAPI等がストリーミングをサポートしているため、それと組み合わせてリアルタイムに処理を行う例が増えています。
```ts:example.ts
app.post("/chat", async (c) => {
    return streamSSE(c, async (stream) => {
        stream.onAbort(() => {
            stream.close()
        })

        const chatStream = openai.beta.chat.completions.stream({
            model: "gpt-3.5-turbo",
            messages: [{ role: "user", content: "hoge" }],
            stream: true,
        });

        for await (const message of chatStream) {
            stream.writeSSE({ // Server-Sent Events
                data: JSON.stringify({
                    message: message.choices[0].message.content,
                    //...
                })
            })
        }
        stream.close();
    });
});
```
上記のように、ストリーミングを扱いやすくでき、かつ**StreamingAPI**には`onAbort`や`pipe`が実装されているため、ストリーミングの中断やReadableStreamの繋ぎ合わせ等も実装が可能です。

:::message
ストリーミングヘルパーの中で起きたエラーは、`onError`で捕捉することができないため、ヘルパー内のエラーは第三引数で処理する必要があります。
```ts
app.get('/stream', (c) => {
  return stream(
    c,
    async (stream) => {
      //...
    },
    (err, stream) => {
      stream.writeln('An error occurred!')
      console.error(err)
    }
  )
})
```
:::

公式でもVercelのSDKと組み合わせた例が紹介されています。
@[tweet](https://x.com/honojs/status/1776714886019785174)

### WebSocket
Honoでは**WebSocket**を扱いやすくするためのヘルパーも用意されています。
https://hono.dev/docs/helpers/websocket

特に**RPCモード**を使用した場合に、Server/Client間で非常に簡単にSocketオブジェクトを扱うことが可能になります。
```ts:example.ts
// server.ts
const wsApp = app.get(
  '/ws',
  upgradeWebSocket((c) => {
    //...
  })
)

export type WebSocketApp = typeof wsApp

// client.ts
const client = hc<WebSocketApp>('http://localhost:8787')
const socket = client.ws.$ws()
```

## セキュリティについて

Honoではセキュリティに関するミドルウェアやヘルパーが用意されています。

### Secure Headers Middleware

基本的なセキュリティヘッダを設定するためのミドルウェアです。`strictTransportSecurity`などHTTPSを強制するヘッダや、XSSのフィルタリングを行う`xXssProtection`等のヘッダを設定することができます。

https://hono.dev/docs/middleware/builtin/secure-headers

```ts:example.ts
const app = new Hono()
app.use(
  '*',
  secureHeaders({
    strictTransportSecurity: 'max-age=63072000; includeSubDomains; preload',
    xXssProtection: '1',
  })
)
```

**nonce属性**を使用したCSPの設定も可能です。
```ts:example.ts
import { secureHeaders, NONCE } from 'hono/secure-headers'
import type { SecureHeadersVariables } from 'hono/secure-headers'

// 変数の型を指定して`c.get('secureHeadersNonce')`を推論：
type Variables = SecureHeadersVariables

const app = new Hono<{ Variables: Variables }>()

// 事前定義されたnonce値を`scriptSrc`に設定：
app.get(
  '*',
  secureHeaders({
    contentSecurityPolicy: {
      scriptSrc: [NONCE, 'https://allowed1.example.com'],
    },
  })
)

// `c.get('secureHeadersNonce')`から値を取得：
app.get('/', (c) => {
  return c.html(
    <html>
      <body>
        {/** contents */}
        <script src='/js/client.js' nonce={c.get('secureHeadersNonce')} />
      </body>
    </html>
  )
})
```

nonce値はContext経由で取得できるようになっています。

https://github.com/honojs/hono/blob/b9799e4f45da70e1fe49957b7c29e35208405d91/src/middleware/secure-headers/secure-headers.ts#L116C1-L120C2

内部実装は下記のように、`secureHeadersNonce`というキーでnonce値をContextに保存しています。
```ts:example.ts
const generateNonce = () => {
  const buffer = new Uint8Array(16)
  crypto.getRandomValues(buffer)
  return encodeBase64(buffer)
}

export const NONCE: ContentSecurityPolicyOptionHandler = (ctx) => {
  const nonce =
    ctx.get('secureHeadersNonce') ||
    (() => {
      const newNonce = generateNonce()
      ctx.set('secureHeadersNonce', newNonce)
      return newNonce
    })()
  return `'nonce-${nonce}'`
}
```

### Authentication Middleware

Honoでは**Basic, Bearer, JWT**などの認証を簡単に実装するためのミドルウェアが用意されています。

後述しますがOpenAPIHonoと組み合わせてSwagger UIを表示することができるサードパーティライブラリがあります。**特定のページのみBasicAuthをつける**等の柔軟な認証設定が可能です。
```ts:example.ts
import { swaggerUI } from '@hono/swagger-ui'
import { basicAuth } from 'hono/basic-auth'
//...
  app.use('/swagger-ui', basicAuth({
    username: 'xxx',
    password: 'yyy',
  }))
  app.use('/doc', basicAuth({
    username: 'xxx',
    password: 'yyy',
  }))
  app.doc('/doc', {
    openapi: '3.1.0',
    info: {
      version: '1.0.0',
      title: 'nozomi-chat-api',
      description: 'Nozomi API',
    },
  })
  app.get('/swagger-ui', swaggerUI({ url: '/doc' }))
```

### JWT
**JWT**はヘルパーで基本的な操作（デコード、署名、検証）を行えるようになっており、一般的な対称トークンであれば検証も行うことができます。
https://hono.dev/docs/helpers/jwt#verify
```ts:example.ts
import { verify } from 'hono/jwt'

const tokenToVerify = 'token'
const secretKey = 'mySecretKey'

const decodedPayload = await verify(tokenToVerify, secretKey)
```

**HMAC**や**RSA**等、基本的なアルゴリズムはサポートしていますが、Auth0やClerk等で使用される**非対称トークンの公開鍵検証は実装されていない**ため、**jose**等のライブラリを組み合わせる必要があります。

https://github.com/honojs/hono/issues/672

## Hono Proxy

Honoの[Routerは正規表現やワイルドカードに対応している](https://hono.dev/docs/concepts/routers)ため、特定のパス以下すべてに対してリクエストをプロキシする等の処理を簡単に書くことができます。

```ts:example.ts
import { Hono } from 'hono'

const app = new Hono()

app.get('/posts/:filename{.+.png$}', (c) => {
  const referer = c.req.header('Referer')
  if (referer && !/^https:\/\/example.com/.test(referer)) {
    return c.text('Forbidden', 403)
  }
  return fetch(c.req.url)
})

app.get('*', (c) => {
  return fetch(c.req.url)
})

export default app
```

差し込みのミドルウェアを使用してETagやキャッシュを挟むことも容易です。


Cloudflare Workersを使用したプロキシパターンとして紹介されています。
https://zenn.dev/yusukebe/articles/647aa9ba8c1550


## Request ID Middleware
Hono v4.5.0からRequest IDを自動で生成しContextで扱える便利ミドルウェアが追加されました。

https://hono.dev/docs/middleware/builtin/request-id

```ts
import { Hono } from 'hono'
import { requestId } from 'hono/request-id'

const app = new Hono()

app.use('*', requestId())

app.get('/', (c) => {
  return c.text(`Your request id is ${c.get('requestId')}`)
})
```

このrequestIDですが、IDの`generator`として`crypto.randomUUID()`を使用しています。大体どのランタイムでも動くと思いますが、自分でカスタマイズも可能になっています。
```ts
app.use('*', requestId({
  generator: () => {
    return 'custom-request-id'
  }
}))
```

さらに`headerName`を指定してあげることで、リクエストヘッダーからrequestIDを取得することも可能です。(デフォルトは`X-Request-ID`)

```ts
app.use('*', requestId({
  headerName: 'X-Custom-Request-ID'
}))
```
この機能により、フロントエンドからバックエンドまで透過的にRequest IDを扱うことも可能になります。

## IP Restrict Middleware

Hono v4.5.0からIP制限をかけるためのミドルウェアが追加されました。
このミドルウェアによりIPv4, IPv6のアドレスを指定してアクセス制限をかけることができます。
**allow/deny**, **CIDR**形式の指定も可能です。

https://hono.dev/docs/middleware/builtin/ip-restriction

```ts
import { Hono } from 'hono'
import { getConnInfo } from 'hono/bun'
import { ipRestriction } from 'hono/ip-restriction'

const app = new Hono()

app.use(
  '*',
  ipRestriction(getConnInfo, {
    denyList: [],
    allowList: ['127.0.0.1', '::1']
  })
)
```

上記で使用している`getConnInfo`は`Context`からIPアドレスを取得するためのhelperです。`getConnInfo`自体はadaptorとして各ランタイム、ホスティング先ごとに実装されています。

https://hono.dev/docs/helpers/conninfo#conninfo-helper

## Cloudflare Pages Middleware

Hono v4.5.0より、Cloudflare Pages Functionのミドルウェアとしてそのまま扱えるアダプターが追加されました。

https://hono.dev/docs/getting-started/cloudflare-pages#cloudflare-pages-middleware

```ts:example.ts
import { handleMiddleware } from 'hono/cloudflare-pages'
import { basicAuth } from 'hono/basic-auth'

export const onRequest = handleMiddleware(
  basicAuth({
    username: 'hono',
    password: 'acoolproject'
  })
)
```

Pages FunctionとしてHonoを使用する際には、このアダプターを使用することで簡単にミドルウェアを適用することができます。
https://developers.cloudflare.com/pages/functions/middleware/

## Service Worker Adapter
Hono v4.5.0より、Service Worker上でHonoが使用できるアダプターが追加されました。
ブラウザ上でのバックグラウンド処理や、オフライン対応のためにService Workerを使用する際に、Honoの統合が容易になっています。

https://hono.dev/docs/getting-started/service-worker

```ts:sw.ts
import { Hono } from 'hono'
import { handle } from 'hono/service-worker'

const app = new Hono().basePath('/sw')
app.get('/', (c) => c.text('Hello World'))

self.addEventListener('fetch', handle(app))
```

このファイルをService Workerとしてmain側で登録することで、Honoアプリケーションは/sw.tsへのアクセスをinterceptし、Honoのルーターを使用してリクエストを処理します。


## executionCtx

HonoのContextには**executionCtx**というオブジェクトのプロパティがあります。
https://hono.dev/docs/api/context#executionctx

**Cloudflare Workers等のServerless環境**では、レスポンスが返った後に処理を行うことができません。
そのため、重い処理はキューイングしたり、タイムアウトを気にしつつ同期的に処理を行う必要があります。
このような状況下で`executionCtx.waitUntil`を使用することで、**バックグラウンドで非同期処理**を行うことが可能です。
```ts:example.ts
// ExecutionContext object
app.get('/foo', async (c) => {
  c.executionCtx.waitUntil(
    c.env.KV.put(key, data)
  )
  ...
})
```

具体的なユースケースとしては**DBのコネクションの解放**や、**ログやメトリクスのemit/flush処理**などが挙げられます。
```ts:example.ts
const PostUserApi = (app: App) =>
    app.openapi(postUserRoute, async (c: AppContext) => {
        const { db } = c.get("services");
        // ...
        const res = await db.query.transaction(async (tx) => {
            const u = await tx.insert(UserTableSchema).values(user).returning().execute();
            if (u.length < 1) {
                await tx.rollback();
                return null;
            }
            return u.at(0);
        });

        if (!res) {
            throw new CustomApiError({
                code: "INTERNAL_SERVER_ERROR",
                message: "Failed to insert user",
            });
        }

        c.executionCtx.waitUntil(db.client.end()); // connectionの解放
        return c.json(res, 200);
    });
```

下記記事では`waitUntil`と`Cache`を使用して、ISRを再現する例が紹介されており非常に参考になります。

https://zenn.dev/monica/articles/a9fdc5eea7f59c
https://yusukebe.com/posts/2022/dcs/

## Hono Zod OpenAPIで実現するスキーマ駆動開発

実際にAPIを開発する際は**OpenAPIをベース**とした**スキーマ駆動開発**を行う場合が多いと思います。
Honoではサードパーティライブラリである`zod-openapi`と`swagger-ui`を使用することで、スキーマ駆動開発を円滑に行えるようになっています。

https://hono.dev/snippets/zod-openapi

下記のように`OpenAPIHono`を定義し、routeを登録します。

```ts:entrypoint.ts
import { OpenAPIHono } from '@hono/zod-openapi'

const app = new OpenAPIHono()

app.openapi(route, (c) => {
  const { id } = c.req.valid('param')
  return c.json({
    id,
    age: 20,
    name: 'Ultra-man',
  })
})
```

オリジナルのHonoインスタンスとの違いは`createRoute`を元にhttpメソッドおよびpath等を指定する点です。
```ts:route.ts
import { createRoute } from '@hono/zod-openapi'

const route = createRoute({
  method: 'get',
  path: '/users/{id}',
  request: {
    params: ParamsSchema,
  },
  responses: {
    200: {
      content: {
        'application/json': {
          schema: UserSchema,
        },
      },
      description: 'Retrieve the user',
    },
  },
})
```

上記の**OpenAPIHonoインスタンス**に対して`openapi`メソッドを通じてRouterを登録すると、**ZodのSchemaから自動的にOpenAPI Documentationを作成してくれる**ようになります。
また、`swagger-ui`を使用することで、自動生成されたOpenAPI DocumentationをSwagger UIとして表示することができます。

https://hono.dev/snippets/swagger-ui

さて、ここで[ミドルウェアの設計](#ミドルウェアの設計)で紹介したfactoryパターンを使用し、OpenAPIHonoインスタンスを作成してみます。
```ts:example.ts
const newApp = () => {
    const app = new OpenAPIHono<HonoEnv>({
        defaultHook: handleZodError,
    });
    app.use(prettyJSON());
    app.onError(handleError);

    app.use('/swagger-ui')
    app.use('/doc')
    app.doc('/doc', {
        openapi: '3.1.0',
        info: {
            version: '1.0.0',
            title: 'api',
            description: 'API',
        },
    })
    app.get('/swagger-ui', swaggerUI({ url: '/doc' }))

    app.openAPIRegistry.registerComponent("securitySchemes", "bearerAuth", {
        bearerFormat: "root key",
        type: "http",
        scheme: "bearer",
    });
    return app;
}
```
OpenAPIHonoには、Honoに標準で用意されているメソッドはもちろん、`defaultHook`や`openAPIRegistry`のメソッドがあります。

### defaultHook

`defaultHook`は**Zodのエラーハンドリング**を行うための**フック**です。
https://github.com/honojs/middleware/tree/main/packages/zod-openapi#a-dry-approach-to-handling-validation-errors

defaultHookはOpenAPIHonoクラスのインスタンスにおいて、**バリデーションが失敗した場合や処理後に実行されるフック**として定義されます。

内部的には`openapi`メソッドで登録されたRouteにアクセスが来た際に**zValidator**を通じてバリデーションを行い、エラーが発生した場合に`defaultHook`内のresultがfalseになります。
https://github.com/honojs/middleware/blob/023e07be0ac689cbbd659a017d833aaf67e19c75/packages/zod-openapi/src/index.ts#L292-L396

そのため、ルートのインスタンスで`defaultHook`を使用することで、バリデーションエラーに対するエラーハンドリングを一括で行うことができます。

下記のようなhandlerを定義し、zodのエラーメッセージを返すことで、通常の`HTTPException`と同様のスキーマでエラーハンドリングを行うことが可能になります。
```ts:example.ts
const handleZodError = (
    result:
        | {
            success: true;
            data: any;
        }
        | {
            success: false;
            error: ZodError;
        },
    c: Context<HonoEnv>,
) => {
    if (!result.success) {
        return c.json<ErrorResponse, StatusCode>(
            {
                error: {
                    code: "BAD_REQUEST",
                    message: parseZodErrorMessage(result.error),
                    requestId: c.get("requestId"),
                },
            },
            { status: 400 },
        );
    }
}
```

また、route定義時に**ErrorSchema**も定義することが可能なため、汎用的なエラースキーマを定義しておくと使い手のエラーハンドリングが楽になります。
```ts:error.schema.ts
const errorSchemaFactory = (code: z.ZodEnum<any>) => {
    return z.object({
        error: z.object({
            code: code.openapi({
                description: "error code.",
                example: code._def.values.at(0),
            }),
            message: z
                .string()
                .openapi({ description: "explanation" }),
            requestId: z.string().openapi({
                description: "requestId",
                example: "req_1234",
            }),
        }),
    });
}

const errorResponses = {
    400: {
        description:
            "The server cannot or will not process the request due to something that is perceived to be a client error.",
        content: {
            "application/json": {
                schema: errorSchemaFactory(z.enum(["BAD_REQUEST"])).openapi("ErrBadRequest"),
            },
        },
    },
    401: {
        description: `Although the HTTP standard specifies "unauthorized", semantically this response means "unauthenticated". ...`,
        content: {
            "application/json": {
                schema: errorSchemaFactory(z.enum(["UNAUTHORIZED"])).openapi("ErrUnauthorized"),
            },
        },
    },
    403: {
        description:
            "The client does not have access rights to the content; ...",
        content: {
            "application/json": {
                schema: errorSchemaFactory(z.enum(["FORBIDDEN"])).openapi("ErrForbidden"),
            },
        },
    },


 // ...
}
```

RouteにErrorSchemaを定義
```ts:route.ts
const postUserRoute = createRoute({
    tags: ["user"],
    operationId: "userKey",
    method: "post" as const,
    path: "/users",
    security: [{ bearerAuth: [] }],
    request: {
        body: {
            required: true,
            content: {
                "application/json": {
                    schema: UserPostBodySchema,
                },
            },
        },
    },
    responses: {
        200: {
            description: "The configuration for an api",
            content: {
                "application/json": {
                    schema: UserResponseSchema,
                },
            },
        },
        ...errorResponses, // 定義したErrorSchemaを使用
    },
});
```

上記を行うと、Swagger UI上でエラーハンドリングが行われた際のスキーマが表示されるようになります。
![alt text](/images/hono/error.png)

### openAPIRegistry

OpenAPIHonoには`openAPIRegistry`というプロパティがあり、OpenAPIのスキーマを登録することができます。

https://github.com/honojs/middleware/tree/main/packages/zod-openapi#the-registry

この機能により、セキュリティスキームを登録し、Routeから使用することが可能になります。
```ts:example.ts
// securitySchemes
app.openAPIRegistry.registerComponent("securitySchemes", "bearerAuth", {
    bearerFormat: "root key",
    type: "http",
    scheme: "bearer",
});

// route
const postUserRoute = createRoute({
    // ...
    security: [{ bearerAuth: [] }],
})
```

![alt text](/images/hono/bearer.png)

# Honoの活用事例

この内容はHono Conference 2024にて発表した内容をもとにしています。
https://speakerdeck.com/sugarcat7/using-hono-in-b2b-saas-application


AI ShiftではHonoを使用してAI WorkerというB2BマルチテナントSaaSアプリケーションの開発を行っています。

https://www.ai-shift.co.jp/3958

設計や使用技術の詳細についてはこちらのブログを参照ください。

https://zenn.dev/aishift/articles/ce9783a0d7acd0

## Tech Stack

AI Workerはお客様環境や自社環境のk8s環境にデプロイするために、各サービスをコンテナ化して扱っています。

![alt text](/images/hono/techstack.png)

その中で、フロントエンドから使用するAPIにHonoを採用しています。

![alt text](/images/hono/techstack2.png)

## ユースケース
AI Workerのキーワードは`AI`と`マルチテナントSaaS`です。

### AI
AI Workerは名前の通りAIを活用したサービスを提供しています。
AIと一括りにすると領域が広すぎるので、ここではLLMの利用について紹介します。

- LLM

LLMというと昨今では**GPT**や**Claude**の利用が挙げられます。

特にOpenAI社が提供するGPTのAPIでは、LLMの回答生成時に**Stream(Server Sent Event)**による回答の生成方法をサポートしています。

![alt text](/images/hono/stream.gif)

これにより、インタラクティブな対話型UIを実現することが可能になります。

前述した[SSE](#ストリーミング)のヘルパーを使用することで、Hono上で簡単にストリーミングを実装することができます。

- RAG

このLLMによるテキスト生成を活用するために、弊社ではRAGを取り入れた社内文書検索の仕組みを取り入れています。

RAGはユーザ**Input**(Query) をもとに**Vector Store**から関連性に基づいて文書を検索し、その結果を**Promptに埋め込み回答LLMで生成する**という仕組みです。(とてもざっくり)
![alt text](/images/hono/rag1.png)
![alt text](/images/hono/rag2.png)
![alt text](/images/hono/rag3.png)
![alt text](/images/hono/rag4.png)


このような仕組みを活用するためには、あらかじめ検索に使用するためのドキュメントを何らかの方法でVector Storeに登録しておく必要があります。
![alt text](/images/hono/rag5.png)

ここでHonoの[ファイルアップロード](#ファイルアップロード)が活躍します。

### B2BマルチテナントSaaS

AI WorkerはB2BマルチテナントSaaSアプリケーションです。
お客様ごとに異なる設定やセキュリティポリシーを持つため、データを適切に分離する必要があります。
![alt text](/images/hono/b2b.png)

テナントごとのDB特定のために、`TenantID`を使用してテナントごとのデータを特定しています。

HonoのContextを使用して、リクエストごとに`TenantID`を保持することで、テナントごとのデータを取得することができます。

```ts:example.ts
const getUser = (c: AppContext): Result<User> => {
    const p: Payload = c.get('jwtPayload')
    if (!p.org_id) {
        return Result.fail(new AppError(StatusCode.forbidden, 'Please select an organization'))
    }
    c.set('sessionUser', {
        userId: p.sub,
        tenantId: p.org_id,
    })
}

const middleware = (): MiddlewareHandler<HonoEnv> => {
    return async (c, next) => {
        const res = await getUser(c)

        if (!res.isSuccess) {
            return c.json({ message: res.getError().message }, res.getError().code)
        }
        const { logger } = c.get('services')
        logger.info({
            message: '[Request Started]',
            method: c.req.method,
            url: c.req.url,
            userInfo: c.get('sessionUser'),
        })
        return await next()
    }
}
```

任意のIdPから取得したJWTのペイロードからtenantIdを取得し、Contextにセットし伝播させることでテナントデータを扱いやすくします。
```ts
c.set('sessionUser', {
    userId: p.sub,
    tenantId: p.org_id,
})
```

この際にLoggerを同時に仕込み、アクセスログとして残すことも可能です。(どのTenantのユーザがどのリクエストを投げたかを残す)
```ts
const { logger } = c.get('services')
logger.info({
    message: '[Request Started]',
    method: c.req.method,
    url: c.req.url,
    userInfo: c.get('sessionUser'),
})
```

- Security

最後に簡単なセキュリティ対策の例を紹介します。
[セキュリティについて](#セキュリティについて)で紹介した`Secure Headers Middleware`の使用や、JWTの適切な検証はもちろんのこと、アプリケーション内での人が起こすミスを防ぐことも必要です。

例えば弊社ではアプリケーションの構築にClean Architectureを採用しており、各レイヤー間のEntityの詰め替え処理を行う際にZodによるバリデーションを行っています。

![alt text](/images/hono/clean.png)

これは例ですが、仮に`User`というEntityに`password`というフィールドが含まれている場合、そのままDBに保存したり、レスポンスに含めることはセキュリティ上好ましくありません。
そこで、Zodによるparseを活用し、`password`フィールドを含めないようにすることが可能です。
![alt text](/images/hono/clean2.png)


# まとめ

Honoの一連の機能を使いこなすことで、開発効率を向上させながら堅牢なAPIを開発することができます。
ここでは紹介しきれていないHono RPCやHonoX等、さまざまな機能があるので公式ドキュメントを参照しながら、ぜひHonoを使って開発を進めてみてください。

## 参考
https://hono.dev/
https://speakerdeck.com/yusukebe/shi-jian-etuziyusukesu
https://zenn.dev/yusukebe/articles/a00721f8b3b92e
https://zenn.dev/monica/articles/a9fdc5eea7f59c
https://yusukebe.com/posts/2022/dcs/
