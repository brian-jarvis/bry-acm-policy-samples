---
# Perses UI/core issue: `JoinByColumnValue` requires every per-query-suffixed
# column name to be listed explicitly

While debugging an "Installed operators" table that always rendered exactly
one row (instead of one row per operator, ~24 expected) in the `ocp-lifecycle`
dashboard in this folder, we traced the behavior to Perses' own `Join by
column value` transform implementation, not to anything `percli` generated
incorrectly. This document exists to be filed as an upstream issue against
[`perses/perses`](https://github.com/perses/perses) (the core table
transform/UI behavior, implemented in
[`perses/shared`](https://github.com/perses/shared)).

---

## Summary

A Table panel with more than one query gets every label key — not just the
synthetic `value` column — suffixed with ` #<queryIndex+1>` before any
`transforms` run (this is documented/expected Perses behavior, confirmed in
`perses/plugins`, `table/src/table-data-utils.ts`,
`buildTimeSeriesTableData()`). However, the `Join by column value`
(`JoinByColumnValue`) transform's own `columns` option does **not** know
about this suffixing: it matches column names by exact string equality
against each row's actual (already-suffixed) keys. If a dashboard author
writes the obvious, intuitive config — `columns: [cluster]` — naming just the
bare label that every query groups by, the transform matches **zero** keys on
every row, which causes *every row in the table to silently collapse into a
single merged row*, with no error, warning, or any other indication that
anything went wrong. The only way to get correct behavior is to list every
per-query-suffixed instance of the join column(s) explicitly (e.g. `['cluster
#1', 'cluster #2', ..., 'cluster #9']` for a panel with 9 queries), which is
unintuitive, verbose, and easy to get subtly wrong (see "Why this is
dangerous" below).

## Source: the transform implementation

`perses/shared`, `components/src/utils/transform-data.ts`,
`applyJoinTransform()`:

```ts
export function applyJoinTransform(
  data: Array<Record<string, unknown>>,
  columns: string[],
): Array<Record<string, unknown>> {
  if (columns.length === 0) {
    return data;
  }
  const rowHashed: { [key: string]: Record<string, unknown> } = {};
  for (const row of data) {
    const rowHash = Object.keys(row)
      .filter((k) => columns.includes(k))
      .map((k) => row[k])
      .join('|');
    const rowHashedValue = rowHashed[rowHash];
    if (rowHashedValue) {
      rowHashed[rowHash] = { ...rowHashedValue, ...row };
    } else {
      rowHashed[rowHash] = { ...row };
    }
  }
  return Object.values(rowHashed);
}
```

`columns.includes(k)` is a plain exact-string match against each row's raw
object keys. It never strips, normalizes, or fuzzy-matches the
` #<queryIndex+1>` suffix that the panel's own data-building step
(`buildTimeSeriesTableData()` in `perses/plugins`,
`table/src/table-data-utils.ts`) unconditionally adds to every label key
whenever the panel has more than one query:

```ts
// If there are multiple queries, add query index to value key and label keys to avoid conflicts
const valueColumnName = queryResults.length === 1 ? 'value' : `value #${queryIndex + 1}`;
const labels =
  queryResults.length === 1
    ? ts.labels
    : Object.entries(ts.labels ?? {}).reduce((acc, [key, value]) => {
        if (key) acc[`${key} #${queryIndex + 1}`] = value;
        return acc;
      }, {} as Labels);
```

So by the time `applyJoinTransform` runs, a row never has a bare `cluster`
key to match against — only `cluster #1`, `cluster #2`, etc.

## What was generated / observed

Dashboard config (8 queries, all grouping by `cluster`, `operator_namespace`,
`package` among other per-query labels — see this folder's
`ocp-lifecycle-perses.yml` for the full panel):

```yaml
transforms:
- kind: JoinByColumnValue
  spec:
    columns:
    - join_key
```

(`join_key` here is itself a label synthesized via a PromQL `label_join(...,
"cluster", "operator_namespace", "package")` in every query, specifically to
work around `JoinByColumnValue` not supporting a true composite multi-column
key — but the underlying suffixing problem affects a single bare column name
just as much, e.g. the simpler `columns: [cluster]` case below.)

Live behavior: confirmed via direct Prometheus query against the dashboard's
own datasource that the underlying data genuinely contains 24 distinct
`join_key` values (one per installed operator). Despite this, the rendered
table — and a CSV export of it — always showed **exactly one row**. For every
row, `Object.keys(row).filter((k) => columns.includes(k))` evaluates to `[]`
(since every real key is suffixed, e.g. `join_key #1`..`#8`, and none of them
equal the bare string `join_key`), so `[].join('|')` produces the same empty
string `''` as the `rowHash` for all 24 rows, and the accumulating `{
...rowHashedValue, ...row }` spread silently merges all of them into one.

## Why this is dangerous (a second, worse manifestation)

A second panel in the same dashboard ("Clusters") was, at the time, assumed
to be working correctly with the simpler, intuitive config:

```yaml
transforms:
- kind: JoinByColumnValue
  spec:
    columns:
    - cluster
```

This panel also hits the exact same bug — `columns: [cluster]` matches
nothing against real rows keyed `cluster #1`..`cluster #9` — but it happened
to *look* correct, because this particular environment only has one managed
cluster. The "merge everything into one row" failure mode is
indistinguishable from the "correctly joined one row per cluster" success
case when there is only one cluster's worth of data. This makes the bug a
silent, **latent** data-correctness issue: it will not surface until a second
managed cluster (or any other dimension with >1 real value) is added, at
which point two clusters' rows would silently merge into one with no error.
A dashboard author relying on the `columns: [cluster]` form has no way to
detect this ahead of time short of already knowing to list the suffixed
variants.

## What was expected

Either of the following would resolve this:

1. **(Preferred)** `applyJoinTransform` (and any other transform/column
   config that accepts a bare column name) should match against the
   suffix-stripped form of each key, so `columns: [cluster]` correctly
   matches `cluster #1`, `cluster #2`, etc. across all of a panel's queries.
   This is the behavior a dashboard author would reasonably expect, since the
   panel's own `columnSettings` already requires this same suffixed-vs-bare
   distinction to be handled manually (see this folder's `percli-issues.md`,
   issue 5) — the join transform doing the same normalization internally
   would make the two mechanisms consistent.
2. **(Minimum)** If exact-match-only behavior is intentional, the transform
   should at minimum warn or error when `columns` contains a name that
   matches zero keys on every input row (a strong signal of exactly this
   misconfiguration), rather than silently collapsing all rows. The Table
   panel editor UI should also surface the *actual* available column names
   (i.e., the suffixed ones) when a user is choosing columns for a
   `JoinByColumnValue` transform, rather than requiring the user to infer the
   suffix naming convention from documentation/source.

What we had to configure in practice, to work around the bug today (listing
every per-query-suffixed instance of the join column explicitly):

```yaml
transforms:
- kind: JoinByColumnValue
  spec:
    columns:
    - 'cluster #1'
    - 'cluster #2'
    - 'cluster #3'
    - 'cluster #4'
    - 'cluster #5'
    - 'cluster #6'
    - 'cluster #7'
    - 'cluster #8'
    - 'cluster #9'
```

This is correct, but unintuitive (nothing in the Table panel editor UI or the
`JoinByColumnValue` documentation at
[perses.dev/plugins/docs/table](https://perses.dev/plugins/docs/table/)
mentions that bare column names must be manually expanded to every
per-query-suffixed variant), verbose (grows linearly with query count, up to
15 entries in one of our panels), and brittle (adding or removing a query
from the panel silently breaks the join unless every `columns` list in that
panel is updated to match the new query count — again with no warning if
missed).

## Environment used to reproduce

- Perses version: `v0.54.0` (confirmed as the pinned upstream version
  compiled into Red Hat's Cluster Observability Operator 1.5.3 downstream
  build, via `rhobs/konflux-coo`'s `release-1.5` branch `Dockerfile.perses`,
  `ARG VERSION=v0.54.0`)
- Reproduced both against a disposable local `persesdev/perses:v0.54.0`
  server and against the Red Hat downstream build shipped by Cluster
  Observability Operator 1.5.3 (`perses-rhel9` image) on an OpenShift/ACM
  cluster
- Affected panel type: Table (`perses/plugins`, `table` plugin)
- Affected transform: `Join by column value` (`JoinByColumnValue`)
