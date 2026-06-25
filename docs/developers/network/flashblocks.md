---
title: Flashblocks
description: Read pre-confirmed transaction state on Ronin in about 250 milliseconds using the pending block tag.
---

Ronin produces a full block about every 2 seconds. Flashblocks let your
application read sequencer-ordered state before that block finalizes: the
sequencer pre-confirms transactions and emits a partial block roughly every 250
milliseconds. Instead of waiting a full block to see a submitted transaction
take effect, you read the pre-confirmed state as soon as the sequencer orders it.

Flashblocks is live on both Ronin mainnet and Saigon testnet. There is no
separate endpoint and no new RPC method.

:::info Reference
Flashblocks is an OP Stack feature. For background, see:

- https://docs.optimism.io/op-stack/features/flashblocks
- https://chainstack.com/what-are-flashblocks/

:::

:::caution Pre-confirmations are not final
Pre-confirmed state is provisional. If the sequencer reorganizes, a transaction
you saw under `pending` can change or disappear. This is the same risk that
already applies to pending-state reads, so treat Flashblocks as a fast signal
for responsive UX and keep relying on confirmed blocks for settlement.
:::

## Reading pre-confirmed state

Pass the `"pending"` block tag to a standard JSON-RPC method and it resolves
against the latest Flashblock instead of the last finalized block. The following
methods support the `"pending"` tag on Ronin:

- `eth_getBalance`
- `eth_getCode`
- `eth_getStorageAt`
- `eth_getTransactionCount`
- `eth_getTransactionReceipt`
- `eth_getLogs`
- `eth_call`
- `eth_estimateGas`
- `eth_simulateV1`

### curl

This call reads the pre-confirmed RON balance of an address. Swap `"pending"`
for `"latest"` and you read the last finalized block instead.

```sh
curl -X POST https://api.roninchain.com/rpc \
-H "Content-Type: application/json" \
-d '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x22cefc91e9b7c0f3890ebf9527ea89053490694e","pending"],"id":1}'
```

Response

```json
{ "jsonrpc": "2.0", "id": 1, "result": "0x21e0c0d9e7e9b0000" }
```

For Saigon testnet, send the same request to
`https://saigon-testnet.roninchain.com/rpc`.

## Using viem

viem reads pre-confirmed state through its `experimental_preconfirmationTime`
chain option. Extend the `ronin` (or `saigon`) chain with
`experimental_preconfirmationTime: 250`. A client built on that chain applies the
`"pending"` block tag to supported actions by default, so you don't have to pass
`blockTag` per call (see viem's [`experimental_blockTag`](https://viem.sh/docs/clients/public#experimental_blocktag-optional)).
Base does the same with `basePreconf`, but routes pre-confirmations through
a separate RPC endpoint; Ronin serves them from its standard RPC, so you set the
timing and keep the default endpoint.

Install the latest version of viem:

```sh
npm install viem
```

The script below sends 0.000000001 RON to the zero address on Saigon, measures
how long the sequencer takes to pre-confirm it, then reads the recipient's
balance again to confirm the pre-confirmed state already reflects the transfer.

```javascript
import {
  createWalletClient,
  defineChain,
  http,
  parseEther,
  publicActions,
} from "viem";
import { saigon } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";

const saigonPreconf = defineChain({
  ...saigon,
  experimental_preconfirmationTime: 250,
  rpcUrls: {
    default: { http: ["https://rpc-saigon-testnet-cc58e966ql.t.conduit.xyz/"] },
  },
});

const account = privateKeyToAccount("0x.."); // your private key

const client = createWalletClient({
  account,
  chain: saigonPreconf,
  transport: http(),
}).extend(publicActions);

const recipient = "0x0000000000000000000000000000000000000000";

const value = parseEther("0.000000001");

// blockTag defaults to "pending" because the chain sets
// experimental_preconfirmationTime, so these reads return pre-confirmed state.
const before = await client.getBalance({
  address: recipient,
  // blockTag: "pending", // redundant: the chain config sets it
});

const t0 = performance.now();
await client.sendTransactionSync({
  to: recipient,
  value,
});
const ms = Math.round(performance.now() - t0);

console.log(`block confirmed in: ${ms} ms`);

const after = await client.getBalance({
  address: recipient,
  // blockTag: "pending", // redundant: the chain config sets it
});

if (after - before === value) console.log("success, balance changed");
else console.log("fail");
```

A sequencer reorg can still revert a pre-confirmed receipt. When a flow must
wait for settlement, wait for a confirmed block:

```javascript
await client.waitForTransactionReceipt({
  hash: receipt.transactionHash,
  confirmations: 1, // wait for a full confirmed block
});
```

## See also

- [Network information](/developers/network)
- [EIP-1559](./eip-1559/)
