<div align="center">

# SuperNode

### Self-hosted multi-chain RPC aggregator in a single binary

One stable JSON-RPC endpoint over a self-healing pool of public nodes across **11 blockchains** — with smart routing, failover, circuit breaking and caching. No API keys, no Redis, no infrastructure. Just run it.

[![Release](https://img.shields.io/github/v/release/odrium-dev/odrium-supernode?label=release&color=2d9bf0)](https://github.com/odrium-dev/odrium-supernode/releases/latest)
[![Platforms](https://img.shields.io/badge/platforms-macOS%20%7C%20Linux-1f9f79)](https://github.com/odrium-dev/odrium-supernode/releases/latest)
[![Built with Go](https://img.shields.io/badge/built%20with-Go%201.23-00ADD8?logo=go&logoColor=white)](https://go.dev)
[![Dependencies](https://img.shields.io/badge/runtime%20deps-none-success)](#run-it-in-60-seconds)

[Download](https://github.com/odrium-dev/odrium-supernode/releases/latest) ·
[Quick start](#run-it-in-60-seconds) ·
[API](#api) ·
[Networks](#supported-networks)

</div>

---

## What is SuperNode

Public RPC nodes are free but unreliable — they rate-limit, lag the chain tip, and go offline without warning. SuperNode puts a smart layer in front of a pool of them: it health-checks every node continuously, routes each request to the fastest healthy upstream, fails over instantly when one dies, and caches hot responses. Your client talks to **one stable endpoint** and feels like it owns a local full node — for every chain at once.

It ships as a **single static binary**. Run it on your laptop or a $5 VPS and you have a working multi-chain RPC API in seconds.

## Supported networks

| | Symbol | Network | Endpoint |
|---|---|---|---|
| Bitcoin | `BTC` | Bitcoin | `/v1/bitcoin` |
| Ethereum | `ETH` | Ethereum | `/v1/ethereum` |
| BNB | `BNB` | BNB Smart Chain | `/v1/bsc` |
| Arbitrum | `ARB` | Arbitrum One | `/v1/arbitrum` |
| Solana | `SOL` | Solana | `/v1/solana` |
| XRP | `XRP` | XRP Ledger | `/v1/xrp` |
| Litecoin | `LTC` | Litecoin | `/v1/litecoin` |
| Tron | `TRX` | Tron | `/v1/tron` |
| TON | `TON` | TON | `/v1/ton` |
| Monero | `XMR` | Monero | `/v1/monero` |
| Zcash | `ZEC` | Zcash | `/v1/zcash` |

Live health of every node is shown on the built-in status page at `/`.

## Run it in 60 seconds

Download the binary for your platform from the [latest release](https://github.com/odrium-dev/odrium-supernode/releases/latest), make it executable, and run it. No Redis, no API keys, no config files.

**macOS (Apple Silicon)**
```bash
curl -L -o supernode https://github.com/odrium-dev/odrium-supernode/releases/download/v1.0.0/supernode_1.0.0_darwin_arm64
chmod +x supernode
xattr -d com.apple.quarantine supernode 2>/dev/null   # clear Gatekeeper quarantine
./supernode
```

**Linux (x86-64)**
```bash
curl -L -o supernode https://github.com/odrium-dev/odrium-supernode/releases/download/v1.0.0/supernode_1.0.0_linux_amd64
chmod +x supernode
./supernode
```

That's it. The aggregator starts and the API is live:

```
SuperNode 1.0.0
SuperNode running in STANDALONE mode (no Redis, keyless, public bind)
SuperNode gateway listening on 0.0.0.0:8080
```

Open `http://localhost:8080/` for the live network status page, or call the API directly:

```bash
curl -s localhost:8080/v1/ethereum \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber"}'
# {"jsonrpc":"2.0","id":1,"result":"0x1839a79"}
```

> Other builds (Intel macOS, Linux ARM64) and `SHA256SUMS` are on the [releases page](https://github.com/odrium-dev/odrium-supernode/releases/latest).
> Verify a download with `shasum -a 256 -c SHA256SUMS_1.0.0.txt`.

## Features

- **Smart routing & failover** — per-node EWMA latency/success scoring picks the best upstream and fails over instantly when one dies.
- **Circuit breaking** — flaky nodes trip a breaker (closed → open → half-open) and are dropped from routing until they recover.
- **Block-lag aware health** — a background checker probes every node concurrently and drops anything lagging the chain tip.
- **Two-tier caching** — sharded in-process L1, with single-flight de-duplication on the hot path (optional Redis L2 in cluster mode).
- **One stable endpoint** — transparent JSON-RPC proxy with batch support across all 11 chains.
- **Live status page** — real-time grid of supported coins and per-node health, streamed over SSE.
- **Zero dependencies** — a single static binary. Standalone mode needs no Redis, no keys, no config.

## API

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/{chain}` | Transparent JSON-RPC proxy (single or batch) |
| `GET` | `/v1/status` | Health snapshot of every node (JSON) |
| `GET` | `/v1/status/stream` | Live health feed (Server-Sent Events, 1 frame/sec) |
| `GET` | `/healthz` | Liveness probe |
| `GET` | `/` | Live status landing page |
| `GET` | `/console` | Operator control panel + JSON-RPC playground |
| `GET` | `/docs` | API documentation |

**Single call**
```bash
curl -s localhost:8080/v1/solana \
  -d '{"jsonrpc":"2.0","id":1,"method":"getSlot"}'
```

**Batch**
```bash
curl -s localhost:8080/v1/ethereum \
  -d '[{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber"},
       {"jsonrpc":"2.0","id":2,"method":"eth_gasPrice"}]'
```

**Live status**
```bash
curl -s localhost:8080/v1/status        # JSON snapshot
curl -N localhost:8080/v1/status/stream # live SSE feed
```

## Run modes

The binary auto-selects **standalone** mode — everything in-process, public bind, no keys. Override behavior with environment variables:

| Variable | Effect |
|---|---|
| `SUPERNODE_LISTEN` | Listen address (default `0.0.0.0:8080`), e.g. `:9000` |
| `SUPERNODE_STANDALONE=1` | Force standalone mode (no Redis, keyless, public bind) |
| `SUPERNODE_LOCAL=1` | Loopback-only sidecar mode (parent-watched) |
| `ADMIN_TOKEN=…` | Enable the `/admin` analytics console |

For multi-user production deployments, a Redis-backed **cluster** mode adds a distributed L2 cache, per-key rate limiting and API-key auth.

## Run as a service (Linux)

```ini
# /etc/systemd/system/supernode.service
[Unit]
Description=SuperNode RPC aggregator
After=network.target

[Service]
ExecStart=/usr/local/bin/supernode
Environment=SUPERNODE_LISTEN=0.0.0.0:8080
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now supernode
```

---

<div align="center">
Built by Odrium · <a href="https://github.com/odrium-dev/odrium-supernode/releases/latest">Download v1.0.0</a>
</div>
