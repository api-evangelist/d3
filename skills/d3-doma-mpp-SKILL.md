---
name: doma-mpp
description: How to consume the Doma MPP domain registration API — a payment-gated endpoint using the Machine Payments Protocol (HTTP 402). Use this skill whenever the user wants to register domains via MPP, call the Doma /register API, make paid API requests with mppx, or work with HTTP 402 payment flows for domain registration. Also trigger when code imports mppx, mppx/client, or the user mentions MPP, paid APIs, or payment-gated endpoints in the context of domain registration.
metadata:
  version: "1.0.0"
---

# Consuming the Doma MPP /register API

The Doma domain registration API is gated behind the Machine Payments Protocol (MPP). When you call the endpoint, the server returns HTTP 402 Payment Required with a payment challenge. The `mppx` client library handles this automatically — it pays via Tempo stablecoins and retries the request, so your code just calls `fetch()` and gets back the result.

## Setup

Install dependencies:

```bash
npm install mppx viem
```

Create an account from a private key:

```ts
import { privateKeyToAccount } from "viem/accounts";
import { Mppx, tempo } from "mppx/client";

const account = privateKeyToAccount("0x<PRIVATE_KEY>");

// This polyfills global fetch to auto-handle 402 payment challenges
Mppx.create({ methods: [tempo({ account })] });
```

If you prefer not to polyfill global fetch:

```ts
const mppx = Mppx.create({
  polyfill: false,
  methods: [tempo({ account })],
});

// Use mppx.fetch instead of global fetch
const response = await mppx.fetch(url);
```

## The /register endpoint

```
POST /register
Content-Type: application/json

{ "domain": "example.com", "buyerAddress": "0x...", "contact": { ... } }
```

| Parameter      | Location | Required | Description                                                                |
| -------------- | -------- | -------- | -------------------------------------------------------------------------- |
| `domain`       | body     | yes      | Full domain including TLD (e.g. `example.com`)                             |
| `buyerAddress` | body     | yes      | EVM address of the domain buyer (0x-prefixed, 40 hex chars)                |
| `contact`      | body     | yes      | Object with registrant contact info (see required fields below)            |

**Supported TLDs:** `com`, `xyz`, `ai`, `io`, `net`, `cash`, `live`, `fyi`

**Cost:** Dynamic — the server looks up the real USD price from the Interstellar.xyz registrar API. Payment is handled automatically via MPP/Tempo.

### Registrant contact format

The `contact` parameter is a JSON object with ICANN registrant contact fields.

**Required fields:** `firstName`, `lastName`, `email`, `phone`, `phoneCountryCode`, `street`, `city`, `state`, `postalCode`, `countryCode`

**Optional fields:** `organization`, `fax`, `faxCountryCode`

```ts
interface RegistrantContact {
  firstName: string;      // required
  lastName: string;       // required
  email: string;          // required
  phone: string;          // required
  phoneCountryCode: string; // required, e.g. "+1"
  street: string;         // required
  city: string;           // required
  state: string;          // required
  postalCode: string;     // required
  countryCode: string;    // required, ISO 3166-1 alpha-2, e.g. "US"
  organization: string;   // optional
  fax: string;            // optional
  faxCountryCode: string; // optional
}
```

### Example request

```ts
import { privateKeyToAccount } from "viem/accounts";
import { Mppx, tempo } from "mppx/client";

const account = privateKeyToAccount("0x<PRIVATE_KEY>");
Mppx.create({ methods: [tempo({ account })] });

const domain = "example.com";
const buyerAddress = account.address;

const contact = {
  firstName: "Jane",
  lastName: "Smith",
  organization: "Acme Inc",
  email: "jane@acme.com",
  phone: "4151234567",
  phoneCountryCode: "+1",
  fax: "4151234567",
  faxCountryCode: "+1",
  street: "123 Main St",
  city: "San Francisco",
  state: "CA",
  postalCode: "94103",
  countryCode: "US",
};

const response = await fetch("https://mpp-testnet.doma.xyz/register", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ domain, buyerAddress, contact }),
});
const data = await response.json();
console.log(data);
```

### Success response (200)

```json
{
  "success": true,
  "domain": "example.com",
  "network": "testnet",
  "txHash": "0xabc123...",
  "paymentContract": "0x9aC6761B5A1006E09C60a0BE10cd1C9d32911e96",
  "voucherAmount": "29990000000000000000",
  "order": {
    "domain": "example.com",
    "amount": "29.99",
    "buyerAddress": "0x...",
    "registrantContact": { ... },
    "voucher": { ... },
    "voucherSignature": "0x..."
  }
}
```

### Error responses

| Status  | Meaning                                                                          |
| ------- | -------------------------------------------------------------------------------- |
| **400** | Missing/invalid domain, buyerAddress, contact (or missing required contact fields), unsupported TLD, or invalid order metadata |
| **402** | Payment required (handled automatically by `mppx`)                               |
| **409** | Domain not available for registration                                            |
| **500** | On-chain registration transaction failed                                         |

## How the payment flow works under the hood

You don't need to manage this yourself — `mppx` does it automatically — but for understanding:

1. Client sends `POST https://mpp-testnet.doma.xyz/register` with JSON body `{ domain, buyerAddress, contact }`
2. Server checks domain availability via the Interstellar.xyz Partner API and creates a payment order (voucher)
3. Server returns **402 Payment Required** with a `WWW-Authenticate` header containing a payment challenge. The challenge includes HMAC-signed metadata (domain, amount, voucher) so the order survives the retry.
4. `mppx` extracts the challenge, signs a payment credential using the Tempo account
5. `mppx` retries the request with an `Authorization` header containing the credential
6. Server reconstructs the order from the HMAC-verified challenge metadata, executes the on-chain payment contract (`pay()` function), waits for the transaction receipt, and returns the result

## Reading the payment receipt

After a successful request, you can inspect the payment receipt:

```ts
import { Receipt } from "mppx";

const response = await fetch("https://mpp-testnet.doma.xyz/register", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ domain: "example.com", buyerAddress: "0x...", contact: { /* ... */ } }),
});
const receipt = Receipt.fromResponse(response);

console.log(receipt.status); // "success"
console.log(receipt.reference); // transaction hash
console.log(receipt.timestamp); // payment timestamp
```

## Endpoints

| Environment | Base URL                       |
| ----------- | ------------------------------ |
| Testnet     | `https://mpp-testnet.doma.xyz` |
| Mainnet     | `https://mpp.doma.xyz`         |

## CLI usage

You can also test from the command line:

```bash
npx mppx account create    # Create a Tempo account (auto-funded on testnet)
npx mppx -X POST -H "Content-Type: application/json" -d '{"domain":"example.com","buyerAddress":"0x...","contact":{...}}' "https://mpp-testnet.doma.xyz/register"
```

### Mainnet CLI usage

The `mppx` CLI defaults to the Tempo **testnet** chain when no RPC URL is provided. For mainnet requests (against `mpp.doma.xyz`), you **must** specify the mainnet RPC — otherwise the CLI will connect to testnet and fail because the mainnet USDC token doesn't exist on testnet.

Use the `-r` flag to specify the mainnet RPC:

```bash
npx mppx --account main -r https://rpc.tempo.xyz -X POST -H "Content-Type: application/json" -d '{"domain":"example.com","buyerAddress":"0x...","contact":{...}}' "https://mpp.doma.xyz/register"
```

| Network | RPC URL                          |
| ------- | -------------------------------- |
| Mainnet | `https://rpc.tempo.xyz`          |
| Testnet | `https://rpc.moderato.tempo.xyz` |

## After Registration: Managing Your Domain

After registering a domain, use the `doma` CLI to manage DNS records and nameservers. For full CLI documentation including token trading, subdomain management, bridging, and more, install the Doma skill:

```bash
npx skills add d3-inc/doma-skill
```

### Install Doma CLI

```bash
npm install -g @doma-protocol/cli
```

### Configure

```bash
doma config set privateKey "0x..."
```

The private key should be for the wallet where the domain was tokenized on the Doma chain. This defaults to the `buyerAddress` wallet used for MPP registration. On macOS, the private key is stored in Keychain.

### Manage DNS Records

Once using Doma nameservers, set DNS records on-chain:

```bash
doma dns set example.com @ A 76.76.21.21 -y          # Apex A record
doma dns set example.com www CNAME cname.vercel-dns.com -y  # Subdomain CNAME
doma dns list example.com                              # View all records
```

DNS and nameserver write operations on the Doma chain use sponsored gas (gasless) via ERC-4337 — no ETH balance is needed on Doma for these operations.

### Set Nameservers

Point your domain to custom nameservers:

```bash
doma nameservers set example.com ns1.myprovider.com ns2.myprovider.com
```
