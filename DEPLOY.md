# eRPC gateway — production deploy

A self-hosted eRPC gateway serving **29 networks** (every chain the aa-security-auditor
tool scans, plus the previously-live set), on free public RPC + optional keyed vendors, with
**cross-check consensus** (don't trust one source), a **persistent Redis cache**, and
**secret auth** on the public endpoint.

## What's in this deploy

| File | Role |
|------|------|
| `erpc.yaml` | Gateway config (chains, cache, consensus, auth, providers). **Tracked in git — keep it that way** (no secrets inside; keys come from `${ENV}` placeholders). Versioning it is what prevents server config drift on redeploy. |
| `docker-compose.prod.yml` | Stack: `erpc` + `redis` (+ optional `monitoring`). |
| `erpc.env.example` | API-key / secret template → copy to `.env`. |
| `Dockerfile` | Builds `erpc-server` (already in repo). |

## Chains served

Mainnets (24): ethereum `1` · base `8453` · arbitrum `42161` · optimism `10` · bsc `56` ·
polygon `137` · gnosis `100` · avalanche `43114` · unichain `130` · monad `143` ·
robinhood `4663` · fantom `250` · cronos `25` · linea `59144` · worldchain `480` ·
hyperevm `999` · megaeth `4326` · sei `1329` · blast `81457` · mantle `5000` ·
katana `747474` · opbnb `204` · apechain `33139` · taiko `167000`
Testnets (5): sepolia `11155111` · base-sepolia `84532` · bsc-testnet `97` ·
unichain-sepolia `1301` · monad-testnet `10143`

Chains unlikely to be covered by the public catalog carry a **static upstream** in
`erpc.yaml` (katana, opbnb, apechain, taiko, robinhood, hyperevm — URLs validated live by
the bounty tool's `lib/chains.json`). After every deploy, check `GET /healthcheck` and
prune/fix any network that isn't `OK`. Known limitation: **Monad mainnet (143) has no
`eth_simulateV1`** on public endpoints.

URL shape: `https://rpc.7kt.org/<slug>` (public, via Cloudflare → predix-nginx, which rewrites
`/<slug>` → `/main/<slug>`). Slugs are eRPC **network aliases** in `erpc.yaml` (`/base`,
`/ethereum`, `/linea`, …). Chain-id addressing also works: public `/evm/<chainId>`
(→ `/main/evm/<chainId>`) or same-host `http://127.0.0.1:4000/main/evm/<chainId>`.

## Live staging deployment (rpc.7kt.org) — the actual model

The staging gateway runs **in place** at `/home/ubuntu/erpc/` on the shared EC2, **not** via
`docker-compose.prod.yml`. Verified 2026-07-21:

- **Stack**: `docker-compose.yml` run with the standalone **`docker-compose`** v2 (the `docker compose`
  plugin is not installed); containers `erpc-gateway` + `erpc-redis-gw` (+ `erpc-prometheus`,
  `erpc-grafana`); pinned image `ghcr.io/erpc/erpc:main@sha256:…`. erpc-gateway joins the external
  `predix` network so `predix-nginx` (Cloudflare origin, TLS) can reach it; redis is on
  `erpc_internal`. Host port `127.0.0.1:4000` is published for same-host callers; nothing else is exposed.
- **Config** = this repo's `erpc.yaml`: **project `main`** (29 networks via slug aliases, 2-of-3
  consensus, the curated archive-log getLogs pool, Redis + `memory` cache) **+ project `plinko`**
  (isolated no-consensus lane for the on-host Plinko resolver — keep it). Redis connector:
  `redis://erpc-redis-gw:6379`.
- **Auth** (project `main` only; `plinko` stays open): `network` (allowLocalhost + `allowedIPs`
  = the same-host consumer's public IP) **OR** `secret` (`${ERPC_AUTH_SECRET}`). eRPC resolves the
  real client IP from nginx's `X-Real-IP`, trusted because `server.trustedIPForwarders` includes the
  predix subnet `172.18.0.0/16` (+ Cloudflare ranges); spoofing is blocked (rightmost-untrusted).
- **Update in place** (additive): edit `/home/ubuntu/erpc/erpc.yaml` → validate with the pinned image
  → `docker-compose up -d --force-recreate --no-deps erpc` (recreates ONLY erpc-gateway;
  redis/monitoring + every predix container untouched). Rolling `erpc.yaml.bak.*` for rollback.
  ⚠️ **`--force-recreate` is mandatory when the config changed.** `erpc.yaml` is a *file* bind-mount,
  so replacing it on the host (`mv`/`scp`) creates a NEW inode while the running container keeps the
  OLD one mounted. Plain `up -d` sees an unchanged compose file and just reports `Container … Running`
  — the new config is silently **not** applied, and `restart` does not help either (the mount is fixed
  at container creation). Only a recreate re-resolves the path. Verify the change actually took effect
  with a real request, never by reading the host file.

## Deploy (on the server)

```bash
# 1. Get the files onto the server:
#    - Building from source: copy the whole repo (Dockerfile builds erpc-server).
#    - Using the published image: you only need erpc.yaml + docker-compose.prod.yml + .env
#      (in docker-compose.prod.yml, comment `build:` and uncomment `image: ghcr.io/erpc/erpc:main`).

# 2. Configure .env:
cp erpc.env.example .env
# REQUIRED now: ERPC_AUTH_SECRET — auth is ENABLED in erpc.yaml.
openssl rand -hex 32   # paste into .env
# Recommended: ENVIO_API_KEY (free, fast getLogs) and one of ALCHEMY_API_KEY /
# DRPC_API_KEY (archive depth + a trusted consensus vote on weak chains).

# 3. Sanity-check the config:
docker run --rm -v "$PWD/erpc.yaml:/erpc.yaml:ro" ghcr.io/erpc/erpc:main validate --config /erpc.yaml
#   (or, if you have the binary: ./erpc-server validate --config erpc.yaml)

# 4. Launch:
docker compose -f docker-compose.prod.yml up -d            # erpc + redis
docker compose -f docker-compose.prod.yml logs -f erpc     # watch boot (first request per chain warms the catalog)

# 5. Smoke test (same-host: no secret needed):
curl -s http://127.0.0.1:4000/main/evm/8453 \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'

# 6. Warm each chain once (first request per chain pays the catalog fetch):
for id in 1 8453 42161 10 56 137 100 43114 130 4663 59144 480 999 81457 5000 747474 204 33139 167000 250 25 1329 4326 143; do
  curl -s -m 20 http://127.0.0.1:4000/main/evm/$id -H 'content-type: application/json' \
    -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}' > /dev/null
done
curl -s http://127.0.0.1:4000/healthcheck   # every network should read OK

# Optional monitoring (Grafana :3000, Prometheus :9090):
docker compose -f docker-compose.prod.yml --profile monitoring up -d
```

## Auth (enabled — how clients authenticate)

`erpc.yaml` enables two strategies:

- **`network`**: a request with **no credential** passes ONLY from localhost (`allowLocalhost`)
  or an allowlisted IP (`allowedIPs`). On staging the same-host consumer reaches the gateway from
  the EC2's public IP via nginx, so that IP is allowlisted — it needs no secret and no change.
  Safe ONLY because `server.trustedIPForwarders` trusts the proxy hop and eRPC reads the real IP
  from `X-Real-IP` / `CF-Connecting-IP`.
  **Host-port callers:** a call to `127.0.0.1:4000` is SNAT'd by docker's userland proxy, so eRPC
  sees the bridge **gateway** (`172.22.0.1`), not `127.0.0.1` — `allowLocalhost` therefore does NOT
  cover it. That gateway address is allowlisted as a single host IP so on-host tools (aggregator,
  cron, probes) work **without a secret** — which matters because putting `?secret=` in a URL leaks
  it into job/verify records. This does not widen access: the port binds `127.0.0.1` only (verified:
  a container gets `Connection refused` on `172.18.0.1:4000`) and a container calling
  `erpc-gateway:4000` presents its own IP and is still rejected `401` (both verified 2026-07-21).
  **Never** widen this to a `/16` or `/24` — the box is multi-tenant and that would admit every container.
- **`secret`**: everyone else, three equivalent forms:
  - Basic-auth password in the URL (username ignored):
    `https://x:$ERPC_AUTH_SECRET@rpc.7kt.org/base` (public slug), or the chain-id form the
    **bounty tool** uses — set `ERPC_URL=https://x:$SECRET@rpc.7kt.org/evm` (nginx makes it
    `/main/evm`); the tool appends `/<chainId>` and needs no code change. (Heads-up: the tool's
    `redact()` does not strip userinfo — avoid pasting crash logs publicly.)
  - Header: `X-ERPC-Secret-Token: $SECRET`
  - Query: `?secret=$SECRET`

If `ERPC_AUTH_SECRET` is empty the gateway still boots but rejects every presented
credential (fail-closed). Rotate: change the env, `restart erpc`, update clients.

## Data integrity (the "don't trust one source" part)

Drain-decision reads (`eth_simulateV1`, `eth_call`, `eth_getCode`, `eth_getStorageAt`) that are
**pinned to an explicit block** run through **consensus**: eRPC fans out to 3 upstreams and returns
a value only if **2 agree**; a node that repeatedly disagrees is cordoned. On a
chain with <2 sim-capable endpoints it degrades to single-endpoint — add an Alchemy/dRPC
key to restore the cross-check. `latest`-tag reads are NOT consensus-checked (public nodes
differ by height). A dispute returns an error by design — clients must re-queue, never
fabricate (the bounty tool already treats `-32603` as transient).

## Persistence

`redis` holds the **finalized** cache (reorg-immune, permanent) so it survives restarts and
redeploys; recent/unfinalized reads stay in an in-memory tier. Redis uses `appendonly` + a
1 GB LRU cap (`redis_data` volume). Liveness reads (balance/deposit at `latest`) are never
cached.

## Reliability (cold-start & the catalog)

The `repository` (public) provider fetches its endpoint catalog from a remote URL and has **no
static fallback** — if that fetch fails, a chain served *only* by public endpoints gets no
upstream until the catalog is reachable (`ErrRemoteCacheCold`). The static upstreams in
`erpc.yaml` cover the six thinnest chains regardless of catalog health. To harden further:
- **Set at least one keyed provider** (`DRPC_API_KEY` is ideal — dRPC ships a built-in chain
  snapshot; Envio also ships one). A keyed provider gives every chain a catalog-independent
  upstream.
- **Warm each chain before serving traffic** (step 6 above).

## Ops

- **Update config:** edit `erpc.yaml`, commit, then `docker compose -f docker-compose.prod.yml restart erpc`.
- **Update eRPC:** `git pull` + `docker compose -f docker-compose.prod.yml up -d --build` (or repull the image).
- **Logs:** JSON, capped at 3×10 MB per container.
- **Health:** `GET http://127.0.0.1:4000/` and `GET /healthcheck` (per-network state);
  metrics at `:4001/metrics` (host-only).
