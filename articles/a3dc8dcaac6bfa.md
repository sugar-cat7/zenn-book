---
title: "Honoを使い倒したい"
emoji: "❤️‍🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに

こんにちは、AI Shift バックエンドエンジニアの[@sugar235711](https://twitter.com/sugar235711)です。
この記事では、Honoの使い方をおさらいした上で、API開発を通じてHonoの実際の開発で使い倒すためのTipsを紹介します。

Honoそもそものコンセプトや網羅的な実装例については、公式ドキュメントを参照してください。
https://hono.dev/concepts/motivation

# 基本編

この章では、Honoの基本的な使い方について紹介します。

## App/Contextオブジェクトの使い方

Honoでは、プライマリオブジェクトであるHonoインスタンスを生成し、そのインスタンスをもとにAPIのエンドポイントを定義します。
```ts
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.text('Hono!'))
export default app
```

https://hono.dev/api/hono#app-hono


Honoにはリクエスト/レスポンスを扱いやすくする**Context**というオブジェクトがあります。上記の例で言えば、`c`がContextオブジェクトです。

https://hono.dev/api/context#context

この**Context**オブジェクトは、ヘッダーやリクエストボディ、レスポンスボディなどの操作を行うためのメソッドを提供はもちろん、環境変数やカスタムミドルウェアを設定することもできます。

```ts
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

上記のように、`Hono<Env>`とすることで、function等をContext経由で型安全に別メソッドに伝播することができます。

さて、このままでも十分便利ですが実際にAPIを作成する際はカスタムのHonoインスタンス(Factory)を作成して使い回すとファイル分割等を行った際に便利です。

- ReturnTypeを使用し、Bindingの型付けがされたHonoインスタンスを返す関数を作成する

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


- 作成したHonoインスタンスを使ってAPIを作成する
```ts:routes/hoge.ts
export const hogeApi = (app: App) => {
    app.get('/hoge', (c) => c.text('hoge'))
}
```

- Honoインスタンスをシングルトンで使い回す
```ts:entrypoint.ts
import { newApp } from './customHono';
import { hogeApi } from './routes/hoge';

const app = newApp();
hogeApi(app)
//...

```

上記のように自身で定義することも可能ですが、Honoの公式HelperとしてFactoryメソッドが用意されているのでそれらを使用するのも良いと思います。
https://hono.dev/helpers/factory

## ミドルウェアの設計
APIにおけるミドルウェアの責務は、ユーザー認証・認可、ロギング、キャッシュ等のAPI呼び出し時の共通処理を行うことです。
Honoでは、ビルトインで用意されているミドルウェアや自身で作成したカスタムのミドルウェアを使用することが可能です。

```ts:example.ts
const example = (): MiddlewareHandler => {
    return async (c, next) => {
        console.log('middleware start')
        await next()
    }
}

// or
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

ミドルウェア自体は定義順に実行されるため、下記のような実行順序になります。
```
example() -> cors() -> post handler
```

nextの後に処理を追加しStackのような処理を実現することも可能です。詳細については、公式ドキュメントを参照してください。
https://hono.dev/guides/middleware#execution-order


このミドルウェアの中では任意の処理を行うことができますが、基本的には1リクエストのスコープに閉じた動作を行うべきです。(例えば、ユーザーID等のリクエストごとに一意に扱いたい情報をContextに詰め伝播させるなど)

下記は、リクエスト時に初期化を行うミドルウェアの例です。
要点としては、リクエストごとに一意なIDを生成し、リクエストごとに一意なロガーを生成してContextに詰めています。
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

上記によって、リクエストごとに一意なロガーを生成し、Contextを通してhandlerから安全にLoggerを利用することができます。


## エラーハンドリングについて

Honoでは、`onError`メソッドによりthorwされたエラーをキャッチしトップレベルでエラーハンドリングを行うことができます。


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

上記のようにHonoには`HTTPException`クラスが用意されており、認証時のエラーなどのHTTPステータスコードを指定してエラーをthrowするように推奨されています。

https://hono.dev/api/exception#exception


- エラーハンドリングの例
より実践的な例として、HonoのCustom Instance内にonErrorメソッドを定義し、各エラータイプごとに処理を出し分けるハンドラーを定義しエラーハンドリングを行います。

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
                    docs: "https://example.com/docs",
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
                docs: "https://example.com/docs",
                message: err.message ?? "something unexpected happened",
                requestId: c.get("requestId"),
            },
        },
        { status: 500 },
    );
}
```


### バリデーションエラーについて

致命的なエラー以外にも、バリデーションエラーなどのエラーをハンドリングすることがあります。
HonoではZodやVailbotなどのライブラリを組み合わせ、リクエストのスキーマに対するバリデーションを簡単に実装することができます。
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

後述するOpenAPIHonoとdefaultHooksを組み合わせることで、zodエラメッセージを加工する例を紹介します。



## 環境変数について
昨今ではJavaScriptを取り巻く環境下の多様化により、環境変数の読み込み方も多様化しています。
例えば下記のような主要なランタイムでは、それぞれ異なる環境変数の読み込み方法があります。
- Workerd
 - wrangler.toml/.dev.vars
- Deno
  - Deno.env
  - .env file
- Bun
  - Bun.env
  - process.env
- Node
  - process.env

そのため、仮にCloudflare Workersで開発していたアプリケーションをコンテナ化して別のランタイムで動かす場合、環境変数の読み込み方法が異なるためコードを書き換える必要があります。
このような負担を減らすため、Honoではランタイムによらず環境変数を読み込むためのヘルパーが用意されています。

https://hono.dev/helpers/adapter#env

```ts:example.ts
import { env } from 'hono/adapter'

app.get('/env', (c) => {
  const { NAME } = env<{ NAME: string }>(c)
  return c.text(NAME)
})
```

adapterを通すことでランタイムによらない環境変数の読み込みを行うことが可能となります。
さらに下記のようにzodと組み合わせることで型安全に環境変数の値を取り出すことができます。

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

一般的に構造化ロギングを行おうとする際に、RequestIDを付与したContextualなLoggerを作成することが多いです。
しかし、Node.js等の環境ではシングルスレッドで動作するため、非同期処理を行う際にグローバルにRequestIDを保持することが難しいです。
そのため、AsyncLocalStorageを組み合わせることでこれを回避することができます。

```ts
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
`getRequestId()`を任意の処理で呼び出すことで、リクエストごと一意なRequestIDを保持したloggerを作成できます。


上記が一般的なロギングのプラクティスかと思いますが、HonoではContextオブジェクトを伝播させることでも、この問題を解決することができます。
例えば、[ミドルウェアの設計](#ミドルウェアの設計)で紹介した初期化処理でLoggerをContextに詰めることで、リクエストごとのLoggerを保持することができます。
```ts
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

ただし、実務だとCleanArchitrctureに則り、レイヤー分けがされている場合が多くレイヤー間でContextを伝播させない選択をする場合は依然としてAsyncLocalStorageを使用することになると思います。


## オブザーバビリティについて

オブザーバビリティはログ、トレース、メトリクスの3つの観点からシステムの内部状態を可視化し、蚊観測性を向上させます。OpenTelemetryはこの3つの観点を統一的に扱うためのライブラリです。
このOpenTelemetryのSDKと組み合わせることでHonoでもトレースを取ることができます。
ロギング同様に、Context経由でTracerを伝播させることで、リクエストごとのトレースを取ることができます。

https://opentelemetry.io/

```ts
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

例えばCloudflare Workersで計装を行えるOTelライブラリがあります。
https://github.com/evanderkoogh/otel-cf-workers
このライブラリとHonoを組み合わせ、Cloudflare Workers上で簡単に計装を行うことが可能です。
下記ではランダムに複数ユーザーを登録するAPIを計装しています。(わざとBulk Insertをしていません)
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
                    s.end();
                }
                return insertedUsers;
            });
            dbSpan.end();
            //...
            return c.json(res, 200);
        });
    });
```
ビジネスロジックに計装を行い、リクエストごとのトレースを取ることで、APIのパフォーマンスを可視化することができます。
下記はExporterをBaselimeとして設定すると下記のようにトレースを可視化することができます。
![alt text](/images/hono/instrument.png)

# 応用編
ここからは3rdpatyライブラリとの組み合わせや、より実践的な開発Tipsについて紹介します。

## 様々なRequest/Responseの扱い

### File Upload
ファイルアップロードといえば`multipart/form-data`ですが、HonoではHonoRequestに`parseBody`が実装されており、これにより簡単にmultipart/form-dataを扱うことができます。
https://hono.dev/docs/api/request

```ts:example.ts
const body = await c.req.parseBody({ all: true })
```

`@hono/zod-openapi`との組み合わせで、複数ファイルアップロードのスキーマを定義することも可能です。
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

またBody Limit Middlewareと組み合わせることで、アップロードされたファイルのサイズに対して制限をかけることも可能です。

https://hono.dev/docs/middleware/builtin/body-limit#body-limit-middleware

### streaming
Honoではstreamを扱いやすくするhelperが用意されています。
https://hono.dev/docs/helpers/streaming

最近ではOpenAIのAPI等がStreamをサポートしているため、それと組み合わせてリアルタイムに処理を行う例が増えています。
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
            stream.writeSSE({
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
上記のように、streamを扱いやすくでき、かつStreamingAPIには`onAbort`や`pipe`が実装されているため、Streamingの中断やReadableStreamの繋ぎ合わせ等も実装が可能となっています。

:::message
Stream Helperの中で起きたエラーは、`onError`で補足することができないため、helper内のエラーは第三引数で処理する必要があります。
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


公式でもVercelのSDKと組み合わせた例も紹介されています。
https://x.com/honojs/status/1776714886019785174/photo/1

- WebSocket
HonoではWebSocketを扱いやすくするためのHelperも用意されています。
https://hono.dev/docs/helpers/websocket

特にRPCモードを使用した場合に、Server/Client間で非常に簡単にSocketオブジェクトを扱うことが可能になります。
```ts
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
Honoではセキュリティに関するミドルウェアやhelperが用意されています。

### Secure Headers Middleware
基本的なセキュリティヘッダを設定するためのミドルウェアです。
`strictTransportSecurity`などHTTPSを強制するヘッダや、XSSのフィルタリングを行う`xXssProtection`等のヘッダを設定することができます。

https://hono.dev/docs/middleware/builtin/secure-headers#secure-headers-middleware

```ts
const app = new Hono()
app.use(
  '*',
  secureHeaders({
    strictTransportSecurity: 'max-age=63072000; includeSubDomains; preload',
    xXssProtection: '1',
  })
)
```

nonce attributeを使用したCSPの設定も可能です。
```ts
import { secureHeaders, NONCE } from 'hono/secure-headers'
import type { SecureHeadersVariables } from 'hono/secure-headers'

// Specify the variable types to infer the `c.get('secureHeadersNonce')`:
type Variables = SecureHeadersVariables

const app = new Hono<{ Variables: Variables }>()

// Set the pre-defined nonce value to `scriptSrc`:
app.get(
  '*',
  secureHeaders({
    contentSecurityPolicy: {
      scriptSrc: [NONCE, 'https://allowed1.example.com'],
    },
  })
)

// Get the value from `c.get('secureHeadersNonce')`:
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

nonce値はctx経由で取得できるようになっています。
https://github.com/honojs/hono/blob/b9799e4f45da70e1fe49957b7c29e35208405d91/src/middleware/secure-headers/secure-headers.ts#L116C1-L120C2
```ts
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
HonoではBasic, Bearer, JWTなどの認証を簡単に実装するためのミドルウェアが用意されています。

後述しますがOpenAPIHonoと組み合わせてSwagger UIを組み合わせることができる3rd partyライブラリが存在します。このページのみBasicAuthをつけるなどの柔軟な認証設定が可能です。
```ts
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



JWTはhelperで基本的な操作(decode, sign, verify)を行えるようになっており、対称トークンであれば検証も行うことができます。
https://hono.dev/docs/helpers/jwt#verify
```ts
import { verify } from 'hono/jwt'

const tokenToVerify = 'token'
const secretKey = 'mySecretKey'

const decodedPayload = await verify(tokenToVerify, secretKey)
```

HMACやRSA等基本的なアルゴリズムはサポートしていますが、Auth0やClerk等で使用される非対称トークンの公開鍵検証は実装されていないため、jose等のライブラリを組み合わせる必要があります。

https://github.com/honojs/hono/issues/672



## Hono Proxy
Honoの[Routerは正規表現やワイルドカードに対応している](https://hono.dev/docs/concepts/routers)ため、特定のPath以下全てに対してリクエストをProxyをする等の処理を簡単に書くことができます。


```ts
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

差し込みのミドルウェアを使用してETagやキャッシュを挟むことも用意です。
:::message
Cloudflare Workersを使用したProxyパターンとして紹介されています。
https://zenn.dev/yusukebe/articles/647aa9ba8c1550
:::

## executionCtx
HonoのContextにはexecutionCtxというオブジェクトのプロパティがあります。
https://hono.dev/docs/api/context#executionctx

Cloudflare Workers等のServerless環境では、レスポンスが返った後に処理を行うことができません。
そのため、重い処理はQueuingしたり、タイムアウトを気にしつつ同期的に処理を行う必要があります。
このような状況下で`executionCtx.waitUntil`を使用することで、バックグラウンドで非同期処理を行うことが可能です。
```ts
// ExecutionContext object
app.get('/foo', async (c) => {
  c.executionCtx.waitUntil(
    c.env.KV.put(key, data)
  )
  ...
})
```

具体的なユースケースとしてはDBのコネクションの解放や、ログやメトリクスのemit/flush処理などが挙げられます。
```ts
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

以下ではwaitUntilとCacheを使用して、ISRを再現する例が紹介されており非常に参考になります。

https://zenn.dev/monica/articles/a9fdc5eea7f59c
https://yusukebe.com/posts/2022/dcs/


## Hono Zod OpenAPIで実現するスキーマ駆動開発
実際にAPIを開発する際はOpenAPIをベースとしたスキーマ駆動開発を行う場合が多いと思います。Honoでは3rd partyライブラリである`zod-openapi`と`swagger-ui`を使用することで、スキーマ駆動開発を円滑に行えるようになっています。

https://hono.dev/snippets/zod-openapi


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

オリジナルとのHonoインスタンスの差分は`createRoute`を元にhttp methodおよびpath等を指定するようになります。
```ts
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

上記をOpenAPIHonoインスタンスに対して`openapi`メソッドを通じてRouterを登録するとZodのSchemaから自動的にOpenAPI Documentationを作成してくれるようになります。
また、`swagger-ui`を使用することで、自動生成されたOpenAPI DocumentationをSwagger UIとして表示することができます。


さて、ここで[## ミドルウェアの設計]()でのfactoryパターンを使用し、OpenAPIHonoインスタンスを作成し手みます。
```ts
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
Honoに標準で用意されているメソッドはもちろん、OpenAPIHonoには`defaultHook`や`openAPIRegistry`の機能があります。

### defaultHook
`defaultHook`はZodのエラーハンドリングを行うためのフックです。
https://github.com/honojs/middleware/tree/main/packages/zod-openapi#a-dry-approach-to-handling-validation-errors

defaultHookはOpenAPIHonoクラスのインスタンスにおいて、バリデーションが失敗した場合や処理後に実行されるフックとして定義されます。
内部的には`openapi`メソッドで登録されたRouteにアクセスが来た際にzValidatorを通じてバリデーションを行い、エラーが発生した場合に`defaultHook`内のresultがfalseになります。
https://github.com/honojs/middleware/blob/023e07be0ac689cbbd659a017d833aaf67e19c75/packages/zod-openapi/src/index.ts#L292-L396

そのため、rootで`defaultHook`を使用することで、バリデーションエラーに対するエラーハンドリングを一括で行うことができます。
下記のようなhandlerを定義し、zodのエラーメッセージを好みのメッセージにしてメッセージとして返すことで、通常の`HTTPException`と同様のスキーマでエラーハンドリングを行うことが可能になります。
```ts
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

またroute定義時にErrorSchemaも定義することが可能なため、汎用的なエラースキーマを定義しておくと使い手のエラーハンドリングが楽になります。
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

- RouteにErrorSchemaを定義
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
        ...errorResponses,
    },
});
```

上記を行うと、Swagger UI上でエラーハンドリングが行われた際のスキーマが表示されるようになります。
![alt text](/images/hono/error.png)

### openAPIRegistry

OpenAPIHonoには`openAPIRegistry`というプロパティがあり、OpenAPIのスキーマを登録することができます。

https://github.com/honojs/middleware/tree/main/packages/zod-openapi#the-registry

この機能により、セキュリティスキーム等を自動登録されたRouteのスキームと同じように登録することができます。
```ts
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

# まとめ

Honoの一連の機能を使いこなすことで、開発効率を向上させながら堅牢なAPIを開発することができます。
ぜひ、Honoを使って開発を進めてみてください。

## 参考


