# onChainOrder

Contract for placing limit and market orders.

> ⚠️ **Important:** Orders are placed on-chain and executed by the protocol execution engine when price conditions are met.

## Contents

- [onChainOrder](#onchainorder)
  - [Contents](#contents)
  - [Order](#order)
    - [Contract Information](#contract-information)
    - [Script Hash](#script-hash)
    - [Reference UTxO](#reference-utxo)
    - [Script CBOR](#script-cbor)
    - [Transaction Structure](#transaction-structure)
    - [Inputs](#inputs)
    - [Outputs](#outputs)
      - [Order Output](#order-output)
    - [SDK Datum Type](#sdk-datum-type)
  - [Datum Building](#datum-building)
    - [Price Calculation](#price-calculation)
    - [Input Amount](#input-amount)
    - [Address Serialization](#address-serialization)
    - [Beacon Calculation](#beacon-calculation)
  - [Output Building](#output-building)
    - [Deposit ADA Calculation](#deposit-ada-calculation)
  - [Transaction Building](#transaction-building)
  - [Order Types](#order-types)
    - [Limit Order](#limit-order)
    - [Market Order](#market-order)
  - [Important Notes](#important-notes)
  - [Refund](#refund)
    - [Execution Units \& Redeemer](#execution-units--redeemer)
    - [Transaction Structure](#transaction-structure-1)
    - [Refund Flow](#refund-flow)
    - [Important Notes](#important-notes-1)

## Order

### Contract Information

### Script Hash

```
9e130eb0323f31df50bfbb731f35f5a795a630ef727b4ba272460b55
```

### Reference UTxO

```typescript
{
  txHash: '2a6b847da15d7bfa3ac23af67590c2d6c98e8b485545d883dd72da322ccfd38e',
  outputIndex: 0
}
```

### Script CBOR

```
5902080101003229800aba2aba1aba0aab9faab9eaab9dab9a488888896600264646644b30013370e900118031baa00289919912cc004c966002601600315980099b8948010c0280062d13370e90011805000a0128b20183754601a00d132330010013756601c601e601e601e601e601e601e60166ea8014896600200314a115980099baf012300b300f0018a518998010011808000a014403515980099b8748000c024dd5000c4cc896600266e1d2000300b3754003133225980099199119801001000912cc00400629422b30013371e6eb8c05000400e2946266004004602a00280790121bac301230133013301330133013301330133013300f37540126eb8c044c048c048c048c048c048c048c048c038dd5002456600266ebcc004c038dd5180098071baa3011300e37540066002601c6ea800a266ebcc044c038dd5001180098071baa0048a50403114a08060c03cdd618079808180818061baa0062301030110018b2014300d300a375400264660020026eb0c038c02cdd5002912cc0040062980103d87a80008992cc004cdd7980818069baa001005899ba548000cc03c0052f5c11330030033011002402c601e002806a29450082010300b001300b300c00130073754005164014601060120026010004601000260066ea802229344d959001130122d87a9f581c5955b1d938f5808537f00a8e252ac63ac853fe8c375cb32bffbcaacdff0001
```

### Transaction Structure

### Inputs

Any user UTxOs to cover:
- Input asset amount (what user wants to sell/swap)
- Transaction fees
- Order collateral (minimum 2 ADA)
- Deposit ADA for outputs

### Outputs

#### Order Output

Send to script address with [script hash](#script-hash) and user's stake credentials (optional).

**Address Construction:**

```typescript
import { credentialsToBech32Address } from '@splashprotocol/sdk';

// See Contract Information > Script Hash
const SCRIPT_HASH = /* script hash from above */;

const orderAddress = await credentialsToBech32Address(
  network,
  {
    hash: SCRIPT_HASH,
    type: 'script',
  },
  userStakeCredentials
    ? {
        hash: userStakeCredentials,
        type: 'pubKey',
      }
    : undefined
);
```

**Value:**

| Asset           | Amount                                             |
| --------------- | -------------------------------------------------- |
| Input Asset     | `inputAmount`                                      |
| ADA (fee)       | `feePerExStep * (orderType === 'limit' ? 3 : 1)`   |
| ADA (deposit)   | `depositAdaForOrder + depositAdaForReceive`        |
| ADA (collateral)| Additional if total < 2 ADA                        |

Where:
- `feePerExStep` = `600_000` lovelace (0.6 ADA per execution step)
- Limit orders require 3x fee (1.8 ADA), market orders 1x fee (0.6 ADA)
- `depositAdaForOrder` = minimum ADA to store the order UTxO
- `depositAdaForReceive` = minimum ADA for user's output after execution
- Additional collateral added if total < 2 ADA minimum

**Datum:**

```typescript
import { Datum } from '@splashprotocol/sdk';

const orderDatum = Datum.constr(0, {
  type: Datum.bytes(), // "09" (order type identifier)
  address: Datum.constr(0, {
    paymentCredentials: Datum.anyOf([
      Datum.constr(0, {
        paymentKeyHash: Datum.bytes(), // user payment key hash
      }),
      Datum.constr(1, {
        scriptHash: Datum.bytes(), // script hash (alternative)
      }),
    ]),
    stakeCredentials: Datum.anyOf([
      Datum.constrAnyOf(0, [
        Datum.constrAnyOf(0, [
          Datum.constr(0, {
            paymentKeyHash: Datum.bytes(), // user stake key hash
          }),
          Datum.constr(1, {
            scriptHash: Datum.bytes(), // script stake hash (alternative)
          }),
        ]),
        Datum.constr(1, {
          slotNumber: Datum.integer(),
          transactionIndex: Datum.integer(),
          certificateIndex: Datum.integer(),
        }),
      ]),
      Datum.constr(1, {}), // no stake credentials
    ]),
  }),
  inputAsset: Datum.list(Datum.bytes()), // [policyId, tokenName]
  inputAmount: Datum.integer(), // amount to sell
  feePerExStep: Datum.integer(), // 600_000
  outputAsset: Datum.list(Datum.bytes()), // [policyId, tokenName]
  price: Datum.list(Datum.integer()), // [numerator, denominator]
  cancelPkh: Datum.bytes(), // user payment key hash (for cancellation)
  beacon: Datum.bytes(), // order beacon (see Beacon Calculation)
});
```

**Datum Field Details:**

- `type`: `"09"` in hex format
- `address`: User's address where funds will be returned after execution
  - `paymentCredentials`: User's payment key hash (constructor 0 for pubkey)
  - `stakeCredentials`: User's stake key hash (constructor 0 → 0 → 0 for pubkey), or constructor 1 for no stake
- `inputAsset`: `[policyId, tokenName]` - what user is selling
  - For ADA: `["", ""]` (empty strings)
  - For tokens: `[policyId, tokenName]` in hex
- `inputAmount`: Amount of input asset (in base units)
- `feePerExStep`: `600000` (0.6 ADA)
- `outputAsset`: `[policyId, tokenName]` - what user wants to receive
- `price`: `[numerator, denominator]` - price ratio
  - Buy order: `[quote_denom, quote_num]` (inverted)
  - Sell order: `[num, denom]` (as-is)
- `cancelPkh`: User's payment key hash for order cancellation
- `beacon`: 20-byte order identifier (see calculation below)

**Real Example:**

Buy order: 50 ADA → NIGHT token at ~4.2 tokens per ADA

```json
{
  "constructor": "0",
  "fields": [
    {
      "bytes": "09"
    },
    {
      "constructor": "0",
      "fields": [
        {
          "constructor": "0",
          "fields": [
            {
              "bytes": "3533ded9539c6ed7dce55b29a5fd341d78d3fca4bebabc4fb9f1894b"
            }
          ]
        },
        {
          "constructor": "0",
          "fields": [
            {
              "constructor": "0",
              "fields": [
                {
                  "constructor": "0",
                  "fields": [
                    {
                      "bytes": "f985dec9000fe612fc20520200f91df5520aa5444059a5cebe411295"
                    }
                  ]
                }
              ]
            }
          ]
        }
      ]
    },
    {
      "list": [
        {
          "bytes": ""
        },
        {
          "bytes": ""
        }
      ]
    },
    {
      "int": "50000000"
    },
    {
      "int": "600000"
    },
    {
      "list": [
        {
          "bytes": "0691b2fecca1ac4f53cb6dfb00b7013e561d1f34403b957cbb5af1fa"
        },
        {
          "bytes": "4e49474854"
        }
      ]
    },
    {
      "list": [
        {
          "int": "209790827"
        },
        {
          "int": "50000000"
        }
      ]
    },
    {
      "bytes": "3533ded9539c6ed7dce55b29a5fd341d78d3fca4bebabc4fb9f1894b"
    },
    {
      "bytes": "016b44032ba4b5e00b0e883e37b20e66e6ba34ca"
    }
  ]
}
```

Transaction: [c77d82d0592252d86a175576665bcf92cbf689ddf92a496cd106cdad75930e79](https://cardanoscan.io/transaction/c77d82d0592252d86a175576665bcf92cbf689ddf92a496cd106cdad75930e79?tab=utxo)

### SDK Datum Type

For integrators using `@splashprotocol/sdk`:

```typescript
import { Datum, InferDatum } from '@splashprotocol/sdk';

export const onChainOrderDatum = Datum.constr(0, {
  type: Datum.bytes(),
  address: Datum.constr(0, {
    paymentCredentials: Datum.anyOf([
      Datum.constr(0, {
        paymentKeyHash: Datum.bytes(),
      }),
      Datum.constr(1, {
        scriptHash: Datum.bytes(),
      }),
    ]),
    stakeCredentials: Datum.anyOf([
      Datum.constrAnyOf(0, [
        Datum.constrAnyOf(0, [
          Datum.constr(0, {
            paymentKeyHash: Datum.bytes(),
          }),
          Datum.constr(1, {
            scriptHash: Datum.bytes(),
          }),
        ]),
        Datum.constr(1, {
          slotNumber: Datum.integer(),
          transactionIndex: Datum.integer(),
          certificateIndex: Datum.integer(),
        }),
      ]),
      Datum.constr(1, {}),
    ]),
  }),
  inputAsset: Datum.list(Datum.bytes()),
  inputAmount: Datum.integer(),
  feePerExStep: Datum.integer(),
  outputAsset: Datum.list(Datum.bytes()),
  price: Datum.list(Datum.integer()),
  cancelPkh: Datum.bytes(),
  beacon: Datum.bytes(),
});

export type OnChainOrderDatum = InferDatum<typeof onChainOrderDatum>;
```

## Datum Building

### Price Calculation

Price is always calculated as `output / input` ratio and stored as `[numerator, denominator]` in the datum.

For **Buy** orders (buying token with ADA):
- Input: ADA
- Output: Token
- Price: `token / ADA` = `[tokenAmount, adaAmount]`

For **Sell** orders (selling token for ADA):
- Input: Token
- Output: ADA
- Price: `ADA / token` = `[adaAmount, tokenAmount]`

**Infinity Slippage:**

Set `numerator = 1` to accept any price (market order with no price limit).

Example:
- Buy 100 tokens for 50 ADA → price = `[100, 50]` = 2 tokens per ADA
- Sell 100 tokens for 50 ADA → price = `[50, 100]` = 0.5 ADA per token
- Market order (any price) → price = `[1, denominator]`

### Input Amount

- **Buy order:** `inputAmount` = ADA amount user is willing to spend
- **Sell order:** `inputAmount` = Token amount user is selling

### Address Serialization

Extract user's payment and stake credentials from their Cardano address:

- **Payment credentials:** User's payment key hash (required)
- **Stake credentials:** User's stake key hash (optional)

Datum address structure:
```json
{
  "paymentCredentials": {
    "paymentKeyHash": "<hex-encoded payment key hash>"
  },
  "stakeCredentials": {
    "paymentKeyHash": "<hex-encoded stake key hash>"
  }
}
```

If user has no stake credentials, use empty object `{}` for `stakeCredentials`.

### Beacon Calculation

Order beacon is a 20-byte unique identifier that prevents beacon collisions.

**Beacon Structure:**
- Byte 0: Order type (0 = limit, 1 = market)
- Bytes 1-19: First 19 bytes of blake2b-224 hash

**Algorithm:**

1. **Create empty beacon:** 28 bytes of zeros (hex: `0000...0000`)

2. **Serialize datum with empty beacon:** Convert the datum (with empty beacon) to CBOR format

3. **Hash the datum:** Apply blake2b-224 to the serialized datum CBOR bytes

4. **Build beacon preimage:** Concatenate in order:
   - Transaction hash (32 bytes) of one of the transaction inputs
   - Output index (8 bytes, **big-endian** uint64) of that input
   - Order index (8 bytes, **big-endian** uint64) - index of order output in this transaction
   - Datum hash (28 bytes) from step 3

5. **Calculate final beacon:**
   - Apply blake2b-224 to the preimage
   - Take first 19 bytes
   - Prefix with order type byte (0 or 1)
   - Result: 20 bytes total

> ⚠️ **Important:** All numeric indices must be encoded as **big-endian** uint64 (8 bytes).

**Implementation example:**

```typescript
import { blake2b224, bytesToHex, hexToBytes } from '@splashprotocol/sdk';
import { Uint64BE } from 'int64-buffer';

const EMPTY_BEACON = bytesToHex(Uint8Array.from(new Array(28).fill(0)));

// Step 1: Serialize datum with empty beacon
const datumWithEmptyBeacon = await orderDatum.serialize({
  ...datumObject,
  beacon: EMPTY_BEACON,
});

// Step 2: Hash datum
const datumHash = await blake2b224(
  C.PlutusData.from_cbor_hex(datumWithEmptyBeacon).to_cbor_bytes()
);

// Step 3: Build beacon preimage
const beaconPreimage = Uint8Array.from([
  ...C.TransactionHash.from_hex(outputReference.txHash).to_raw_bytes(),
  ...new Uint64BE(Number(outputReference.index)).toArray(),
  ...new Uint64BE(Number(orderIndex)).toArray(),
  ...hexToBytes(datumHash),
]);

// Step 4: Hash and prefix with order type
const beacon = bytesToHex(
  new Uint8Array([
    orderType === 'limit' ? 0 : 1, // 0 for limit, 1 for market
    ...hexToBytes(await blake2b224(beaconPreimage)).slice(0, 19),
  ])
);
```

Where:
- `outputReference`: One of transaction inputs (UTxO reference)
- `orderIndex`: Index of order output in transaction (BigInt)
- `orderType`: `'limit'` or `'market'`

## Output Building

### Deposit ADA Calculation

Order output must contain sufficient ADA to cover:

1. **Execution fee:**
   - Limit order: `1.8 ADA` (3 × 0.6 ADA)
   - Market order: `0.6 ADA` (1 × 0.6 ADA)

2. **Minimum UTxO deposit:** Calculated based on:
   - Output value size (input asset + ADA)
   - Datum size (~200-300 bytes)
   - Use Cardano's `minUTxOValue` formula or protocol parameters

3. **Collateral:** Minimum total `2 ADA`
   - If (execution fee + UTxO deposit) < 2 ADA, add difference as additional collateral

4. **Receiving deposit:** ADA needed for user's output after order execution
   - Depends on expected output asset amount and address

**Final Output Value:**

```
Order Output = Input Asset
             + Execution Fee (0.6 or 1.8 ADA)
             + UTxO Deposit (calculated)
             + Additional Collateral (if needed to reach 2 ADA minimum)
             + Receiving Deposit (for user's output)
```

Example for Buy order (50 ADA → Token):
- Input Asset: 50 ADA
- Execution Fee: 0.6 ADA (market order)
- UTxO Deposit: ~1.5 ADA (estimated)
- Receiving Deposit: ~1.5 ADA (for token output)
- Additional Collateral: 0 ADA (total already > 2 ADA)
- **Total Output: ~53.6 ADA**

## Transaction Building

Complete flow:

1. **Select input UTxO** covering all costs (input amount + fees + deposits)
2. **Extract user credentials** from address (payment and stake key hashes)
3. **Calculate order index** - position of order output in transaction
4. **Build datum** with all fields except beacon
5. **Calculate beacon** using first input UTxO reference and order index
6. **Serialize datum** with calculated beacon to CBOR
7. **Calculate deposits** for order and receiving outputs
8. **Add collateral** if needed to reach 2 ADA minimum
9. **Create order output** with [script hash](#script-hash), stake credentials, value, and datum
10. **Add change output** for user with remaining funds

See [real transaction example](https://cardanoscan.io/transaction/c77d82d0592252d86a175576665bcf92cbf689ddf92a496cd106cdad75930e79?tab=utxo).

## Order Types

### Limit Order

- Fee: `1.8 ADA` (3 × 0.6 ADA)
- Beacon prefix: `0x00`
- Executes only when market price reaches specified price
- Can be cancelled by user

### Market Order

- Fee: `0.6 ADA` (1 × 0.6 ADA)
- Beacon prefix: `0x01`
- Executes at current market price
- Can be cancelled by user

## Important Notes

1. **First UTxO**: The beacon calculation uses the first transaction input, so ensure you add the UTxO to transaction inputs before calculating the beacon.

2. **Order Index**: Must match the actual index of the order output in the transaction outputs array.

3. **Minimum Collateral**: Total deposits must be at least 2 ADA. Additional collateral is added automatically if needed.

4. **Stake Credentials**: Optional but recommended for proper fund ownership and staking rewards.

5. **Price Inversion**: Buy orders require inverted price ratios (denom/num instead of num/denom).

6. **Asset Format**: ADA is represented as `["", ""]`, tokens as `[policyId, tokenName]` in hex.

7. **Execution**: Orders are executed by protocol batchers when conditions are met. Execution timing depends on batcher availability and market conditions.

## Refund

Orders can be refunded by the user who created them.

Use [script hash](#script-hash), [script CBOR](#script-cbor), and [reference UTxO](#reference-utxo) from Contract Information.

### Execution Units & Redeemer

| Parameter        | Value                                    |
| ---------------- | ---------------------------------------- |
| Memory           | 162,000                                  |
| Steps            | 60,000,000                               |
| Redeemer         | `Constr 0 []`                            |
| Redeemer CBOR    | `d87980`                                 |
| Required Signer  | User's payment key hash from `cancelPkh` |

### Transaction Structure

**Inputs:**

1. Order UTxO to cancel
2. User's UTxO for fees

**Input (Order UTxO):**
- **Redeemer:** Empty constructor `Constr 0 []` (CBOR: `d87980`)
- **Required signer:** User's payment key hash (from `cancelPkh` field in datum)
- **Script reference:** Use [reference UTxO](#reference-utxo) or attach [script CBOR](#script-cbor)
- **Execution units:** See table above

**Outputs:**

1. User's address with all funds from refunded order (including collateral)
2. User's change output (if any)

### Refund Flow

1. **Find order UTxO** by transaction hash and output index
2. **Deserialize datum** to extract user's address and `cancelPkh`
3. **Add order UTxO as input** with:
   - Empty redeemer: `Constr 0 []`
   - Required signer: payment key hash from `cancelPkh`
   - Script reference or inline script
   - Execution units from contract info
4. **Create output** to user's address with full order value
5. **Sign transaction** with user's payment key

### Important Notes

- Only the user who created the order can refund it (signature verified via `cancelPkh`)
- All ADA from order (including collateral and fees) is returned to user
- User's address is extracted from datum `address` field
- Transaction must be signed by the key corresponding to `cancelPkh`
