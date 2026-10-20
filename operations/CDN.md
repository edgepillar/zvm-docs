# Serving NFT media from `nft.zenon.foo`

The node stores every NFT image and metadata document under the SHA-256 of its
bytes and serves it immutable. That is the whole reason a CDN can sit in front of
it: a URL can never mean anything else, so a cache may keep it forever, and
anyone who has the bytes can re-host them and be checked against the name. The
hostname is where the bytes are usually found, not what makes them the right
bytes.

This is the deployment. Nothing here is required to run the ZVM — without it the
pages load the same objects from the node itself.

---

## What the node already does

```
-media-dir  /data/media            where objects live
-media-url  https://nft.zenon.foo  the canonical base, written into token URIs
-media-cdn  false                  whether pages should load from that base yet
```

With `-media-url` set, the node answers requests whose `Host` is that hostname
with **objects only** — no explorer, no API, no uploads:

```
GET https://nft.zenon.foo/<sha256>.png     an image
GET https://nft.zenon.foo/<sha256>.json    a metadata document
GET https://nft.zenon.foo/<sha256>/7       that document with {id} replaced by 7
```

Every answer carries `Cache-Control: public, max-age=31536000, immutable`, an
`ETag` of the hash, `X-Content-Type-Options: nosniff`, and a `Content-Security-Policy`
that sandboxes the object — an SVG served from this hostname can draw itself and
nothing else.

The same objects are always available from the node at `/media/<sha256>.png`, and
the page prefers whichever `-media-cdn` says, so art shows before the hostname
exists and keeps showing if it ever goes away.

## Putting Cloudflare in front of it

`zenon.foo` already uses Cloudflare's nameservers, so this is one hostname in a
zone you hold. The tunnel means nothing has to be exposed to the internet.

1. **Create a tunnel.** In the Cloudflare dashboard: Zero Trust → Networks →
   Tunnels → Create a tunnel (Cloudflared). Copy the tunnel token.

2. **Add the public hostname** to that tunnel:
   - Subdomain `nft`, domain `zenon.foo`
   - Service `HTTP`, URL `node:8080`
   Leave the host header alone: the node routes on it.

3. **Run the connector** beside the node:

   ```sh
   TUNNEL_TOKEN=… docker compose --profile cdn up -d
   ```

   That starts the `cloudflared` service in `docker-compose.yml`, on the same
   network as the node.

4. **Make the zone cache it.** Cloudflare caches images by extension by default
   but not `.json`, and not an extension-less path. Add one Cache Rule:
   - **If** `Hostname equals nft.zenon.foo`
   - **Then** Eligible for cache, Edge TTL: *Use cache-control header if present*,
     Browser TTL: *Respect origin*.

5. **Tell the node it is live**, so pages load from the CDN rather than from it:

   ```yaml
   - -media-cdn=true
   ```

   Token URIs do not change — they always said `https://nft.zenon.foo/…`.

### Checking it

```sh
# twice: MISS, then HIT
curl -sI https://nft.zenon.foo/<hash>.png | grep -i 'cf-cache-status\|cache-control\|content-type'
```

A `cf-cache-status: HIT` on the second request means the edge is serving it and
the node is not being asked.

## Taking something down

An operator's domain is theirs to answer for. To stop serving one object:

```sh
docker exec zvm-node zvm media hide -db /data/zvm.db <sha256>
docker exec zvm-node zvm media list -db /data/zvm.db     # what is stored
```

The node then answers `451 Unavailable For Legal Reasons` for it. **Purge the CDN
too** — Cloudflare is holding its own copy: Caching → Configuration → Purge
custom URL, with `https://nft.zenon.foo/<hash>.<ext>`.

The bytes stay on disk (`zvm media show` puts an object back). Deleting them is a
file in `-media-dir`, named by its hash.

## R2 instead of a tunnel

If you would rather not have the node be the origin, an R2 bucket with
`nft.zenon.foo` as its custom domain serves the same URLs, and objects can be
copied into it because they are named by their hash:

```sh
docker exec zvm-node tar -C /data/media -cf - . | tar -xf - -C ./media
rclone copy ./media r2:zvm-media --immutable
```

The node stays the source of truth: it is what the indexer reads metadata from,
and what an upload writes to. Keep `-media-cdn=true` and the URLs are identical.

## What this does not do

- **It does not make the art permanent.** If the domain lapses and nobody
  re-hosts, a token URI stops resolving — the same trade every HTTP-hosted
  collection makes. What content addressing buys is that re-hosting is enough:
  anyone with the bytes can serve them from anywhere and be checked.
- **It does not moderate uploads.** Uploads take a wallet signature and are rate
  limited per account, which makes them attributable and bounded, not reviewed.
