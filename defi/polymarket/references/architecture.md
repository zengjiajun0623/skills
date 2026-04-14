# Polymarket × Elytro Architecture

**Audience:** developers and agents that want the full mental model behind `elytro subkey` + `elytro polymarket --subkey`. Read this before modifying the integration or before explaining "why does this skill need a subkey" to a user.

## 1. The problem this design solves

Polymarket's CTF Exchange (on Polygon, `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E`) verifies orders via `verifyEOASignature`:

```solidity
function verifyEOASignature(address signer, address maker, bytes32 structHash, bytes memory signature)
    internal pure returns (bool) {
    return (signer == maker) && verifyECDSASignature(signer, structHash, signature);
}
```

For `SignatureType.EOA (0)`, the only checks are:
1. `signer == maker`, and
2. `ecrecover(structHash, signature) == signer`.

No registry lookup, no gnosis-safe factory check. **Any EOA with an on-chain USDC.e allowance to the Exchange can trade** as long as it signs order structs with its private key.

The convenient thing this enables: a wallet doesn't need Polymarket-specific onboarding. The painful thing: the signing key is a regular, fully-privileged EOA private key.

### Why the naive path is unsafe

If Elytro's wallet were to reuse the **smart-account owner key** as the Polymarket signer, the same key would be responsible for:

1. Signing `UserOperation`s for the smart account (the wallet's master authority), and
2. Signing Polymarket EIP-712 order structs (trading authority).

Any process that has access to path #2 — a CLI subprocess, a skill, a malicious prompt injection — effectively has access to path #1 as well. `scopedSignTypedData` guards at the wrapper level (reject non-Polymarket domains) but cannot stop an attacker who reaches the underlying private key. Defense in depth requires **key separation**, not just domain filtering.

## 2. The subkey primitive

Elytro's vault now holds two disjoint key sets:

```typescript
interface VaultData {
  owners: OwnerKey[];       // signs UserOperations for smart accounts
  currentOwnerId: Address;  // active owner
  subkeys?: SubKey[];       // scoped trading/agent keys; disjoint from owners
}

interface SubKey {
  id: Address;              // EOA address
  key: Hex;                 // private key, scrubbed to keyBuffers at rest
  label: string;            // unique label, e.g. "polymarket-main"
  scope: string;            // advisory hint, e.g. "polymarket-clob"
  boundAccount: Address;    // smart account this subkey is bound to (sweep target)
  boundChainId: number;     // chain of the bound smart account
  createdAt: number;        // unix ms
}
```

Both owners and subkeys are:
- encrypted at rest with the same vault key (AES-256-GCM, vault key stored in the OS keychain)
- hydrated to scrubbable `Uint8Array` buffers in memory, indexed by address in `keyBuffers`
- zero-filled on CLI exit (`keyring.lock()`)

The differences matter at the **authority** level, not at the storage level:

| Capability | Owner key | Subkey |
|---|---|---|
| Sign a UserOperation for a smart account | ✓ | ✗ (has no authority over any smart account) |
| Sign a Polymarket EIP-712 typed-data struct | ✓ (current polymarket flow without `--subkey`) | ✓ (the recommended path) |
| Sign an arbitrary ETH transaction | ✓ | ✓ (it's still an EOA) |
| Generate new subkeys | ✓ (indirectly via `elytro subkey create` which writes to the vault) | ✗ |
| Reset the vault | ✓ | ✗ |

`boundAccount` is **metadata, not authority** — it records where the sweep command should return funds, and is semantically "which smart account does this subkey belong to". The subkey has no on-chain power over `boundAccount`; the smart account still requires an owner signature to move its funds.

## 3. Signing paths

### Owner path (UserOp signing)

```
keyring.signDigest(userOpHash)
  → uses vault.currentOwnerId's private key
  → used by elytro tx send, elytro subkey fund, elytro pm deposit (legacy)
```

The owner signs a UserOperation; the EntryPoint validates via ERC-4337 + the smart-account contract's `validateUserOp`. If SecurityHook is installed, the hook also validates (2FA, spending limits, etc.) before the bundler submits.

### Subkey path (scoped EIP-712 typed data + direct EOA txs)

```
keyring.getSubkeyAccount(label)
  → returns a viem LocalAccount for the subkey
  → consumers call .signTypedData or .signTransaction / .sendTransaction directly
```

The subkey signs whatever is asked of it. In the polymarket skill, all subkey signing routes through `scopedSignTypedData(account, params)`, which enforces:

```typescript
const ALLOWED_DOMAINS = new Set(['ClobAuthDomain', 'Polymarket CTF Exchange']);
```

Any EIP-712 domain outside this allowlist is rejected. This is a **defense-in-depth** layer on top of the key-separation layer. If an attacker somehow reaches `PolymarketService` with a different domain, they still can't produce a valid signature for arbitrary contracts. Combined with the subkey being disjoint from owners, the blast radius is bounded by:
- What the subkey's balance holds (a known, user-chosen trading budget)
- What Polymarket-domain messages can do (trade on Polymarket)

## 4. Fund and sweep flows

These are the two flows where value crosses the Owner↔Subkey boundary. Both are designed so the smart-account security envelope (SecurityHook) is the authoritative gatekeeper.

### Fund: Smart Account → Subkey

```
┌─────────────────┐                       ┌─────────────┐
│  Smart Account  │ ─── UserOp ─────────→ │   Subkey    │
│  (kind-cape)    │  (batched, 2 ops)     │  (0xd1e0..) │
│  0x4c2e74...    │                       │             │
└─────────────────┘                       └─────────────┘
         ▲                                        ▲
         │                                        │
         │                                        │
    Owner key                                 USDC.e + POL
    signs UserOp                              land on subkey
```

- One UserOperation contains two `execute(target, value, data)` calls:
  - USDC.e transfer: `execute(USDC.e, 0, encodeTransfer(subkey, usdcAmount))`
  - Native transfer: `execute(subkey, nativeAmount, 0x)`
- Signed by the current **owner key** (because smart-account control is required to move its assets)
- If SecurityHook is installed, `preUserOpValidation` intercepts the UserOp; the user must approve via email OTP; the hook then co-signs; the signature is packed with both owner + hook signatures
- Funding is the **authorization boundary**: everything that can happen to trading money is ratified here, then the subkey is free to operate without further hook involvement until sweep time

### Trading: Subkey ↔ Polymarket CLOB

```
┌─────────────┐         ┌──────────────────┐      ┌──────────────────┐
│   Subkey    │ sign ─→ │ Polymarket CLOB  │──→   │  CTFExchange     │
│             │         │ (off-chain match)│      │  (Polygon)       │
└─────────────┘         └──────────────────┘      └──────────────────┘
       │                                                    │
       │                                                    ▼
       │                                          ┌────────────────────┐
       └── USDC.e allowance ─────────────────→    │ USDC.e → exchange  │
                                                  │ CTF tokens → subkey│
                                                  └────────────────────┘
```

- Subkey signs CLOB-auth EIP-712 typed data and order struct EIP-712 typed data
- `signatureType = 0 (EOA)` in the order
- `maker == signer == subkey.address`
- Polymarket's off-chain matcher batches fills and calls CTFExchange on-chain; the exchange pulls USDC.e from the subkey (allowance) and mints CTF tokens to the subkey
- **Owner key is never involved.** SecurityHook is not involved — the smart account is not in the path.

### Sweep: Subkey → Smart Account

```
┌─────────────┐                              ┌─────────────────┐
│   Subkey    │ ─── USDC.e (EOA tx) ───────→ │  Smart Account  │
│             │ ─── Native POL (EOA tx) ───→ │   (kind-cape)   │
└─────────────┘                              └─────────────────┘
      │
  subkey signs both legs
  (owner uninvolved)
```

Both legs return to the bound smart account as plain EOA transactions signed by the subkey.

- **USDC.e** sweeps via `ERC20.transfer(smartAccount, balance)`. `transfer()` only updates internal balances; it does not execute recipient code, so any contract address accepts it regardless of `receive()` support.
- **Native POL** sweeps via a plain value transfer from the subkey to the smart account. Because the smart account is an ERC-1967 proxy, a plain value call triggers `delegatecall` into the implementation's `receive()` / fallback, which consumes more than the 21000 gas a simple EOA-to-EOA transfer reserves (typical Elytro smart accounts need ~26000 gas for the proxy + impl receive path). The sweep command probes the actual cost via `eth_estimateGas` before sending and uses **max(estimatedGas × 1.5, 40000)** as the tx gas limit, aborting with a clear error if the probe itself reverts (signaling the implementation lacks a payable receive).
- Neither leg touches SecurityHook — funds flow back toward safety, so no gate is required.
- **No owner-EOA gas tank.** Earlier versions routed native POL to the smart account's owner EOA to sidestep the proxy's receive-gas issue. That workaround created an unprotected "owner gas tank" that had to be refilled via a direct owner-EOA → subkey path, bypassing the SecurityHook invariant. The modern design eliminates both: sweep returns everything to the smart account, and any subsequent subkey refill goes through `subkey fund` (UserOp, hook-gated). See troubleshooting entry #5 for the gas-probe details.
- **CTF position tokens are NOT swept.** They stay at the subkey address. If you want to reclaim them, you need to (a) sell them before sweeping, (b) wait for market resolution and call the CTF contract's `redeemPositions`, or (c) leave them orphaned if you `subkey remove` without resolving them.

## 5. CLOB credential isolation

The Polymarket CLOB issues HMAC API credentials per EOA. Elytro stores these credentials in its local filesystem store under a key derived from the signer address:

```
polymarket-creds:<signer-lowercase-address>
```

So the owner EOA and each subkey each get their own credentials. Switching `--subkey` in a command switches which stored creds are loaded. This prevents auth leakage: if you somehow signed-in as the owner EOA earlier (via a legacy `elytro pm auth` without `--subkey`), that auth token cannot be used to post orders from a subkey, and vice versa.

**Migration note:** elytro 0.8.9 used a global key `polymarket-creds` for the owner's credentials. After upgrading to the subkey-aware version, owners need to re-auth once via `elytro pm auth` (without `--subkey`) to populate the new per-address slot.

## 6. Gas cost economics

Typical Polygon mainnet gas at normal load:

| Step | Typical cost | Notes |
|---|---|---|
| `subkey create` | 0 POL | Vault-only write, no on-chain |
| `subkey fund` (batched USDC + native) | ~0 POL | Pimlico public sponsor handles UserOp gas |
| `pm approve` (USDC + CTF) × 1 target | ~0.01 POL | Paid by subkey's own POL budget |
| `pm approve --neg-risk` | ~0.02 POL | Two targets = 4 txs |
| `pm order BUY/SELL` | 0 POL | Polymarket matcher pays on-chain settlement |
| `subkey sweep` (USDC + native) | ~0.01 POL | Two EOA txs from subkey |
| `subkey remove` | 0 POL | Vault-only |

So a reasonable per-session POL budget for a subkey is **~0.05 POL** (allows generous headroom for gas spikes and a sweep). The `fund` command lets you size `--native` exactly for this.

Polygon gas can spike by 3-5x during busy periods. viem's automatic `eth_feeHistory` fee estimation tends to over-reserve on Polygon (by 10x or more), which is why Elytro sets explicit `gasPrice` on direct-EOA txs. If you're writing similar tools, do not trust viem's automatic Polygon fee computation.

## 7. Why not ERC-1271 / POLY_1271?

The CTF Exchange contract repo includes a `POLY_1271` signature type (value `3`) that checks ERC-1271 `isValidSignature` on a contract wallet. In theory, a 4337 smart account that implements ERC-1271 could sign Polymarket orders directly.

In practice, as of April 2026:

- Polymarket's public API documentation still only lists `EOA (0)`, `POLY_PROXY (1)`, `POLY_GNOSIS_SAFE (2)` as supported signature types
- `@polymarket/clob-client` (the official TypeScript library) does not expose `POLY_1271`
- Orders posted with `signatureType = 3` via direct REST may or may not be routed through the matcher

So the subkey pattern is the **shippable** path today. If Polymarket enables `POLY_1271` at the CLOB API layer, the Elytro integration can add a direct-signing path where the smart account itself is the maker, and the subkey primitive becomes redundant — but the architectural win (key separation, per-skill isolation) remains useful for any skill that talks to a protocol Polymarket's scheme doesn't cover.

## 8. Invariants the skill must maintain

Skill authors and reviewers should check these before accepting a change:

1. **Owner key never leaves the Elytro process.** Any external tool (polymarket-cli, hyperliquid-cli, a custom script) should receive a **subkey**, never the owner key. If a tool demands a private key file, generate a fresh subkey for it, fund it, use it, sweep, and remove.
2. **Every money-moving operation that uses the owner key must flow through SecurityHook.** That's the single checkpoint where 2FA and spending-limit gates apply. If you're tempted to bypass it (`--no-hook`), you're building an unsafe product.
3. **Subkey boundAccount metadata is binding for sweep defaults, not for authority.** Don't check boundAccount to authorize a trade — it's not an authorization field, just a sweep-target hint.
4. **Don't `subkey remove` if there's non-dust balance at the subkey address.** The command warns but does not refuse. Agents must check `subkey info` first and require explicit user acknowledgment if balances are present.
5. **`scopedSignTypedData` is the only place that should call `account.signTypedData` inside the Polymarket service.** Any bypass of this helper weakens the domain-allowlist defense.
6. **CLOB credentials are per-signer, not per-vault.** The storage key is `polymarket-creds:<address>`. Do not introduce a global creds key.

---

For the gotchas that bit us during the spike (flaky geoblock, $1 min notional on limit price, CTF residuals, smart accounts rejecting plain native transfers), see [troubleshooting.md](troubleshooting.md).
