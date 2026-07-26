---
name: doma-trade-tokens
description: >
  Buy or sell fractionalized Doma domain tokens across every venue — bonding
  curve launchpad, failed launch (sellOnFail), and Uniswap V3 post-graduation.
  Trigger on: buy tokens, sell tokens, swap tokens, trade fractional,
  purchase fractional, fractional buy, launchpad, bonding curve, sellOnFail,
  graduated, Uniswap, post-graduation, failed launch. Uses doma CLI, not web
  search.
license: MIT
metadata:
  author: doma-protocol
  version: "2.1.0"
compatibility: Requires Node.js 18+ and the latest @doma-protocol/cli (the skill verifies via `npm view` at runtime and prompts the user to upgrade if the local version is behind).
allowed-tools: Bash(doma *), Bash(npx -y @doma-protocol/cli *), Bash(which doma), Bash(node *)
argument-hint: [domain-name]
---

# Doma Trade Tokens

Help users buy or sell fractionalized Doma domain tokens across every venue. A Doma domain can be tokenized and fractionalized; fractional tokens trade on a launchpad bonding curve, then graduate onto Uniswap V3. Some launches fail — those tokens can still be sold back at their original purchase price via `sellOnFail`. This skill covers all of it.

`doma swap` auto-detects launchpad vs Uniswap routing internally. Use `--quiet --format json` so every call emits pure JSON on stdout — no narration — and parse the result with `node -e`. Errors always go to stderr and surface to the user verbatim.

## Wallet modes

The `doma` CLI signs writes in one of two modes. The "Doma wallet" path lets the user sign through the same Doma wallet they already use in the launchpad — the launchpad signs on their behalf under a USD-capped allowance, no key on the user's machine. The "private key" path is the user's own raw key, set as an env var.

| Mode | What signs the tx | How the user sets it up |
|---|---|---|
| `agent` (Doma wallet) | Doma launchpad signs on the user's behalf; no key on the user's machine | `doma auth login` (browser consent for a $200 USD spend ceiling) → mints session into `~/.doma/credentials.json` |
| `private-key` (default) | Local raw key, signed with viem/ethers | `export DOMA_PRIVATE_KEY=...` and restart the session |

**CLI version.** The Doma wallet path lands in `@doma-protocol/cli` 0.5.0. At skill version 2.1.0, npm `latest` was 0.4.0 (no `auth` command) — so any install pulled from npm today is private-key-only. The skill runtime-detects via `doma auth --help` exit code (the `AUTH_CAPABLE` check in Common Setup Step 3) instead of parsing the version string, so this section stays correct once 0.5.0 ships.

### Choosing a mode

Ask exactly once when the user has neither configured `walletMode` nor exported `DOMA_PRIVATE_KEY` (Common Setup Step 3's bash output disambiguates this). Anyone with `walletMode` already set or a key already exported skips this — their existing setup is the answer.

```
AskUserQuestion: "You already use Doma — would you like to sign with your Doma wallet here, or insert a private key?"
  → Use my Doma wallet — opens a browser to authorize a $200 USD spend ceiling on Doma + Base chains
  → Insert my own private key — set DOMA_PRIVATE_KEY yourself outside this chat, then restart
```

- **Doma wallet** → run `doma auth login` directly. The CLI opens the launchpad's consent screen in the user's default browser; the user clicks Authorize there (that screen is the real authorization moment — no separate "are you sure" prompt from the skill). On `Authorized.`, run `doma config set walletMode agent` so the choice persists, then proceed. If a session is later missing (revoked / expired), re-run `auth login` directly without re-asking — picking "Doma wallet" is standing consent (rule #4).
- **Private key** → tell the user the steps and stop until they confirm:
  1. `export DOMA_PRIVATE_KEY=0x<64-hex>` in their shell.
  2. (Optional) `doma config set walletMode private-key` — already the default, but explicit.
  3. Restart this conversation and re-run the skill.
   Never paste the key in chat. Don't proceed until they confirm the key is set.

`doma swap` (the only write in this skill) goes through the chosen mode; `doma token` / `doma quote` are read-only and hit the public RPC directly in both modes. The discover → preview → confirm → execute flow is byte-identical; only auth setup and a few error patterns (see Gotchas) differ.

## Rules

1. **Ask, don't assume** — always ask network, always ask intent (buy or sell), always preview before executing, always confirm before spending.
2. **One network per session** — once chosen, every command and URL stays on that network. Never mention the other.
3. **Private keys** — never handle, log, or display. Direct the user to set `DOMA_PRIVATE_KEY` themselves.
4. **Wallet mode is the user's choice, asked once** — see Wallet modes → Choosing a mode. After they pick, don't re-ask and don't switch modes on their behalf. Picking "Doma wallet" is standing consent for `doma auth login` whenever a session is later missing — the launchpad's consent screen is the actual authorization moment, not a skill-level prompt.
5. **Clean language** — no error codes, ticket numbers, stack traces, or developer jargon surfaced to the user.
6. **Numbers from JSON, never by hand** — parse `doma token` / `doma quote` JSON with `node -e`. Do not compute prices, decimals, or slippage bounds mentally.
7. **State is authoritative** — route by `status` + `tradingVenue` + `fillPercent` from `doma token`. Never assume a venue based on the domain name, the blog, or stale knowledge.
8. **Sell discipline** — sells follow the same preview → confirm → execute sequence as buys. sellOnFail has no slippage choice (fixed refund rate), but still confirm before executing.
9. **Never mix venues in one session** — once a token has graduated, do not reference its bonding curve. If GRADUATION_FAILED, explain sellOnFail semantics before executing.

---

## State × Action Matrix

Run `doma token <domain> --quiet --format json` first. Read `status` + `tradingVenue` + `fillPercent`. Then jump to the section below matching the user's intent (buy or sell):

**Check `status` FIRST.** Some `status` values (like `BOUGHT_OUT`) co-exist with a truthy `graduatedAt`, which would otherwise mislead the agent into routing to the Uniswap flows. Always branch on `status` before looking at `tradingVenue` or `graduatedAt`.

| If `status` is… | `tradingVenue` / extra | `fillPercent` | Buy | Sell |
|-----------------|------------------------|---------------|-----|------|
| `BOUGHT_OUT` (has `boughtOutAt` set) | any (often Uniswap V3) | — | refuse — already redeemed | refuse — already redeemed |
| `FRACTIONALIZED` | Launchpad (bonding curve) | < 100% | → §Flow-A Launchpad Buy | → §Flow-B Launchpad Sell |
| `FRACTIONALIZED` | Launchpad (bonding curve) | = 100% | refuse — graduation pending | refuse — graduation pending |
| `GRADUATION_FAILED` | Launchpad (failed - sell only) | — | refuse — launch failed | → §Flow-C sellOnFail |
| `GRADUATION_SUCCESSFUL` or `graduatedAt` set | Uniswap V3 | — | → §Flow-D Uniswap Buy | → §Flow-E Uniswap Sell |
| anything else (null, unknown) | — | — | refuse — unknown state, do not guess | refuse — unknown state, do not guess |

For refused cases, tell the user the exact reason and stop:

- **Bought out**: "This fractional token was bought out on `<boughtOutAt>` — the underlying domain NFT has been redeemed by a single buyer at the buyout price. The fractional token is no longer tradeable. If you already hold tokens from before the buyout, they represent a pro-rata claim on the buyout proceeds (check the token contract for the claim function, or contact Doma support)."
- **Graduation pending**: "The bonding curve is full and graduation is in progress. The token is not tradeable for the next few minutes. Try again shortly."
- **Launch failed, buy attempt**: "This launch didn't fill before its deadline. You can't buy it, but if you already own tokens you can sell them back at your original purchase price via this skill."
- **Launch failed, sell attempt**: route to Flow-C.
- **Unknown state**: "`doma token <domain>` returned an unexpected state (`<status value>`). I won't guess which flow applies — please check the domain on the Doma dashboard and try again."

---

## Common Setup

All flows start here.

### 1. Ask network

```
AskUserQuestion: "Which network is this domain on?"
  → Mainnet (chain 97477, api.doma.xyz)
  → Testnet (chain 97476, api-testnet.doma.xyz)
```

### 2. Set config

Rewrite config atomically — never patch single fields. Substitute `<NETWORK>` below with `mainnet` or `testnet` (exactly one of those two strings, matching the answer from Step 1) before running the command. Every other value is hard-coded from the network mapping and requires no substitution.

```bash
node -e "
const fs = require('fs');
const path = require('path');
const network = '<NETWORK>'; // substitute 'mainnet' or 'testnet' before running
const isTestnet = network !== 'mainnet';
const configPath = path.join(require('os').homedir(), '.doma', 'config.json');
const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
config.testnet = isTestnet;
config.chainId = isTestnet ? '97476' : '97477';
config.apiUrl = isTestnet ? 'https://api-testnet.doma.xyz' : 'https://api.doma.xyz';
fs.writeFileSync(configPath, JSON.stringify(config, null, 2));
console.log('Config updated for ' + network);
"
```

### 3. Check prerequisites

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

- **CLI missing** → install (`npm i -g @doma-protocol/cli@latest`) or use `npx -y @doma-protocol/cli`.
- **API key** → already in `~/.doma/config.json` for most users. The CLI reads it automatically.

### 4. Ask intent

```
AskUserQuestion: "Do you want to buy or sell <domain> tokens?"
  → Buy
  → Sell
```

### 5. Discover state

```bash
doma token <domain> --quiet --format json
```

Parse:

```bash
echo '<JSON>' | node -e "
const t = JSON.parse(require('fs').readFileSync('/dev/stdin','utf8'));
console.log(JSON.stringify({
  status: t.status,
  venue: t.tradingVenue,
  fillPercent: t.fillPercent ?? null,
  graduated: !!t.graduatedAt,
  boughtOut: !!t.boughtOutAt,
  boughtOutAt: t.boughtOutAt ?? null,
  poolAddress: t.poolAddress,
  launchpadAddress: t.launchpadAddress,
  priceUsd: t.priceUsd
}));
"
```

Use the §State × Action Matrix above to route. Each flow below assumes you've reached it legitimately.

---

## Flow-A: Launchpad Buy

`FRACTIONALIZED`, `fillPercent < 100%`, user wants to buy.

### 1. Ask spend amount

```
AskUserQuestion: "How much USDC do you want to spend on <domain>?"
  → <user enters amount>
```

### 2. Preview

```bash
doma quote USDC <domain> <usdc-amount> --quiet --format json
```

Parse and present: `venue`, `amountIn`, `amountOut`, `executionPrice`, `feeBps`, `fillPercent`. Call out any of:
- `priceImpact > 5%` as a risk (if the quote returns one — launchpad quotes don't, but two-step routing may).
- `fillPercent > 95%` as a graduation-race risk.
- `venue: "two-step"` as a multi-tx buy — if this appears, tell the user before the slippage question: "This buy requires two on-chain transactions — first swapping your input token to USDC via Uniswap, then buying domain tokens on the launchpad. Price can move between the two steps. If the first step succeeds but the second reverts, you will hold USDC instead of domain tokens." Then continue to Step 3.

### 3. Slippage

```
AskUserQuestion: "Slippage tolerance? (default 100 bps / 1% for launchpad)"
  → 100 (default)
  → <user enters bps>
```

Launchpad default is higher than Uniswap's (50 bps) because the curve moves faster. Apply this rule before asking the question:

- If `fillPercent < 20%` (from the Step 5 state check), suggest 200–500 bps as the default instead of 100 — thin curves amplify every buy.
- If `fillPercent > 95%`, suggest 100 bps but also warn about the graduation-race risk (see Gotcha "Graduation race").
- Otherwise, 100 bps.

### 4. Confirm

```
AskUserQuestion: "Buy ~<amountOut> <symbol> for <amountIn> USDC at up to <slippage-pct>% slippage?"
  → Yes
  → No, cancel
```

### 5. Execute

```bash
doma swap USDC <domain> <usdc-amount> --yes --quiet --slippage <bps> --format json
```

Expected success JSON:
```json
{"Status":"Success","Venue":"Launchpad","Tx Hash":"https://<explorer>/tx/0x...","Gas Used":"...","Block":"..."}
```

If the CLI writes to stderr, or the JSON response has `"Status":"Failed"`, surface the message verbatim — no retry, no rewording.

### 6. Report

On success:
```
✓ Purchase confirmed!
  Received ~<amountOut> <symbol> for <amountIn> USDC.
  Venue: Launchpad
  TX: <Tx Hash>
```

On failure: surface the CLI's stderr verbatim. Suggest the user check balance, slippage, or that the curve hasn't graduated while they were confirming. Stop. Do not retry automatically.

---

## Flow-B: Launchpad Sell

`FRACTIONALIZED`, `fillPercent < 100%`, user wants to sell.

### 1. Ask amount

```
AskUserQuestion: "How many <symbol> do you want to sell?"
  → <user enters token amount>
```

### 2. Preview

```bash
doma quote <domain> USDC <token-amount> --quiet --format json
```

Parse and present: `amountIn`, `amountOut` (USDC), `executionPrice`, `feeBps` (sell fee). The curve moves down as you sell — large sales cause meaningful slippage. Call out:
- `fillPercent > 95%` as a graduation-race risk — a sell at 99% fill can race with the final buyer and revert if the curve graduates between quote and execute. Warn the user before continuing.

### 3. Slippage

```
AskUserQuestion: "Slippage tolerance? (default 100 bps / 1% for launchpad)"
  → 100 (default)
  → <user enters bps>
```

### 4. Confirm

```
AskUserQuestion: "Sell <token-amount> <symbol> for ~<amountOut> USDC at up to <slippage-pct>% slippage?"
  → Yes
  → No, cancel
```

### 5. Execute

```bash
doma swap <domain> USDC <token-amount> --yes --quiet --slippage <bps> --format json
```

If the CLI writes to stderr, or the JSON response has `"Status":"Failed"`, surface the message verbatim — no retry, no rewording.

### 6. Report

On success:
```
✓ Sale confirmed!
  Sold <token-amount> <symbol> for ~<amountOut> USDC.
  Venue: Launchpad
  TX: <Tx Hash>
```

On failure: surface the CLI's stderr verbatim. Suggest the user check balance, slippage, or that the curve hasn't graduated while they were executing. Stop. Do not retry automatically.

---

## Flow-C: sellOnFail

`GRADUATION_FAILED`, user wants to sell (refund).

No quote step — the refund rate is fixed at the original purchase price (not current market). Explain this before executing.

### 1. Explain semantics

> "This launch didn't fill before its deadline. The `sellOnFail` contract lets you return your tokens for exactly what you paid — USDC in, at your original purchase rate. You won't get more if the market moved, and you won't get less. Continue?"

### 2. Ask amount

```
AskUserQuestion: "How many <symbol> do you want to sell back?"
  → <user enters token amount>
```

### 3. Confirm

```
AskUserQuestion: "Sell back <token-amount> <symbol> at your original purchase rate?"
  → Yes
  → No, cancel
```

### 4. Execute

```bash
doma swap <domain> USDC <token-amount> --yes --quiet --format json
```

The CLI auto-detects `GRADUATION_FAILED` status and routes to `sellOnFail` internally. No `--slippage` flag needed — there's no slippage on a fixed refund.

If the CLI writes to stderr, or the JSON response has `"Status":"Failed"`, surface the message verbatim — no retry, no rewording.

### 5. Report

On success:
```
✓ Refund processed.
  Returned <token-amount> <symbol>, received original purchase amount in USDC.
  Venue: Launchpad (sellOnFail)
  TX: <Tx Hash>
```

On failure: surface the CLI's stderr verbatim. Common causes: the user's wallet holds fewer than `<token-amount>` tokens, or the launchpad contract state is stale. Stop. Do not retry automatically.

---

## Flow-D: Uniswap Buy

Post-graduation, user wants to buy.

### 1. Ask spend amount

```
AskUserQuestion: "How much USDC do you want to spend on <domain>?"
  → <user enters amount>
```

### 2. Preview

```bash
doma quote USDC <domain> <usdc-amount> --quiet --format json
```

Parse and present: `amountIn`, `amountOut`, `executionPrice`, `route`, `priceImpact`. Call out price impact > 5% as a risk. If `amountOut === 0`, the Uniswap pool is effectively dry — see Gotchas and abort.

### 3. Slippage

```
AskUserQuestion: "Slippage tolerance? (default 50 bps / 0.5%)"
  → 50 (default)
  → <user enters bps>
```

Apply this rule before asking the question:

- If the Step 2 preview showed `priceImpact > 5%` or `amountOut` is small relative to the pool's TVL (thin pool), suggest 200–500 bps as the default instead of 50 — shallow Uniswap pools trip `TRANSFER_FROM_FAILED` or `Too little received` at 50 bps.
- Otherwise, 50 bps.

### 4. Confirm

```
AskUserQuestion: "Buy ~<amountOut> <symbol> for <amountIn> USDC at up to <slippage-pct>% slippage?"
  → Yes
  → No, cancel
```

### 5. Execute

```bash
doma swap USDC <domain> <usdc-amount> --yes --quiet --slippage <bps> --format json
```

Expected success JSON: `{"Status":"Success","Venue":"Uniswap V3","Tx Hash":"...","Gas Used":"...","Block":"..."}`.

If the CLI writes to stderr, or the JSON response has `"Status":"Failed"`, surface the message verbatim — no retry, no rewording.

### 6. Report

On success:
```
✓ Purchase confirmed!
  Received ~<amountOut> <symbol> for <amountIn> USDC.
  Venue: Uniswap V3
  TX: <Tx Hash>
```

On failure: surface the CLI's stderr verbatim. Suggest the user check balance, slippage, or pool liquidity, and stop. Do not retry automatically.

---

## Flow-E: Uniswap Sell

Post-graduation, user wants to sell.

### 1. Ask amount

```
AskUserQuestion: "How many <symbol> do you want to sell?"
  → <user enters token amount>
```

### 2. Preview

```bash
doma quote <domain> USDC <token-amount> --quiet --format json
```

Parse: `amountIn`, `amountOut` (USDC), `executionPrice`, `route`, `priceImpact`. Same price-impact checks as Flow-D.

### 3. Slippage

```
AskUserQuestion: "Slippage tolerance? (default 50 bps / 0.5%)"
  → 50 (default)
  → <user enters bps>
```

### 4. Confirm

```
AskUserQuestion: "Sell <token-amount> <symbol> for ~<amountOut> USDC at up to <slippage-pct>% slippage?"
  → Yes
  → No, cancel
```

### 5. Execute

```bash
doma swap <domain> USDC <token-amount> --yes --quiet --slippage <bps> --format json
```

If the CLI writes to stderr, or the JSON response has `"Status":"Failed"`, surface the message verbatim — no retry, no rewording.

### 6. Report

On success:
```
✓ Sale confirmed!
  Sold <token-amount> <symbol> for ~<amountOut> USDC.
  Venue: Uniswap V3
  TX: <Tx Hash>
```

On failure: surface the CLI's stderr verbatim. Suggest the user check balance, slippage, or pool liquidity, and stop. Do not retry automatically.

---

## Gotchas

Real failure modes. Check before blaming the CLI.

**Config:**
- `~/.doma/config.json` drifts between networks — `testnet: "false"` (string not boolean), or `chainId: "97476"` while `apiUrl` points at mainnet. Rewrite the whole file atomically in §Common Setup Step 2; never patch single fields.
- API keys are network-specific. A testnet key fails on mainnet. If lookups error with "Invalid API key", ask for the right one for the chosen network — don't retry.

**Private key:**
- macOS Keychain holds the CLI's key for interactive use, but `doma swap` reads it from `DOMA_PRIVATE_KEY` env when set. If you see `invalid private key, expected hex or 32 bytes, got string`, the env var is empty / malformed / still has a `"0x..."` placeholder. Must be `0x` + 64 hex chars (66 total). Validate:
  ```bash
  node -e "console.log(/^0x[0-9a-fA-F]{64}$/.test(process.env.DOMA_PRIVATE_KEY||''))"
  ```

**Pool liquidity (Uniswap):**
- `doma quote` returning `amountOut: 0`, `executionPrice: 0`, or `priceImpact` near 100 means the Uniswap pool has essentially no output-token liquidity. A swap will still submit, revert on-chain, and eat gas for ~0 tokens. Abort and tell the user the pool is dry.
- Very low-TVL pools accept buys but move price drastically. If `priceImpact > 5%` for a small order, the pool is thin — warn, suggest a smaller amount or higher slippage, and re-confirm before executing.

**Slippage (Uniswap and Launchpad market swaps only — does NOT apply to sellOnFail):**
- The default 50 bps (0.5%) is tuned for deep Uniswap pools. On a recently-graduated token with shallow liquidity, a 50 bps `amountOutMin` will often trip `TRANSFER_FROM_FAILED` or `Too little received`. Launchpad default is 100 bps for the same reason (bonding curve moves faster). If the user accepts higher impact, bump to 200–500 bps. sellOnFail has no slippage — the refund rate is fixed at the original purchase price.

**Bonding curve front-running:**
- Thin launchpad curves let price move 10%+ between quote and execute on a small order. If the quote was taken more than 30 seconds ago, re-quote before executing. Default 100 bps slippage is for moderately-filled curves — below 20% fill, bump to 200–500 bps.

**Graduation race:**
- When `fillPercent` is 95-100%, the `buy` can revert mid-flight because another buyer filled the remaining curve supply first. Re-quote right before execute. If `remainingSupply < amountIn` after the re-quote, warn the user that the available supply won't cover their full order, and re-confirm with the smaller number.

**Launchpad state out of sync with DB (pre-launch or post-expiry):**
- The launchpad contract has an on-chain state machine (`PRE_LAUNCH` → `LAUNCHED` → `LAUNCH_SUCCEEDED` / `LAUNCH_FAILED`) that is authoritative for whether buys/sells are allowed. The `doma token` JSON today reflects the backend's DB state, which is eventually consistent with on-chain state — there is a lag at both the start and the end of the sale window.
- **Pre-launch window**: `status: FRACTIONALIZED` appears as soon as the launchpad is deployed, but the sale doesn't open until `block.timestamp >= launchStartTime`. Buys before start revert with an undecoded custom error (commonly `0xa1fa02b3`). The `doma token` JSON does NOT expose `launchStartTime` today, so the skill can't refuse proactively.
- **Post-expiry, pre-backend-crank** (up to ~30 seconds): when `launchEndTime` passes and the curve hasn't filled, the launchpad transitions to `LAUNCH_FAILED` on-chain. The backend's `LaunchpadCrankTask` updates the DB to `status: GRADUATION_FAILED` on its next tick (every 30s), but between those two events the DB still says `FRACTIONALIZED`. Buys and market sells revert in this window; only `sellOnFail` works.
- **Recovery**: if a launchpad buy or sell reverts with `ContractFunctionExecutionError` + an unknown signature — especially when `fillPercent: 0` (untouched curve, likely pre-launch) or when the current time is near/past `launchEndTime` — the cause is one of these state-sync gaps, not slippage or bad calldata. Surface the revert to the user, explain the likely state (pre-launch or just-expired), and suggest either waiting for the sale to open or retrying in 30-60 seconds (which gives the backend crank time to move the DB to `GRADUATION_FAILED`, at which point Flow-C becomes available for refund-style sells).
- **Optional RPC verification** (if the agent has direct RPC access): call `launchStatus()` on the `launchpadAddress` — `0` = PRE_LAUNCH, `1` = LAUNCHED, `3` = LAUNCH_FAILED. Most agents won't do this; the DB-state + revert signature + time-window heuristic above is sufficient.

**Two-step routing:**
- Buying a launchpad token with non-USDC input (e.g., ETH) runs TWO on-chain transactions: Uniswap tokenIn → USDC, then Launchpad USDC → token. Price can move between the two. Explain this clearly. Both txs use the same slippage setting. If Step 1 succeeds but Step 2 reverts, the user will hold USDC (not the target token) — surface the exact state and don't retry automatically.

**Two-step buy revert recovery:**

Flow-A buys (non-USDC input → USDC → token) approve TWICE: tokenIn → Permit2 (step 1) and USDC → launchpad (step 2). If step 2 reverts (slippage, insufficient launchpad supply, etc.), both approvals remain outstanding on-chain.

To revoke both:
```
doma approve <tokenIn> --revoke --yes
doma approve USDC --spender <launchpad-addr> --revoke --yes
```

The launchpad address is in the swap output's `route` field or the quote JSON. Only revoke if the user wants to clean up — the next swap attempt would auto-re-approve anyway.

**Graduation detection:**
- `tradingVenue` is the authoritative signal, not the blog's "$250K threshold" (that's a narrative, not an on-chain invariant). The contract triggers migration on `tokensSold == launchTokensSupply`. Trust the JSON field.
- `status: "GRADUATION_FAILED"` means the launch deadline passed without filling. Refuse buy flows; route sell flows to Flow-C.

**sellOnFail semantics:**
- The CLI auto-detects `GRADUATION_FAILED` and calls `sellOnFail` under the hood — users don't pass a special flag. Refund amount is fixed at the original purchase price (not current market, not highest price reached). A user who bought at $1 and now sees a "market" price of $0.10 still gets $1 back per token (up to their original purchase count). Explain before executing.

**CLI flag interaction:**
- `doma swap --quiet` requires `--yes`. Without `--yes` the command exits 1 with `Error: --quiet requires --yes for non-interactive execution.` That guard exists because the confirm prompt would otherwise hang — this is expected behavior, not a bug. Pass both.
- Narration from `swap` goes to stdout without `--quiet`. Piping to `jq` without `--quiet` will fail on the first non-JSON line. Always use `--quiet --format json` when piping.

**Tx index lag:**
- After a successful swap, `doma token <domain>` may still show the pre-swap `tvlUsd` / `holders` / `tokensSold` for 1–2 minutes. The on-chain receipt is the source of truth; don't re-run on a whim.

**Doma wallet errors (`walletMode=agent`):**
- `Authorization revoked` (HTTP 402) — user revoked the session via `doma auth revoke` or the launchpad UI. Re-run `doma auth login` directly; the launchpad's consent screen is the actual re-authorization moment (the user clicks Authorize there).
- `Allowance exhausted` (HTTP 402) — per-session USD budget is spent. Tell the user `$N USD spent of $M ceiling`, then run `doma auth refill` to reset to the default ceiling. The next swap succeeds.
- `Allowance not found` / `Session invalid` / `no_user_jwt` (HTTP 401) — session JWT is gone or stale. Run `doma auth login` to mint a fresh session (mode pick is standing consent).
- Buys / sells in this mode go through the launchpad backend (`POST /api/agent/execute`) — the user's USD allowance ticks down by the on-chain spend per call. If their refill cycle is short, surface the remaining allowance from `doma auth status` before a large swap so they can refill proactively.

**`auth refill` recovery:**

If `doma auth refill` errors with `invalid_refill_request` (a known bug that PR D3-6955/D3-6956 fixes on the launchpad backend + CLI), fall back to:
```
doma auth revoke
doma auth login
```

The fresh session resets the $200 budget. The user has to re-authorize in the browser, but the flow works end-to-end on any launchpad version.

**Approvals:**

ERC20 swaps on Doma require an on-chain approval to Permit2 (`0x000000000022D473030F116dDEE9F6B43aC78BA3` — canonical, same on every chain) before the wallet can sell that token through the Universal Router. The CLI handles this automatically inside `doma swap` — you don't normally think about it.

If a swap ever fails with an approval-related error (rare; mostly historical, see "When this matters" below), or if you want to pre-approve a token outside the swap flow:

| Action | Command | Notes |
|---|---|---|
| Check the current on-chain allowance | `doma approve <token> --check` | No tx sent. Shows allowance against Permit2 by default. |
| Approve unlimited | `doma approve <token> --yes` | Sends `approve(Permit2, maxUint160)` — the canonical "unlimited" amount. One-time per token. |
| Approve a bounded amount | `doma approve <token> --amount 100 --yes` | Useful for testing or for users who prefer not to grant unlimited. The CLI will re-approve as needed before each swap. |
| Revoke (set to 0) | `doma approve <token> --revoke --yes` | Useful when migrating off a wallet, or as a security hygiene step. |
| Approve to a non-Permit2 spender | `doma approve <token> --spender <addr> --yes` | Escape hatch — for future flows (e.g. Seaport conduit, launchpad bonding-curve contracts). The default of Permit2 covers all Uniswap V3 swaps. |

All variants work in both `walletMode=private-key` and `walletMode=agent`. Routing differs by sub-command:

- **`--check`** is a read-only RPC call (`allowance(owner, spender)`) — direct to the public RPC in BOTH wallet modes. It does NOT hit the launchpad's agent endpoint and does NOT consume any session allowance. Safe to call at any time, including after a session has expired.
- **`--yes` / `--amount` / `--revoke`** (write paths) — in `walletMode=agent`, the actual approval transaction routes through the launchpad's agent endpoint, which prices the approve at $0 (approvals don't move funds, so they don't consume your session allowance). In `walletMode=private-key`, the CLI signs and broadcasts directly.

**Native token (ETH) handling:**

ETH (and any native token) doesn't have an ERC20 `approve()` — there's nothing to approve. The CLI handles native input automatically:

- `doma swap ETH <token> <amount>` skips the approval step entirely (the Universal Router pulls ETH via `msg.value`, not via `transferFrom`).
- `doma approve ETH` deliberately errors: `Error: native ETH cannot be approved (ERC20 approvals only).` This is correct behavior — don't try to recover from it; surface to the user as "ETH doesn't need approval" if they asked.

When ETH is the **input** to a swap, the wallet's ETH balance must cover BOTH the swap amount AND gas. The pre-flight checks the combined requirement and emits `details.required_human` accordingly (e.g. for swap 1 ETH with 0.001 ETH gas estimate, `required_human ≈ "1.001"`). If you parse an `insufficient_balance` error with `details.type === "gas"` on a native-input swap, present the FULL required amount to the user, not just the gas portion.

When ETH is the **output** of a swap (`doma swap USDC ETH …`), only the input token (USDC here) needs the standard Permit2 approval flow. The output is delivered directly to the wallet as native ETH; no unwrap or approval step required from the user.

**When approvals matter:**

The CLI's swap flow auto-approves on first use. You only need to think about `doma approve` if:

- The auto-approve step fails for any reason and you want to retry just the approval leg.
- You want to script "approve once, then swap many times" (e.g. for a fixed-budget trading loop).
- You want to audit or hygiene the on-chain allowance.
- You're on an older CLI build that pre-dates the agent-mode approval fix, and `doma swap` returns "Allowance exhausted ($0 of $X)" during the approval step. Workaround in that case: approve from the launchpad web UI (which bypasses the broken path), then re-run `doma swap`.

**Balance pre-flight:**

Both `doma swap` and `doma approve` run a structured balance check before broadcasting any tx. If the wallet has insufficient ETH (for gas) or insufficient input-token balance (for swap), the CLI exits with code 2 and a machine-readable error.

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

`details.type` is `"gas"` (native ETH for tx fee) or `"input_token"` (the ERC20 the user is paying with — typically USDC).

**What to do as a skill consumer:** check exit code first. If it's `2`, parse the JSON for `error === "insufficient_balance"` and surface:

> You don't have enough `{details.token_symbol}` to complete this {swap/approve}. You have `{details.balance_human}`, you need at least `{details.required_human}`. Top up the wallet at `{details.wallet}` on `{details.chain.name}` and retry.

Don't suggest `doma auth refill` for balance issues — that's for the agent-mode session allowance (a USD budget on the launchpad side), which is a different concept entirely. Refill is for "the launchpad session is out of budget"; `insufficient_balance` is "the wallet itself is out of tokens".

**Optional defensive pattern:** if you want to pre-check before even starting a swap (e.g. to avoid the user clicking through approval prompts on a wallet that can't afford the swap), run `doma balance <token>` first and verify the amounts. The pre-flight inside `doma swap` catches it either way, but an explicit pre-check is cheaper than starting the flow and bailing out.

**Token-specific edges:**

- **Fee-on-transfer tokens** (rare): pre-flight balance check reads `balanceOf` only and won't catch a token that takes a fee on `transferFrom`. Swap can pass pre-flight then revert at broadcast with exit code 1 + stderr containing "revert". If this happens for a known-FoT token, suggest a smaller amount or contact Doma support.
- **USDT-style stale-allowance tokens**: USDT itself isn't in Doma's well-known token list today (ETH/WETH/USDC/USDC.E/USDTEST), so this only matters if a user supplies a USDT-style token address directly (e.g. via a custom token deployment or a future asset listing). Such tokens require `approve(spender, 0)` before any non-zero new approval. If `doma approve <custom-token> --amount X` ever fails with a revert about "non-zero allowance", first run `doma approve <custom-token> --revoke --yes`, then retry the new amount. Do NOT surface USDT to users as a supported swap input unless they've explicitly provided a token contract.
- **Pausable / blacklisted tokens**: a `transferFrom` revert can mean the token contract is paused or the user is blacklisted. The error surfaces as exit 1 with revert reason in stderr — surface verbatim to the user; not all reverts are actionable from the CLI side.

## Resources

- [reference/lifecycle.md](reference/lifecycle.md) — Tokenization → fractionalization → graduation → Uniswap lifecycle
- [reference/state-machine.md](reference/state-machine.md) — The 4 states, their transitions, and on-chain signals
