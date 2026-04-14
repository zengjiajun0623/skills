---
name: polymarket
description: >
  Trade Polymarket prediction markets from the Elytro CLI through a scoped subkey
  (not the smart-account owner key). Vault holds USDC.e on Polygon; a per-session
  subkey signs CLOB orders with bounded blast radius. Covers create/fund/approve/
  auth/buy/sell/close/sweep. Use for: placing prediction-market bets as an agent
  without exposing the wallet's primary signing key.
allowed-tools: []
required-skills: ["elytro"]
related-skills: ["defi", "elytro"]
metadata:
  openclaw:
    product-homepage: https://polymarket.com
    chain: Polygon (137)
    exchange: '0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E'
    requires:
      bins:
        - elytro
      node: '>=18.0.0'
      # Note: this skill uses the `elytro subkey` primitive (see Install
      # section). Probe availability with `elytro subkey --help` at runtime
      # rather than pinning a version — the primitive is on the
      # feat/subkey-primitive branch and will land in a future minor release.
    emoji: '🎯'
  user-invocable: true
  disable-model-invocation: false
---

# Polymarket Trading via Elytro Subkeys

**Install (Node >= 18). Pick the path that matches the CLI state on npm:**

This skill depends on the `subkey` primitive (`elytro subkey create|fund|sweep`) and the `--subkey <label>` option on `elytro pm` subcommands. These are on the `feat/subkey-primitive` branch upstream and will ship in a future minor release of `@elytro/cli`.

**Preferred (once the subkey primitive is published):**
```bash
npm install -g @elytro/cli
elytro --version
# then verify the subkey command exists:
elytro subkey --help
```
If `elytro subkey --help` prints a help page, you're good. If it errors with "unknown command `subkey`", the published npm version does not yet include the primitive — fall back to the source install below.

**Fallback (source install until the primitive lands on npm):**
```bash
git clone https://github.com/Elytro-eth/cli.git elytro-cli
cd elytro-cli
git checkout feat/subkey-primitive
bun install
bun run build
npm link
elytro subkey --help   # should succeed
```

Do **not** pin the install to a specific version like `@^0.9.0` — that would fail before the release lands on npm, blocking the entire skill. The `elytro subkey --help` probe is the authoritative "is the primitive available" check, not the version number.

**Architecture deep-dive:** [references/architecture.md](references/architecture.md)
**Gotchas + troubleshooting:** [references/troubleshooting.md](references/troubleshooting.md)

---

## Core concept: Vault and Subkey are different keys

Polymarket's CLOB requires orders signed by an EOA that acts as `maker == signer`. If you sign with the smart-account owner key directly, a compromised signing path drains the entire smart account. Instead, this skill uses a **subkey**: a separate EOA stored in the same encrypted vault but **disjoint from any smart-account owner**. The subkey:

- is generated fresh per trading context (`elytro subkey create <label>`)
- gets funded with a bounded budget via a SecurityHook-gated UserOp (`elytro subkey fund`)
- signs all Polymarket-scoped EIP-712 typed data (CLOB auth + order struct)
- holds the CTF position tokens after fills
- is swept back to the bound smart account when trading is done (`elytro subkey sweep`)

**Trust model:** compromise of the subkey = loss of the subkey's trading balance only. Compromise of the owner key = loss of everything (never expose the owner key to Polymarket-facing code).

---

## Prerequisites

1. **Elytro initialized:** `elytro init` done, vault unlocked at every CLI invocation via OS keychain.
2. **Polygon smart account active and deployed:** `elytro account list` must show a row with `chain: "Polygon"` / `chainId: 137`, `active: true`, AND `deployed: true`. If a Polygon row already exists, **do not create a duplicate** — creating a second account strands balances and security settings across the two. Resolve by running the two steps below **in order**, skipping any that are already satisfied:

   - **Step A — ensure a Polygon row exists and is active.**
     - If no Polygon row exists: create one. This creates the account in an `active: false`, `deployed: false` state, which is normal.
       ```bash
       elytro account create --chain 137 --alias <alias>
       ```
     - If a Polygon row exists but `active: false` (common right after `account create`, or if another chain is currently active): switch to it.
       ```bash
       elytro account switch <polygon-alias>
       ```
     - If a Polygon row is already `active: true`: skip this step.

   - **Step B — ensure the (now active) Polygon account is deployed.** Deployment is a separate on-chain step that has to run against an active account. A fresh `account create` does not deploy, and `account switch` does not either — always re-check `deployed` and run activate when needed:
     ```bash
     elytro account activate <polygon-alias>
     ```
     Skip only if `account list` already shows `deployed: true`.

   After both steps, re-run `elytro account list` and confirm the Polygon row now shows **both** `active: true` AND `deployed: true` before proceeding. Do not skip this verification — an "active but not deployed" account will pass the Prerequisites glance-test and then fail every subsequent `subkey fund` / `pm` command because the smart-account contract doesn't exist on-chain yet.
3. **SecurityHook installed (recommended for production):** `elytro security status` reports `hookInstalled: true` AND `emailVerified: true`. If not, run this exact sequence — **order matters and the OTP step is not optional**:

   ```bash
   # Step 1: Deploy the on-chain 2FA hook. Required first.
   elytro security 2fa install

   # Step 2: Bind the email. This WILL typically return `otpPending` — the
   # email is NOT actually bound until you complete it with `otp submit`.
   elytro security email bind <email>
   # → returns { otpPending: { challengeId, maskedEmail, expiresAt } }
   # The user checks the mailbox and tells you the 6-digit code.
   elytro otp submit <challengeId> <6-digit-code>
   # → email bind is now finalized

   # Step 3: Set the daily spending limit. Writes to the hook's backend.
   elytro security spending-limit 100
   # (this may also return otpPending depending on whether the change
   # requires step-up verification; if it does, run `otp submit` again)

   # Step 4: Verify the whole stack is actually live.
   elytro security status
   # → expect hookInstalled: true, emailVerified: true, dailyLimitUsd: 100
   ```

   **Critical:** if `security status` after step 4 does not show the hook installed AND the email verified, stop and re-run whichever step failed. The rest of this skill assumes a live hook-gated path — an unverified email or a noop spend-limit write makes `subkey fund` a false-positive safety story, not a real one.

   Without the hook, `subkey fund` will still work but is **not** 2FA-gated. For production trading the hook must be live — see the safety rules below.
4. **Smart account is funded with USDC.e + some POL.** USDC.e is the bridged USDC (`0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`), **not native USDC (`0x3c499c54...`)** — Polymarket's CTF Exchange only accepts USDC.e.

---

## Workflow

### 1. Create a scoped subkey

```bash
elytro subkey create polymarket-main --scope polymarket-clob
```

Output includes the subkey's EOA address and a "hint" about the next step. The subkey is bound to the currently-active smart account, recorded via `boundAccount`. Use a label like `polymarket-main` per vault; if you're trading multiple portfolios, create one subkey per portfolio.

### 2. Fund the subkey (triggers 2FA if SecurityHook installed)

```bash
elytro subkey fund polymarket-main --usdc 20 --native 0.1
```

- `--usdc <amount>`: USDC.e transferred to the subkey via the smart account
- `--native <amount>`: POL sent to the subkey to pay for approvals + sweep gas
- Batched into **one** UserOp via the smart account. Gas is typically sponsored by Pimlico's public endpoint on Polygon.

If SecurityHook is installed and the amount exceeds the `spending-limit`, the command returns an `otpPending` challenge. Follow the normal OTP flow:

1. Read `maskedEmail` from the error and ask the user to fetch the code from that mailbox.
2. When the user provides the 6-digit code, submit with `elytro otp submit <challengeId> <code>`.
3. The fund completes after OTP submission.

**Rule of thumb:** fund only what you're willing to lose in a worst-case subkey compromise. Top up later via another `fund` call — each top-up re-triggers the SecurityHook gate.

### 3. Set trading approvals (one-time per subkey + market class)

```bash
elytro pm --subkey polymarket-main approve              # non-neg-risk markets
elytro pm --subkey polymarket-main approve --neg-risk   # neg-risk markets (separate allowance)
```

This sends direct EOA transactions **from the subkey**:
1. `USDC.approve(exchange, MAX_UINT256)` — lets the exchange pull USDC on fill
2. `CTF.setApprovalForAll(exchange, true)` — lets the exchange transfer CTF position tokens

**Idempotent:** re-running `approve` is safe — it reads current allowance/approval state on-chain first and skips anything already set. If a prior partial failure left you with only one of the two approvals, just run `approve` again and it'll fill in the missing tx. Each tx waits for confirmation before the next is sent (some public RPCs silently drop back-to-back sends).

The subkey pays gas for these from its POL balance. Typical cost on Polygon at normal fees: ~0.01 POL for the CTFExchange pair, ~0.02 POL for the NegRisk pair (4 txs). During gas spikes it can be 2-3x higher. If the target market is `negRisk: true`, use `--neg-risk` which approves the `NegRiskExchange` + `NegRiskAdapter` contracts instead. **You can tell if a market is neg-risk** via `elytro pm market <id>` — look at the `negRisk` field.

If `approve` fails with "insufficient funds for gas", the subkey is out of POL. **Do not bypass the smart account for gas refills** — top up via the normal SecurityHook-gated path:

```bash
elytro subkey fund polymarket-main --native 0.02
```

If the smart account itself is low on POL, fund the smart account externally from your own wallet (the smart account address is shown in `elytro account info`) and then re-run `subkey fund --native`. Every owner-key POL movement into a trading subkey must go through this path — there is intentionally no direct owner-EOA → subkey shortcut, because that would bypass the SecurityHook invariant that protects the vault.

### 4. Authenticate with the CLOB (derive HMAC credentials)

```bash
elytro pm --subkey polymarket-main auth
```

Signs an EIP-712 ClobAuth struct with the **subkey** (not the owner) and exchanges it for HMAC credentials that persist in the Elytro store at `polymarket-creds:<subkey-address>`. Per-signer: different subkeys have different API keys. If you switch subkeys you must re-auth.

### 5. Discover markets

```bash
elytro pm markets --active --limit 10                                # all-time top (default sort)
elytro pm markets --active --limit 10 --order volume24hr             # TRENDING TODAY
elytro pm markets --active --limit 10 --order liquidityClob          # deepest order books
elytro pm markets --search "ceasefire" --limit 5                     # keyword search
elytro pm market <slug-or-id>                                        # details incl. clobTokenIds + negRisk
elytro pm book <token_id>                                            # live order book (top 5 each side)
elytro pm midpoint <token_id>                                        # mid price
```

**Sort fields** (for `--order`):
- `volume_num` — all-time cumulative volume. Default. Good for "most important markets ever," not so good for "what's hot right now" (stale large events dominate).
- `volume24hr` — **24-hour volume. Use this for "trending today" or "what's people actively betting on."** The right default when a user asks a time-sensitive question.
- `liquidityClob` — deepest CLOB order books. Use when you want markets where large orders won't move the price.
- `volume1wk`, `volume1mo`, `volume1yr` — other trailing-window aggregates.

Sort is always **descending** by default (top N by the field). Pass `--ascending` if you need the reverse.

The `clobTokenIds` field is an array of two strings: `[yesTokenId, noTokenId]`. Use these as the `--token` argument when placing orders.

### 6. Place an order

```bash
elytro pm --subkey polymarket-main order \
  --token <token_id> \
  --side BUY \
  --price 0.20 \
  --size 5 \
  --type FAK
```

**Order semantics — READ CAREFULLY:**

- `--price` is your **limit price**, not the expected fill price.
- `--size` is the **number of shares** you want at your limit price.
- For marketable orders, `makerAmount = price × size` must be **≥ $1 USDC** (Polymarket's minimum notional).
- FAK (Fill and Kill) fills what it can immediately at the best available prices and cancels the rest.
- **Surprise behavior:** a FAK BUY at a limit price *above* the current ask fills at the ask (good for you) but can fill **more shares than `size`** because the matcher spends up to `makerAmount = price × size` worth of USDC. Example: `--price 0.20 --size 5` = $1.00 budget; if the ask is $0.12, you may receive ~8 shares for $1.00. This is by design.
- If you need a hard share count, use `--type FOK` (Fill or Kill) or compute your limit price exactly at the expected fill.
- Share minimum per market: typically 5 (from the market's `orderMinSize` field). Check with `elytro pm market <id>`.

### 7. Close / sell

```bash
elytro pm --subkey polymarket-main order \
  --token <same_token_id> \
  --side SELL \
  --price <below-best-bid> \
  --size <your-share-count> \
  --type FAK
```

Same `$1` minimum notional rule applies: `size × price ≥ 1`. If your position is smaller than 5 shares OR worth less than $1 at the bid, it may be stuck — see "CTF residuals" in troubleshooting.

### 8. Cancel any live CLOB orders first

**This step is mandatory before sweep/remove — do not skip it.** Any standing order (especially GTC, GTD, FOK that didn't fill) remains executable on Polymarket's CLOB until explicitly cancelled. `subkey sweep` does NOT touch live orders or CTF positions, and `subkey remove` destroys the local signing key — meaning a GTC SELL placed earlier could fill *after* cleanup, transferring CTF tokens the vault can no longer sign for.

```bash
# 1. Enumerate everything still open
elytro pm --subkey polymarket-main orders

# 2. Cancel all of them. Use --all for the nuclear option, or pass
#    individual order IDs from the step above.
elytro pm --subkey polymarket-main cancel --all
# Or: elytro pm --subkey polymarket-main cancel <order_id>

# 3. Re-check to confirm no residual open orders before moving on.
elytro pm --subkey polymarket-main orders
# Expected: empty list.
```

Also check CTF positions: `elytro pm --subkey polymarket-main balance --token-id <id>` for **every** token you've touched this session. If **any** non-zero balance exists — fractional *or* whole, integer 5 shares is as orphanable as 0.666 shares — resolve it via the paths in the residuals troubleshooting entry before continuing. `subkey sweep` does not touch CTF tokens, and `subkey remove` destroys the signing key. Any non-zero position that survives both commands is permanently unrecoverable from this vault.

### 9. Sweep the subkey back to the vault

```bash
elytro subkey sweep polymarket-main
```

Both legs return to the bound smart account:
- **USDC.e → smart account** (ERC-20 `transfer`, works on any contract)
- **Native POL → smart account** (the sweep probes `eth_estimateGas` first to verify the proxy's `receive()` path and sends with a safe gas floor of 40000; smart-account proxies need more than the 21000 stipend)

CTF position tokens are **not** swept by this command — they live at the subkey address. If you want to cash out winning positions post-resolution, the merge/redeem path is a future addition; for now, resolve positions before trying to remove the subkey.

### 10. Remove the subkey (optional)

```bash
elytro subkey remove polymarket-main
```

Scrubs the subkey's private key from the vault and keyBuffers. **Any residual on-chain balance at the subkey address — live CLOB orders, CTF positions, USDC, or native — becomes unrecoverable from this vault.** Always complete steps 8 (cancel + CTF check) and 9 (sweep) first. The command warns you but does not block removal if residuals exist; it's your responsibility to verify the subkey is empty before invoking it.

---

## Full example: $10 round-trip on a random market

```bash
# 1. Create and fund a scoped subkey
elytro subkey create demo --scope polymarket-clob
elytro subkey fund demo --usdc 10 --native 0.1

# 2. Find a liquid market and INSPECT IT before approving anything. This
#    matters because non-neg-risk and neg-risk markets need different
#    approvals. Running the default approve before knowing which kind of
#    market you're trading can leave you unable to place the order.
elytro pm markets --active --limit 5 --order volume24hr

# 3. Fetch the chosen market's details. Note the negRisk field — this
#    determines which approve variant to run next.
elytro pm market <slug-or-id>
# → look for  "negRisk": true/false  in the output
elytro pm book <yes_token_id>         # confirm the best ask before ordering

# 4. Run the matching approve variant for this market.
#    IF the market shows negRisk: false:
elytro pm --subkey demo approve
#    IF the market shows negRisk: true:
elytro pm --subkey demo approve --neg-risk
#    (approve is idempotent — re-running is a no-op if the allowances are
#    already set. But the two variants target DIFFERENT contracts, so you
#    must pick the right one for the market you're actually trading.)

# 5. Authenticate with the CLOB (per-subkey API key, does not depend on
#    which approve variant ran).
elytro pm --subkey demo auth

# 6. Place a marketable buy as FOK — hard-caps size so you get exactly 20 shares
#    (FAK would let the matcher spend the entire price×size budget, possibly
#    filling MORE than 20 shares at a lower price. FOK locks the share count.)
elytro pm --subkey demo order \
  --token <yes_token_id> \
  --side BUY --price 0.25 --size 20 --type FOK

# 7. Later: close the full 20-share position at a sensible limit
#    The $1 minimum notional rule means size × price ≥ 1. With 20 shares,
#    your minimum safe SELL price is 0.05. Pick a limit below the current bid
#    (check with `elytro pm book`) that still clears the floor comfortably.
elytro pm --subkey demo order \
  --token <yes_token_id> \
  --side SELL --price 0.10 --size 20 --type FOK

# 8. Cancel any still-open orders BEFORE sweeping. Standing orders remain
#    executable on the CLOB after cleanup, and removing the subkey destroys
#    the signing key needed to cancel them later.
elytro pm --subkey demo orders                      # list anything live
elytro pm --subkey demo cancel --all                # or cancel <order_id> by ID
elytro pm --subkey demo orders                      # confirm empty list

# 9. Confirm ZERO CTF balance on every token touched this session.
#    Any non-zero balance — fractional OR whole — will be orphaned by remove.
#    If you see anything non-zero, go back and close/redeem it before removing.
elytro pm --subkey demo balance --token-id <yes_token_id>

# 10. Sweep liquid balance back to the smart account and remove the subkey
elytro subkey sweep demo
elytro subkey remove demo
```

Expected outcome: USDC.e minus spread back on the smart account, POL residual also back on the smart account (the sweep uses enough gas for the smart account's `receive()` path), subkey removed from the vault. No CTF dust, no live orders, no owner-EOA gas tank, no SecurityHook bypass.

**Why FOK not FAK in the example:** FAK (Fill and Kill) uses `price × size` as a **USDC budget** and will spend it all if the market is favorable — so a BUY at `0.25 × 20` can fill 40+ shares if the ask is 0.12. FOK (Fill or Kill) treats `--size` as a **hard share count**: you get exactly 20 shares or the order is rejected. Use FOK anywhere the share count matters for later cleanup (closing the position, calculating SELL notional, sweeping the subkey). Use FAK only when you genuinely want "spend up to $X regardless of share count."

---

## Safety rules (enforce strictly — this is trading with real money)

1. **Always check `geoblock` and TOS eligibility before trading.** `elytro pm info` + geolocation check. Polymarket restricts US users; running this skill from a US IP will fail with a `403 Trading restricted in your region` response. The platform does its own detection and it can be flaky — one rejection does not always mean permanent block, but do not circumvent geo-restrictions.
2. **Never expose the owner key to Polymarket.** If a user omits `--subkey`, `elytro pm order` will default to the smart-account owner key, which is the unsafe path. **Always pass `--subkey <label>` explicitly.** If the user asks to skip the subkey, warn them in plain language that they're trading with their vault's master key.
3. **Never bypass SecurityHook on `subkey fund`.** Do not pass `--no-hook`. If `elytro security status` shows `hookInstalled: false` OR `emailVerified: false` OR no dailyLimitUsd, stop immediately and run the **full four-step setup** from the Prerequisites section (`2fa install` → `email bind` → `otp submit` → `spending-limit` → `security status` verification). Do not accept a partial setup — installing the hook without completing the email OTP leaves `emailVerified: false`, which means later `subkey fund` calls will appear to run but provide no actual 2FA protection. This is the specific failure mode that makes a fund bypass feel safe when it isn't.
4. **Simulate-equivalent checks for orders:** before placing an order, fetch `elytro pm book <token_id>` to confirm the current bid/ask and that the order will actually match at the price you expect.
5. **$1 minimum notional.** Reject any order request where `price × size < 1` before submitting. Explain to the user why the order was rejected and suggest a larger size or a different price.
6. **Geoblock retry once and no more.** If the first order POST returns `403 Trading restricted`, retry exactly one time after 30 seconds. If it fails again, report to user and abort. Never keep retrying silently.
7. **CTF residuals are real money too.** After a close, check the subkey's CTF balance for every token touched this session. Two distinct cases:
   - **Balance ≥ 5 shares (even if fractional like 16.666):** you are NOT stuck. `orderMinSize` is a minimum, not a granularity — sell all of it in one order as long as `size × price ≥ $1`. See the "sell all fractional" example in the troubleshooting doc. Do not "buy more to reach a round number" — that wastes money.
   - **Balance < 5 shares:** actually stuck. Real exits: (a) buy enough to cross the 5-share minimum, then SELL everything in one order, (b) wait for resolution and call `redeemPositions` directly on the CTF contract (requires collateralToken + parentCollectionId + conditionId + indexSets — see troubleshooting #4 for the full `cast send`), or (c) accept the loss if dust < gas + spread cost. Never `subkey remove` with any non-zero CTF balance; those tokens become permanently unrecoverable.

---

## Approval-required commands

Get explicit user confirmation before running:

- `elytro subkey fund <label> ...` — **any** variant (`--usdc`, `--native`, or both) moves owner-key money from the smart account into the trading subkey and may trigger 2FA. All variants require user confirmation; there is no "gas-only is automatic" carveout.
- `elytro pm --subkey <label> approve` — sends on-chain transactions, irreversible
- `elytro pm --subkey <label> order …` — places real trades with real money
- `elytro pm --subkey <label> cancel <id>` / `cancel --all` — state-changing market action that removes the user's standing exposure. Always confirm before running, especially `--all` which is non-reversible in a single command.
- `elytro subkey sweep <label>` — moves all trading funds back; harmless-ish but the user should know
- `elytro subkey remove <label>` — scrubs key; warn about any residual balance first

Read-only commands that do NOT need approval: `pm markets`, `pm market`, `pm book`, `pm midpoint`, `pm spread`, `pm info`, `pm orders`, `pm balance`, `subkey list`, `subkey info`.

---

## Command reference (cheat sheet)

### Public market data (no wallet, no subkey)

```bash
elytro pm markets [--active] [--limit N] [--search "query"] [--order <field>] [--ascending]
elytro pm market <id-or-slug>
elytro pm book <token_id>
elytro pm midpoint <token_id>
elytro pm spread <token_id>
elytro pm price <token_id> --side BUY
```

Useful `--order` values: `volume_num` (all-time, default), `volume24hr` (trending today), `liquidityClob` (deepest books), `volume1wk`/`volume1mo`/`volume1yr` (trailing windows). Descending by default — add `--ascending` to flip.

### Subkey lifecycle

```bash
elytro subkey create <label> [--scope <scope>]
elytro subkey list [--scope <scope>] [--bound <smart-account-address>]
elytro subkey info <label>
elytro subkey fund <label> [--usdc <n>] [--native <n>] [--no-sponsor]
# NOTE: `--no-hook` exists on the CLI but is INTENTIONALLY omitted here.
# Do not add it. See safety rule #3 — skipping the hook on fund breaks the
# entire security model this skill is built on.
elytro subkey sweep <label> [--usdc-only] [--native-only]
elytro subkey remove <label>
```

### Subkey-scoped trading

```bash
elytro pm --subkey <label> auth
elytro pm --subkey <label> approve [--neg-risk]
elytro pm --subkey <label> balance [--token-id <id>]
elytro pm --subkey <label> orders [--market <conditionId>]
elytro pm --subkey <label> order --token <id> --side BUY|SELL --price <0-1> --size <n> [--type GTC|FOK|FAK] [--neg-risk]
elytro pm --subkey <label> cancel [<order_id> | --all]
elytro pm --subkey <label> info
```

---

## How to explain results to the user

Use the shapes defined in the [`elytro` skill](../../elytro/SKILL.md#how-to-explain-results). Polymarket-specific additions:

**Order placed:**
`Done: placed <side> <size> shares at limit <price> on <market question>.`
`Status: <matched | open | cancelled>.` `Fill: <fill details if matched>.`
`Subkey balance: <remaining USDC.e>.`

**Order rejected (min notional):**
`Couldn't complete: order size × price is <n>, below Polymarket's $1 minimum.`
`Try: increase --size or --price so the notional is at least $1.`

**Geoblock hit:**
`Couldn't complete: Polymarket returned "Trading restricted in your region" for this IP.`
`This is a platform-level block. Retry once after 30 seconds is allowed; beyond that, abort.`

**CTF residual found (balance ≥ 5 shares):**
`Note: your subkey still holds <n> shares on market <market>. orderMinSize is a minimum not a granularity, so you can close all <n> in one SELL as long as size × price ≥ $1. Want me to place that close order now?`

**CTF residual warning after close (balance < 5 shares — actually stuck):**
`Warning: your position closed with <n> residual shares (below Polymarket's 5-share orderMinSize floor). You cannot sell this residual via CLOB. Options: (1) buy enough additional shares to cross the 5-share minimum and sell all at once, (2) wait for market resolution and redeem via CTF redeemPositions (full cast send example is in troubleshooting #4 — requires collateralToken + parentCollectionId + conditionId + indexSets), (3) accept the loss if recovery cost exceeds residual value.`
