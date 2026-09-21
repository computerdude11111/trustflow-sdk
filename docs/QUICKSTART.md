# TrustFlow SDK Quick Start

A walkthrough of escrow, dispute, multi-sig, pagination, juror, profile, storage and event APIs.

Each TypeScript block is independent and assumes an environment with top-level `await`
(for example an ES module). Set the environment variables listed below before running
a block; use valid testnet addresses and existing escrow/dispute IDs for your deployment.
Backend examples also require a reachable API and valid credentials.

Some contract helpers currently return prepared-call metadata or placeholder hashes
(see their source); an `ok` result alone is not proof of an on-chain transfer. Use the
[API reference](./API.md) and deployment-specific signing flow for live transactions.

## Install

```bash
npm install @trustflow/sdk
# or
yarn add @trustflow/sdk
```

---

## 1. Connect to the Network

```typescript
import { TrustFlowClient } from '@trustflow/sdk';

const client = new TrustFlowClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET', // or 'MAINNET'
});

await client.connect();
console.log('Connected to', client.network); // 'TESTNET'
console.log('Config:', client.getConfig());
```

---

## 2. Create an Escrow

### Using `TrustFlowEscrowClient` + `EscrowBuilder` (recommended)

```typescript
import { TrustFlowEscrowClient, EscrowBuilder } from '@trustflow/sdk';

const escrowClient = new TrustFlowEscrowClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET',
  rpcUrl: 'https://soroban-testnet.stellar.org',
  networkPassphrase: 'Test SDF Network ; September 2015',
});

const params = new EscrowBuilder()
  .setDepositor(process.env.DEPOSITOR_ADDRESS!)
  .setBeneficiary(process.env.BENEFICIARY_ADDRESS!)
  .setAmount('50') // XLM
  .setDeadline(17280) // ~1 day in ledgers
  .build();

const result = await escrowClient.createEscrow(params);
if (result.ok) {
  console.log('Escrow ID:', result.data.escrowId);
  console.log('Tx Hash:', result.data.txHash);
} else {
  console.error('Error:', result.error);
}
```

### Using `createEscrow` function directly

```typescript
import { TrustFlowClient } from '@trustflow/sdk';
import { createEscrow } from '@trustflow/sdk/escrow';
import { xlmToStroops } from '@trustflow/sdk/utils';

const client = new TrustFlowClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET',
});
await client.connect();

const escrow = await createEscrow(client, {
  sender: process.env.DEPOSITOR_ADDRESS!,
  recipient: process.env.BENEFICIARY_ADDRESS!,
  amountStroops: xlmToStroops('50'), // 50 XLM → stroops
  durationBlocks: 17280,
  metadata: { orderId: 'ORD-001', description: 'Freelance payment' },
});

console.log('Escrow created:', escrow.id);
console.log('Amount (stroops):', escrow.amount.toString());
```

---

## 3. Fund and Release an Escrow

### Fund

```typescript
import { TrustFlowEscrowClient, xlmToStroops } from '@trustflow/sdk';

const escrowClient = new TrustFlowEscrowClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET',
  rpcUrl: 'https://soroban-testnet.stellar.org',
  networkPassphrase: 'Test SDF Network ; September 2015',
});

const result = await escrowClient.fund(
  process.env.ESCROW_ID!,
  process.env.DEPOSITOR_ADDRESS!,
  xlmToStroops('50'), // 50 units of an asset with 7 decimal places
  process.env.USDC_TOKEN_CONTRACT_ID, // optional C... token contract on this network
);
if (result.ok) console.log('Fund call result:', result.data.txHash);
else console.error(result.error);
```

Omit the fourth argument to use the escrow's native asset. For USDC, provide its
Soroban token contract address for the selected network, not an account address.

### Release (browser wallet)


```typescript
import { TrustFlowClient } from '@trustflow/sdk';
import { releaseEscrow } from '@trustflow/sdk/escrow';
import { connectWallet } from '@trustflow/sdk/wallet';

const wallet = await connectWallet('freighter');

const client = new TrustFlowClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET',
});
await client.connect();

const txHash = await releaseEscrow(client, {
  escrowId: process.env.ESCROW_ID!,
  caller: wallet.publicKey,
});

console.log('Released! Transaction:', txHash);
```

---

## 4. Check Balance

```typescript
import { TrustFlowClient } from '@trustflow/sdk';

const client = new TrustFlowClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET',
});
await client.connect();
const balance = await client.getBalance(process.env.DEPOSITOR_ADDRESS!);
console.log(`Balance: ${balance} XLM`);
```

---

## 5. Raise a Dispute

```typescript
import { DisputeClient } from '@trustflow/sdk';

const disputes = new DisputeClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET',
  rpcUrl: 'https://soroban-testnet.stellar.org',
  networkPassphrase: 'Test SDF Network ; September 2015',
  apiBaseUrl: process.env.TRUSTFLOW_API_URL!,
  apiKey: process.env.API_KEY!, // backend bearer credential
});

const result = await disputes.raiseDispute({
  escrowId: process.env.ESCROW_ID!,
  reason: 'Work not delivered as agreed',
  evidence: 'https://evidence.example.com/proof.pdf',
});

if (result.ok) {
  console.log('Dispute raised:', result.data.disputeId);
  // getDispute takes the escrow ID, not the returned dispute ID.
  const details = await disputes.getDispute(process.env.ESCROW_ID!);
  if (details.ok) console.log(details.data);
  else console.error(details.error);
} else {
  console.error(result.error);
}
```

---

## 6. Multi-Sig Escrow (M-of-N)

Collect signatures from multiple approvers before broadcasting:

```typescript
import { MultiSigEscrowClient } from '@trustflow/sdk';
import { Networks } from '@stellar/stellar-sdk';

const client = new MultiSigEscrowClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET',
  rpcUrl: 'https://soroban-testnet.stellar.org',
  networkPassphrase: Networks.TESTNET,
});

const APPROVER_A = process.env.APPROVER_A!;
const APPROVER_B = process.env.APPROVER_B!;
const SIGNED_XDR_A = process.env.SIGNED_XDR_A!;
const SIGNED_XDR_B = process.env.SIGNED_XDR_B!;

// Register a 2-of-2 release operation
const init = client.initMultiSigOperation({
  escrowId: process.env.ESCROW_ID!,
  signers: [APPROVER_A, APPROVER_B],
  threshold: 2,
  operationType: 'release',
  unsignedXdr: process.env.UNSIGNED_RELEASE_XDR!,
  networkPassphrase: Networks.TESTNET,
});

if (!init.ok) throw new Error(init.error);
const { operationId } = init.data;

// Each approver submits their signed XDR independently
client.addSignature({ operationId, signerAddress: APPROVER_A, signedXdr: SIGNED_XDR_A });
client.addSignature({ operationId, signerAddress: APPROVER_B, signedXdr: SIGNED_XDR_B });

// Broadcast once threshold is met
const result = await client.submitWhenReady(operationId, 'https://horizon-testnet.stellar.org');
if (result.ok) console.log('Released! tx:', result.data.txHash);
```

---

## 7. Paginated Gig Listing

```typescript
import { TrustFlowEscrowClient } from '@trustflow/sdk';

const escrowClient = new TrustFlowEscrowClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET',
  rpcUrl: 'https://soroban-testnet.stellar.org',
  networkPassphrase: 'Test SDF Network ; September 2015',
  apiBaseUrl: process.env.TRUSTFLOW_API_URL!,
  apiKey: process.env.API_KEY,
});

let cursor: string | undefined;
do {
  const page = await escrowClient.getGigs({ cursor, limit: 20, status: 'active' });
  if (!page.ok) { console.error(page.error); break; }
  console.log(page.data.data);
  cursor = page.data.nextCursor ?? undefined;
} while (cursor);
```

---

## 8. Juror Voting

```typescript
import { JurorClient } from '@trustflow/sdk';

const jurors = new JurorClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET',
  rpcUrl: 'https://soroban-testnet.stellar.org',
  networkPassphrase: 'Test SDF Network ; September 2015',
});
const result = await jurors.vote({
  disputeId: process.env.DISPUTE_ID!,
  jurorAddress: process.env.JUROR_ADDRESS!,
  vote: { encrypted: false, choice: 'approve' },
});
if (result.ok) console.log('Vote call result:', result.data);
else console.error(result.error);
```

See [JurorClient](../src/juror/client.ts) and the [README](../README.md)
for encrypted vote payloads and the current contract integration status.

## 9. Profiles

```typescript
import { ProfileClient } from '@trustflow/sdk';

const profiles = new ProfileClient(
  process.env.TRUSTFLOW_API_URL!,
  process.env.AUTH_TOKEN!,
);
const result = await profiles.getProfile(process.env.PROFILE_ADDRESS!);
if (result.ok) console.log(result.data.displayName);
else console.error(result.error);
```

See [ProfileClient](../src/profile/client.ts) for `updateProfile` and the
[API reference](./API.md) for the broader SDK surface.

## 10. IPFS Upload

```typescript
import { readFile } from 'node:fs/promises';
import { TrustFlowClient } from '@trustflow/sdk';

const client = new TrustFlowClient({
  contractId: process.env.TRUSTFLOW_CONTRACT_ID!,
  network: 'TESTNET',
  ipfs: {
    apiUrl: process.env.IPFS_API_URL!,
    apiKey: process.env.IPFS_API_KEY!,
  },
});
const file = await readFile(process.env.UPLOAD_FILE!);
const result = await client.storage.upload(file, { filename: 'contract.pdf' });
if (result.ok) console.log(result.data.cid, result.data.url);
else console.error(result.error);
```

Use an upload service that accepts a raw file body and returns `{ cid }`.
See [IPFSStorage](../src/storage/ipfs.ts) for gateway and content-type options.

## 11. Parse and Deliver Events

```typescript
import { readFile } from 'node:fs/promises';
import { EscrowMonitor, parseEvents, type RawContractEvent } from '@trustflow/sdk';

// A JSON array in the SDK's RawContractEvent shape, captured from your event feed.
const rawEvents: RawContractEvent[] = JSON.parse(
  await readFile(process.env.EVENTS_FILE!, 'utf8'),
);
const events = parseEvents(rawEvents, process.env.TRUSTFLOW_CONTRACT_ID!);
const monitor = new EscrowMonitor();
monitor.on('escrow_created', (event) => {
  if (event.type === 'escrow_created') console.log(event.data.escrowId);
});
monitor.onError((error, context) => console.error(context.phase, error));
monitor.deliver(events);
```

`parseEvents` filters by contract ID and returns the typed events that
`EscrowMonitor` consumes. See [event types/parser](../src/events.ts) and
[EscrowMonitor](../src/escrow/monitor.ts) for polling (`startPolling` with your
own fetch function; call `stopPolling` during cleanup). Validate untrusted JSON
before treating it as `RawContractEvent[]`.

---

## Environment Variables

Set only the variables needed by the example you run. Keep keys, tokens and signed
XDR out of source control. `!` is a TypeScript assertion, not runtime validation.

```bash
TRUSTFLOW_CONTRACT_ID=C...          # Contract on the chosen network
TRUSTFLOW_API_URL=https://...      # Reachable TrustFlow backend
API_KEY=...                       # Dispute bearer credential / pagination API key
AUTH_TOKEN=...                    # Profile backend bearer token
DEPOSITOR_ADDRESS=G...            # Valid depositor/funder account
BENEFICIARY_ADDRESS=G...          # Valid beneficiary account
ESCROW_ID=...                     # Existing escrow for fund/release/dispute
USDC_TOKEN_CONTRACT_ID=C...       # Optional network-specific USDC token contract
APPROVER_A=G...                   # First multi-sig approver
APPROVER_B=G...                   # Second multi-sig approver
UNSIGNED_RELEASE_XDR=...          # Prepared multi-sig release transaction
SIGNED_XDR_A=...                  # Same transaction signed by APPROVER_A
SIGNED_XDR_B=...                  # Same transaction signed by APPROVER_B
DISPUTE_ID=...                    # Existing dispute for juror voting
JUROR_ADDRESS=G...                # Valid juror account
PROFILE_ADDRESS=G...              # Profile account to fetch
IPFS_API_URL=https://...          # Compatible raw-body upload endpoint
IPFS_API_KEY=...                  # Upload service credential
UPLOAD_FILE=./contract.pdf        # Local file to upload
EVENTS_FILE=./events.json         # Saved RawContractEvent[] JSON
```

See [API Reference](./API.md) for the full method list and [examples/](../examples/) for runnable scripts.
