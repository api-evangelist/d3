---
name: doma-secondary-sales
description: >
  Buy domains listed on Doma marketplace — .ai, .com, .xyz, .net, .fyi, .io and
  all ICANN TLDs. Trigger on: buy domain, purchase domain, secondary sale, make
  offer, Doma, Seaport. Uses doma CLI, not web search.
license: MIT
metadata:
  author: doma-protocol
  version: "1.2.0"
compatibility: Requires Node.js 18+ and the latest @doma-protocol/cli (the skill verifies via `npm view` at runtime and prompts the user to upgrade if the local version is behind).
allowed-tools: Bash(doma *), Bash(npx -y @doma-protocol/cli *), Bash(node *), Bash(npx esbuild *), Bash(npm *), Bash(which doma), Bash(curl *)
argument-hint: [domain-name]
---

# Doma Secondary Sales

Help users buy domains listed on the Doma Names Marketplace. Doma tokenizes ICANN domains as DOT NFTs traded via Seaport.

Use the `doma` CLI — do NOT web search.

## Wallet modes

The `doma` CLI signs writes in one of two modes. The "Doma wallet" path lets the user sign through the same Doma wallet they already use in the launchpad — the launchpad signs on their behalf under a USD-capped allowance, no key on the user's machine. The "private key" path is the user's own raw key, set as an env var.

| Mode | What signs the tx | How the user sets it up |
|---|---|---|
| `agent` (Doma wallet) | Doma launchpad signs on the user's behalf; no key on the user's machine | `doma auth login` (browser consent for a $200 USD spend ceiling) → mints session into `~/.doma/credentials.json` |
| `private-key` (default) | Local raw key, signed with viem/ethers | `export DOMA_PRIVATE_KEY=...` and restart the session |

**CLI version.** The Doma wallet path lands in `@doma-protocol/cli` 0.5.0. At skill version 1.2.0, npm `latest` was 0.4.0 (no `auth` command) — so any install pulled from npm today is private-key-only. The skill runtime-detects via `doma auth --help` exit code (the `AUTH_CAPABLE` check in Step 1d) instead of parsing the version string, so this section stays correct once 0.5.0 ships.

### Choosing a mode

Ask exactly once when the user has neither configured `walletMode` nor exported `DOMA_PRIVATE_KEY` (Step 1d's bash output disambiguates this). Anyone with `walletMode` already set or a key already exported skips this — their existing setup is the answer.

```
AskUserQuestion: "You already use Doma — would you like to sign with your Doma wallet here, or insert a private key?"
  → Use my Doma wallet — opens a browser to authorize a $200 USD spend ceiling on Doma + Base chains
  → Insert my own private key — set DOMA_PRIVATE_KEY yourself outside this chat, then restart
```

- **Doma wallet** → run `doma auth login` directly. The CLI opens the launchpad's consent screen in the user's default browser; the user clicks Authorize there (that screen is the real authorization moment — no separate "are you sure" prompt from the skill). On `Authorized.`, run `doma config set walletMode agent` so the choice persists, then proceed. If a session is later missing (revoked / expired), re-run `auth login` directly without re-asking — picking "Doma wallet" is standing consent (rule #4).
- **Private key** → tell the user the steps and stop until they confirm:
  1. `export DOMA_PRIVATE_KEY=0x<64-hex>` (or write it to `/tmp/doma-buy/.env`).
  2. (Optional) `doma config set walletMode private-key` — already the default, but explicit.
  3. Restart this conversation and re-run the skill.
   Never paste the key in chat. Don't proceed until they confirm the key is set.

The transactional flow (look up domain, check funds, bridge, buy) is byte-identical in both modes; only auth setup and a few error patterns (see Gotchas) differ.

## Rules

1. **Ask, don't assume** — always ask network, always ask before bridging, always confirm before spending.
2. **One network per session** — once chosen, every command, URL, and prompt stays on that network. Never mention the other.
3. **Private keys** — never handle, log, or display. Direct user to set env vars themselves.
4. **Wallet mode is the user's choice, asked once** — see Wallet modes → Choosing a mode. After they pick, don't re-ask and don't switch modes on their behalf. Picking "Doma wallet" is standing consent for `doma auth login` whenever a session is later missing — the launchpad's consent screen is the actual authorization moment, not a skill-level prompt.
5. **Clean language** — no error codes, ticket numbers, or developer jargon to the user.
6. **CLI-first** — use `doma marketplace` commands when available. Fall back to scripts only if marketplace subcommand doesn't exist.
7. **Price conversion (fallback only)** — use `listing.currency.decimals` from the API, compute in code, never by hand. Use `externalId` for orderId, not `id`.

---

## Flow

Linear — each step completes before the next.

### 1. Setup

**a) Ask network first** (before anything else):
```
AskUserQuestion: "Which network is this domain on?"
  → Mainnet (chain 97477, api.doma.xyz)
  → Testnet (chain 97476, api-testnet.doma.xyz)
```

**b) Set config to match the chosen network.** Rewrite the full config to avoid stale/conflicting values:
```bash
node -e "
const fs = require('fs');
const path = require('path');
const configPath = path.join(require('os').homedir(), '.doma', 'config.json');
const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
config.testnet = <NETWORK === 'mainnet' ? false : true>;
config.chainId = '<CHAIN_ID>';
config.apiUrl = '<API_URL>';
fs.writeFileSync(configPath, JSON.stringify(config, null, 2));
console.log('Config updated for <NETWORK>');
"
```

**c) Check if marketplace commands are available:**
```bash
doma marketplace --help 2>&1 | grep -q "marketplace" && echo "CLI_MARKETPLACE=yes" || echo "CLI_MARKETPLACE=no"
```

**If `CLI_MARKETPLACE=yes`:** Skip Steps 1d and 1e — the CLI handles API keys, SDK deps, and private key access from Keychain internally. Proceed to Step 2.

**If `CLI_MARKETPLACE=no` (older CLI):** Continue with fallback setup:

**d) Check prerequisites** (parallel):
```bash
# Check CLI is at the latest npm version. Uses `sort -V` for semver-aware
# comparison so a locally-built newer CLI (e.g. 0.6.0-dev when npm latest is
# 0.5.2) isn't flagged. Falls back to "unknown" when npm is offline, rate-
# limited, or `doma` is missing — the LLM should treat unknown the same as
# yes when `which doma` also fails.
LATEST=$(npm view @doma-protocol/cli version 2>/dev/null)
CURRENT=$(doma --version 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1)
if [ -z "$LATEST" ] || [ -z "$CURRENT" ]; then
  echo "CLI_OUTDATED=unknown LATEST=${LATEST:-?} CURRENT=${CURRENT:-?}"
elif [ "$(printf '%s\n%s\n' "$LATEST" "$CURRENT" | sort -V | head -1)" = "$CURRENT" ] && [ "$CURRENT" != "$LATEST" ]; then
  echo "CLI_OUTDATED=yes LATEST=$LATEST CURRENT=$CURRENT"
else
  echo "CLI_OUTDATED=no"
fi
which doma || echo "not installed"
doma auth --help >/dev/null 2>&1 && echo "AUTH_CAPABLE=yes" || echo "AUTH_CAPABLE=no"
# `doma config get walletMode` prints `(default: private-key)` when unset — filter to literal mode values so the matrix sees `(unset)` correctly.
WALLET_MODE_CONFIGURED=$([ "$AUTH_CAPABLE" = "yes" ] && doma config get walletMode 2>/dev/null | grep -E "^(agent|private-key)$" || echo "(unset)")
echo "WALLET_MODE_CONFIGURED=$WALLET_MODE_CONFIGURED"
[ -n "$DOMA_PRIVATE_KEY" ] && echo "PK_SET=yes" || echo "PK_SET=no"
if [ "$WALLET_MODE_CONFIGURED" = "agent" ]; then
  # `auth status` prints `Session    : Authorized` (capital A) iff a session exists; case-sensitive grep avoids matching the always-present "Wallet mode:" line or the "not authorized" miss.
  doma auth status 2>&1 | grep -q "Authorized" && echo "SESSION=ok" || echo "SESSION=missing"
fi
[ -n "$DOMA_API_KEY" ] && echo "api set" || echo "api missing"
```

Use the bash output to pick a row in this decision matrix:

- **`CLI_OUTDATED=yes`** → STOP. Tell the user:
  > "Your `doma` CLI is on version `$CURRENT`, but the latest is `$LATEST`. Please run:
  > ```
  > npm i -g @doma-protocol/cli@latest
  > ```
  > Then restart this conversation."

  Do NOT proceed with any other commands until the user confirms they've upgraded. Older CLI versions may be missing commands (`doma approve`, balance pre-flight, etc.) that this skill expects.

- **`CLI_OUTDATED=unknown`** → npm couldn't be queried (offline, registry rate-limited, corporate proxy, etc.) OR `doma` isn't installed locally. Check the `which doma` line in the same block:
  - If `which doma` returned a path AND `AUTH_CAPABLE=yes`, the CLI exists — proceed cautiously and tell the user up-front: "I couldn't verify your `doma` CLI is current (npm registry unreachable). If commands fail with `unknown option` or missing flags, please run `npm i -g @doma-protocol/cli@latest` and retry."
  - If `which doma` returned "not installed", treat as `CLI_OUTDATED=yes` — STOP and direct the user to install per the message above.

| `AUTH_CAPABLE` | `WALLET_MODE_CONFIGURED` | `PK_SET` | `SESSION` | What to do |
|---|---|---|---|---|
| `no` | (any) | `yes` | n/a | Proceed (private-key — only path on this CLI). |
| `no` | (any) | `no` | n/a | Wallet modes → Private key branch. Stop until user confirms key is set. |
| `yes` | `(unset)` | `yes` | n/a | Proceed (CLI default = private-key). |
| `yes` | `(unset)` | `no` | n/a | Wallet modes → Choosing a mode. Act on their pick. |
| `yes` | `agent` | n/a | `ok` | Proceed. |
| `yes` | `agent` | n/a | `missing` | Run `doma auth login` directly (mode pick is standing consent). Wait for browser; on `Authorized.`, proceed. |
| `yes` | `private-key` | `yes` | n/a | Proceed. |
| `yes` | `private-key` | `no` | n/a | Wallet modes → Private key branch. Stop until user confirms key is set. |

- **CLI missing** → ask: install globally or use npx.
- **API key missing** → read from `~/.doma/config.json` (`apiKey` field, safe to read). If that key fails against the chosen network's API, ask user to provide the correct key for that network.

**e) Validate API key against chosen network:**
```bash
curl -s -X POST "<API_BASE>/graphql" \
  -H "Content-Type: application/json" \
  -H "Api-Key: $DOMA_API_KEY" \
  -d '{"query":"{ name(name: \"doma.xyz\") { name } }"}' | node -e "
const d=JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
if(d.errors) { console.log('INVALID_KEY: ' + d.errors[0].message); process.exit(1); }
console.log('API key valid');
"
```

If invalid → ask user for the correct API key for the chosen network. Do not proceed until key is validated.

**f) Pre-install SDK deps:**
```bash
mkdir -p /tmp/doma-buy && cd /tmp/doma-buy
npm init -y 2>/dev/null
npm install @doma-protocol/orderbook-sdk ethers dotenv viem esbuild 2>/dev/null
```

### 2. Look up domain + balances (parallel)

```bash
doma domain <domain> -f json
doma balance -f json
doma balance --chain base -f json
doma balance --chain ethereum -f json
```

If domain "not found" → retry once after 10s (index lag). Still not found → ask user to confirm domain name and network.

### 3. Get listing details

**CLI method** (if `CLI_MARKETPLACE=yes`):
```bash
doma marketplace get <domain> -f json
```

Returns `{ domain, listed, owner, listings: [{ orderId, price, currency, expiresAt }] }`. Price is already human-readable.

**Fallback** (if `CLI_MARKETPLACE=no`):
```bash
curl -s -X POST "<API_BASE>/graphql" \
  -H "Content-Type: application/json" \
  -H "Api-Key: $DOMA_API_KEY" \
  -d '{"query":"{ name(name: \"<domain>\") { name tokens { tokenId ownerAddress tokenAddress listings { id externalId price expiresAt currency { symbol decimals } } } } }"}'
```
Parse and convert price:
```bash
node -e "
const r = JSON.parse(\`<GRAPHQL_RESPONSE>\`);
const l = r.data.name.tokens[0].listings[0];
if (!l) { console.log('NO_LISTING'); process.exit(0); }
const hp = Number(BigInt(l.price)) / Math.pow(10, l.currency.decimals);
const buffer = hp * 0.02;
console.log(JSON.stringify({ externalId: l.externalId, price: hp.toFixed(2), buffer: buffer.toFixed(2), total: (hp + buffer).toFixed(2), symbol: l.currency.symbol }));
"
```
Use `externalId` as orderId, not `id`.

**No listing** → the domain exists on Doma but is not currently listed for sale. Ask:

```
AskUserQuestion: "<domain> is not listed for sale. What would you like to do?"
  → Make an offer to the owner (gasless, no bridging needed)
  → Cancel
```

If user wants to make an offer, go to the **Offer Flow** section below.

### 4. Check funds + bridge if needed

From Step 2 balances, determine what's needed:

```bash
node -e "
const domaETH = <DOMA_ETH_BALANCE>;
const domaUSDC = <DOMA_USDC_BALANCE>;
const listingPrice = <LISTING_PRICE>;
const needETH = domaETH < 0.001;
const needUSDC = domaUSDC < listingPrice;
const bridgeUSDC = needUSDC ? (listingPrice * 1.10).toFixed(2) : '0';
console.log(JSON.stringify({ needETH, needUSDC, bridgeETH: needETH ? '0.0005' : '0', bridgeUSDC, sufficient: !needETH && !needUSDC }));
"
```

**If sufficient** → go to Step 5.

**If insufficient** → show all balances, identify which chain has funds, and ask:

```
AskUserQuestion: "You need <listingPrice> <symbol> + gas on Doma. Your funds are on <chain>. Bridge?"
  → Yes, bridge from <chain> (ETH: <bridgeETH>, USDC: <bridgeUSDC>)
  → No, cancel
```

If user confirms, bridge in this order:

**ETH first** (needed for gas on subsequent bridge + purchase txs):
```bash
doma bridge ETH <bridgeETH> --from <chain> --to doma -y -f json
```

**Then USDC** — bridge listing price + 10% buffer to cover relay fees:
```bash
doma bridge USDC <bridgeUSDC> --from <chain> --to doma -y -f json
```

Bridge commands hang for 10-30s (CLI polls Relay internally). Do NOT interrupt.

Verify after bridge:
```bash
doma balance -f json
```

If still insufficient, tell user and offer to bridge more.

### 5. Confirm purchase

```
AskUserQuestion: "Buy <domain> for <price> <symbol>?"
  → Yes (Price: X.XX <symbol>. Fees deducted from seller. Balance: X.XX)
  → No, cancel
```

### 6. Execute purchase

**CLI method** (if `CLI_MARKETPLACE=yes`):
```bash
doma marketplace buy <domain> -y -f json
```

Done. The CLI handles everything: listing lookup, SDK init, Keychain key, Seaport execution.

**Fallback** (if `CLI_MARKETPLACE=no`):

Write a buy script dynamically for the chosen network, bundle with esbuild, and run. See [examples/buy-listing.ts](examples/buy-listing.ts) for the template. Key values to replace:
- `<CHAIN_ID>`: 97477 (mainnet) or 97476 (testnet)
- `<RPC_URL>`: `https://rpc.doma.xyz` or `https://rpc-testnet.doma.xyz`
- `<API_BASE>`: `https://api.doma.xyz` or `https://api-testnet.doma.xyz`

```bash
cd /tmp/doma-buy
npx esbuild buy.ts --bundle --platform=node --format=cjs --outfile=buy.cjs
node buy.cjs <orderId>
```

### 7. Verify

```bash
doma domain <domain> -f json
```

Owner may show zero address for 1-2 minutes (index lag). The TX is the source of truth.

```
✓ Purchase confirmed! TX: 0x...
  Manage at: app.doma.xyz/domain/<domain>
```

---

## Offer Flow (domain not listed)

**CLI method** (if `CLI_MARKETPLACE=yes`):
```bash
doma marketplace offer <domain> <amount> --currency USDC -y -f json
```

Gasless off-chain signature. No bridging, no ETH needed.

**Fallback** (if `CLI_MARKETPLACE=no`):

Ask user how much to offer, convert to smallest units (`amount × 10^decimals`), write offer script dynamically. See [examples/make-offer.ts](examples/make-offer.ts). Bundle and run with esbuild.

---

## Gotchas

Built from real production failures.

**Config:**
- `~/.doma/config.json` gets into bad states — `testnet: "false"` (string not boolean), `chainId: "97476"` while `apiUrl` points to devnet. After network selection, rewrite the entire file atomically with Step 1b. Never patch individual fields.
- API keys are network-specific. Validate before lookups.
- Private key in macOS Keychain — CLI reads it, but SDK scripts need `DOMA_PRIVATE_KEY` env var.

**Bridging:**
- Bridge ETH first, then USDC. USDC bridge needs ETH for gas on source chain.
- Relay deducts fees from bridged amount — bridge 10% extra.
- Never bridge $0 — errors. Use buffer calculation.
- Bridge takes 10-30s, not the ~2s estimate. CLI polls internally. Do NOT interrupt.

**Price:**
- Buyer pays listing price. Fees (0.5% protocol + 2.5% royalty) deducted from seller.
- GraphQL `price` is smallest units — divide by `10^decimals`. CLI `marketplace get` returns human-readable.
- Use `externalId`, not `id` for orderId.

**SDK (fallback only):**
- Requires `source`, `chains`, `apiClientOptions` with `defaultHeaders`.
- Private key must have `0x` prefix.
- ESM breaks on Node 22+ — always esbuild CJS.
- First buy requires token approval (2 txs).

**Doma wallet errors (`walletMode=agent`):**
- `Authorization revoked` (HTTP 402) — user revoked the session via `doma auth revoke` or the launchpad UI. Re-run `doma auth login` directly; the launchpad's consent screen is the actual re-authorization moment (the user clicks Authorize there).
- `Allowance exhausted` (HTTP 402) — per-session USD budget is spent. Tell the user `$N USD spent of $M ceiling`, then run `doma auth refill` to reset to the default ceiling. The next call succeeds.
- `Allowance not found` / `Session invalid` / `no_user_jwt` (HTTP 401) — session JWT is gone or stale. Run `doma auth login` to mint a fresh session (mode pick is standing consent).
- **`auth refill` recovery:** if `doma auth refill` errors with `invalid_refill_request` (a known bug that PR D3-6955/D3-6956 fixes on the launchpad backend + CLI), fall back to:
  ```
  doma auth revoke
  doma auth login
  ```
  The fresh session resets the $200 budget. The user has to re-authorize in the browser, but the flow works end-to-end on any launchpad version.
- Off-chain marketplace ops (`marketplace offer`, off-chain `cancel`, off-chain `accept` against an off-chain offer) currently fail Seaport's strict offerer-vs-signer check in this mode. On-chain `buy` and on-chain `cancel` work fine. If the user hits this, suggest temporarily switching to `private-key` mode for that one operation.
- **`doma marketplace offer` requires `walletMode=private-key`.** Seaport offers fail in agent mode because the embedded wallet's signer doesn't match the offerer address. DO NOT attempt offers in agent mode. Instead, instruct the user to set `DOMA_PRIVATE_KEY` and run `doma config set walletMode private-key` for the offer flow, then revert to `agent` for swaps if desired. `doma marketplace buy` and `doma marketplace cancel` work in both modes.

**Marketplace command compatibility:**

| Command | private-key | agent | Notes |
|---|---|---|---|
| `doma marketplace buy` | ✅ | ✅ | Works in both modes. |
| `doma marketplace cancel` | ✅ | ✅ | Works in both modes. |
| `doma marketplace offer` | ✅ | ❌ | Agent mode fails Seaport's offerer/signer check. Switch to private-key. |
| `doma marketplace accept` | ✅ | ❌ | Same reason as offer. |

If unsure which mode the user is in, run `doma auth status` — `Wallet mode: ...` is the first line.

**Approvals:**

Buying a domain on Doma involves Seaport. The buyer-side approval target is the **Seaport contract directly** at `0x0000000000000068F116a894984e2DB1123eB395` (canonical Seaport 1.6 — same on Doma testnet, Doma mainnet, and every EVM chain Seaport is deployed on). Payment-token approval (WETH/USDC) goes to that address; the buyer's `fulfillerConduitKey` is `bytes32(0)` (the no-conduit path).

`doma marketplace buy` and `doma marketplace offer` delegate to `@doma-protocol/orderbook-sdk` for the buy. The SDK auto-handles approval internally — you don't normally think about it. **Note**: the SDK is a black box from doma-cli's perspective (no explicit approve calls in `src/clients/orderbook.ts`); if the SDK's internal default ever changes (e.g. starts using a non-zero `fulfillerConduitKey`), the auto-approve target could shift. If you hit an `allowance`-shaped error on a buy/offer, the safe manual fallback is approving Seaport itself directly (see below).

**Manual fallback** (if the SDK's auto-approve fails or you want to pre-approve outside the buy flow): approve the payment token directly to the Seaport contract — same address on every chain:
```
doma approve USDC --spender 0x0000000000000068F116a894984e2DB1123eB395 --yes
doma approve USDC --spender 0x0000000000000068F116a894984e2DB1123eB395 --check
doma approve USDC --spender 0x0000000000000068F116a894984e2DB1123eB395 --revoke --yes
```

Substitute the relevant payment token symbol (USDC, USDTEST on Doma testnet, WETH, etc.). This is the "no-conduit fulfillment" path — verified in Doma's own reference implementation. It works regardless of what `@doma-protocol/orderbook-sdk` does internally, because `fulfillerConduitKey = bytes32(0)` makes Seaport pull payment via its own allowance.

**Two things buyers never have to think about:**

1. **`extraData`** — Seaport on Doma requires a non-empty `extraData` blob per-fulfiller-address. The Doma orderbook REST API returns it embedded in the listing payload, and `@doma-protocol/orderbook-sdk` plumbs it through. If you're using `doma marketplace buy`, you'll never see this field; it's only relevant if you're building a direct Seaport integration outside the CLI.
2. **`setApprovalForAll` on the domain NFT** — this is the **seller's** approval (granting Seaport permission to transfer the NFT to the buyer). Sellers do it once when creating a listing, typically via the Doma web app. Buyers never call `setApprovalForAll`; `doma approve` doesn't support it (it's only for ERC20 `approve`). If you need to revoke an NFT approval, use the launchpad web UI.

For the more common pre-acquire flow — buying or topping up the payment token via Uniswap V3 — the spender defaults to Permit2 (`0x000000000022D473030F116dDEE9F6B43aC78BA3`):

```bash
doma approve USDC --yes              # approve USDC to Permit2 with maxUint160
doma approve USDC --check            # read-only verification
doma approve USDC --revoke --yes     # set to 0
```

All variants work in both `walletMode=private-key` and `walletMode=agent`. Routing differs by sub-command:

- **`--check`** is a read-only RPC call (`allowance(owner, spender)`) — direct to the public RPC in BOTH wallet modes. It does NOT hit the launchpad's agent endpoint and does NOT consume any session allowance. Safe to call at any time, including after a session has expired.
- **`--yes` / `--amount` / `--revoke`** (write paths) — in `walletMode=agent`, the approval transaction routes through the launchpad's agent endpoint, which prices the approve at $0 (approvals don't move funds, so they don't consume your session allowance). In `walletMode=private-key`, the CLI signs and broadcasts directly.

**Native token (ETH) handling:**

Domain listings on Doma's Seaport marketplace can be priced in three currencies: ETH (native), WETH (wrapped), or USDC. Each has different approval semantics:

| Payment token | Approval needed? | Notes |
|---|---|---|
| **ETH** (native) | **No** | Buyer sends ETH as `msg.value`. The CLI handles this directly — no `doma approve` step. Do NOT call `doma approve ETH`; it will error correctly. |
| **WETH** | Yes | Approve WETH to **Seaport directly** (`0x0000000000000068F116a894984e2DB1123eB395`, canonical). CLI auto-handles within `doma marketplace buy/offer` via `@doma-protocol/orderbook-sdk`. |
| **USDC** | Yes | Approve USDC (or USDTEST on Doma testnet) to **Seaport directly** at the same address. CLI auto-handles via the SDK. |

For ETH-priced buys, the buyer wallet's ETH balance must cover the listing price PLUS gas. The pre-flight check (inside `doma swap`, not currently inside `doma marketplace buy` — see the Balance pre-flight subgroup) handles the combined requirement when native is the input.

When parsing a `doma marketplace buy` failure for an ETH-priced listing, look for `"insufficient funds for gas * price + value"` in the stderr — that's the classic native-balance-too-low signal. Surface to the user as "your wallet doesn't have enough ETH to cover the price plus gas".

**When approvals matter:**

The CLI's offer/buy flow auto-approves on first use against the right spender. You only need `doma approve` if:

- You want to pre-approve before bulk-placing offers (script-friendly: "approve once, place many").
- The auto-approve step fails and you want to retry just the approval leg.
- You're hygiene-revoking after a session.

**Balance pre-flight:**

`doma swap` and `doma approve` run a structured balance check before broadcasting. If the wallet has insufficient ETH (for gas) or insufficient input-token balance, the CLI exits with code 2 and a machine-readable error.

(`doma marketplace buy` and `doma marketplace offer` do NOT currently run this pre-flight — they'll surface balance failures as generic RPC errors with exit code 1. If you're using a marketplace command and want a balance pre-check, run `doma balance <token>` first.)

Exit codes:
- `0` — success
- `1` — generic error (invalid args, RPC down, tx revert, etc.) — **may also indicate balance issues** if pre-flight failed silently (see below)
- `2` — insufficient balance (clean structured JSON, this section)

**Exit-1 fallback for balance issues:** the pre-flight is best-effort. If `estimateGas` fails for any reason (RPC blip, network flake), the CLI logs a warning and proceeds to broadcast; the real out-of-gas error then surfaces with exit code **1**, not 2. So when parsing CLI failures, **also check exit-1 stderr for strings like `"insufficient funds for gas"`, `"transfer amount exceeds balance"`, or `"revert"`** — these indicate balance/gas issues that pre-flight didn't catch.

Error shape in `--format json` mode (written to stdout):

```json
{
  "error": "insufficient_balance",
  "details": {
    "type": "gas",
    "token_symbol": "ETH",
    "balance_human": "0.0001",
    "balance_raw": "100000000000000",
    "required_human": "0.01",
    "required_raw": "10000000000000000",
    "wallet": "0x...",
    "chain": { "name": "Doma Testnet", "chainId": 97476 }
  }
}
```

`details.type` is `"gas"` (native ETH for tx fee) or `"input_token"` (the ERC20 the user is paying with — typically USDC or WETH for marketplace buys).

**What to do as a skill consumer:** check exit code first. If it's `2`, parse the JSON for `error === "insufficient_balance"` and surface:

> You don't have enough `{details.token_symbol}` to complete this purchase. You have `{details.balance_human}`, you need at least `{details.required_human}`. Top up the wallet at `{details.wallet}` on `{details.chain.name}` and retry.

For marketplace buys specifically, the input_token is whatever you're paying with (WETH for ETH-priced listings, USDC for stablecoin-priced ones). The shape is the same regardless.

**Token-specific edges:**

- **Fee-on-transfer tokens** (rare): pre-flight balance check reads `balanceOf` only and won't catch a token that takes a fee on `transferFrom`. Swap can pass pre-flight then revert at broadcast with exit code 1 + stderr containing "revert". If this happens for a known-FoT token, suggest a smaller amount or contact Doma support.
- **USDT-style stale-allowance tokens**: USDT itself isn't a payment currency on Doma's marketplace today (listings price in ETH/WETH/USDC/USDTEST). This only matters if a future listing accepts a USDT-style token, or if a user supplies a custom payment-token address. Such tokens require `approve(spender, 0)` before any non-zero new approval. If `doma approve <custom-token> --amount X` ever fails with a revert about "non-zero allowance", first run `doma approve <custom-token> --revoke --yes`, then retry the new amount. Do NOT surface USDT to users as a supported buy currency unless the listing explicitly priced in it.
- **Pausable / blacklisted tokens**: a `transferFrom` revert can mean the token contract is paused or the user is blacklisted. The error surfaces as exit 1 with revert reason in stderr — surface verbatim to the user; not all reverts are actionable from the CLI side.

## Resources

- [examples/buy-listing.ts](examples/buy-listing.ts) — Buy script (fallback)
- [examples/make-offer.ts](examples/make-offer.ts) — Offer script (fallback)
- [reference/contracts.md](reference/contracts.md) — Contract addresses
- [reference/api-endpoints.md](reference/api-endpoints.md) — API endpoints
