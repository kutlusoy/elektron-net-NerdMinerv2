- **Version:** 0.1 (draft)
- **Date:** August 30, 2026
- **Audience:** Firmware maintainers of this fork, pool operators integrating hobby-miner firmware
- **Reference implementation:** [`elektron-net`](https://github.com/kutlusoy/elektron-net) — `src/validation.cpp` (`ValidateUTXOCheckpoint`, `ComputeBlockUTXOAttestationHash`), `src/node/miner.cpp` (`CreateNewBlock`) — treat as ground truth for anything this doc references
- **Fork base:** [`BitMaker-hub/NerdMiner_v2`](https://github.com/BitMaker-hub/NerdMiner_v2)
- **Consumer:** [`elektron-net-ppool`](https://github.com/kutlusoy/elektron-net-ppool)
- **See also:** `doc-elektron/mining-pool-integration.md` §3.5, §3.6, §3.7 in `elektron-net`; `elektron-net-ppool/src/models/stratum-messages/SubscriptionMessage.ts`; `elektron-net-ppool/src/models/stratum.constants.ts`

---

## 1. Problem

Workers running this firmware against `elektron-net-ppool` connect successfully (subscribe, authorize, receive jobs) but every submitted share/block is rejected by the node with `bad-utxo-attestation`.

## 2. Root Cause

Elektron Net commits a UTXO-set hash into the coinbase of every block (see `ComputeBlockUTXOAttestationHash` in `elektron-net`). That hash is computed by the node against a coinbase whose `scriptSig` is exactly `coinbase_script_sig_prefix` from `getblocktemplate`, nothing appended. Any byte spliced into `scriptSig` after template creation changes the coinbase txid and invalidates the commitment.

To keep older Stratum v1 firmware connected without breaking this rule, `elektron-net-ppool` decouples the two extranonce slots (`stratum.constants.ts`, `SubscriptionMessage.ts`):

- `extranonce1` in the subscribe response: non-empty per-session hex string, sent only so hobby-miner firmware does not abort on an empty value. The pool never splices it into the coinbase it submits.
- `extranonce2_size`: always `0`. The pool expects workers to iterate nothing and treat `coinb1` as the complete coinbase (`coinb2` is always the empty string).

This firmware does not respect that contract. In `src/utils.cpp`, `calculateMiningData()`:

```cpp
// lines 216-226 (before fix)
if (mWorker.extranonce2_size == 2)
    mWorker.extranonce2 = "0001";
else if (mWorker.extranonce2_size == 4)
    mWorker.extranonce2 = "00000001";
else if (mWorker.extranonce2_size == 8)
    mWorker.extranonce2 = "0000000000000001";
else
{
    Serial.println("Unknown extranonce2");
    mWorker.extranonce2 = "00000001";   // extranonce2_size == 0 falls through here
}
```

and further down (lines 232-234), the coinbase is always built as:

```cpp
snprintf(coinbase_buffer, sizeof(coinbase_buffer), "%s%s%s%s",
         mJob.coinb1.c_str(), mWorker.extranonce1.c_str(),
         mWorker.extranonce2.c_str(), mJob.coinb2.c_str());
```

With `extranonce2_size == 0`, the correct behavior is `coinbase = coinb1 + coinb2` (both extranonce slots contribute zero bytes). Instead this firmware always appends the non-empty `extranonce1` session label plus a hardcoded 4-byte `extranonce2` ("00000001"), so the coinbase it hashes and submits diverges from `coinbase_script_sig_prefix`. The connection stays open (the `stratum.cpp:78-83` empty-`extranonce1` abort check is satisfied), but the resulting coinbase txid, and therefore the UTXO attestation hash, no longer matches what the node committed to at template time.

## 3. Fix

Two changes in `src/utils.cpp`, `calculateMiningData()`:

1. Handle `extranonce2_size == 0` explicitly instead of falling into the `else` branch:

```cpp
if (mWorker.extranonce2_size == 0) {
    mWorker.extranonce2 = "";
} else if (mWorker.extranonce2_size == 2)
    mWorker.extranonce2 = "0001";
else if (mWorker.extranonce2_size == 4)
    mWorker.extranonce2 = "00000001";
else if (mWorker.extranonce2_size == 8)
    mWorker.extranonce2 = "0000000000000001";
else {
    Serial.println("Unknown extranonce2");
    mWorker.extranonce2 = "00000001";
}
```

2. Skip splicing `extranonce1` as well when `extranonce2_size == 0`, since on Elektron Net it is a wire-only session label, not coinbase material:

```cpp
if (mWorker.extranonce2_size == 0) {
    snprintf(coinbase_buffer, sizeof(coinbase_buffer), "%s%s",
             mJob.coinb1.c_str(), mJob.coinb2.c_str());
} else {
    snprintf(coinbase_buffer, sizeof(coinbase_buffer), "%s%s%s%s",
             mJob.coinb1.c_str(), mWorker.extranonce1.c_str(),
             mWorker.extranonce2.c_str(), mJob.coinb2.c_str());
}
```

Change (1) alone is not sufficient: `extranonce1` still needs to be excluded from the coinbase build, or the pool's non-empty session label leaks into `scriptSig` and the attestation still mismatches.

Behavior against a classic Stratum v1 pool (non-zero `extranonce2_size`) is unchanged, since that path still goes through the original four-branch splice.

## 4. Checklist

- [ ] Apply both changes to `src/utils.cpp`
- [ ] Build and flash against `elektron-net-ppool` in hobby mode (non-empty `extranonce1`, `extranonce2_size = 0`)
- [ ] Confirm `submitblock` no longer returns `bad-utxo-attestation` for this worker
- [ ] Confirm classic Stratum v1 behavior (non-zero `extranonce2_size`, e.g. against a Bitcoin-style test pool) is unaffected
- [ ] Point `getHeightAPI`, `getGlobalHash`, `getFees` in `src/monitor.h` at the operator's own `elektron-net-mempool` instance instead of `mempool.space`

## 5. Open Questions

1. `getBTCAPI` in `src/monitor.h` currently queries Coingecko for BTC/USD. Sourcing an ELEK price from `elektron-net-mempool` instead requires that backend to have its own ELEK price feed configured; this is separate work in the `elektron-net-mempool` repo, not covered by this fix.