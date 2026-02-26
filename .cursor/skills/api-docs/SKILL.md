---
name: api-docs
description: Create and maintain Shadowbook API documentation pages. Use when adding a new endpoint, updating response types, adding response codes, or editing anything in the api-docs/ directory. Source of truth for all API details is API.md.
---

# Shadowbook API docs

## Source of truth

Always read `API.md` before making any changes to `api-docs/`. It contains the authoritative endpoint definitions, type schemas, query parameters, and response codes.

## Page structure

Each endpoint gets its own MDX file in `api-docs/`. Required frontmatter:

````mdx
---
title: SHORT_ACTION_PHRASE
description: "ONE_SENTENCE."
api: "GET /trading-view/your-endpoint"
---
````

The `api` field pairs with the `server` in `docs.json` (`https://api.shadowbook.io`). No auth is required — these are public read-only endpoints.

After creating a page, add it to the `"API"` group in `docs.json`.

## Component patterns

### Query parameters

````mdx
<ParamField query="base" type="string" required>
  Base asset identifier. Use `ADA` for Cardano's native asset, or `POLICY_ID_HEX:ASSET_NAME_HEX` for native tokens.
</ParamField>
````

Use `query="…"` for URL query params. Mark all required params with `required`.

### Response fields

````mdx
<ResponseField name="spot" type="DecimalPrice | null">
  Most recent execution price. `null` if no trades have occurred.
</ResponseField>
````

For nested objects, wrap child fields in `<Expandable>`:

````mdx
<ResponseField name="pairs" type="PairInfo[]">
  Array of trading pairs.
  <Expandable title="PairInfo fields">
    <ResponseField name="base" type="string">…</ResponseField>
    <ResponseField name="last_spot" type="DecimalPrice | null">…</ResponseField>
  </Expandable>
</ResponseField>
````

### Response codes table

Add a `## Response codes` section (left column) to document status codes and their conditions:

````mdx
## Response codes

| Status | Body | Condition |
|--------|------|-----------|
| `200 OK` | `GetOrderBookResponse` | Pair exists in the index |
| `404 Not Found` | `{"error": "..."}` | Pair not found |
````

Omit the Condition column if there is only one possible status code.

### Right-panel examples

Use `<RequestExample>` and `<ResponseExample>` to pin examples in the right sidebar. Place these **after** all left-column content.

````mdx
<RequestExample>

```bash Request
curl "https://api.shadowbook.io/trading-view/order-book?base=ADA&quote=TOKEN"
```

</RequestExample>

<ResponseExample>

```json 200
{ … }
```

```json 404
{ "error": "Pair ADA/UNKNOWN not found" }
```

</ResponseExample>
````

Each code block title inside `<ResponseExample>` is the HTTP status code (`200`, `404`, etc.). Include one block per possible response code.

## Shared types

These types are defined in `api-docs/overview.mdx` and referenced across endpoint pages.

| Type | Description |
|------|-------------|
| `DecimalPrice` | Decimal string — quote/base ratio (e.g. `"0.00001"`) |
| `PriceLevel` | `{ price: DecimalPrice, accumulated_liquidity: u64 }` |
| `PairInfo` | `{ base, quote, last_spot: DecimalPrice \| null }` |
| `GetOrderBookResponse` | `{ base, quote, spot, bids: PriceLevel[], asks: PriceLevel[] }` |
| `PairListResponse` | `{ pairs: PairInfo[] }` |

## Asset identifier format

| Asset | Format |
|-------|--------|
| ADA | `"ADA"` |
| Native token | `"POLICY_ID_HEX:ASSET_NAME_HEX"` (56-char policy ID, hex asset name) |

## Full page example

See `api-docs/order-book.mdx` for a complete reference page with all sections.
