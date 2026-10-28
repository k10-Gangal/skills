# Conventions

> Project defaults. The user or an existing project pattern overrides these —
> ask the user how their existing models are named and follow that.

## Layers

```
  staging.stg_<source>      1:1 with a source table. cast, rename, light clean.
                            kind VIEW. no joins, no aggregates.
  marts.<subject>_<grain>   business logic. joins, aggregates.
                            reads staging.*, never raw sources.
```

Build staging before marts. Reuse an existing staging model instead of adding a
second one over the same source.

## Names

```
  snake_case everywhere
  _at          timestamps          placed_at
  _date        dates               order_date
  is_ / has_   booleans            is_active, has_prior_order
  _count       counts              order_count
  _rate        0–1                 conversion_rate
  _pct         0–100               churn_pct
  _id          keys                customer_id
```

Name a mart for its grain: `revenue_by_region_month`, `customer_daily`.

## Files

One model per file, named after the model without its schema:

```
  models/staging/stg_orders.sql
  models/marts/revenue_by_region.sql
```

The path is convention only: the MODEL `name` is the real name (unless the
project sets `infer_names`). Keep the two in sync.

## Every model has

`kind` explicit · `grain` · `description` · at least one meaningful audit ·
every output column cast and aliased.
