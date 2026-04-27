# relationshipSearchColumns Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `relationshipSearchColumns` config property to `TablePageConfigColumn` that LEFT-JOINs one-to-one related table columns into the main query when a search value is present, making those columns searchable.

**Architecture:** The type is declared in `@kottster/common`. The JOIN logic lives in `DataSourceAdapter.buildTableRecordsQuery` in `@kottster/server` — it scans the column config for `relationshipSearchColumns`, resolves the relationship, adds a LEFT JOIN with a deterministic alias (`{targetTable}_via_{foreignKeyColumn}`), and appends the qualified column refs to `searchableColumns` before passing them to `getSearchBuilder`. A small fix to `KnexBetterSqlite3.getSearchBuilder` is required because its `??` binding doesn't correctly handle dot-qualified column references (`alias.column`).

**Tech Stack:** TypeScript, Knex, Jest / ts-jest, better-sqlite3 (for integration tests)

---

## File Map

| Action | File | What changes |
|--------|------|-------------|
| Modify | `packages/common/lib/models/tablePage.model.ts` | Add `relationshipSearchColumns?: string[]` to `TablePageConfigColumn` |
| Modify | `packages/server/lib/adapters/knex/knexBetterSqlite3.ts` | Fix `getSearchBuilder` to handle `alias.column` dot-notation references |
| Modify | `packages/server/lib/models/dataSourceAdapter.model.ts` | Add LEFT JOIN logic in `buildTableRecordsQuery` when `input.search` is set |
| Create | `packages/server/__tests__/adapters/relationshipSearchColumns.spec.ts` | Integration tests using in-memory SQLite |

---

## Task 1: Add `relationshipSearchColumns` to `TablePageConfigColumn`

**Files:**
- Modify: `packages/common/lib/models/tablePage.model.ts`

- [ ] **Step 1: Add the property**

In `packages/common/lib/models/tablePage.model.ts`, find the `TablePageConfigColumn` interface. After the `relationshipPreviewColumns` property, add:

```typescript
  /** 
   * Columns from the related (foreign key) table to include in the global text search.
   * The related table is resolved via the column's detected one-to-one relationship.
   * When a search value is present, these columns are LEFT-JOINed into the main query.
   *
   * @example
   * // Makes plaats.kloeke_code and plaats.naam searchable when searching opname rows
   * { column: 'plaats_id', relationshipSearchColumns: ['kloeke_code', 'naam'] }
   */
  relationshipSearchColumns?: string[];
```

- [ ] **Step 2: Build the common package to verify no type errors**

```bash
cd packages/common && pnpm run build
```

Expected: exits 0, `dist/` updated.

- [ ] **Step 3: Commit**

```bash
cd packages/common
git add lib/models/tablePage.model.ts
git commit -m "feat(common): add relationshipSearchColumns to TablePageConfigColumn"
```

---

## Task 2: Write the failing integration test

**Files:**
- Create: `packages/server/__tests__/adapters/relationshipSearchColumns.spec.ts`

The test uses an in-memory SQLite database and the `KnexBetterSqlite3` adapter directly. `getTableRecords` is the public entry point that calls `buildTableRecordsQuery` internally.

- [ ] **Step 1: Create the test file**

```typescript
// packages/server/__tests__/adapters/relationshipSearchColumns.spec.ts
import knex, { Knex } from 'knex';
import { KnexBetterSqlite3 } from '../../lib/adapters/knex/knexBetterSqlite3';
import { RelationalDatabaseSchema, TablePageConfig } from '@kottster/common';

// Shared schema describing opname + plaats so getTableData can resolve columns / relationships
const databaseSchema: RelationalDatabaseSchema = {
  name: 'main',
  tables: [
    {
      name: 'opname',
      columns: [
        { name: 'id',       type: 'integer', fullType: 'integer', nullable: false, primaryKey: { autoIncrement: true }, contentHint: 'number' },
        { name: 'title',    type: 'text',    fullType: 'text',    nullable: true,  contentHint: 'string' },
        { name: 'plaats_id',type: 'integer', fullType: 'integer', nullable: true,  contentHint: 'number',
          foreignKey: { table: 'plaats', column: 'id' } },
      ],
    },
    {
      name: 'plaats',
      columns: [
        { name: 'id',          type: 'integer', fullType: 'integer', nullable: false, primaryKey: { autoIncrement: true }, contentHint: 'number' },
        { name: 'kloeke_code', type: 'text',    fullType: 'text',    nullable: true,  contentHint: 'string' },
        { name: 'naam',        type: 'text',    fullType: 'text',    nullable: true,  contentHint: 'string' },
      ],
    },
  ],
};

// Table page config with relationshipSearchColumns on the FK column
const tablePageConfig: TablePageConfig = {
  fetchStrategy: 'databaseTable',
  table: 'opname',
  columns: [
    {
      column: 'plaats_id',
      relationshipPreviewColumns: ['kloeke_code', 'naam'],
      relationshipSearchColumns: ['kloeke_code', 'naam'],
    },
  ],
};

describe('relationshipSearchColumns', () => {
  let db: Knex;
  let adapter: KnexBetterSqlite3;

  beforeAll(async () => {
    db = knex({
      client: 'better-sqlite3',
      connection: { filename: ':memory:' },
      useNullAsDefault: true,
    });

    await db.schema.createTable('plaats', (t) => {
      t.increments('id').primary();
      t.text('kloeke_code');
      t.text('naam');
    });

    await db.schema.createTable('opname', (t) => {
      t.increments('id').primary();
      t.text('title');
      t.integer('plaats_id').references('id').inTable('plaats').nullable();
    });

    await db('plaats').insert([
      { id: 1, kloeke_code: 'A001', naam: 'Amsterdam' },
      { id: 2, kloeke_code: 'B002', naam: 'Rotterdam' },
    ]);

    await db('opname').insert([
      { id: 1, title: 'Recording 1', plaats_id: 1 },
      { id: 2, title: 'Recording 2', plaats_id: 2 },
      { id: 3, title: 'Recording 3', plaats_id: null },
    ]);

    adapter = new KnexBetterSqlite3(db);
    adapter.setData({ name: 'test', type: 'knex_better_sqlite3' } as any);
    adapter.setTablesConfig({});
  });

  afterAll(async () => {
    await db.destroy();
  });

  it('returns rows whose related naam matches the search term', async () => {
    const result = await adapter.getTableRecords(
      tablePageConfig,
      { page: 1, pageSize: 10, search: 'Amsterdam' },
      databaseSchema,
    );

    expect(result.records).toHaveLength(1);
    expect(result.records[0].id).toBe(1);
  });

  it('returns rows matched by kloeke_code', async () => {
    const result = await adapter.getTableRecords(
      tablePageConfig,
      { page: 1, pageSize: 10, search: 'B002' },
      databaseSchema,
    );

    expect(result.records).toHaveLength(1);
    expect(result.records[0].id).toBe(2);
  });

  it('returns no rows when search matches nothing', async () => {
    const result = await adapter.getTableRecords(
      tablePageConfig,
      { page: 1, pageSize: 10, search: 'nonexistent_xyz' },
      databaseSchema,
    );

    expect(result.records).toHaveLength(0);
  });

  it('preserves rows with null FK (they appear when no search is set)', async () => {
    const result = await adapter.getTableRecords(
      tablePageConfig,
      { page: 1, pageSize: 10 },
      databaseSchema,
    );

    expect(result.records).toHaveLength(3);
  });

  it('does not duplicate rows when search matches in both naam and kloeke_code would', async () => {
    // Insert a row where kloeke_code and naam both contain 'ster'
    await db('plaats').insert({ id: 3, kloeke_code: 'ster-01', naam: 'Kottster' });
    await db('opname').insert({ id: 4, title: 'Recording 4', plaats_id: 3 });

    const result = await adapter.getTableRecords(
      tablePageConfig,
      { page: 1, pageSize: 10, search: 'ster' },
      databaseSchema,
    );

    // Should return exactly 1 row, not 2 (no JOIN duplication)
    expect(result.records).toHaveLength(1);
    expect(result.records[0].id).toBe(4);

    // Clean up
    await db('opname').where({ id: 4 }).del();
    await db('plaats').where({ id: 3 }).del();
  });
});
```

- [ ] **Step 2: Run the test to confirm it fails**

```bash
cd packages/server && pnpm run test -- --testPathPattern="relationshipSearchColumns"
```

Expected: 4–5 tests FAIL. The error will be something like `expect(received).toHaveLength(1)` since no JOIN is added yet and the search returns 0 results.

---

## Task 3: ~~Fix `KnexBetterSqlite3.getSearchBuilder`~~ — already done

> **This task is complete.** The fix was applied on branch `fix/sqlite-search-qualified-column-refs`
> (commit `a5fbcf9`) and cherry-picked into this branch (commit `065de0f`).
> A separate PR from that branch to `main` (upstream) is pending.
>
> **Why the fix was needed:** `whereRaw('?? LIKE ?', [column])` treats the full value as a
> single identifier, so `'alias.column'` becomes `"alias.column"` (one quoted name with a literal
> dot) instead of `"alias"."column"`. Qualified references now use `where(column, 'like', ...)`,
> which Knex correctly splits on the dot and quotes each segment separately.

- [x] **Applied via cherry-pick** — no further action needed in this branch.

---

## Task 4: Implement JOIN logic in `buildTableRecordsQuery`

**Files:**
- Modify: `packages/server/lib/models/dataSourceAdapter.model.ts`

- [ ] **Step 1: Replace the search block in `buildTableRecordsQuery`**

Find the `// Search` comment block (around line 401). It currently reads:

```typescript
    // Search
    if (input.search) {
      const searchValue = input.search.trim();
      if (tablePageProcessedConfig.searchableColumns?.length > 0) {
        query.where(this.getSearchBuilder(tablePageProcessedConfig.searchableColumns, searchValue));
        countQuery?.where(this.getSearchBuilder(tablePageProcessedConfig.searchableColumns, searchValue));
      }
    }
```

Replace it with:

```typescript
    // Search
    if (input.search) {
      const searchValue = input.search.trim();

      // Collect qualified column refs from one-to-one relationships configured with relationshipSearchColumns
      const extraSearchColumns: string[] = [];
      const joinedAliases = new Set<string>();

      tablePageProcessedConfig.columns?.forEach(col => {
        if (!col.relationshipSearchColumns?.length) return;

        const relationship = (tablePageProcessedConfig.relationships ?? [])
          .find(r => r.relation === 'oneToOne' && (r as OneToOneRelationship).foreignKeyColumn === col.column) as OneToOneRelationship | undefined;

        if (!relationship?.targetTable || !relationship?.targetTableKeyColumn) return;

        const alias = `${relationship.targetTable}_via_${col.column}`;

        if (!joinedAliases.has(alias)) {
          query.leftJoin(
            `${relationship.targetTable} as ${alias}`,
            `main.${col.column}`,
            `${alias}.${relationship.targetTableKeyColumn}`,
          );
          countQuery?.leftJoin(
            `${relationship.targetTable} as ${alias}`,
            `main.${col.column}`,
            `${alias}.${relationship.targetTableKeyColumn}`,
          );
          joinedAliases.add(alias);
        }

        col.relationshipSearchColumns.forEach(searchCol => {
          extraSearchColumns.push(`${alias}.${searchCol}`);
        });
      });

      const allSearchColumns = [
        ...(tablePageProcessedConfig.searchableColumns ?? []),
        ...extraSearchColumns,
      ];

      if (allSearchColumns.length > 0) {
        query.where(this.getSearchBuilder(allSearchColumns, searchValue));
        countQuery?.where(this.getSearchBuilder(allSearchColumns, searchValue));
      }
    }
```

`OneToOneRelationship` is already imported at the top of this file — no import change needed.

---

## Task 5: Verify, rebuild, and commit

- [ ] **Step 1: Run the server tests**

```bash
cd packages/server && pnpm run test -- --testPathPattern="relationshipSearchColumns"
```

Expected: all 5 tests PASS.

- [ ] **Step 2: Run the full server test suite**

```bash
cd packages/server && pnpm run test
```

Expected: all existing tests still pass (no regressions).

- [ ] **Step 3: Build the server package**

```bash
cd packages/server && pnpm run build
```

Expected: exits 0.

- [ ] **Step 4: Commit**

```bash
cd packages/server
git add \
  lib/adapters/knex/knexBetterSqlite3.ts \
  lib/models/dataSourceAdapter.model.ts \
  __tests__/adapters/relationshipSearchColumns.spec.ts
git commit -m "feat(server): add relationshipSearchColumns — LEFT JOIN related columns into search query"
```

---

## Self-Review Notes

- **Spec coverage:** All spec requirements covered — type definition (Task 1), JOIN-on-search logic (Task 4), LEFT JOIN for null FK safety (Task 4: `leftJoin`), alias strategy for duplicate target tables (Task 4: `joinedAliases` Set), no JOIN when search is absent (Task 4: guarded by `if (input.search)`), no changes to `getSearchBuilder` signature or `applyFilters` (confirmed).
- **Future work preserved:** The alias convention (`{table}_via_{column}`) is documented in the spec and used here — it can be reused in `applyFilters` for filter panel support without changes to this code.
- **No placeholders.**
- **Type consistency:** `OneToOneRelationship`, `relationshipSearchColumns`, `extraSearchColumns`, `allSearchColumns` — names are consistent across all tasks.
