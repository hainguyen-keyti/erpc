# eRPC gateway — production deploy

A self-hosted eRPC gateway for **Ethereum, Base, BSC, Unichain, Monad** (mainnet + testnet),
on free public RPC, with **cross-check consensus** (don't trust one source) and a **persistent
Redis cache**. Built to front the AA-hunter scanner, but usable by any client.

## What's in this deploy

| File | Role |
|------|------|
| `erpc.yaml` | Gateway config (chains, cache, consensus, providers). **gitignored — local only.** |
| `docker-compose.prod.yml` | Stack: `erpc` + `redis` (+ optional `monitoring`). |
| `erpc.env.example` | API-key / secret template → copy to `.env`. |
| `Dockerfile` | Builds `erpc-server` (already in repo). |

## Chains served

| Chain | Mainnet | Testnet | Public sim support |
|-------|--------:|--------:|--------------------|
| Ethereum | `1` | Sepolia `11155111` | ✅ consensus (many endpoints) |
| Base | `8453` | Sepolia `84532` | ✅ consensus |
| BSC | `56` | testnet `97` | ✅ consensus |
| Unichain | `130` | Sepolia `1301` | ✅ consensus (mainnet: fewer) |
| Monad | `143` | testnet `10143` | testnet ✅ · **mainnet: 1 public endpoint → no cross-check** (add a key) |

URL shape: `http://<host>:4000/main/evm/<chainId>` — e.g. `/main/evm/8453` (Base).

## Deploy (on the server)

```bash
# 1. Get the files onto the server:
#    - Building from source: copy the whole repo (Dockerfile builds erpc-server).
#    - Using the published image: you only need erpc.yaml + docker-compose.prod.yml + .env
#      (in docker-compose.prod.yml, comment `build:` and uncomment `image: ghcr.io/erpc/erpc:main`).

# 2. Configure keys (all optional; blank = pure public RPC):
cp erpc.env.example .env
# edit .env — recommended: set ENVIO_API_KEY (free, fast getLogs) and one of
# ALCHEMY_API_KEY / DRPC_API_KEY (gives Monad-mainnet sim + a trusted consensus vote).

# 3. Sanity-check the config:
docker run --rm -v "$PWD/erpc.yaml:/erpc.yaml:ro" ghcr.io/erpc/erpc:main validate --config /erpc.yaml
#   (or, if you have the binary: ./erpc-server validate --config erpc.yaml)

# 4. Launch:
docker compose -f docker-compose.prod.yml up -d            # erpc + redis
docker compose -f docker-compose.prod.yml logs -f erpc     # watch boot (first request per chain warms the catalog)

# 5. Smoke test:
curl -s http://127.0.0.1:4000/main/evm/8453 \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'

# Optional monitoring (Grafana :3000, Prometheus :9090):
docker compose -f docker-compose.prod.yml --profile monitoring up -d
```

## Security

- **RPC (4000) and metrics (4001) bind to `127.0.0.1` on the host** — not reachable from the
  internet. Same-host clients (the scanner) use `http://127.0.0.1:4000/...`.
- **To expose remotely:** put a reverse proxy (Caddy/nginx) with **TLS** in front, AND enable
  `auth` in `erpc.yaml` (uncomment the `secret` strategy, set `ERPC_AUTH_SECRET`). Never bind
  4000 to `0.0.0.0` without auth — public endpoints have no rate-limit protection for you.

## Data integrity (the "don't trust one source" part)

Drain-decision reads (`eth_simulateV1`, `eth_call`, `eth_getCode`, `eth_getStorageAt`) that are
**pinned to an explicit block** run through **consensus**: eRPC fans out to 3 upstreams and returns
a value only if **2 agree** (grouped by hash); a node that repeatedly disagrees is cordoned. On a
chain with <2 sim-capable endpoints (e.g. Monad mainnet on pure public) it degrades to single-
endpoint — add an Alchemy/dRPC key to restore the cross-check. This is majority agreement, not
cryptographic proof. `latest`-tag reads are NOT consensus-checked (public nodes differ by height).

## Persistence

`redis` holds the **finalized** cache (reorg-immune, permanent) so it survives restarts and
redeploys; recent/unfinalized sims stay in an in-memory tier. Redis uses `appendonly` + a 1 GB
LRU cap (`redis_data` volume). Liveness reads (balance/deposit at `latest`) are never cached.

## Reliability (cold-start & the catalog)

The `repository` (public) provider fetches its endpoint catalog from a remote URL and has **no
static fallback** — if that fetch fails (network blip, air-gapped host), a chain served *only* by
public endpoints gets no upstream until the catalog is reachable (`ErrRemoteCacheCold`). To harden
a production deploy:
- **Set at least one keyed provider** (`DRPC_API_KEY` is ideal — dRPC ships a built-in chain
  snapshot, so it works even if its remote fetch fails; Envio also ships a 61-chain snapshot). A
  keyed provider gives every chain a catalog-independent upstream.
- **Warm each chain before serving traffic**: the first request per chain pays the catalog fetch
  (retried with backoff). Hit `/main/evm/<chainId>` once per chain after `up`.
- Envio fast-path note: **Unichain Sepolia (1301) is not in Envio's static chain map** (mainnet 130
  is), so its `eth_getLogs` uses the public endpoints, not HyperRPC — functional, just not the fast
  path. All other testnets (Sepolia, Base Sepolia, BSC testnet, Monad testnet) are covered.

## Ops

- **Update config:** edit `erpc.yaml`, then `docker compose -f docker-compose.prod.yml restart erpc`.
- **Update eRPC:** `git pull` + `docker compose -f docker-compose.prod.yml up -d --build` (or repull the image).
- **Logs:** JSON, capped at 3×10 MB per container.
- **Health:** `GET http://127.0.0.1:4000/` returns a health envelope; metrics at `:4001/metrics`.
