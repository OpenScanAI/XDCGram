# Legacy Custom Chain Compatibility: Upgrading XDPoSChain from v2.6.1 to v2.8.2

**Repository:** `xdc-subnet`  
**Network:** Custom XDPoS private subnet, chain ID `7670`  
**Source release:** `XDPoSChain v2.6.1`  
**Target release:** `XDPoSChain v2.8.2`  
**Document version:** 1.0  
**Date:** 2026-07-02

---

## 1. Background

The `xdc-subnet` repository contains a reproducible 3-validator XDPoS private subnet (chain ID `7670`) deployed on a single host. It was originally generated with the `xinfinorg/subnet-generator` Docker image and ran under Docker using the XDPoSChain `v2.6.1` binary.

During a planned upgrade to the native `v2.8.2` binary, the subnet failed to import or produce blocks. The logs showed repeated `max fee per gas less than block base fee` errors on the BlockSigner (coinbase signer) transaction, even though the same configuration had worked on `v2.6.1`.

This document records the investigation, root cause, minimal fix, and validation that restored compatibility for legacy custom chains on `v2.8.2`.

---

## 2. Root Cause Analysis

### 2.1 The observable failure

When a `v2.8.2` validator started against the existing data directory and genesis, it repeatedly logged:

```text
WARN [...] Propagated block import failed
    err="max fee per gas less than block base fee: address xdc994e38673C40B1F21B087dAFd04AcBBD6bcd970B, maxFeePerGas: 0 baseFee: 12500000000"
```

The block at which the failure occurred had already been produced and accepted by `v2.6.1`. The failure prevented the `v2.8.2` node from staying in sync or participating in consensus.

### 2.2 How the BlockSigner transaction is handled

XDPoS uses a special zero-gas-price transaction called the **BlockSigner transaction** (or signing transaction) to sign each block. The account that sends this transaction is the block's coinbase. The transaction:

- Has `maxFeePerGas = 0`
- Is **not** a normal user transaction
- Must be routed through the `ApplySignTransaction` path rather than the standard `ApplyMessage` / `state_transition` path
- Must not be subjected to EIP-1559 base-fee validation

The `ApplySignTransaction` path is gated by `IsTIPSigning(num)` returning `true`. If `IsTIPSigning` returns `false`, the BlockSigner transaction is treated as an ordinary transaction, the EIP-1559 base-fee check is applied, and `maxFeePerGas < baseFee` fails the block.

### 2.3 Why `IsTIPSigning` returned `false` on v2.8.2

In `v2.6.1`, fork predicates in `params/config.go` used a dual fallback:

```go
func (c *ChainConfig) IsTIPSigning(num *big.Int) bool {
    return isForked(common.TIPSigningBlock, num) || isForked(c.TIPSigningBlock, num)
}
```

`common.TIPSigningBlock` was a **process-wide global** populated by `common.CopyConstants`. For a custom chain ID that was not mainnet, testnet, or devnet, `CopyConstants` copied values from `localConstant`, where every fork block was set to `0`. This made `IsTIPSigning` (and `IsTIPRandomize`, `IsBerlin`, `IsLondon`, `IsMerge`, `IsShanghai`, `IsEIP1559`, `IsCancun`) return `true` for every block on a legacy custom chain.

In `v2.8.2`, the global constants were removed. The fork predicates now evaluate only the stored `ChainConfig`:

```go
func (c *ChainConfig) IsTIPSigning(num *big.Int) bool {
    return isForked(c.TIPSigningBlock, num)
}
```

The genesis generator for this custom subnet stored `TIPSigningBlock`, `TIPRandomizeBlock`, `EIP1559Block`, `BerlinBlock`, `LondonBlock`, `MergeBlock`, `ShanghaiBlock`, and `CancunBlock` as `nil`. With `nil` fork blocks, every predicate returned `false` on `v2.8.2`, which broke the BlockSigner path, EIP-1559 semantics, and the selected EVM instruction set.

### 2.4 Why this matters for the whole EVM, not just the BlockSigner

Because `IsTIPSigning` was not the only predicate returning `false`, the EVM jump table also regressed. On `v2.6.1`, the chain effectively ran with `cancunInstructionSet` from block 0. On unpatched `v2.8.2`, it fell back to `frontierInstructionSet`, disabling Berlin, London, Merge, Shanghai, EIP-1559, and Cancun semantics. Even if the BlockSigner path had been fixed, state-root compatibility would still have been at risk.

---

## 3. Timeline of Investigation

| Step | Date | Activity | Outcome |
| ---- | ---- | -------- | ------- |
| 1 | 2026-07-01 | Reproduce failure by running `v2.8.2` validator against existing data | `max fee per gas less than block base fee` confirmed |
| 2 | 2026-07-01 | Compare `v2.6.1` and `v2.8.2` `params/config.go` / `params/config_forks.go` | Identified removal of global `common.XBlock` fallbacks |
| 3 | 2026-07-01 | Inspect `common.CopyConstants` and `localConstant` values | Confirmed all custom-chain fork blocks were historically `0` |
| 4 | 2026-07-01 | Trace `IsTIPSigning` → `ApplySignTransaction` → `state_transition` | Confirmed BlockSigner routing depends on `IsTIPSigning` |
| 5 | 2026-07-01 | Trace `IsEIP1559` → `jump_table` → `BASEFEE` / `PREVRANDAO` / `PUSH0` | Confirmed broader EVM regression without the fallbacks |
| 6 | 2026-07-01 | Draft `fork-predicate-compatibility-analysis.md` | Proposed `effectiveForkBlock` helper for all affected predicates |
| 7 | 2026-07-02 | Apply minimal patch in `params/config_forks.go` | All fork predicates restored for non-built-in custom chains |
| 8 | 2026-07-02 | Rebuild `XDC` binary at `/home/shalaka/xdposchain-v2.8.2/build/bin/XDC` | Binary reports `XDC/v2.8.2-testnet-f2d66480-20260623/linux-amd64/go1.26.0` |
| 9 | 2026-07-02 | Start Validator 1 with patched binary | No errors, but cannot reach consensus alone |
| 10 | 2026-07-02 | Update startup scripts for Validators 2 and 3 | All three validators run patched `v2.8.2` |
| 11 | 2026-07-02 | Three-validator validation | Peers = 2, block height increasing, no fee errors |

---

## 4. Blockers Encountered and Resolutions

### Blocker 1: `max fee per gas less than block base fee` on block import

**Cause:** `IsTIPSigning` returned `false` because `c.TIPSigningBlock` was `nil` and `v2.8.2` had no global fallback.

**Impact:** BlockSigner transactions were routed through the normal transaction path, which applied EIP-1559 base-fee validation. With `maxFeePerGas = 0` and `baseFee > 0`, every block import failed.

**Resolution:** Add an `effectiveForkBlock` helper in `params/config_forks.go` that falls back to `LocalnetChainConfig.XBlock` for non-built-in chain IDs when the stored `ChainConfig` field is `nil`. Apply it to `IsTIPSigning`, `IsTIPRandomize`, and the Ethereum fork predicates (`IsBerlin`, `IsLondon`, `IsMerge`, `IsShanghai`, `IsEIP1559`, `IsCancun`).

### Blocker 2: Single validator cannot prove consensus

**Cause:** XDPoS requires a quorum of masternodes to produce and commit blocks. With only Validator 1 running, `net_peerCount` was `0` and the chain head did not advance.

**Impact:** Validator 1 could be verified as stable and error-free, but block production, import, and BlockSigner routing could not be validated in isolation.

**Resolution:** Keep Validator 1 running, update `start-native-validator2.sh` and `start-native-validator3.sh` to use the patched `v2.8.2` binary, and start the remaining two validators. Once all three were connected, consensus resumed and the block height began to increase.

### Blocker 3: Default log level hides block-import success messages

**Cause:** The startup scripts use `--verbosity 2`, which only emits `WARN` and higher. Successful block import and commit messages are at lower levels.

**Impact:** It is not possible to read "Imported new chain segment" or "Committed block" messages directly from the log files. Validation must rely on RPC block-height polling and absence of errors.

**Resolution:** Use `eth_blockNumber` and `net_peerCount` RPC calls to verify live progress and log greps to confirm no `max fee per gas`, `invalid baseFee`, or FATAL errors.

---

## 5. Why the Original v2.8.2 Behavior Broke Legacy Custom Chains

| v2.6.1 behavior | v2.8.2 unpatched behavior |
| --------------- | ------------------------- |
| `common.CopyConstants` sets process-wide globals from `localConstant` for custom chains | Globals removed; predicates rely only on `ChainConfig` |
| All fork blocks effectively `0` for custom chain ID `7670` | Stored fork blocks are `nil` |
| `IsTIPSigning`, `IsTIPRandomize`, `IsEIP1559`, `IsBerlin`, `IsLondon`, `IsMerge`, `IsShanghai`, `IsCancun` all return `true` | All return `false` |
| BlockSigner routed correctly; EVM uses `cancunInstructionSet` | BlockSigner treated as regular transaction; EVM falls back to `frontierInstructionSet` |
| Blocks import and validate successfully | Block import fails with `max fee per gas less than block base fee` and potential state-root divergence |

The fundamental breaking change was the removal of the global fallback without updating the genesis/config generation tooling for legacy custom chains. Legacy chains that never wrote explicit fork blocks into `genesis.json` or `ChainConfig` became incompatible with `v2.8.2`.

---

## 6. Why the Implemented Compatibility Fix Works

The fix adds a helper inside `params/config_forks.go`:

```go
func (c *ChainConfig) effectiveForkBlock(field, localnetDefault *big.Int) *big.Int {
    if field != nil {
        return field
    }
    if c.ChainID != nil && !isKnownXDCBuiltInChainID(c.ChainID) {
        return localnetDefault
    }
    return nil
}
```

Each affected predicate is then changed from:

```go
func (c *ChainConfig) IsTIPSigning(num *big.Int) bool {
    return isForked(c.TIPSigningBlock, num)
}
```

to:

```go
func (c *ChainConfig) IsTIPSigning(num *big.Int) bool {
    return isForked(c.effectiveForkBlock(c.TIPSigningBlock, LocalnetChainConfig.TIPSigningBlock), num)
}
```

The same pattern is applied to `IsTIPRandomize`, `IsBerlin`, `IsLondon`, `IsMerge`, `IsShanghai`, `IsEIP1559`, and `IsCancun`.

### Why this is safe

- **Built-in networks are unaffected:** `isKnownXDCBuiltInChainID` returns `true` for mainnet, testnet, and devnet. Those networks continue to use their own fork blocks from the stored config.
- **Localnet is unaffected:** `LocalnetChainConfig` is already defined in `params/config.go` and is the canonical source of truth for a local/custom subnet.
- **No process-wide globals:** The fix is `ChainConfig`-local, matching the architectural direction of `v2.8.2`.
- **No changes to consensus, EVM, txpool, or state-transition logic:** The patch only changes the predicate evaluation, so it is the minimal possible scope.

---

## 7. Validation Methodology

1. **Build patched binary:**
   - Patch `params/config_forks.go` in `xdposchain-v2.8.2`.
   - Run `make XDC`.
   - Verify `XDC version` reports `v2.8.2`.

2. **Update validator scripts:**
   - Change `XDC_BIN` in `start-native-validator1.sh`, `start-native-validator2.sh`, and `start-native-validator3.sh` to point to `/home/shalaka/xdposchain-v2.8.2/build/bin/XDC`.

3. **Start Validator 1:**
   - Verify `web3_clientVersion` returns `v2.8.2`.
   - Verify no FATAL or `max fee per gas` errors in the log.
   - Confirm `net_peerCount` is `0` because no other validators are running.

4. **Start Validators 2 and 3:**
   - Confirm all three processes are running with the patched binary.
   - Confirm each validator reports `net_peerCount = 2`.
   - Confirm all three report the same `eth_blockNumber`.

5. **Monitor block production:**
   - Poll `eth_blockNumber` every 5–10 seconds for 2 minutes.
   - Verify continuous increase at ~2-second intervals.
   - Verify all validators remain in sync.

6. **Check for errors:**
   - Search all three `xdc.log` files for `max fee per gas less than block base fee`, `invalid baseFee`, and FATAL.
   - Confirm no new occurrences after the patched validators started.

---

## 8. Validation Results

### 8.1 Process state

| Validator | Binary | RPC port | P2P port | Status |
| --------- | ------ | -------- | -------- | ------ |
| Validator 1 | `/home/shalaka/xdposchain-v2.8.2/build/bin/XDC` | 8545 | 20303 | Running |
| Validator 2 | `/home/shalaka/xdposchain-v2.8.2/build/bin/XDC` | 8546 | 20304 | Running |
| Validator 3 | `/home/shalaka/xdposchain-v2.8.2/build/bin/XDC` | 8547 | 20305 | Running |

### 8.2 Peer connectivity

All three validators reported:

```json
{"jsonrpc":"2.0","id":1,"result":"0x2"}
```

for `net_peerCount`, meaning each validator saw the other two.

### 8.3 Block height progression

Sample observation on Validator 1 over 120 seconds:

| Time | Block height (hex) | Block height (decimal) |
| ---- | ------------------ | ---------------------- |
| T+0s  | `0xfea9` | 65193 |
| T+10s | `0xfeae` | 65198 |
| T+20s | `0xfeb3` | 65203 |
| T+30s | `0xfeb8` | 65208 |
| T+40s | `0xfebe` | 65214 |
| T+50s | `0xfec3` | 65219 |
| T+60s | `0xfec8` | 65224 |
| T+70s | `0xfecd` | 65229 |
| T+80s | `0xfed2` | 65234 |
| T+90s | `0xfed7` | 65239 |
| T+100s | `0xfedd` | 65245 |
| T+110s | `0xfee2` | 65250 |
| T+120s | `0xfee7` | 65255 |

- **Delta:** 62 blocks in 120 seconds
- **Average block time:** ~1.94 seconds (matches the XDPoS 2-second target)
- All three validators remained at the same block height throughout the observation.

### 8.4 Error checks

Searches across all three `xdc.log` files for the target strings after the patched validators started:

| Error string | Occurrences after patch start | Result |
| ------------ | ----------------------------- | ------ |
| `max fee per gas less than block base fee` | 0 | ✅ Pass |
| `invalid baseFee` | 0 | ✅ Pass |
| `FATAL` | 0 | ✅ Pass |
| New `ERROR` (excluding `db` RPC module warning) | 0 | ✅ Pass |

The only warnings observed were:
- The pre-existing `Unavailable modules in HTTP API list` message for `db` (harmless; the `db` module is not available in the HTTP API set).
- Periodic `[sendTimeout]` messages, which are expected in a 3-validator XDPoS network when a validator is slow to respond.

### 8.5 Consensus stability

The network produced blocks continuously for several minutes without reverting to the pre-patch failure mode. No new consensus or compatibility blockers appeared.

---

## 9. Comparison Table: v2.6.1 vs v2.8.2

| Aspect | v2.6.1 | v2.8.2 unpatched | v2.8.2 patched (this fix) |
| ------ | ------ | ---------------- | -------------------------- |
| **Runtime configuration mechanism** | `common.CopyConstants` sets process-wide globals at startup based on chain ID | Fork blocks read only from stored `ChainConfig` | Fork blocks read from `ChainConfig`; fallback to `LocalnetChainConfig` for non-built-in custom chains |
| **`common.CopyConstants`** | Used; copies `localConstant` values for custom chains | Removed | Not needed; per-chain-config fallback replaces it |
| **Fork activation** | All Ethereum and XDC-specific forks effectively active at block 0 for custom chains | No forks active if stored fork blocks are `nil` | Same effective behavior as v2.6.1 for custom chains; built-ins unchanged |
| **EIP-1559 handling** | Active (`IsEIP1559` returns `true`) | Inactive if `EIP1559Block` is `nil` | Active for custom chains |
| **TIPSigning/TIPRandomize behavior** | Active (`IsTIPSigning`/`IsTIPRandomize` return `true`) | Inactive if `TIPSigningBlock`/`TIPRandomizeBlock` are `nil` | Active for custom chains |
| **BlockSigner routing** | Routed through `ApplySignTransaction`, bypassing base-fee validation | Treated as normal transaction; fails EIP-1559 base-fee check | Correctly routed, bypassing base-fee validation |
| **BaseFee validation** | Applied to user transactions, not BlockSigner | Applied to BlockSigner incorrectly, causing block import failure | Applied only to user transactions |
| **EVM instruction set** | `cancunInstructionSet` from block 0 | `frontierInstructionSet` if all predicates false | `cancunInstructionSet` from block 0 for custom chains |
| **Legacy custom-chain compatibility** | Works | Broken | Restored |
| **Built-in network behavior** | Mainnet/testnet/devnet use their own configs | Mainnet/testnet/devnet use their own configs | Unchanged |
| **Files involved in the fix** | `common/constants*.go`, `params/config.go` | `params/config_forks.go` | `params/config_forks.go` only |
| **Pros of implementation** | Simple global model | Clean, no globals | Clean, no globals; minimal scope; preserves built-in behavior |
| **Cons of implementation** | Globals are error-prone and hard to test | Breaks legacy custom chains | Requires an explicit fallback helper; new custom chains should set fork blocks in genesis |

---

## 10. Flow Diagrams

### 10.1 Fork predicate evaluation (patched v2.8.2)

```text
┌─────────────────┐
│  IsTIPSigning   │
│  (or IsEIP1559  │
│   etc.)         │
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│ c.TIPSigningBlock nil?  │
│ (or other fork block)    │
└────────┬────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌──────┐  ┌──────────────────────────────┐
│ No   │  │ Yes                            │
│      │  │                                │
│ use  │  │ ChainID known built-in?      │
│stored│  │ (mainnet/testnet/devnet)      │
│block │  └────────┬───────────────────────┘
│      │           │
│      │      ┌────┴────┐
│      │      ▼         ▼
│      │  ┌──────┐  ┌──────────────┐
│      │  │ Yes  │  │ No           │
│      │  │      │  │              │
│      │  │return│  │ use Localnet │
│      │  │ nil  │  │ ChainConfig  │
│      │  │      │  │ default      │
│      │  └──┬───┘  │              │
│      │     │      └──────┬───────┘
│      │     │             │
│      │     └──────┬──────┘
│      │            │
│      └────────────┘
│                   │
                   ▼
            ┌─────────────┐
            │ isForked()  │
            │ evaluates   │
            │ active fork │
            └─────────────┘
```

### 10.2 BlockSigner transaction flow

```text
Block received by validator
         │
         ▼
┌─────────────────────┐
│ IsTIPSigning(block) │
│    returns true?      │
└──────────┬────────────┘
           │
      ┌────┴────┐
      ▼         ▼
┌────────┐  ┌──────────────┐
│  Yes   │  │      No      │
│        │  │ (unpatched)  │
└───┬────┘  └──────┬───────┘
    │              │
    ▼              ▼
┌────────────────┐  ┌─────────────────────────┐
│ ApplySignTx()  │  │ ApplyMessage()          │
│ - gas price 0  │  │ - standard EIP-1559     │
│ - no base-fee  │  │   validation            │
│   validation   │  │ - maxFeePerGas = 0      │
│ - accepted     │  │   < baseFee             │
└────────────────┘  │ - BLOCK IMPORT FAILS    │
                   └─────────────────────────┘
```

### 10.3 Validation workflow

```text
Patch params/config_forks.go
         │
         ▼
Rebuild XDC v2.8.2
         │
         ▼
Update start-native-validator*.sh
         │
         ▼
Start Validator 1
         │
         ▼
Check: version, no errors, peer count 0
         │
         ▼
Start Validators 2 & 3
         │
         ▼
Check: all peers connected, block height in sync
         │
         ▼
Poll eth_blockNumber for 2+ minutes
         │
         ▼
Verify: continuous block production, no fee errors
```

---

## 11. Remaining Limitations and Future Considerations

### 11.1 Future custom chains should set fork blocks explicitly

The `effectiveForkBlock` fallback restores compatibility for **legacy** custom chains that were created before `v2.8.2`. New custom chains should ideally write the intended fork blocks directly into `genesis.json` and `ChainConfig`, rather than relying on the fallback. This makes the configuration explicit and easier to reason about.

### 11.2 `common.CopyConstants` is gone

`v2.8.2` removed the `common.CopyConstants` mechanism entirely. Any tooling, scripts, or documentation that relied on it must be updated. The fallback helper is the replacement, but it lives in `params/config_forks.go`, not in `common`.

### 11.3 This fix does not address unrelated v2.8.2 changes

The patch only restores fork-predicate behavior. Other behavioral changes between `v2.6.1` and `v2.8.2` (e.g., gas-limit logic, RPC changes, transaction-pool rules, P2P protocol changes) are not covered. Long-running validation on a non-trivial workload is recommended before any production upgrade.

### 11.4 Log verbosity limits observability

At `--verbosity 2`, successful block import and consensus commit messages are not written to the log. For deeper troubleshooting, consider raising verbosity temporarily, but be aware that `v2.8.2` can produce a large volume of logs at higher levels.

### 11.5 The fallback is scoped to non-built-in chain IDs

If a custom chain was created with a chain ID that collides with a known built-in ID (mainnet, testnet, devnet), the fallback will not apply. This is by design and should not affect properly generated subnets.

---

## 12. Related Documents

- `fork-predicate-compatibility-analysis.md` — detailed predicate-by-predicate analysis that led to the full fix.
- `MIGRATION.md` — Docker-to-native migration guide for this subnet.
- `SUBNET_ANALYSIS.md` — analysis of the subnet configuration and version choices.
- `MANUAL_DEPLOYMENT_PLAN.md` — full manual deployment plan.
- `generated/README.md` — generated documentation for the Docker subnet.

---

## 13. Conclusion

The `v2.6.1` → `v2.8.2` upgrade broke this legacy custom subnet because `v2.8.2` removed the process-wide fork-block globals that custom chains relied on. The implemented `effectiveForkBlock` fallback in `params/config_forks.go` restores the `v2.6.1` runtime behavior for non-built-in custom chains without affecting mainnet, testnet, or devnet.

Validation on the live 3-validator subnet confirmed:

- All three validators run the patched `v2.8.2` binary.
- All three validators connect to each other.
- Block height increases continuously at ~2-second intervals.
- All validators remain in sync.
- No `max fee per gas less than block base fee` errors occur.
- No `invalid baseFee` or fatal errors occur.

The compatibility fix is working as intended.
