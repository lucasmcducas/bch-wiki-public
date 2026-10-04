# In-wallet swaps: ExchangeLab and the public indexer

How this wallet actually swaps tokens on Cauldron, and why it stopped using the
router.

## The short version

Pool state comes from `https://indexer.riften.net` (public, no auth, no key). The
transaction is assembled locally by `@cashlab/cauldron`. The wallet signs its own
input and broadcasts. There is no server in the middle, so there is nothing to
trust and nothing to go stale between a quote and a build.

This is the approach Paytaca — the reference wallet for this protocol — uses for
its own swaps. See [Paytaca's implementation](#paytaca-the-reference-implementation).

## The API

No key, no headers, no cookies. Verified live:

```
GET https://indexer.riften.net/cauldron/pool/active?token=<category>
GET https://indexer.riften.net/cauldron/tokens/search_cached?q=pusd
GET https://indexer.riften.net/cauldron/tokens/list_cached?limit=100
```

A pool record is small and complete:

```json
{
  "txid": "cfd9dd19ecb4d7d100ea3a91f547bc7fd6285d3965a504f053f4b08bf0f446b4",
  "tx_pos": 6,
  "sats": 5592828,
  "tokens": 1771,
  "owner_pkh": "3410fe7cb078ea8d56e07e2137199f5736148c75",
  "pool_id": "dc7c57bb7963b7ed2039150b1eef0d6f21f71f8d33a6aecfce5a23b2f5f2c302",
  "token_id": "2469acc5afa4b10cb5b5c04afb89c3a3ffd61c5da9c01e26d00951cae2a02544"
}
```

`owner_pkh` plus `pool_id` is enough to regenerate the covenant locking bytecode.
We never assemble it by hand — the SDK does, from those two fields.

PUSD = `2469acc5afa4b10cb5b5c04afb89c3a3ffd61c5da9c01e26d00951cae2a02544`,
60 active pools, 388.89 BCH against 122,448 PUSD.

## The SDK surface

`@cashlab/cauldron@1.0.3` exports a small, useful surface:

```js
ExchangeLab
buildPoolV0RedeemScriptBytecode
buildPoolV0UnlockingBytecode
extractInfoFromPoolV0UnlockingBytecode
```

`ExchangeLab` carries the whole AMM:

```
constructTradeBestRateForTargetSupply(sell, buy, amount, pools, feePerByte)
constructTradeBestRateForTargetDemand(...)
constructTradeAvailableAmountBelowTargetRate(...)
createTradeTx(entries, inputCoins, payoutRules, data, feePerByte)
verifyTradeTx(result)
generatePoolV0LockingBytecode({withdraw_pubkey_hash})
```

Five API facts, each learned by hitting it rather than by reading docs that
did not say it:

1. **`NATIVE_BCH_TOKEN_ID` is `'BCH'`, uppercase.** It must match the pool's
   `token_id` exactly. Passing the string `'bch'` throws *"either demand_token_id
   or supply_token_id should be NATIVE_BCH_TOKEN_ID"* — a message that reads like
   a logic error rather than a typo. Import the constant; never spell it.

2. **`createTradeTx` takes payout RULES, not outputs.** Hand-made output objects
   fail with *"Only one change payout_rule is required!"*. The SDK computes every
   amount itself. A `CHANGE` rule sweeps leftover BCH **and** tokens to one
   script; a `FIXED` rule pins an amount.

3. **`input_coins` must be all P2PKH.** `create-trade-tx.js:184` throws on
   anything else. Pool coins are *not* passed in — the builder derives the covenant
   inputs from `trade.entries` itself. That is the entire reason to pass the
   entries.

4. **libauth validates the private key at compile time.** A padded placeholder is
   rejected, even for an unsigned build. Same class of error as the NIP-59
   *"bad point"*: 32 arbitrary bytes are not a valid scalar.

5. **The result is already signed.** See below.

## The signing model — the part that surprises people

The SDK is handed `coin.key`, a **private** key, under a compiler entry named
`user_key`, and there is no `signTransaction` and no finalize step anywhere in
`create-trade-tx.js`. It unlocks the pool covenants itself.

Confirmed by building with a throwaway key and reading the unlocking bytecode:

```
in[0] v41:  69 bytes  44746376a914…   covenant unlock, SDK-generated
in[1] v44:  69 bytes  44746376a914…   covenant unlock, SDK-generated
in[2] v0:  100 bytes  4109c2d313ded…  65-byte DER signature push, ours
```

So `createTradeTx` returns a complete, signed transaction. There is no separate
signing step and no `signExternalTransaction` call in this path.

This means **private keys are handed to a third-party library.** Three things
bound that, and all three are load-bearing:

1. The key passed is the **child** key for the one address holding that UTXO, so
   a mishandled key leaks one input, never the seed.
2. No key is loaded until `--quote-only` has already returned, so the quote path
   touches no key material at all.
3. Nothing moves without `BCH_CONFIRM=yes`, so even a hostile library cannot
   broadcast without a human setting an environment variable.

## A real trade

```
[1/4] quote: 0.31 PUSD across 2 pool(s)
      pool 1: +53214 in -> 17 out
      pool 2: +43822 in -> 14 out
[2/4] 3 UTXO(s) available: 3 coin (1656311 sat)
[3/4] built 636 bytes, 2 payout(s), fee 636 sats (already signed)
      verified: pays 0.31 PUSD against a quote of 0.31
```

Decoded independently with our own libauth — not the SDK's nested one:

```
3 inputs, 4 outputs
  out[0] 16893468 sats + 5396 PUSD   new pool position
  out[1] 14770611 sats + 4719 PUSD   new pool position
  out[2]      651 sats +   31 PUSD   what the user receives
  out[3]   756988 sats                change, ours
```

## The guard that stayed

Ownership checks cannot catch *"right address, wrong amount."* A server that
quotes 36,141 and builds 1 pays a genuine address the user owns and passes every
ownership and pool-count test. The user signs it and receives nothing.

ExchangeLab removes the specific instance of that hazard — there is no server-side
quote to diverge from — but the *class* of defect moved rather than vanished. The
cheap way it comes back is for a caller to pass `buildSwap` a pool list the user
never approved a quote for.

So `verifyAgainstQuote` compares the **built** payout against the **quote**, and
refuses rather than warns. It is exercised by six assertions in
`test-swap-receive-value.mjs`, all of them refusals, because a guard nobody
exercises is a guard that has already decayed.

## Paytaca, the reference implementation

Paytaca does its own swaps in-wallet with no WizardConnect anywhere in the Cauldron
path:

```
src/wallet/cauldron/api.js          → https://indexer.riften.net
src/wallet/cauldron/transact.js     → ExchangeLab + calcTrade*FromAPair
src/wallet/cauldron/send.ts         → prepareSendWithCauldron, executeSendWithCauldron
src/wallet/cauldron/pool-tracker.ts → pool state
src/pages/apps/cauldron/trade.vue   → the UI
```

The indexer-to-SDK bridge is ~35 lines, in `src/wallet/cauldron/utils.js`
(`apiPoolToMicroPool`, `microPoolToPoolV0`). That is the whole integration.

Their route dialog shows something we had been throwing away: a per-exchange
breakdown, not just a pool count.

```
v-for="route in routeGroup.routes"   {{ route.exchange }}: {{ route.percentage }}%
```

## Why the router is gone

The router assembled the swap server-side and named the pool inputs. It named a
parent whose 56 outputs were all spent, on every quote, so the network rejected
every swap with *"Missing inputs"* — an error that reads like a malformed
transaction and is not one.

The liquidity was real; only the position had moved. Re-pointing the router's
inputs would have meant reimplementing the constant-product math, which is a
rewrite, not a fix. And that is what `@cashlab/cauldron` is.

Nothing was preserved "for safety". `verifyTransactionOutputs` was 348 lines and
looked like a net worth keeping. It was not: its arguments are
`expectedPoolCount` and `expectedReceiveAmount`, both taken from the router's own
quote, so its entire threat model is a server that quotes one thing and builds
another. With no server-side quote, that hazard does not exist.

The reasoning that *was* worth keeping is above, and the short-payment case is
now a real test.

## See also

- [[entities/riften-router]] — what the router was, and why it stopped working
- [[security/dex-swap-integration]] — the security model
- [[references/gates-that-pass-when-they-should-fail]] — measurement errors that
  masqueraded as drained liquidity
