---
name: run-sql-ecomowner
description: Use when the user wants to run an arbitrary read-only SQL query / ad-hoc data pull against the company SQL Server database through the ecom-owner MCP. Triggers on requests to "query the database", "run SQL", "select from", "pull data", or any question answerable by a custom SELECT that the curated ecom-owner tools (revenue, stock, products, etc.) don't cover.
---

# Read-only SQL via the ecom-owner MCP

Run arbitrary **read-only** SELECT queries against the company database through
two MCP tools on the `ecom-owner` server. There is no direct database access —
everything goes through these tools, which enforce read-only at two layers
(query-text validation + a `db_datareader`-only login).

## Workflow

1. **Discover the schema first.** Call `get_schema` (optionally with a
   `tablePattern` LIKE filter, e.g. `Order%`) to learn exact table and column
   names, primary keys, and foreign-key relationships. Never guess names.
2. **Compose a single SELECT** (or `WITH`/CTE). Use explicit columns, a `TOP`,
   and a `WHERE` to keep the result small.
3. **Call `run_sql`** with the query. Inspect the response:
   `{ columns, rows, rowCount, truncated, error? }`. If `error` is present, the
   guard rejected the query — read it and fix the query (it names the broken
   rule). If `truncated` is true, the result hit the row/cell cap — narrow the
   query (tighter `WHERE`, smaller `TOP`, aggregate) rather than paging blindly.

## Hard rules (the guard rejects these with a clear message)

- **One statement only.** No `;`-separated statements (a single trailing `;` is
  tolerated and stripped).
- **Must start with `SELECT` or `WITH`.** A `WITH` must be a real CTE (it has to
  contain a `SELECT`).
- **No writes or admin commands:** `INSERT/UPDATE/DELETE/MERGE/DROP/ALTER/CREATE/
  TRUNCATE/GRANT/REVOKE/DENY/BACKUP/RESTORE/SHUTDOWN/RECONFIGURE/DBCC/WAITFOR`,
  `SELECT ... INTO`, and `OPENROWSET/OPENQUERY/OPENDATASOURCE` are all blocked.
- **No stored procedures:** `EXEC/EXECUTE`, `sp_*`, `xp_*`.
- **No SQL comments** (`--` or `/* */`) — they could hide blocked tokens.
- **Results are capped** (defaults: 1000 rows / 50000 cells, 30s timeout).
  `truncated=true` means you hit the cap.

Note: `;`, `--`, and blocked keywords *inside a quoted string literal* are fine —
the guard masks string contents before checking, so
`WHERE name = 'A; B -- C'` is accepted.

## Key domain map

The most useful tables and how they join (always confirm exact columns with
`get_schema` — names below are a starting point, not a contract):

- **`Order`** — one row per order. `Order.ChannelId → Channel.Id`. Active orders:
  `State = 1` (Enabled) and `Cancelled IS NULL`. `OrderDate` is the order time;
  `Amount` is the order total.
- **`Channel`** — sales channels/accounts. `Channel.Name` is the individual
  account name (e.g. 'eBay UK Store'); `Channel.Code` is the channel type/code.
- **`Product`** — catalogue. `Product.PriceBuying` is the fallback unit cost.
- **`Stock`** — stock movements. `Stock.OrderId → Order.Id`,
  `Stock.ProductId → Product.Id`. A sale is `StockTypeId = Out` and
  `StockSubtypeId = Sold`; per-line cost = `Stock.PriceBuying ?? Product.PriceBuying ?? 0`.

## When NOT to use this

Prefer the curated ecom-owner tools when they already answer the question
(`get_revenue_summary`, `get_top_products_by_revenue`, `get_stock_levels`,
`search_product_sales`, etc.) — they apply the correct fee/COGS logic. Use
`run_sql` only for ad-hoc questions those tools don't cover.

## Deployment requirement

The MCP server's `CompanyReadonly` connection MUST use a SQL login restricted to
`db_datareader`. The query-text guard is the first layer; the read-only login is
the real backstop. Do not point this connection at a privileged login.
