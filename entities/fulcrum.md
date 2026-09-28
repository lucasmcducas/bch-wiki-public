
---
pageType: entity
entityType: software
id: entity.fulcrum
description: Fulcrum - high-performance Electrum Cash protocol server by cculianu. Indexes the BCH chain including CashTokens UTXOs. Required for the bot address/token queries since BCHN wallet subsystem does NOT track tokens.
sourceUrl: https://github.com/cculianu/Fulcrum
---

# Fulcrum

> Fulcrum — high-performance Electrum Cash protocol server by Calin Culianu (cculianu). Indexes the BCH chain including CashTokens UTXOs. Required for the bot's address/token queries since the BCHN wallet subsystem does NOT track tokens.

**Repository:** [github.com/cculianu/Fulcrum](https://github.com/cculianu/Fulcrum)
**Latest release:** **v2.1.2 (August 2026)**
**Protocols:** Electrum Cash v1.4.4+ (JSON-RPC over TCP / SSL / WS / WSS); admin RPC on TCP 8000
**Sources:** Fulcrum `doc/fulcrum-example-config.conf` (1100 lines, canonical config reference), `doc/unix-man-page.md`, FulcrumAdmin script source, Electrum Cash Protocol docs at electrum-cash-protocol.readthedocs.io

## Why Fulcrum matters to the bot

BCHN's built-in wallet subsystem (`bch-wallet`) does **not** index CashTokens — it tracks spendable BCH UTXOs only. The bot's queries about *which* UTXOs carry *which* FT/NFT category id (e.g. "what PUSD does this wallet hold?", "what Brainrot NFTs are at this address?") cannot be answered by BCHN RPC. Fulcrum's whole purpose is to maintain those indexes from a bitcoind-compatible daemon and serve them over the Electrum protocol. The moth pattern picks Fulcrum as the indexer layer for exactly this reason.

## Endpoints & versions

| Endpoint           | Purpose             | Default port | Notes                                  |
|--------------------|---------------------|--------------|----------------------------------------|
| TCP (non-SSL)      | Client RPC          | 50001        | Mainnet                               |
| SSL/TLS            | Client RPC          | 50002        | Mainnet                               |
| WS                 | Client RPC          | 50003        | Mainnet                               |
| WSS                | Client RPC          | 50004        | Mainnet                               |
| **Admin RPC**      | Operator commands   | **8000**     | Must bind to a **loopback / private** address — never expose to the internet |

Testnet uses 60001-60004 + 8000 on a separate daemon. The default testnet p2p port is unchanged.

The 2.x series is current. **2.x strongly recommended over 1.x.** 2.0.0 introduced a new database format with atomic-write guarantees — Fulcrum's DB can no longer get corrupted by a hard kill, signal, or power loss during sync. Migrating from 1.x requires running Fulcrum once against the old DB (`--db-upgrade`) or starting fresh. **2.1.2** is the latest as of August 2026; pre-compiled Linux binaries are attached to each release (Fulcrum-2.1.2-x86_64-linux.tar.gz).

The README / release notes do **not** pin a specific BCHN minimum version. They say Fulcrum has been "tested extensively" with BCHN, Bitcoin Cash Unlimited, Flowee, and bchd. The dsproof dependency on bitcoind RPC was established at v22.3.0 (so anything BCHN v22.3.0+ works); newer BCHN releases (2024–2026) have been spot-checked and work fine.

## Required bitcoind config

`txindex=1` is the **only** mandatory flag for Fulcrum's RPC. Strongly recommended:

- `-zmqpubhashtx=tcp://127.0.0.1:29001` — pushed transaction notifications; lets Fulcrum pick up new txs without waiting for block inclusion.
- `-zmqpubhashblock=tcp://127.0.0.1:29002` — new-block notifications.
- `rpcthreads=` matches or exceeds `max_connections` in `fulcrum.conf`. Defaults to 4 — increase to 16+ during initial block sync.
- `rpcworkqueue=` raised above default 1 to keep bitcoind responsive under Fulcrum's parallel queries.

**No pruning.** Fulcrum needs full chain history; the DB rebuild from a pruned chain will not work. Reserve **40–80 GB** for mainnet (growing), more if you're behind on pruning catchup. SSD is required — the random-read pattern during sync will pummel an HDD.

Memory footprint at idle: **1–4 GB RAM** (binary + LMDB + mempool), spike to 8 GB during initial sync. A small VPS or a dedicated machine is fine.

## CashTokens support across versions

| Version | Date       | CashTokens capability                                              |
|---------|------------|---------------------------------------------------------------------|
| 1.8.0   | May 2023   | Indexed CashTokens UTXOs; token-bearing txs included in history     |
| 1.9.0   | Late 2023  | Protocol v1.5.0 — `token_data` field returned in `blockchain.scripthash.listunspent`; `blockchain.scripthash.get_balance` accepts `token_filter`; `server.features()` advertises `cashtokens: true` |
| 2.0.0   | 2024       | New DB format, atomicity guarantees                                 |
| 2.1.2   | Aug 2026   | Latest; minor fixes                                                 |

If you see anything older than 1.9.0 in the wild, it does **not** surface `token_data` and the bot will be flying blind on token UTXOs.

## Methods the bot uses

From the Electrum Cash Protocol reference (`electrum-cash-protocol.readthedocs.io`, current at protocol v1.5.3 per the docs):

- `blockchain.scripthash.get_balance(scripthash[, token_filter])` — returns confirmed + unconfirmed `confirmed`/`unconfirmed` sats; with `token_filter` includes per-category fungible tokens. `token_filter` accepts `"include_tokens"`, `"exclude_tokens"`, or a list of specific category ids. Without the flag, default is `"include_tokens"` on protocol v1.5.0+ servers.
- `blockchain.scripthash.listunspent(scripthash[, token_filter])` — returns the full UTXO set as `[{ tx_hash, tx_pos, value, height, token_data? }, ...]`. **`token_data` is the canonical place tokens surface.** Each `token_data` payload contains `{ category_id, amount, nft? }`.
- `blockchain.scripthash.get_history(scripthash, from_height=0, to_height=-1)` — paginated history; token-touching txs are flagged (txs that *touch* any token UTXO, not just txs whose output is the focused scripthash). Added in 1.1; pagination in 1.5.1.
- `blockchain.scripthash.subscribe(scripthash)` — **this is the address-activity push notification method.** Returns current status; server pushes a notification when the status hash changes (new tx touching this script hash). The server sends `blockchain.scripthash.subscribe` *notifications* — these are JSON-RPC server-pushed messages; the bot must implement an event loop that subscribes and dispatches notifications. Subscription is implicit — server keeps tracking until client disconnects.
- `blockchain.transaction.get(tx_hash, verbose=true)` — full transaction as JSON.
- `blockchain.transaction.broadcast(raw_tx)` — submit; returns `tx_hash` on success or error string.
- `blockchain.headers.subscribe([raw_header])` — **new-block notification.** Returns chain tip header on subscribe; pushes new headers as they're found. This is the *only* "new block" method.
- `blockchain.address.get_first_use(address)` — first height where address appears (added 1.5.2).
- `blockchain.utxo.get_info(tx_hash, tx_pos)` — UTXO details; ignore confirmed-only booleans. Returns `token_data` since 1.9.0.
- `server.features()` — protocol-level handshake. Returns `{ genesis_hash, hash_function, server_version, protocol_min, protocol_max, cashtokens, dsproof, hosts, ...}`. **Check `cashtokens: true` at startup** to fail fast on servers without token support.
- `daemon.passthrough({ method, params })` — admin-mode escape hatch to BCHN RPC from inside Fulcrum. **Don't use this from a wallet bot** unless you really need it; it's an admin-side feature.

## Methods the bot does NOT use (or does not exist)

- **There is NO `server.subscribe()`.** This is a common confusion with BTC Electrum protocol. For new-block notifications on Electrum Cash, use `blockchain.headers.subscribe()`.
- **There is NO `blockchain.token.*` method.** There is no `blockchain.token.get_info`, no `token.subscribe`, no `token.list` — there is no protocol-level token registry. Tokens surface only via UTXO methods (`listunspent`, `get_balance` with `token_filter`). For token metadata (BCMR, CashScript) the bot reads them out-of-band (BCMR registry, NFT commitment content).
- `server.peers.subscribe()` — operator-side peer list; wallets ignore.
- `blockchain.transaction.dsproof.*` — double-spend proof methods. Available when `server.features().dsproof == true`; useful for merchant-side trust but not for a self-custodial wallet bot.

## Token-aware address events

The bot's standard "watch this address" pattern is:

1. Compute scripthash from the cashaddr: `sha256(lockingBytecode)` then reverse-bytes.
2. Call `blockchain.scripthash.subscribe(scripthash)`.
3. On status-change notification, call `blockchain.scripthash.listunspent(scripthash)` to get the current UTXO set; each UTXO has `token_data` if it carries any. Diff against the previous UTXO set; emit "received/sold/transferred" events.

The scripthash is computed client-side from the address's locking bytecode. Libauth has helpers: `cashAddressToLockingBytecode(addr)` then `hash160`-style on the bytecode (but note: Electrum's scripthash is **sha256 of the locking bytecode** in the script's standard (non-reversed) byte order).

## Admin RPC on port 8000

The `FulcrumAdmin` Python script (bundled in repo root, no extension — has shebang) speaks JSON-RPC over TCP. Useful commands:

- `getinfo` — version, uptime, peer count.
- `getclients` — connected clients with their subscribed scripthashes.
- `banlist` / `setban` — peer banning.
- `stop` — graceful shutdown.
- `sendrawtransaction` — alternate broadcast (same as client `blockchain.transaction.broadcast`).
- `broadcasttx` — alternate broadcast variant.

**Bind `admin=` to a loopback address or VPN interface only.** The README is emphatic: *NOT* to be exposed to the internet.

## Client library

The recommended Node.js client is **`electrum-cash`** (npm package by GeneralProtocols — the same group that builds Fulcrum):

- Latest: **v3.2.0** (Oct 5, 2023). 21 versions; mature. Stable; no recent commits but no open issues either.
- License: MIT.
- Dependencies: `ws`, `debug`, `@types/ws`, `async-mutex`, `lossless-json`. Pure-JS WebSocket and TCP client with protocol negotiation.
- Usage:
  ```js
  import { ElectrumCluster, ClusterOrder } from 'electrum-cash';
  const cluster = new ElectrumCluster('my-bot', '1.5.0', false, [ClusterOrder.RANDOM]);
  cluster.addServer('fulcrum.example.com', 50002, 'tls');
  await cluster.ready;
  const balance = await cluster.getBalance(scripthash);  // returns { confirmed, unconfirmed }
  ```
- The cluster abstraction handles multiple servers for failover; the bot can connect to self-hosted + a public fallback.

Alternatively, raw protocol over WebSocket (`ws://host:50003`) is workable and is what the moth bch-wallet skill uses (`@electrum-cash/protocol`).

## Pitfalls

1. **`server.subscribe()` does not exist** on Electrum Cash. The BTC Electrum `server.subscribe` is the old global subscription method and has no analogue here. Use `blockchain.headers.subscribe()` for new blocks and `blockchain.scripthash.subscribe(hash)` for address activity.
2. **`blockchain.token.get_info` does not exist.** Tokens appear only on UTXO-level responses. Build your token index from `listunspent`, not from a token-service RPC.
3. **`txindex=1` MUST be set on bitcoind.** Without it, getrawtransaction lookups for any tx not in the mempool or its block fail. Fulcrum will surface errors that may look like token-data corruption.
4. **DB corruption on 1.x**: Fulcrum's 1.x DB could be corrupted by a hard kill. The community advice ("DO NOT REBOOT OR STOP THE SERVICE DURING THE DB CREATION PROCESS") still applies for 2.x *during initial sync*; afterward 2.x's atomic format protects you. **Still: never `kill -9` Fulcrum mid-sync.**
5. **`blockchain.scripthash.subscribe` notifications are not guaranteed.** A missed notification means stale state. On reconnect, the bot MUST re-fetch the UTXO set via `listunspent` to catch up. Do not assume the notification alone is the source of truth.
6. **`server.features()` may omit `cashtokens: true`** on older Fulcrum instances (pre-1.9.0). Treat it as a soft signal — **always issue a real `listunspent` query with a known token-bearing UTXO test vector** to verify token indexing end-to-end on startup.
7. **Admin RPC port 8000 is unauthenticated.** Bind it to 127.0.0.1 or a tailnet only. `sudo lsof -i :8000` to verify.
8. **`max_connections` in fulcrum.conf** is the **client** connection cap; do not confuse it with BCHN's `maxconnections`. Default is 256 for Fulcrum.
9. **Memory growth during sync** can push RAM to 8 GB transiently. Allocate 12 GB to be safe on a small VPS during initial sync.
10. **Never trust third-party Fulcrum servers for client-side wallet work** if the bot is signing with private keys at that server. Fulcrum can serve malicious responses including a `token_data` field that doesn't match the chain; bot must verify on-chain or via signed oracle before trusting state for high-value decisions.

## Related entities

- `entity.bchn-node` — the BCHN node Fulcrum connects to via RPC and ZMQ.
- `entity.libauth` — for transaction construction from UTXO + token_data the Fulcrum layer returns.
- `entity.moth-bch-wallet` — example usage of `@electrum-cash/protocol` against a Fulcrum server.
- `entity.paryonusd` — the PUSD contract the bot will query via `token_filter` and `listunspent`.
