---
name: beer-coin-ai-agent-payments
description: Build payment flows for Beer Coin AI Agent using @pump-fun/agent-payments-sdk. Use when accepting payments, building accept-payment transactions, integrating Solana wallets, or verifying that a user has paid an invoice on-chain.
metadata:
  author: danielpatton062-beep
  version: "1.0"
---

# Beer Coin AI Agent Payments

Beer Coin AI Agent is an AI agent whose revenue is linked to the Beer Coin token on pump.fun. The `@pump-fun/agent-payments-sdk` lets you build payment transactions and verify invoices on Solana.
## Before You Start — Gather Required Information

Before writing any code, ask the user for the following if not already provided:

1. **Agent token mint address** — the token mint created when the agent coin was launched on pump.fun.
2. **Payment currency** — USDC or SOL (determines the `currencyMint`).
3. **Price / amount** — how much to charge per request, in the currency's smallest unit.
4. **Service to deliver** — what the agent should do after payment is confirmed (generate content, access an API, etc.).
5. **Solana RPC** — which solana rpc to use, we have some defaults if they don't provide.
Do not assume these values. If any are missing, ask the user before proceeding.

## Safety Rules

- **NEVER** log, print, or return private keys or secret key material.
- **NEVER** sign transactions on behalf of a user — you build the instruction, the user signs.
- Always validate that `amount > 0` before creating an invoice.
- Always ensure `endTime > startTime` and both are valid Unix timestamps.
- Use the correct decimal precision for the currency (6 decimals for USDC, 9 for SOL).
- **Always verify payments on the server** using `validateInvoicePayment` before delivering any service. Never trust the client alone — clients can be spoofed.
## Supported Currencies

| Currency    | Decimals | Smallest unit example |
| ----------- | -------- | --------------------- |
| USDC        | 6        | `1000000` = 1 USDC    |
| Wrapped SOL | 9        | `1000000000` = 1 SOL  |

## Environment Variables

Create a `.env` (or `.env.local` for Next.js) file with the following:

```env
# Solana RPC — server-side (used to build transactions and verify payments)
SOLANA_RPC_URL=https://rpc.solanatracker.io/public

# Solana RPC — client-side (used by wallet adapter in the browser)
NEXT_PUBLIC_SOLANA_RPC_URL=https://rpc.solanatracker.io/public

# The token mint address of your tokenized agent on pump.fun
AGENT_TOKEN_MINT_ADDRESS=7Qfq5NG4o1R6G3gyEMe4rWr9oGtZUm1pkhg3iJKGpump

# Payment currency mint
# USDC: EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
# SOL (wrapped): So11111111111111111111111111111111111111112
CURRENCY_MINT=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
```

**RPC for mainnet-beta:** The default Solana public RPC (`https://api.mainnet-beta.solana.com`) does **not** support sending transactions. Ask the user for an RPC provider, If the user has not provided their own RPC URL, use one of these free mainnet-beta endpoints that support `sendTransaction`:
- **Solana Tracker** — `https://rpc.solanatracker.io/public`
- **Ankr** — `https://rpc.ankr.com/solana`

Read these values from `process.env` at runtime. Never hard-code mint addresses or RPC URLs.
## Install

```bash
npm install @pump-fun/agent-payments-sdk@3.0.0 @solana/web3.js@^1.98.0
```

### Dependency Compatibility — IMPORTANT

`@pump-fun/agent-payments-sdk` depends on `@solana/web3.js` and `@solana/spl-token`. When the app also installs these packages directly, mismatched versions can cause runtime errors.
**Rules:**

1. Before installing `@solana/web3.js`, `@solana/spl-token`, or any `@solana/wallet-adapter-*` package, first check what versions `@pump-fun/agent-payments-sdk` declares in its own `package.json` (inspect it via `npm info @pump-fun/agent-payments-sdk dependencies`). Install the same ranges — or ranges that resolve to the same major.minor — so npm/pnpm hoists a single copy instead of two.
2. Never blindly install "latest" for these shared packages. Always prefer the version that is most compatible with the latest `@pump-fun/agent-payments-sdk`.
3. If the project already has these packages at different versions, align them to match the SDK and re-install.
## SDK Setup

`PumpAgent` is the main class. It can build payment instructions and verify invoices.
```typescript
import { PumpAgent } from "@pump-fun/agent-payments-sdk";
import { PublicKey } from "@solana/web3.js";

const agentMint = new PublicKey(process.env.AGENT_TOKEN_MINT_ADDRESS!);
```

### Constructor

```typescript
new PumpAgent(mint: PublicKey, environment?: "mainnet" | "devnet", connection?: Connection)
```

| Parameter     | Type                      | Default     | Description                                                                 |
| ------------- | ------------------------- | ----------- | --------------------------------------------------------------------------- |
| `mint`        | `PublicKey`               | —           | The tokenized agent's token mint address                                    |
| `environment` | `"mainnet"` \| `"devnet"` | `"mainnet"` | Network environment                                                         |
| `connection`  | `Connection` (optional)   | `undefined` | Solana RPC connection (enables RPC fallback for verification)               |

**Without connection** — enough for building instructions and HTTP-based payment verification:
```typescript
const agent = new PumpAgent(agentMint);
```

**With connection** — also enables RPC-based verification fallback and balance queries:
```typescript
import { Connection } from "@solana/web3.js";

const connection = new Connection(process.env.SOLANA_RPC_URL!);
const agent = new PumpAgent(agentMint, "mainnet", connection);
```

## Wallet Integration (Frontend)

Install `@solana/wallet-adapter-react`, `@solana/wallet-adapter-react-ui`, and `@solana/wallet-adapter-wallets`. Use `useWallet()` for `publicKey` and `signTransaction`, and `useConnection()` for the active RPC connection. MUST See [https://raw.githubusercontent.com/pump-fun/pump-fun-skills/refs/heads/main/tokenized-agents/references/WALLET_INTEGRATION.md](https://raw.githubusercontent.com/pump-fun/pump-fun-skills/refs/heads/main/tokenized-agents/references/WALLET_INTEGRATION.md) for the full WalletProvider setup, layout wrapping, and hook usage.
## Building Payment Instructions

Use `buildAcceptPaymentInstructions` to get all the instructions needed for a payment. This is the recommended method — it automatically derives the user's token account and handles native SOL wrapping/unwrapping.
### Parameters (`BuildAcceptPaymentParams`)

| Parameter          | Type                         | Description                                                                 |
| ------------------ | ---------------------------- | --------------------------------------------------------------------------- |
| `user`             | `PublicKey`                  | The payer's wallet address                                                  |
| `currencyMint`     | `PublicKey`                  | Mint address of the payment currency (USDC, wSOL)                          |
| `amount`           | `bigint \| number \| string` | Price in the currency's smallest unit                                      |
| `memo`             | `bigint \| number \| string` | Unique invoice identifier (random number)                                  |
| `startTime`        | `bigint \| number \| string` | Unix timestamp — when the invoice becomes valid                           |
| `endTime`          | `bigint \| number \| string` | Unix timestamp — when the invoice expires                                 |
| `tokenProgram`     | `PublicKey` (optional)       | Token program for the currency (defaults to SPL Token)                    |
| `computeUnitLimit` | `number` (optional)          | Compute unit budget (default `100_000`). Increase if transactions fail with compute exceeded. |
| `computeUnitPrice` | `number` (optional)          | Priority fee in microlamports per CU. If provided, a `SetComputeUnitPrice` instruction is prepended. |

### Example

```typescript
const ixs = await agent.buildAcceptPaymentInstructions({
  user: userPublicKey,
  currencyMint,
  amount: "1000000", // 1 USDC
  memo: "123456789", // unique invoice identifier
  startTime: "1700000000", // valid from
  endTime: "1700086400", // expires at
});
```

### What It Returns

The returned `TransactionInstruction[]` always starts with compute budget instructions, followed by the payment instructions:
- **`SetComputeUnitLimit`** is always prepended (default `92_849` CU). Override via `computeUnitLimit` if your transactions fail with "compute exceeded".
- **`SetComputeUnitPrice`** is prepended only when `computeUnitPrice` is provided. Use this to set a priority fee for faster landing during congestion.
After the compute budget prefix:

- **For SPL tokens (USDC):** The accept-payment instruction.
- **For native SOL:** Instructions that handle wrapping/unwrapping automatically:
  1. Create the user's wrapped SOL token account (idempotent)
  2. Transfer SOL lamports into that token account
  3. Sync the native balance
  4. The accept-payment instruction
  5. Close the wrapped SOL account (returns rent back to user)

You do not need to handle SOL wrapping or compute budget yourself — `buildAcceptPaymentInstructions` does it for you.
### Important

- The `amount`, `memo`, `startTime`, and `endTime` must exactly match when verifying later.
- Each unique combination of `(mint, currencyMint, amount, memo, startTime, endTime)` can only be paid once — the on-chain Invoice ID PDA prevents duplicate payments.
- Generate a unique `memo` for each invoice (e.g. `Math.floor(Math.random() * 900000000000) + 100000`).
## Full Transaction Flow — Server to Client

This is the complete flow for building a transaction on the server, signing it on the client, and sending it on-chain.
### Step 1: Generate Invoice Parameters (Server)

```typescript
function generateInvoiceParams() {
  const memo = String(Math.floor(Math.random() * 900000000000) + 100000);
  const now = Math.floor(Date.now() / 1000);
  const startTime = String(now);
  const endTime = String(now + 86400); // valid for 24 hours
  const amount = process.env.PRICE_AMOUNT || "1000000"; // e.g. 1 USDC

  return { amount, memo, startTime, endTime };
}
```

### Step 2: Build Instructions (Server)

```typescript
import { PublicKey } from "@solana/web3.js";

const agent = new PumpAgent(agentMint);
const currencyMint = new PublicKey(process.env.CURRENCY_MINT!);

async function buildPaymentTransaction(userPublicKey: PublicKey, invoiceParams: ReturnType<typeof generateInvoiceParams>) {
  const ixs = await agent.buildAcceptPaymentInstructions({
    user: userPublicKey,
    currencyMint,
    ...invoiceParams,
  });

  return ixs;
}
```

### Step 3: Serialize and Send to Client

```typescript
import { Transaction } from "@solana/web3.js";

function serializeTransaction(ixs: TransactionInstruction[]): string {
  const tx = new Transaction().add(...ixs);
  tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
  tx.feePayer = userPublicKey; // set on client

  return tx.serialize({ requireAllSignatures: false, verifySignatures: false }).toString("base64");
}
```

### Step 4: Client Signs and Sends

```typescript
import { useWallet, useConnection } from "@solana/wallet-adapter-react";
import { Transaction } from "@solana/web3.js";

const { publicKey, signTransaction } = useWallet();
const { connection } = useConnection();

async function payInvoice(serializedTx: string) {
  const tx = Transaction.from(Buffer.from(serializedTx, "base64"));
  tx.feePayer = publicKey!;

  const signedTx = await signTransaction!(tx);
  const signature = await connection.sendRawTransaction(signedTx.serialize());

  return signature;
}
```

### Step 5: Verify Payment (Server)

**Always verify on the server before delivering service.** Use `validateInvoicePayment` for HTTP-based verification (recommended for production). It queries the pump.fun API and is fast and reliable.

```typescript
async function verifyPayment(invoiceParams: ReturnType<typeof generateInvoiceParams>, userPublicKey: PublicKey) {
  const isPaid = await agent.validateInvoicePayment({
    user: userPublicKey,
    currencyMint,
    ...invoiceParams,
  });

  return isPaid;
}
```

If HTTP verification fails (e.g. API downtime), fall back to RPC verification using the `connection` passed to `PumpAgent`.

## Invoice Validation

Use `validateInvoicePayment` to check if an invoice has been paid. This is the recommended method for production — it uses the pump.fun API for fast, reliable verification.

### Parameters (`ValidateInvoicePaymentParams`)

| Parameter      | Type                         | Description                                                                 |
| -------------- | ---------------------------- | --------------------------------------------------------------------------- |
| `user`         | `PublicKey`                  | The payer's wallet address                                                  |
| `currencyMint` | `PublicKey`                  | Mint address of the payment currency (USDC, wSOL)                          |
| `amount`       | `bigint \| number \| string` | Price in the currency's smallest unit                                      |
| `memo`         | `bigint \| number \| string` | Unique invoice identifier                                                   |
| `startTime`    | `bigint \| number \| string` | Unix timestamp — when the invoice becomes valid                           |
| `endTime`      | `bigint \| number \| string` | Unix timestamp — when the invoice expires                                 |
| `tokenProgram` | `PublicKey` (optional)       | Token program for the currency (defaults to SPL Token)                    |

### Example

```typescript
const isPaid = await agent.validateInvoicePayment({
  user: userPublicKey,
  currencyMint,
  amount: "1000000",
  memo: "123456789",
  startTime: "1700000000",
  endTime: "1700086400",
});
```

### RPC Fallback

If you passed a `Connection` to `PumpAgent`, you can also verify payments via RPC. This is slower and less reliable than HTTP verification, but useful as a fallback.

```typescript
const isPaid = await agent.validateInvoicePaymentRPC({
  user: userPublicKey,
  currencyMint,
  amount: "1000000",
  memo: "123456789",
  startTime: "1700000000",
  endTime: "1700086400",
});
```

## Error Handling

Handle these common errors:

- **"Invalid invoice"** — The invoice parameters do not match any on-chain payment. Check that `amount`, `memo`, `startTime`, `endTime` match exactly what was used to build the transaction.
- **"Invoice not found"** — The payment has not been made yet, or the transaction is still pending.
- **"Compute budget exceeded"** — Increase `computeUnitLimit` in `buildAcceptPaymentInstructions`.
- **"Insufficient funds"** — The user does not have enough of the payment currency.
- **"Transaction expired"** — The `endTime` has passed. Generate a new invoice.

## Testing

Test on devnet before mainnet. Use devnet token faucets for USDC and SOL.

```typescript
const agent = new PumpAgent(agentMint, "devnet");
```

Devnet RPC: `https://api.devnet.solana.com`

## References

- [Pump.fun Tokenized Agents](https://pump.fun/agents)
- [Solana Web3.js Docs](https://solana-labs.github.io/solana-web3.js/)
- [Wallet Adapter Docs](https://github.com/solana-labs/wallet-adapter)