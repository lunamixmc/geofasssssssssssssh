# GeoDash (browser build)

`index.html` is the whole thing: game loader, the online layer and
[libcurl.js](https://github.com/ading2210/libcurl.js) all live in that one file.
The game's own payloads come from jsDelivr, so GitHub Pages needs nothing but
this file — no build step, no service worker, no server of your own.

## Reaching the Geometry Dash servers

`www.boomlings.com` sends no CORS headers, so a browser cannot call it from a
page, and school filters usually block the domain outright. The page therefore
never calls it directly.

The WASM build performs every online request as
`fetch("/__proxy?url=<encoded target>")`. `GDNet` (inlined near the top of
`index.html`) intercepts exactly those calls and replays them through
**libcurl.js** — curl compiled to WebAssembly — over a **WISP** WebSocket tunnel.
That is the same mechanism GUST uses:

- the game server sees the relay's address, never yours;
- the only connection your network can see is a WebSocket to the relay;
- the game gets back a real `Response` (status, headers and a streaming body),
  so level search, profiles, comments, leaderboards and account login work
  unmodified.

Requests carry `User-Agent: " "`. Boomlings' Cloudflare layer rejects normal
browser user agents on `/database/` but accepts the blank one (it is what the
game's own web builds send), so the tunnel returns the real API text instead of
a 403 page.

Relays are tried in order. One that fails is blacklisted for five minutes and
the request is retried on the next; the relay that worked is remembered in
`localStorage`, so later visits skip the dead ones. If the working relay dies
mid-session, the next request picks a new one automatically.

## Controls

Open the console on the page:

```js
GDNet.status()                       // active relay, saved relay, endpoint order
GDNet.check()                        // probe every relay, print a table
GDNet.setWisp("wss://mages.io/wisp/")// pin a relay that works on your network
GDNet.clearWisp()                    // forget it and go back to auto
```

Or add `?wisp=wss://your-relay.example/wisp/` to the URL. The setting is saved
for that browser. A small status pill in the bottom-right corner shows
`GD ONLINE · <relay>` once a relay is live, and `GD OFFLINE` if none answered;
it ignores clicks so it never gets in the way of the game.

Built-in relays:

- `wss://mages.io/wisp/`
- `wss://anura.pro/`
- `wss://invisiproxy.com/wisp/`
- `wss://definitelyscience.com/wisp/`
- `wss://lichology.com/wisp/`
- `wss://motor-cycle-part.org/wisp/`
- `wss://wisp.terbiumon.top/wisp/`
- `wss://wisp.mercurywork.shop/`
- `wss://nebulaproxy.io/wisp/`

If every public relay is blocked, point `GDNet.setWisp()` at a WISP server you
run yourself ([Mercury Workshop's WISP](https://github.com/MercuryWorkshop/wisp-server-node)
is the reference implementation). GitHub Pages does not need to host it.

## Hosting

Drop `index.html` on GitHub Pages (or any static host, or just open it as a
local file). The game streams its ~366 MB of assets from jsDelivr as before —
that is unrelated to the online layer and unchanged.

## Files

- `index.html` — the game, the relay layer and libcurl.js (single file)
- `GeoDashOnline.html` — older wrapper that iframes the upstream jsDelivr build

## Notes and limits

- libcurl.js 0.7.4 is vendored inline (LGPL-3.0, © ading2210), with its WASM
  module embedded as a data URI so the page makes no extra request for it.
- Online play needs at least one reachable WISP relay. Public relays are shared
  and can be slow or rate-limited; `GDNet.check()` tells you which are alive.
- The game's fallback path for loading an audio file does a *synchronous* XHR;
  that cannot be tunnelled (the tunnel is asynchronous), so a sound missing from
  the data file still falls back to the browser. Everything else that goes
  online uses `fetch` and is tunnelled.
