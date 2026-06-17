---
description: Answer an e-commerce data question using the ecom-owner MCP (curated analytics + read-only SQL).
argument-hint: "<a question about sales, stock, products, revenue…>"
---

Answer this e-commerce data question for the business owner using the **ecom-owner** MCP tools:

> $ARGUMENTS

Guidance:

- Prefer the **curated tools** when they fit the question (`get_business_snapshot`,
  `get_revenue_summary`, `get_top_products_by_revenue`, `get_stock_levels`,
  `get_dead_stock`, `search_product_sales`, `get_refunds_and_returns`, etc.) — they
  apply the correct fee/COGS logic.
- For ad-hoc questions the curated tools don't cover, fall back to read-only SQL:
  call `get_schema` first to learn exact table/column names, then `run_sql` with a
  single `SELECT`/`WITH` (explicit columns, a `TOP`, a `WHERE`). If `run_sql` returns
  an `error`, read it and fix the query.
- Present the result as a concise table, and state the period and any filters you used.

If no question was supplied, ask the owner what they'd like to know.
