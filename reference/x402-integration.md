# x402 Payment Integration

## What is x402?

An open payment protocol by Coinbase that uses HTTP 402 status codes for pay-per-request API access. Agents pay with USDC on Base — no accounts, API keys, or subscriptions needed.

- **Spec**: https://www.x402.org
- **Docs**: https://docs.cdp.coinbase.com/x402/welcome
- **GitHub**: https://github.com/coinbase/x402

## How It Works

```
1. Agent sends HTTP request to a service
2. Server responds: 402 Payment Required (with price, token, address)
3. Agent signs a USDC payment authorization (EIP-712)
4. Agent retries request with PAYMENT-SIGNATURE header
5. Server verifies payment via facilitator, returns resource
```

No actual transaction is broadcast by the agent — only a signed authorization. The facilitator settles on-chain.

## Setup (Node.js)

### Install

```bash
npm install @x402/axios @x402/evm @x402/core
```

### Basic Client

```javascript
import { x402Client, withPaymentInterceptor } from "@x402/axios";
import { registerExactEvmScheme } from "@x402/evm/exact/client";
import { privateKeyToAccount } from "viem/accounts";
import axios from "axios";

// Load private key from credentials
const signer = privateKeyToAccount(process.env.PRIVATE_KEY);

// Create x402 client
const client = new x402Client();
registerExactEvmScheme(client, { signer });

// Wrap axios with payment interceptor
const api = withPaymentInterceptor(axios.create(), client);

// Now any 402 response is handled automatically
const response = await api.get("https://some-x402-service.com/api/data");
```

### Using fetch wrapper

```bash
npm install @x402/fetch @x402/evm
```

```javascript
import { x402Client, x402Fetch } from "@x402/fetch";
import { registerExactEvmScheme } from "@x402/evm/exact/client";
import { privateKeyToAccount } from "viem/accounts";

const signer = privateKeyToAccount(process.env.PRIVATE_KEY);
const client = new x402Client();
registerExactEvmScheme(client, { signer });

const fetch402 = x402Fetch(client);
const response = await fetch402("https://some-x402-service.com/api/data");
```

## Setup (Python)

```bash
pip install x402
```

```python
from eth_account import Account
from x402.clients.requests import x402_requests

account = Account.from_key(PRIVATE_KEY)
session = x402_requests(account)

# Automatically handles 402 responses
response = session.get("https://some-x402-service.com/api/data")
```

## Budget Controls

Set daily spending limits to prevent runaway costs:

```javascript
// Example: cap at $1/day
const client = new x402Client({ maxDailySpend: 1.0 });
```

Or implement manually by tracking cumulative payments in credentials file.

## Requirements

- **USDC on Base**: The agent wallet must hold USDC on Base mainnet
- **USDC contract (Base)**: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
- Typical API costs: $0.001 - $0.01 per request

## Testing on Testnet

Use Base Sepolia for testing:

```javascript
// Testnet facilitator
const FACILITATOR_URL = "https://x402.org/facilitator";

// Get test USDC from Base Sepolia faucet
// https://docs.cdp.coinbase.com/x402/quickstart-for-buyers
```

## Discovering x402 Services

### x402 Bazaar

```bash
# Discover available paid services
curl https://bazaar.x402.org/api/listings
```

### x402 Ecosystem

Browse available services at https://www.x402.org/ecosystem

## Verifying x402 Capability

After setup, test with a known x402 endpoint:

```javascript
try {
  const response = await api.get("https://some-x402-service.com/test");
  console.log("x402 payment successful:", response.data);
} catch (e) {
  if (e.response?.status === 402) {
    console.error("Payment failed - check USDC balance");
  }
}
```

## Supported Networks

| Network | CAIP-2 ID | Status |
|---------|-----------|--------|
| Base | `eip155:8453` | Primary (recommended) |
| Ethereum | `eip155:1` | Supported |
| Base Sepolia | `eip155:84532` | Testnet |
| Solana | `solana:mainnet` | Supported (requires SVM signer) |
