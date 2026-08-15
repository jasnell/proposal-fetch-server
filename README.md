# Fetch Server API

**Date:** July 2026

This is a **proposal** for a server-side HTTP API built on the Fetch Standard's `Request` and
`Response` types. It is intended for standardization through ECMA TC55 (WinterTC).

This is a **work in progress** and should not be considered a final design.
The API and specification are evolving as we explore design tradeoffs and gather feedback.

## Why

Every JavaScript runtime uses Fetch's `Request` and `Response` for server-side work, but they've
all diverged on handler signatures, WebSocket upgrade, lifecycle management, and everything else
Fetch doesn't model — trailers, informational responses, extended CONNECT, HTTP Datagrams,
priority.

This specification defines a server-side API that closes those gaps:

* Uses standard `Request` and `Response` without modification.
* Introduces a `ServerContext` for connection metadata, server capabilities, and lifecycle.
* Unifies HTTP/1.1, HTTP/2, and HTTP/3 behind one programming model.
* Handles both request/response exchanges and tunnel protocols (extended CONNECT).
* Defines primitives for the Capsule Protocol and HTTP Datagrams.

## Design Principles

### P1: Standard Fetch Types Without Extension

The `Request` a handler receives is a `Request`. The `Response` a handler returns is a `Response`.
No duck-typing, no structural compatibility concerns, no server-specific subtypes.
`ctx.request instanceof Request` is `true`. Proxying is `return fetch(ctx.request)`.

### P2: Clean Separation of Message and Environment

The HTTP message (`Request`) is separate from the server processing environment (`ServerContext`).
Connection metadata, lifecycle management, and server capabilities are properties of the context,
not the request.

### P3: One Programming Model for All HTTP Versions

A handler should not need to know whether a request arrived over HTTP/1.1, HTTP/2, or HTTP/3.
Protocol-version-specific behavior is the implementation's concern, not the application's.

### P4: Incremental Adoption

A handler that ignores the context and uses only `Request` and `Response` works unchanged.
Server-specific capabilities are available when needed but never required.

### P5: Portability Across Runtimes

The same handler code should run on any conforming runtime without per-runtime adapters.
The handler object is the portable unit — it works with both declarative export and
imperative `serve()`.

### P6: Extensibility for Future Protocols

Extended CONNECT is designed to carry new protocols. The handler model accommodates new
`:protocol` values without API changes and allows for entirely new handlers (e.g., `socket()`
for raw TCP) to be defined.

## Quick Example

```js
export default {
  [Symbol.for('server.protocol')]: 1,

  async fetch(ctx) {
    const { request } = ctx;
    const url = new URL(request.url);

    // Send 103 Early Hints
    if (url.pathname === '/') {
      ctx.sendInformational(103, {
        "Link": '</style.css>; rel=preload; as=style'
      });
    }

    // Proxy a request — ctx.request is a standard Request
    if (url.pathname.startsWith('/proxy/')) {
      return fetch(ctx.request);
    }

    // Background work after response
    ctx.waitUntil(logRequest(request));

    return new Response("Hello!");
  },

  async connect(ctx) {
    switch (ctx.connectProtocol) {
      case 'websocket': {
        const ws = ctx.upgradeWebSocket();
        ws.addEventListener('message', e => ws.send(`Echo: ${e.data}`));
        return;
      }
      case 'webtransport': {
        const session = await ctx.upgradeWebTransport();
        for await (const stream of session.incomingBidirectionalStreams) {
          handleBidiStream(stream);
        }
        return;
      }
      default:
        return new Response(null, { status: 501 });
    }
  },
};
```

## Conformance Layers

The specification has two layers:

| Layer | What it covers | Required? |
|-------|---------------|-----------|
| **Core** (handler model) | `ServerContext`, `ConnectContext`, handler object pattern, callback signatures | MUST implement |
| **Infrastructure** (server lifecycle) | `serve()`, `Server`, `Listener`, `Closeable`, options dictionaries | MAY implement |

Cloudflare Workers would implement the core layer only (the application exports a handler object;
the platform handles everything else). Node.js and Deno would implement both layers.

## Specification

The formal specification is written in [Bikeshed](https://speced.github.io/bikeshed/) format:

* **Source**: [`index.bs`](index.bs)
* **Build**: `npm run build:spec` (or `bash build-spec.sh`)

## Building

```bash
# Install dependencies
npm install

# Build the spec
npm run build:spec

# Build TypeScript (when reference implementation is added)
npm run build

# Run tests (when reference implementation is added)
npm test
```

## Prior Art and Motivation

* [Fetch Is Not Enough](https://jasnell.me/posts/fetch-is-not-enough) — the problem statement
* [Fetch Needs Error Codes](https://jasnell.me/posts/fetch-needs-error-codes) — protocol-level error semantics
* [HTTP Server API: A Draft Specification](https://jasnell.me/posts/http-server-api-draft) — the draft this proposal is based on

## Relationship to Existing Specifications

| Specification | Relationship |
|--------------|-------------|
| [Fetch Standard](https://fetch.spec.whatwg.org/) | `Request`, `Response`, `Headers`, `Body` definitions. Assumes trailer support and `onInformation` callback. |
| [Streams Standard](https://streams.spec.whatwg.org/) | `ReadableStream` and `WritableStream` for bodies, tunnels, capsules, datagrams. `ReadableWritablePair` is the base of `CapsuleStream` and `DatagramStream`. |
| [WebTransport](https://w3c.github.io/webtransport/) | `WebTransportSession` strictly follows the W3C WebTransport API's types (`WebTransportDatagramDuplexStream`, `WebTransportBidirectionalStream`, etc.); they are referenced, not duplicated. |
| [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110) | HTTP semantics (trailers, informational responses). |
| [RFC 9218](https://www.rfc-editor.org/rfc/rfc9218) | Extensible Prioritization Scheme. |
| [RFC 9297](https://www.rfc-editor.org/rfc/rfc9297) | HTTP Datagrams and the Capsule Protocol. |
| [RFC 8441](https://www.rfc-editor.org/rfc/rfc8441) / [RFC 9220](https://www.rfc-editor.org/rfc/rfc9220) | WebSocket via extended CONNECT (HTTP/2, HTTP/3). |
| [RFC 9298](https://www.rfc-editor.org/rfc/rfc9298) | Proxying UDP in HTTP. |
| [RFC 9484](https://www.rfc-editor.org/rfc/rfc9484) | Proxying IP in HTTP. |

## License

[MIT](LICENSE)
