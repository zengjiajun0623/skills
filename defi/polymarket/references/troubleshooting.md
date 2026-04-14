# Polymarket Skill Troubleshooting

Real gotchas encountered during end-to-end validation. Each section: symptom → root cause → fix.

---

## 1. `403 Trading restricted in your region` on order POST

**Symptom:**
```json
{"error":"Status: error(403 Forbidden) making POST call to /order with {\"error\":\"Trading restricted in your region, please refer to available regions - https://docs.polymarket.com/developers/CLOB/geoblock\"}"}
```

**What's happening:** Polymarket's CLOB API runs its own geo-detection at the POST endpoint, on top of (and sometimes inconsistent with) the country check reported by `clob geoblock`. We hit this from a Hong Kong VPN exit IP: `clob geoblock` reported `Blocked: false`, but the first `POST /order` returned 403. Retry ~30 seconds later succeeded with no change on our end. Platform heuristics involve VPN-exit detection beyond the simple country check.

**Fix:**
- **Retry exactly once** after 30 seconds. Do not retry silently beyond that.
- If it continues to fail, treat it as a hard geo-block and abort.
- **Never circumvent** — this is a platform rule. Tell the user it's restricted, suggest an IP/VPN change is their decision not the skill's.
- The skill command `elytro pm order` does not currently retry automatically. The skill's agent layer is responsible for the 30-second retry. If the command fails with a 403, wait 30 seconds, try exactly once more, then bail.

**Related:** `elytro pm info` → checks `geoblock` endpoint which may disagree with the order endpoint. Trust the order endpoint's verdict.

---

## 2. `invalid amount for a marketable BUY order ($X), min size: $1`

**Symptom:** Polymarket rejects a BUY order where the notional is below $1 USDC.e.

**Root cause:** Polymarket enforces a **$1 minimum notional** on marketable (taking-liquidity) orders. The check uses **limit price × size**, not expected fill price.

**Fixes:**
- **Easy:** raise `--size` so that `size × price ≥ 1`. For a $0.12 market: `--size 10` makes notional $1.20.
- **Trick:** place a FAK BUY at a limit price *above* the current ask, e.g. `--price 0.20 --size 5` = $1.00 notional check, but the matcher fills at the market ask (~$0.12) → ~8 actual shares for $1.00. See "Order fills more shares than requested" below.
- **For SELLs:** same $1 rule applies. If you're closing a position worth less than $1 at the bid, you cannot sell back via CLOB. See "CTF residuals" below.

---

## 3. Order fills MORE shares than `--size`

**Symptom:** You request `--size 10` on a FAK BUY, but your on-chain CTF position balance shows ~16 shares after settlement.

**Root cause:** Polymarket's FAK order type uses `makerAmount = price × size` as a **USDC budget**, not a strict share cap. When the order is matched, the matcher fills up to `makerAmount` of USDC spent, and if the fills clear at prices below your limit, you receive proportionally more shares.

Example:
- `--price 0.20 --size 10` → `makerAmount = 2.00 USDC` (your cost ceiling)
- Best ask is $0.12 with 10075 shares available
- Matcher takes 16.666 shares at $0.12 = exactly $2.00 USDC spent
- You get 16.666 shares for your $2.00

This is **by design**, not a bug. It's also the mechanism that lets the "limit above market to satisfy $1 notional" trick work.

**Fixes / choose the right tool:**
- For a **hard share count**, use `--type FOK` (Fill or Kill) which cancels if it can't fill exactly.
- For a **hard USDC budget**, use FAK with a carefully chosen limit price and accept that you'll get ≥ size shares.
- For **marketable intent** ("buy as many shares as I can for $X"), set `price = high limit`, `size = ceil($X / limit_price)`. The matcher will spend up to $X.

---

## 4. CTF residuals below the 5-share minimum

**Symptom:** After closing a position, `elytro pm --subkey X balance --token-id <id>` shows a small residual (e.g. `0.666` shares). Attempting to sell this residual returns `invalid order size, minimum: 5`.

**Root cause:** Every Polymarket market has an `orderMinSize` field (usually 5). **`orderMinSize` is a minimum, not a granularity** — a SELL with `size = 16.666` is perfectly valid as long as `16.666 ≥ 5`. The stuck case is only when your *total* balance on that token is strictly below 5 shares.

**Categorize the residual by balance size, not by whether it's fractional:**

### Case A — balance ≥ 5 shares (even if fractional)

**You are not stuck.** Sell all of it in one order. The `orderMinSize` check passes because `size ≥ 5`; the only other rule to clear is the $1 notional floor (`size × price ≥ 1`).

Example: you hold `16.666` shares after a FAK BUY that spent the full `price × size` budget at a lower fill price. You can SELL all `16.666` in one order:

```bash
elytro pm --subkey demo order \
  --token <yes_token_id> \
  --side SELL --price 0.10 --size 16.666 --type FOK
# 16.666 ≥ 5 (orderMinSize ✓) and 16.666 × 0.10 = 1.67 (notional ✓) → valid
```

The earlier "buy more to reach a multiple of 5" advice was wrong — it treats `orderMinSize` as a granularity, which it is not. You only need to reach the minimum *once*. Fractional balances above the minimum close cleanly with a single order.

### Case B — balance < 5 shares (actually stuck)

This is the real dust case. A SELL with `size < 5` is rejected regardless of the notional value. Three exits:

1. **Aggregate up:** BUY enough additional shares to push the total above 5, then SELL everything in one order. Example: holding 2 shares, BUY 3 more to reach 5, SELL 5. Only economic if the residual's resolution value exceeds the round-trip cost (new BUY spread + new SELL spread + gas).

2. **Wait for resolution and redeem via the CTF contract.** Post-resolution redemption has no share-count minimum — it pays out pro-rata USDC based on the resolved outcome. Call `redeemPositions` directly on the ConditionalTokens contract:

   ```bash
   # CTF ConditionalTokens on Polygon
   CTF=0x4D97DCd97eC945f40cF65F87097ACe5EA0476045
   # Polymarket collateral (USDC.e on Polygon)
   USDCE=0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174
   # parent collection is bytes32(0) for top-level binary markets
   PARENT=0x0000000000000000000000000000000000000000000000000000000000000000
   # conditionId from `elytro pm market <slug>` (same one you used to get the token_id)
   COND=0x813451b970c8aae23a9b2441fd9be8367e8691a636323ced76f6fd1914c1e0a8
   # indexSets: [1] redeems the YES branch, [2] the NO branch, [1,2] both
   # Pass an ABI-encoded uint256[] — cast takes "[1]" for a single-element array.
   cast send $CTF \
     "redeemPositions(address,bytes32,bytes32,uint256[])" \
     $USDCE $PARENT $COND "[1]" \
     --private-key <subkey_priv_key> --rpc-url https://polygon-bor-rpc.publicnode.com
   ```

   **All four arguments are required** — leaving any out fails the selector match. `collateralToken` is USDC.e (not native USDC, not any other address); `parentCollectionId` is `bytes32(0)` for top-level markets (Polymarket does not use nested conditions); `conditionId` comes from the market metadata; `indexSets` selects which outcome branch you own. Elytro does not yet expose a redeem command — this `cast send` is the supported fallback.

3. **Accept the loss.** For a 0.666-share × $0.12 residual (~$0.08 value), the gas + spread to execute either option 1 or option 2 will likely exceed the recovery. Document the loss and move on.

### Cleanup rule

**Never `subkey remove` while ANY non-zero CTF balance exists on the subkey**, regardless of whether it's above or below the 5-share minimum. The command warns but does not block; removing the key makes those shares permanently unrecoverable from this vault. Always check `elytro pm --subkey <label> balance --token-id <id>` for every token you've touched in the session before removal.

---

## 5. Sweep: native POL transfer to smart account reverts with default gas stipend

**Symptom:** A plain EOA-to-smart-account native value send with the default `gas: 21000n` reverts. Receipt shows `status: 0 (failed)`, no value transferred.

**Root cause:** ERC-4337 smart-account proxies route plain value calls through a `delegatecall` into the implementation's `receive()` / fallback. On an Elytro account (ERC-1967 proxy + typical implementation), that path consumes ~26000 gas — roughly SLOAD(2100) + DELEGATECALL(2600) + implementation receive handling. 21000 gas is insufficient and the outer delegatecall runs out before `receive()` can complete.

**Fix applied in the skill:** `subkey sweep` probes the exact cost via `eth_estimateGas` before sending and uses the larger of `estimatedGas × 1.5` or `40000` as the tx gas limit. Rough guide for Elytro accounts:

| Path | Typical gas needed | Sweep uses |
|---|---|---|
| plain EOA → EOA | 21000 | 40000 floor |
| plain EOA → Elytro proxy (receive) | ~26000 | max(estimate × 1.5, 40000) |

If `eth_estimateGas` reverts during the probe, the sweep aborts with a clear error — this signals that the smart account's implementation lacks a payable `receive()`, which would be an upstream Elytro bug, not a skill issue.

**Before this fix:** earlier versions routed native POL to the smart account's owner EOA instead of the smart account, as a workaround. That created an unprotected "owner gas tank" that had to be refilled via a direct owner-EOA → subkey path, bypassing SecurityHook — violating the architectural invariant. The gas-probe fix lets the native leg go where it belongs (the smart account) and eliminates the need for any owner-EOA bypass.

---

## 6. viem over-reserves gas on Polygon → `insufficient funds for transfer`

**Symptom:** When sending direct EOA transactions via viem on Polygon, the `sendTransaction` call throws `The total cost (gas * gas fee + value) exceeds the balance of the account`, even though `cast estimate` shows a gas cost ~100x smaller than viem's reservation.

**Root cause:** viem's automatic fee estimation goes through `eth_feeHistory` to compute `maxFeePerGas` and multiplies by gas limit. On Polygon, `eth_feeHistory` returns very conservative values because priority fees are volatile. The product can exceed available balance.

**Fix (applied in the skill):** Set explicit `gasPrice` and `gas` on every direct-EOA send. Pattern:

```typescript
const rawGasPrice = await walletClient.request({ method: 'eth_gasPrice' });
const gasPrice = (BigInt(rawGasPrice as string) * 120n) / 100n;  // 20% buffer

await walletClient.sendTransaction({
  to,
  data,
  gas: 80000n,     // ERC-20 transfer
  gasPrice,        // legacy tx type
});
```

This uses `eth_gasPrice` (single number, not a distribution) and avoids viem's feeHistory-based maxFee computation entirely. The resulting tx is a legacy (type 0) transaction, which is fine on Polygon.

---

## 7. Two back-to-back sends collide on the same nonce

**Symptom:** First `sendTransaction` returns a hash; second returns a hash too; but the second tx is "not found" on chain when queried later. The network dropped it.

**Root cause:** Without explicit nonces, viem's `sendTransaction` reads the nonce via `eth_getTransactionCount` fresh for each call. If called back-to-back faster than the first tx lands in a block, both calls fetch the same nonce → the second tx has a duplicate nonce → silently dropped from the mempool.

**Fix (applied in the skill's `pm approve` command):** fetch the starting nonce once, then increment manually:

```typescript
const pendingNonce = await walletClient.request({
  method: 'eth_getTransactionCount',
  params: [address, 'pending'],
});
let nonce = parseInt(pendingNonce as string, 16);

await walletClient.sendTransaction({ ...tx1, nonce: nonce++ });
await walletClient.sendTransaction({ ...tx2, nonce: nonce++ });
```

The `subkey sweep` command uses a different workaround: `waitForTransactionReceipt` between sends, so the next call reads the freshly-advanced nonce from the node. Both approaches work.

---

## 8. `pm auth` or `pm order` reports `trader` as the wrong address

**Symptom:** You ran `elytro pm --subkey X auth`, then `elytro pm info` (without --subkey) and saw the owner EOA's API key, not the subkey's. Confusing.

**Root cause:** Per-signer credential storage works correctly; each signer has its own cached creds at `polymarket-creds:<address>`. But `elytro pm info` without `--subkey` reports the **owner's** info, not the last-used subkey's. This is correct behavior (`pm info` reflects the currently-routed signer), but it surprises users who expect the command to remember the last subkey.

**Fix:** Always pass `--subkey <label>` to every `pm` command when you want subkey-scoped state. Don't rely on commands "remembering" a subkey. Agent-side, keep track of which subkey you're using in the session.

---

## 9. Chain config mismatch: command sends tx to wrong RPC

**Symptom:** `elytro pm --subkey X approve` errors with something like "invalid sender" or the tx appears to go to a different chain's RPC (e.g. Optimism Sepolia's RPC).

**Root cause:** Older Elytro code used `ctx.chain.currentChain.endpoint` to pick the RPC, but `ctx.chain.currentChain` reflects the CLI config's default chain (set via `elytro init`) which is often Optimism Sepolia, **not** the active account's chain (Polygon).

**Fix (applied in the skill):** Chain resolution order for any command that sends on-chain txs:
1. If `--subkey <label>` is specified, use `subkey.boundChainId`
2. Otherwise, use `ctx.account.currentAccount.chainId`
3. Never rely on `ctx.chain.currentChain.id` (CLI config) as the source of truth

When contributing commands, **always resolve chain from the account or subkey, never from CLI config.**

---

## 10. SecurityHook is not installed; `subkey fund` completes without 2FA

**Symptom:** `elytro subkey fund <label> --usdc N` succeeds with `hookSigned: false`, no email OTP required.

**Root cause:** If the active smart account does not have SecurityHook installed on-chain, there is no gate to trigger. The `fund` command's hook logic is a no-op pass-through when `hookStatus.installed === false`.

**Fix:** Install the hook AND complete every step of the setup before trading. The full sequence is four steps — partial setup (e.g. install without OTP submit) leaves the skill in exactly the unsafe state this troubleshooting entry is supposed to resolve.

```bash
# Step 1: deploy the on-chain hook
elytro security 2fa install

# Step 2: bind email. Returns otpPending — NOT finalized until otp submit.
elytro security email bind <email>
# → { otpPending: { challengeId, maskedEmail, ... } }
elytro otp submit <challengeId> <6-digit-code>
# → email bind is now actually bound

# Step 3: set the spending limit (may itself return otpPending; complete it
# with another `otp submit` if so)
elytro security spending-limit 100

# Step 4: verify the whole stack is live — ALL three must be true
elytro security status
# Expected: hookInstalled: true, emailVerified: true, dailyLimitUsd: 100
```

**Do not stop after step 1.** An installed hook with `emailVerified: false` is the exact false-positive state that makes `subkey fund` appear to succeed "with hook" while actually providing no 2FA coverage — because there is no verified destination for the OTP to route to. Re-running `subkey fund` after only step 1 will still log `hookSigned: false`. If `security status` shows anything other than the three `true`/set fields above, rerun whichever step failed before touching `subkey fund` again.

**Important:** a skill that silently accepts `hookSigned: false` is not operating under the safety envelope it claims. For any production Polymarket usage, require all four steps to be complete. The skill should refuse to run `fund` if `security status` shows the stack is not fully live — this is a reasonable strictness for a "safe by default" deployment.

---

## 11. `cast receipt <tx>` returns no data for a recent tx

**Symptom:** Immediately after getting a tx hash back from `sendTransaction`, `cast receipt` returns "tx not found".

**Root cause:** Polygon blocks take 2-5 seconds to produce, and public RPCs have their own indexing lag. The tx might be in the mempool or just-included, but the specific RPC you're hitting hasn't caught up.

**Fix:** Wait 5-10 seconds and retry, or use `waitForTransactionReceipt` in viem which polls until the tx is mined. Don't interpret a first-read miss as "tx was dropped."

---

## 12. `polymarket deposit` command fails with obscure errors

**Symptom:** `elytro pm deposit --amount N` errors with weird messages about missing `chainConfig` or wrong function signatures.

**Root cause:** The legacy `pm deposit` command has known API-signature bugs — it calls `estimateUserOp(userOp, account.address)` (should be `{fakeBalance: true}`) and `getFeeData()` without a chainConfig arg. It also bypasses SecurityHook. These bugs pre-date the subkey refactor.

**Fix:** **Use `elytro subkey fund` instead.** It's the supported, SecurityHook-aware path for moving USDC + POL from a smart account to a trading EOA. `pm deposit` will be deleted in a future cleanup.

---

## 13. Subkey runs out of POL mid-session

**Symptom:** `elytro pm --subkey X approve` or any other direct-EOA tx from the subkey fails with `insufficient funds for gas`.

**Root cause:** Gas spikes on Polygon (150+ gwei) combined with an initial subkey POL budget sized for quieter times. After a few approvals and order attempts, the subkey's POL is consumed faster than expected.

**Fix — hook-gated, the only supported path:**

```bash
elytro subkey fund polymarket-main --native 0.02
elytro pm --subkey polymarket-main approve    # retry after the UserOp confirms
```

`subkey fund --native` routes POL through a SecurityHook-gated UserOp from the bound smart account. If SecurityHook is installed and the amount (plus any accumulated 24h spend) is above the daily limit, you'll get an OTP challenge you must complete before the fund lands.

**If the smart account itself is also out of POL:** fund the smart account externally from your own wallet (e.g., from an exchange or another address). The smart account's address is shown in `elytro account info`. Once it has POL, re-run `subkey fund --native`.

**There is intentionally no `subkey topup` command that bypasses SecurityHook.** An earlier iteration of this skill exposed an `elytro subkey topup` command that sent owner-EOA POL directly to the subkey as a plain tx, avoiding the smart account and the hook. That contradicted the architectural invariant that every owner-key money movement into a trading subkey must flow through SecurityHook, and it was removed during the Codex review pass. If you see references to `topup` in old docs or old vault state, that command no longer exists — use `subkey fund --native` instead.

---

## 14. `pm approve` back-to-back tx dropped by RPC even with explicit nonce

**Symptom:** Rare. First tx (USDC.approve) lands and you can see it via `cast receipt`. Second tx (CTF.setApprovalForAll) returns a hash from `sendTransaction` but `cast tx <hash>` says "tx not found" and on-chain state shows no approval.

**Root cause:** Some Polygon public RPC endpoints silently drop a second transaction from the same sender if it arrives while the first is still pending — even when the second has a correctly-incremented nonce. The first eventually mines; the second is just gone.

**Fix (applied in the skill):** `pm approve` now calls `publicClient.waitForTransactionReceipt(hash1)` before sending the second tx. Throughput is lower (one tx at a time), but reliability is 100%. Also: `pm approve` is now idempotent — it reads current allowance/approval state on-chain and skips sends that are already set. So if you hit this drop, just run `pm approve` again and it'll finish the remaining work.

---

## Known open issues / backlog

- **No CTF redeem command.** Residuals below the 5-share minimum can only be recovered by redeeming after market resolution, and Elytro doesn't expose a command for this yet. Workaround: use `cast send` against the CTF contract or the Polymarket UI.
- **Auto-retry on geoblock.** The skill tells agents to retry once after 30 seconds, but there's no built-in retry in the `elytro pm order` command itself. Consider adding.
- **No sweep CTF command.** `subkey sweep` only moves USDC.e and POL, not CTF tokens. If you want to consolidate CTF tokens from subkey to another address, use `elytro pm` + cast manually, or add a `subkey sweep --cft` option.
- **Subkey scope enforcement is advisory.** The `scope` field is documentation, not enforced at the keyring API layer. Future: extend `getSubkeyAccount(label, { requiredScope: 'polymarket-clob' })` to require scope matching.
