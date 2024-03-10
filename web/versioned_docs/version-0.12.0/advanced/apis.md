---
title: Custom HTTP API Endpoints
---

import { ShowForTs, ShowForJs } from '@site/src/components/TsJsHelpers'
import { Required } from '@site/src/components/Tag'

In Wasp, the default client-server interaction mechanism is through [Operations](../data-model/operations/overview). However, if you need a specific URL method/path, or a specific response, Operations may not be suitable for you. For these cases, you can use an `api`. Best of all, they should look and feel very familiar.

## How to Create an API

APIs are used to tie a JS function to a certain endpoint e.g. `POST /something/special`. They are distinct from Operations and have no client-side helpers (like `useQuery`).

To create a Wasp API, you must:

1. Declare the API in Wasp using the `api` declaration
2. Define the API's NodeJS implementation

After completing these two steps, you'll be able to call the API from the client code (via our `Axios` wrapper), or from the outside world.

### Declaring the API in Wasp

First, we need to declare the API in the Wasp file and you can easily do this with the `api` declaration:

<Tabs groupId="js-ts">
<TabItem value="js" label="JavaScript">

```wasp title="main.wasp"
// ...

api fooBar { // APIs and their implementations don't need to (but can) have the same name.
  fn: import { fooBar } from "@src/apis",
  httpRoute: (GET, "/foo/bar")
}
```
</TabItem>
<TabItem value="ts" label="TypeScript">

```wasp title="main.wasp"
// ...

api fooBar { // APIs and their implementations don't need to (but can) have the same name.
  fn: import { fooBar } from "@src/apis",
  httpRoute: (GET, "/foo/bar")
}
```
</TabItem>
</Tabs>

Read more about the supported fields in the [API Reference](#api-reference).


### Defining the API's NodeJS Implementation

<ShowForTs>

:::note
To make sure the Wasp compiler generates the types for APIs for use in the NodeJS implementation, you should add your `api` declarations to your `.wasp` file first _and_ keep the `wasp start` command running.
:::
</ShowForTs>

After you defined the API, it should be implemented as a NodeJS function that takes three arguments:

1. `req`: Express Request object
2. `res`: Express Response object
3. `context`: An additional context object **injected into the API by Wasp**. This object contains user session information, as well as information about entities. The examples here won't use the context for simplicity purposes. You can read more about it in the [section about using entities in APIs](#using-entities-in-apis).

<Tabs groupId="js-ts">
<TabItem value="js" label="JavaScript">

```ts title="src/apis.js"
export const fooBar = (req, res, context) => {
  res.set("Access-Control-Allow-Origin", "*"); // Example of modifying headers to override Wasp default CORS middleware.
  res.json({ msg: `Hello, ${context.user ? "registered user" : "stranger"}!` });
};
```

</TabItem>
<TabItem value="ts" label="TypeScript">

```ts title="src/apis.ts"
import { FooBar } from "wasp/server/api"; // This type is generated by Wasp based on the `api` declaration above.

export const fooBar: FooBar = (req, res, context) => {
  res.set("Access-Control-Allow-Origin", "*"); // Example of modifying headers to override Wasp default CORS middleware.
  res.json({ msg: `Hello, ${context.user ? "registered user" : "stranger"}!` });
};
```

</TabItem>
</Tabs>

<ShowForTs>

#### Providing Extra Type Information

We'll see how we can provide extra type information to an API function.

Let's say you wanted to create some `GET` route that would take an email address as a param, and provide them the answer to "Life, the Universe and Everything." 😀 What would this look like in TypeScript?

Define the API in Wasp:

```wasp title="main.wasp"
api fooBar {
  fn: import { fooBar } from "@src/apis",
  entities: [Task],
  httpRoute: (GET, "/foo/bar/:email")
}
```

We can use the `FooBar` type to which we'll provide the generic **params** and **response** types, which then gives us full type safety in the implementation.

```ts title="src/apis.ts"
import { FooBar } from "wasp/server/api";

export const fooBar: FooBar<
  { email: string }, // params
  { answer: number } // response
> = (req, res, _context) => {
  console.log(req.params.email);
  res.json({ answer: 42 });
};
```

</ShowForTs>

## Using the API

### Using the API externally

To use the API externally, you simply call the endpoint using the method and path you used.

For example, if your app is running at `https://example.com` then from the above you could issue a `GET` to `https://example/com/foo/callback` (in your browser, Postman, `curl`, another web service, etc.).

### Using the API from the Client

To use the API from your client, including with auth support, you can import the Axios wrapper from `wasp/client/api` and invoke a call. For example:

<Tabs groupId="js-ts">
<TabItem value="js" label="JavaScript">

```jsx title="src/pages/SomePage.jsx"
import React, { useEffect } from "react";
import { api } from "wasp/client/api";

async function fetchCustomRoute() {
  const res = await api.get("/foo/bar");
  console.log(res.data);
}

export const Foo = () => {
  useEffect(() => {
    fetchCustomRoute();
  }, []);

  return <>// ...</>;
};
```
</TabItem>
<TabItem value="ts" label="TypeScript">

```tsx title="src/pages/SomePage.tsx"
import React, { useEffect } from "react";
import { api } from "wasp/client/api";

async function fetchCustomRoute() {
  const res = await api.get("/foo/bar");
  console.log(res.data);
}

export const Foo = () => {
  useEffect(() => {
    fetchCustomRoute();
  }, []);

  return <>// ...</>;
};
```
</TabItem>
</Tabs>

#### Making Sure CORS Works

APIs are designed to be as flexible as possible, hence they don't utilize the default middleware like Operations do. As a result, to use these APIs on the client side, you must ensure that CORS (Cross-Origin Resource Sharing) is enabled.

You can do this by defining custom middleware for your APIs in the Wasp file.

<Tabs groupId="js-ts">
<TabItem value="js" label="JavaScript">

For example, an `apiNamespace` is a simple declaration used to apply some `middlewareConfigFn` to all APIs under some specific path:

```wasp title="main.wasp"
apiNamespace fooBar {
  middlewareConfigFn: import { fooBarNamespaceMiddlewareFn } from "@src/apis",
  path: "/foo"
}
```

And then in the implementation file:

```js title="src/apis.js"
export const apiMiddleware = (config) => {
  return config;
};
```

</TabItem>
<TabItem value="ts" label="TypeScript">

For example, an `apiNamespace` is a simple declaration used to apply some `middlewareConfigFn` to all APIs under some specific path:

```wasp title="main.wasp"
apiNamespace fooBar {
  middlewareConfigFn: import { fooBarNamespaceMiddlewareFn } from "@src/apis",
  path: "/foo"
}
```

And then in the implementation file (returning the default config):

```ts title="src/apis.ts"
import { MiddlewareConfigFn } from "wasp/server";
export const apiMiddleware: MiddlewareConfigFn = (config) => {
  return config;
};
```

</TabItem>
</Tabs>

We are returning the default middleware which enables CORS for all APIs under the `/foo` path.

For more information about middleware configuration, please see: [Middleware Configuration](../advanced/middleware-config)

## Using Entities in APIs

In many cases, resources used in APIs will be [Entities](../data-model/entities.md).
To use an Entity in your API, add it to the `api` declaration in Wasp:

<Tabs groupId="js-ts">
<TabItem value="js" label="JavaScript">

```wasp {3} title="main.wasp"
api fooBar {
  fn: import { fooBar } from "@src/apis",
  entities: [Task],
  httpRoute: (GET, "/foo/bar")
}
```
</TabItem>
<TabItem value="ts" label="TypeScript">

```wasp {3} title="main.wasp"
api fooBar {
  fn: import { fooBar } from "@src/apis",
  entities: [Task],
  httpRoute: (GET, "/foo/bar")
}
```
</TabItem>
</Tabs>

Wasp will inject the specified Entity into the APIs `context` argument, giving you access to the Entity's Prisma API:

<Tabs groupId="js-ts">
<TabItem value="js" label="JavaScript">

```ts title="src/apis.js"
export const fooBar = (req, res, context) => {
  res.json({ count: await context.entities.Task.count() });
};
```

</TabItem>
<TabItem value="ts" label="TypeScript">

```ts title="src/apis.ts"
import { FooBar } from "wasp/server/api";

export const fooBar: FooBar = (req, res, context) => {
  res.json({ count: await context.entities.Task.count() });
};
```

</TabItem>
</Tabs>

The object `context.entities.Task` exposes `prisma.task` from [Prisma's CRUD API](https://www.prisma.io/docs/reference/tools-and-interfaces/prisma-client/crud).

## API Reference

<Tabs groupId="js-ts">
<TabItem value="js" label="JavaScript">

```wasp title="main.wasp"
api fooBar {
  fn: import { fooBar } from "@src/apis",
  httpRoute: (GET, "/foo/bar"),
  entities: [Task],
  auth: true,
  middlewareConfigFn: import { apiMiddleware } from "@src/apis"
}
```
</TabItem>
<TabItem value="ts" label="TypeScript">

```wasp title="main.wasp"
api fooBar {
  fn: import { fooBar } from "@src/apis",
  httpRoute: (GET, "/foo/bar"),
  entities: [Task],
  auth: true,
  middlewareConfigFn: import { apiMiddleware } from "@src/apis"
}
```
</TabItem>
</Tabs>

The `api` declaration has the following fields:

- `fn: ExtImport` <Required />

  The import statement of the APIs NodeJs implementation.

- `httpRoute: (HttpMethod, string)` <Required />

  The HTTP (method, path) pair, where the method can be one of:

  - `ALL`, `GET`, `POST`, `PUT` or `DELETE`
  - and path is an Express path `string`.

- `entities: [Entity]`

  A list of entities you wish to use inside your API. You can read more about it [here](#using-entities-in-apis).

- `auth: bool`

  If auth is enabled, this will default to `true` and provide a `context.user` object. If you do not wish to attempt to parse the JWT in the Authorization Header, you should set this to `false`.

- `middlewareConfigFn: ExtImport`

  The import statement to an Express middleware config function for this API. See more in [middleware section](../advanced/middleware-config) of the docs.