# Design: `relationshipSearchColumns` — Search Across One-to-One Related Columns

**Date:** 2026-04-27  
**Status:** Draft

---

## Problem

Kottster already supports displaying data from one-to-one related tables via `relationshipPreviewColumns`. For example, a table `opname` with a `plaats_id` foreign key can show `kloeke_code` and `naam` from the related `plaats` table. This display is powered by a separate batch query executed after the main query.

Because those related columns never appear in the main SQL query, they cannot participate in `WHERE` clauses. As a result, the global search box cannot search on related values — a user cannot search an `opname` list by `plaats.naam`.

---

## Goal

Add a new opt-in column config property, `relationshipSearchColumns`, that causes specified related-table columns to be LEFT-JOINed into the main query when a search value is present, making them eligible for the global text search.

---

## Non-Goals

- Per-column UI filter panel support (deferred — tracked as future work; the JOIN mechanism introduced here is designed to be reusable for that purpose).
- Support for one-to-many relationships (only one-to-one / foreign-key-based joins are in scope).
- Auto-discovery of searchable related columns without explicit configuration.

---

## API Design

### New property on `TablePageConfigColumn`

```typescript
interface TablePageConfigColumn {
  // ...existing props...

  /**
   * Columns from the related (foreign key) table to include in the global search.
   * The related table is resolved via the column's foreign key relationship.
   * Requires the column to have a detected one-to-one relationship.
   *
   * @example
   * // opname.plaats_id → plaats.kloeke_code and plaats.naam become searchable
   * { column: 'plaats_id', relationshipSearchColumns: ['kloeke_code', 'naam'] }
   */
  relationshipSearchColumns?: string[];
}
```

### Example developer configuration

```typescript
createApp({
  schema,
  pages: {
    opname: {
      fetchStrategy: 'databaseTable',
      table: 'opname',
      columns: [
        {
          column: 'plaats_id',
          relationshipPreviewColumns: ['kloeke_code', 'naam'],
          relationshipSearchColumns: ['kloeke_code', 'naam'],  // NEW
        },
      ],
    },
  },
});
```

---

## Implementation Design

### Where the change lives

All query-building logic is in `DataSourceAdapter.buildTableRecordsQuery` (`packages/server/lib/models/dataSourceAdapter.model.ts`). The new logic is added there, scoped to the `input.search` branch. No changes to `getSearchBuilder`, `applyFilters`, or any enrichment logic.

### Join alias strategy

To prevent conflicts when two FK columns in the same table reference the same related table, each JOIN gets a deterministic alias:

```
{targetTable}_via_{foreignKeyColumn}
```

Example: `plaats_via_plaats_id`

### What gets added to the query

When `input.search` is set:

1. Walk `tablePageProcessedConfig.columns` for entries that have `relationshipSearchColumns`.
2. For each such column, look up its one-to-one relationship from `tablePageProcessedConfig.relationships` (matched by `relationship.foreignKeyColumn === column.column`).
3. If a relationship is found:
   - Add `LEFT JOIN {targetTable} AS {alias} ON main.{foreignKeyColumn} = {alias}.{targetTableKeyColumn}`
   - Append `{alias}.{searchCol}` for each entry in `relationshipSearchColumns` to the `extraSearchColumns` list.
4. Pass `[...searchableColumns, ...extraSearchColumns]` to `getSearchBuilder`.

### Data flow summary

```
input.search present?
  └─ yes → scan columns for relationshipSearchColumns
              └─ for each match: resolve relationship → add LEFT JOIN → collect qualified column refs
            pass [searchableColumns + joined refs] to getSearchBuilder
            getSearchBuilder generates ILIKE WHERE conditions for all of them
  └─ no  → no JOINs added (no performance impact)
```

### JOIN type

`LEFT JOIN` — records with a NULL FK value (no related row) are preserved in results and simply don't match the search term on the joined columns.

---

## Files Changed

| File | Package | Change |
|------|---------|--------|
| `lib/models/tablePage.model.ts` | `@kottster/common` | Add `relationshipSearchColumns?: string[]` to `TablePageConfigColumn` |
| `lib/models/dataSourceAdapter.model.ts` | `@kottster/server` | In `buildTableRecordsQuery`: detect `relationshipSearchColumns`, add LEFT JOINs, extend search columns |

No changes required to: `getSearchBuilder`, `applyFilters`, enrichment logic, any adapter subclasses, or any common utilities.

---

## Future Work

- **Filter panel support**: The same JOIN alias mechanism can be reused in `applyFilters`. A `FilterItem` referencing a related column would trigger the same LEFT JOIN pattern. UI discovery of related columns as filter targets is a separate, deferred design question.
- **`relationshipFilterColumns`**: A parallel property for filter panel support, following the same naming convention.
