# Putting the chain-69 stack online as `devnet.zenon.foo`

> These are the procedures for the chain 69 deployment behind devnet.zenon.foo,
> which is operated from a deployment tree on the devnet's host (the deprecated
> `zvm` repository, not published). The compose files, gateway and scripts they
> name are there; the services they run are built from the published
> repositories. A local devnet to work against is `zvm-fleet/devnet/`.

One hostname, everything under it at its own path:

| Path | What answers | Served from |
|---|---|---|
| `/explorer/` | zvm-explorer | static bundle in `gateway/site/explorer` |
| `/glyphswap/` | zvm-glyphswap | static bundle in `gateway/site/glyphswap` |
| `/karum/` | zvm-karum | static bundle in `gateway/site/karum` |
| `/zvm/rpc`, `/zvm/l1rpc`, `/zvm/api/…`, `/zvm/healthz`, `/zvm/media/…` | the ZVM node | `node:8080` |
| `/zenon` | chain 69's znnd, HTTP and WebSocket on the same URL | `host.docker.internal:35997` / `:35998` |
| `/status/` | the status page, and `/status/api/…` behind it | static bundle, and `status:8090` |
| `/sentinelpoc/` | the Sentinel lab console of `zsentm3`, another chain's (76), with no login | `host.docker.internal:8760`, a Go program on the host (`zenon-sentinels/lab/cmd/demo`) |

`/` redirects to `/explorer/`. Any other `/zvm/…` path redirects to the same
path under `/explorer/`, because glyphswap and karum link a transaction as
`<node>/tx/<hash>` and the standalone explorer has the same routes. The node's
own embedded UI is not published: it assumes it owns the origin's root and
breaks under a prefix.

What a wallet is given:

- **EVM wallet** (MetaMask, Rabby): `https://devnet.zenon.foo/zvm/rpc`, chain 7340469.
- **Zenon wallet or SDK**: `wss://devnet.zenon.foo/zenon`, or `https://…/zenon`
  for HTTP. Prefer wss from a page: a WebSocket needs no CORS preflight.

## How a request gets there

`cloudflared` dials **out** to Cloudflare, so nothing on this machine is
port-forwarded. That is forced rather than chosen: the default route is a VPN
(Mullvad), which forwards no inbound ports and would make any dynamic-DNS
updater publish the VPN's exit address.

The tunnel has exactly one route, `devnet.zenon.foo → http://gateway:80`. The
gateway is a Caddy container (`gateway/Caddyfile`) that owns every path. The
paths live there and not as per-path rules in the Zero Trust dashboard because
a tunnel rule can neither strip a prefix nor serve files, and because a change
to a file is a reviewed diff while a dashboard edit is not.

The front ends are built for this host by `gateway/build-sites.ps1`: each with
its prefix as Vite's base, `VITE_ZVM_NODE` set to `https://devnet.zenon.foo/zvm`
and `VITE_GATEWAY` to `https://devnet.zenon.foo`. No app has a node built in,
and each build refuses to run without them. Each app's checkout needs its
zvm-core submodule (`git submodule update --init`). The GitHub Pages builds set
the same two from the organisation's Actions variables.

**Not** reachable, on purpose: op-reth's own ports (49545/49546/49551), the
relayer, bridge, signers, verifier, and the faucet except through the node's
`/api/faucet`.

- **No ZVM WebSocket RPC.** The node proxies HTTP `POST /rpc` only; op-reth's
  `:8546` is not behind it and also serves `debug_`. `eth_subscribe` does not
  work; polling does.
- **`debug_`, `txpool_`, `admin_` and `trace_` are refused** on `/zvm/rpc`
  (`-rpc-deny` in the compose file). A trace over a large block is a cheap way
  to keep op-reth busy. The filter fails closed: a request it cannot read is
  refused, not forwarded. Until 2026-09-23 it could be walked past in two ways,
  both confirmed against this host. One was a second `Method` key: Go's JSON
  decoder folds key case and keeps the last value, while op-reth reads the exact
  key `method`. The other was one malformed element, which made a whole batch
  unreadable to the filter so it was forwarded unchecked.
  `zvm-node/api/ethrpc_test.go` pins both.
- **znnd is not filtered.** Every namespace it can serve is marked `Public` in
  `go-zenon/rpc/apis.go` and none holds a key. It is still a shared devnet's
  node that was not built for public load; the rate limit below is its only
  protection.

## Is it up?

`https://devnet.zenon.foo/status/` answers that, without asking anyone: the
`status` service checks every row of the table above every five minutes — plus
the relayer and the bridge, which are not public but are what a stuck
transaction or withdrawal is waiting on — keeps thirty days, and emails when one
goes down. It probes the **gateway**, so a red
row is that service; its one `edge` row is checked through Cloudflare, and is
the row that goes red when the tunnel rather than a service is the problem.

Alerts need an SMTP relay with a login, in `zvm/.env`:

```
SMTP_HOST=smtp.example.net
SMTP_PORT=587
SMTP_USER=...
SMTP_PASS=...
SMTP_FROM=REPLACE_WITH_SENDER_MAILBOX
ALERT_TO=REPLACE_WITH_RECIPIENT_MAILBOX
```

Then prove it, once, rather than finding out during an outage:

```powershell
docker compose -f docker-compose.chain69.yml --profile public run --rm status -test-alert
```

Direct delivery to the recipient's MX is not an option here: the default route
is a VPN, and VPN and residential exits are blocked on port 25 by most
providers and distrusted by the rest. Without `SMTP_HOST` the outage is still
recorded and the page says alerts are not configured.

The thing it cannot report is this machine being off, because it runs on it.
That needs a prober somewhere else, and `zvm-status/README.md` says so too.

## Steps on this machine

1. **Build the sites** (and again whenever one of the apps changes; the gateway
   reads `site/` per request, so no restart is needed):

   ```powershell
   pwsh gateway/build-sites.ps1
   ```

2. **Start the gateway, the prober and the connector.** The tunnel token is the
   existing `ZVM69_TUNNEL_TOKEN` in `zvm/.env`; the tunnel does not change, only
   its route.

   ```powershell
   docker compose -f docker-compose.chain69.yml --profile public up -d --build gateway status cloudflared
   ```

   The gateway runs with Caddy's admin API off, so nothing can rewrite its
   routes at runtime. The cost is that a `Caddyfile` edit needs
   `docker restart zvm69-gateway`, not `caddy reload`.

3. **Check the routes without Cloudflare**, on the gateway's loopback port:

   ```sh
   curl -sI http://127.0.0.1:49080/explorer/            # 200
   curl -s  http://127.0.0.1:49080/zvm/healthz          # {"ok":true,...}
   curl -s -X POST -H 'content-type: application/json' \
     -d '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' \
     http://127.0.0.1:49080/zvm/rpc                     # {"result":"0x7001b5"}
   curl -s -X POST -H 'content-type: application/json' \
     -d '{"jsonrpc":"2.0","id":1,"method":"ledger.getFrontierMomentum","params":[]}' \
     http://127.0.0.1:49080/zenon                       # chainIdentifier 69
   ```

## Steps in Cloudflare

The tunnel `zvm-69` already exists and its connector is running; this adds the
new hostname to it and retires the old one.

1. **Add the route.** Zero Trust → Networks → Tunnels → `zvm-69` → Edit →
   *Public hostname* (called *Published application routes* in newer
   dashboards) → Add:
   - Subdomain `devnet`, domain `zenon.foo`, path **empty**
   - Service type `HTTP`, URL `gateway:80`

   Saving creates the proxied `devnet` CNAME to `<tunnel-id>.cfargotunnel.com`.
   The connector picks the route up live; nothing restarts.

2. **Check it from outside** (under Mullvad, check DNS with DoH —
   `https://cloudflare-dns.com/dns-query?name=devnet.zenon.foo&type=CNAME` with
   `accept: application/dns-json` — because plain DNS to Cloudflare times out):

   ```sh
   curl -s https://devnet.zenon.foo/zvm/healthz
   curl -s -X POST -H 'content-type: application/json' \
     -d '{"jsonrpc":"2.0","id":1,"method":"debug_traceCall","params":[]}' \
     https://devnet.zenon.foo/zvm/rpc                   # -32601, "not available"
   ```

   Then open `https://devnet.zenon.foo/explorer/`, `/glyphswap/` and `/karum/`.
   Each should connect to the node without anyone picking one.

3. **WebSockets on.** Zone `zenon.foo` → Network → WebSockets. It is on by
   default; `/zenon` over wss fails without it.

4. **Rate-limit the expensive paths.** Security → WAF → Rate limiting rules.
   Edit the `testnet` rule if one was ever made, or create one (the free plan
   allows one):
   - **If** `Hostname equals devnet.zenon.foo` **and** `URI Path is in`
     `/zvm/rpc` `/zvm/l1rpc` `/zvm/api/faucet` `/zenon`

   Leave `/status/api/summary` out of it: the page polls it every 30 seconds,
   and rate-limiting the status page is how an outage becomes two outages.
   - Count by IP; block for 10 s once over the threshold.

   An explorer page makes a burst of `/zvm/rpc` calls, so pick the threshold
   from a day of *Security → Events*, not by guessing. A wss connection to
   `/zenon` counts once, however many calls ride on it.

5. **Retire `testnet.zenon.foo`.** Once step 2 passes: in the same tunnel,
   delete the `testnet.zenon.foo → node:8080` route, then DNS → Records → delete
   the `testnet` CNAME (removing the route does not always remove the record).

   Old links will then fail rather than redirect. A redirect is possible — keep
   the record, and add Rules → Redirect Rules, hostname `testnet.zenon.foo`,
   dynamic target `concat("https://devnet.zenon.foo/zvm", http.request.uri.path)`,
   308 — but only a browser loading a
   page follows it. A wallet that saved `https://testnet.zenon.foo/rpc` POSTs to
   it, and most JSON-RPC clients do not follow a redirect on POST. Anyone with
   the old URL saved has to change it either way.

## The faucet's cooldown

The faucet keys its cooldown on the requester's address, read from
`X-Forwarded-For` past the peers it trusts (`-trust-proxies=172.20.0.0/16`,
the subnet of `zvm-69_default`). The chain is edge → cloudflared → gateway →
node → faucet, and every hop from cloudflared on is inside that subnet.

The gateway is the one hop that could break this: Caddy replaces the header of
an untrusted peer with the peer's own address, which would make every visitor
cloudflared. `trusted_proxies static private_ranges` in the Caddyfile makes it
append instead. Checked on 2026-09-21: a request carrying
`X-Forwarded-For: 203.0.113.9` from the compose network arrived as
`203.0.113.9, 172.20.0.14`.

If `zvm-69_default` is recreated on another subnet the cooldown **fails
closed** — everyone shares one key — and `docker network inspect
zvm-69_default` gives the value to put in `-trust-proxies`.

## What this does not do

- **It is only as available as this machine and Docker Desktop.** The tunnel
  reconnects by itself; it cannot answer while the host is asleep or the stack
  is down.
- **Cloudflare sees the traffic.** It terminates TLS.
- **The GitHub Pages copies still exist** and keep deploying from `main`.
  Glyphswap's and karum's preset node there is now
  `https://devnet.zenon.foo/zvm/rpc`; the explorer's is still
  `http://localhost:47000`.
