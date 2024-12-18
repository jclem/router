# Router

Router is a minimal router for building Bun HTTP services.

## Installation

```shell
bun add @jclem/router
```

## Use

### Routing

Routing in Router is a matter of assigning handlers to HTTP methods and paths.
Here is an example of a simple Router router:

```ts
import { createRouter } from "@jclem/router";
import { expect } from "bun:test";

const router = createRouter()
  .get("/", () => Response.json({ ok: true }))
  .get("/:foo", ({ parameters }) => Response.json(parameters))
  .get("/:foo/bar/*", ({ parameters }) => Response.json(parameters));

let response = await router.handle(new Request("http://example.com"));
expect(await response.json<unknown>()).toEqual({ ok: true });

response = await router.handle(new Request("http://example.com/foo"));
expect(await response.json<unknown>()).toEqual({ foo: "foo" });

response = await router.handle(
  new Request("http://example.com/foo/bar/baz/qux"),
);
expect(await response.json<unknown>()).toEqual({ foo: "foo", "*": "baz/qux" });
```

#### Sub-routers

Router supports sub-routers, where a new router is mounted at a given path. This router
will inherit the middleware stack of the parent router.

```ts
import { createRouter, createMiddleware } from "@jclem/router";
import { expect } from "bun:test";

const router = createRouter()
  .use(createMiddleware(() => ({ a: 1 })))
  .route("/foo", (t) =>
    t
      .use(createMiddleware(() => ({ b: 2 })))
      .get("/", (ctx) => Response.json(ctx.locals))
      .route("/bar", (t) =>
        t
          .use(createMiddleware(() => ({ c: 3 })))
          .get("/", (ctx) => Response.json(ctx.locals)),
      ),
  )
  .get("/", (ctx) => Response.json(ctx.locals));

let resp = await router.handle(new Request("http://example.com"));
expect(resp.status).toBe(200);
expect(await resp.json<unknown>()).toEqual({ a: 1 });

resp = await router.handle(new Request("http://example.com/foo"));
expect(resp.status).toBe(200);
expect(await resp.json<unknown>()).toEqual({ a: 1, b: 2 });

resp = await router.handle(new Request("http://example.com/foo/bar"));
expect(resp.status).toBe(200);
expect(await resp.json<unknown>()).toEqual({ a: 1, b: 2, c: 3 });
```

Note that Router isn't an HTTP _server_, it's just a router. In order to invoke
the router, just pass it a `Request` via its `handle(request: Request)` method,
like the one you get from a Bun HTTP server handler. This `handle` method
returns a `Response` or a Promise resolving to a `Response`.

### Middleware

Router uses one method for attaching middleware, called `use`. Middleware is
always invoked before route matching happens, so even when no route is matched,
middleware is still invoked.

The easiest way to write middleware is to use the `createMiddleware` function
provided by Router. The return value (if one is present) of the function given to
`createMiddleware` will be merged into the request "locals", which will be
available to middleware further down the stack.

As seen below in `logRequest`, `createMiddleware` can also accept a second
function, which will run after the handler is called. It receives the context
argument as well as the response.

```ts
import { createRouter, createMiddleware } from "@jclem/router";
import crypto from "node:crypto";

const assignRequestID = createMiddleware(({ request }) => {
  const requestID = request.headers.get("Request-ID") || crypto.randomUUID();
  return { requestID };
});

const logRequest = createMiddleware(
  ({ request, locals }) => {
    return { startTime: process.hrtime.bigint() };
  },
  ({ request, locals }, response) => {
    const elapsed = process.hrtime.bigint() - locals.startTime;

    console.log(
      `${request.method} ${request.url} ${response.status} ${elapsed}ns`,
    );
  },
);

const router = createRouter()
  .use(assignRequestID)
  .use(logRequest)
  .get("/", () => Response.json({ ok: true }));
```

Note that `createMiddleware` is just a convenience function. You can also
manually write middleware, should you choose to do so, but it is a little more
difficult to deal with types (they'll be under-specified, but not inaccurate).
We could write the above module without `createMiddleware`. In order to do so: A
middleware takes two arguments: The incoming `BeforeCtx<Locals>` object, and a
`Next<NewLocals>` callback function, which returns a `Response`.

So, the basic raw middlware flow looks like this:

1. Do something with the incoming context before the request.
2. Call `next`, with the new locals. The calls the remaining middleware and the
   request handler.
3. Do something with the response and the new locals.
4. Return the response.

Note that due to how middlewares are stacked, only the first "half" of all
remaining middleware will run when `next()` in called. This allows you to
effectively have "before" and "after" middleware using the same function.

It looks like this:

```ts
import { BeforeCtx, Next, createRouter } from "@jclem/router";
import crypto from "node:crypto";

function assignRequestID(
  { request, locals }: BeforeCtx<{}, {}>,
  next: Next<{ requestID: string }>,
) {
  const requestID = request.headers.get("Request-ID") || crypto.randomUUID();
  return next({ ...locals, requestID });
}

async function logRequest(
  { request, locals }: BeforeCtx<{ requestID: string }, {}>,
  next: Next<{ requestID: string }>,
) {
  const startTime = process.hrtime.bigint();
  const response = await next(locals);
  const elapsedMs = Number(process.hrtime.bigint() - startTime) / 1e6;
  console.log(
    `${request.method} ${request.url} ${response.status} ${elapsedMs}ms`,
  );
  return response;
}

const router = createRouter()
  .use(assignRequestID)
  .use(logRequest)
  .get("/", () => Response.json({ ok: true }));
```

To write this in a more type-safe manner, use the exported types such as
`Next<O>` and `Middleware<I, O>` provided by Router:

```ts
import { Middleware, createRouter } from "@jclem/router";
import crypto from "node:crypto";

function assignRequestID<I, P>(): Middleware<I, I & { requestID: string }, P> {
  return function ({ request, locals }, next) {
    const requestID = request.headers.get("request-id") || crypto.randomUUID();
    return next({ ...locals, requestID });
  };
}

function logRequest<I extends { requestID: string }, P>(): Middleware<I, I, P> {
  return async function ({ request, locals }, next) {
    const startTime = process.hrtime.bigint();
    const response = await next(locals);
    const elapsedMs = Number(process.hrtime.bigint() - startTime) / 1e6;
    console.log(
      `${locals.requestID} ${request.method} ${request.url} ${response.status} ${elapsedMs}ms`,
    );
    return response;
  };
}

const router = createRouter()
  .use(assignRequestID())
  .use(logRequest())
  .get("/", () => Response.json({ ok: true }));
```

### Serving Real Requests

In order to serve real requests, just call `router.handle` in a Bun HTTP server
request handler.

```ts
import { createRouter } from "router";

const router = createRouter().get("/", () => Response.json({ ok: true }));

Bun.serve({
  port: 3000,
  fetch(request) {
    return router.handle(request);
  },
});
```
