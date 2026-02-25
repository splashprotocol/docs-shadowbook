# Market API

Read-only HTTP API for querying the Shadow DEX order book and trading pair data.

## Base URL

`https://api.shadowbook.io/`

## Type reference

### Asset identifier

Assets are passed as plain strings in query parameters and returned as strings in responses.

| Asset | Format | Example |
|-------|--------|---------|
| ADA (Cardano native) | `"ADA"` | `ADA` |
| Native token | `"<policy_id_hex>:<asset_name_hex>"` | `279c909f348e533da5808898f87f9a14bb2c3dfbbacccd631d927a3f:534e454b` |

- `policy_id_hex` — 56 lowercase hex characters (28 bytes).
- `asset_name_hex` — hex-encoded asset name bytes (variable length, may be empty).

**Examples:**

```
ADA
279c909f348e533da5808898f87f9a14bb2c3dfbbacccd631d927a3f:534e454b    (SNEK)
8db269c3ec630e06ae29f74bc39edd1f87c819f1056206e879a1cd61:dda5fdb1002f7389b33e036b6afee82a27189467772a31503f6185a66cb3d2   (djed)
```

### `DecimalPrice`

A price value stored and transmitted as a decimal number. Serialised via `rust_decimal::Decimal`.

```
0.00001
1.5
100000
```

Represents a Quote/Base ratio — how many quote-asset units equal one base-asset unit.

### `PriceLevel`

A single rung of the order book with the aggregated liquidity available at that price.

```json
{
  "price": "0.00001",
  "accumulated_liquidity": 5000000000
}
```

| Field | Type | Description |
|-------|------|-------------|
| `price` | `DecimalPrice` | Limit price (quote units per base unit) |
| `accumulated_liquidity` | `u64` | Total base-asset amount available at this price |

### `GetOrderBookResponse`

```json
{
  "base": "ADA",
  "quote": "279c909f348e533da5808898f87f9a14bb2c3dfbbacccd631d927a3f:534e454b",
  "spot": "0.00001",
  "bids": [
    { "price": "0.000011", "accumulated_liquidity": 3000000000 },
    { "price": "0.000010", "accumulated_liquidity": 1500000000 }
  ],
  "asks": [
    { "price": "0.000012", "accumulated_liquidity": 2000000000 },
    { "price": "0.000013", "accumulated_liquidity": 800000000 }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `base` | `string` | Base asset identifier (echoed from query) |
| `quote` | `string` | Quote asset identifier (echoed from query) |
| `spot` | `DecimalPrice \| null` | Most recent execution price; `null` if no trades have occurred |
| `bids` | `PriceLevel[]` | Buy-side levels, highest price first |
| `asks` | `PriceLevel[]` | Sell-side levels, lowest price first |

### `PairInfo`

```json
{
  "base": "ADA",
  "quote": "279c909f348e533da5808898f87f9a14bb2c3dfbbacccd631d927a3f:534e454b",
  "last_spot": "0.00001"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `base` | `string` | Base asset identifier |
| `quote` | `string` | Quote asset identifier |
| `last_spot` | `DecimalPrice \| null` | Last execution price; `null` if no trades yet |

### `PairListResponse`

```json
{
  "pairs": [
    {
      "base": "ADA",
      "quote": "279c909f348e533da5808898f87f9a14bb2c3dfbbacccd631d927a3f:534e454b",
      "last_spot": "0.00001"
    },
    {
      "base": "ADA",
      "quote": "8db269c3ec630e06ae29f74bc39edd1f87c819f1056206e879a1cd61:dda5fdb1002f7389b33e036b6afee82a27189467772a31503f6185a66cb3d2",
      "last_spot": null
    }
  ]
}
```

### Error response

```json
{ "error": "Pair ADA/UNKNOWN not found" }
```

---

## Endpoints

### `GET /trading-view/order-book`

Returns a snapshot of the order book for a trading pair, including price levels on both sides and the current spot price.

**Query parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `base` | string | yes | Base asset identifier |
| `quote` | string | yes | Quote asset identifier |

**Responses**

| Status | Body | Condition |
|--------|------|-----------|
| `200 OK` | `GetOrderBookResponse` | Pair exists in the index |
| `404 Not Found` | `{"error": "..."}` | Pair not found |

**Example request**

```
https://api.shadowbook.io/trading-view/order-book?base=0691b2fecca1ac4f53cb6dfb00b7013e561d1f34403b957cbb5af1fa:4e49474854&quote=ADA
```

**Example response — 200 OK**

```json
{
  "base": "ADA",
  "quote": "279c909f348e533da5808898f87f9a14bb2c3dfbbacccd631d927a3f:534e454b",
  "spot": "0.00001",
  "bids": [
    { "price": "0.0000110", "accumulated_liquidity": 3000000000 },
    { "price": "0.0000100", "accumulated_liquidity": 1500000000 },
    { "price": "0.0000090", "accumulated_liquidity": 500000000 }
  ],
  "asks": [
    { "price": "0.0000120", "accumulated_liquidity": 2000000000 },
    { "price": "0.0000130", "accumulated_liquidity": 800000000 }
  ]
}
```

**Example response — 404 Not Found**

```json
{
  "error": "Pair ADA/UNKNOWN not found"
}
```

---

### `GET /trading-view/pair-list`

Returns all trading pairs currently tracked in the market index along with their last known spot price.

**Query parameters**

None.

**Responses**

| Status | Body |
|--------|------|
| `200 OK` | `PairListResponse` |

**Example request**

```
GET https://api.shadowbook.io/trading-view/pair-list
```

**Example response — 200 OK**

```json
{
  "pairs": [
    {
      "base": "ADA",
      "quote": "279c909f348e533da5808898f87f9a14bb2c3dfbbacccd631d927a3f:534e454b",
      "last_spot": "0.00001"
    },
    {
      "base": "ADA",
      "quote": "8db269c3ec630e06ae29f74bc39edd1f87c819f1056206e879a1cd61:dda5fdb1002f7389b33e036b6afee82a27189467772a31503f6185a66cb3d2",
      "last_spot": null
    }
  ]
}
```
