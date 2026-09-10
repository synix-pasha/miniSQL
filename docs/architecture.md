# MiniSQL — Architecture Documentation

A from-scratch SQL engine written in C++ with a REPL front end, a hand-written
recursive-descent-style parser, a rule-based query optimizer, and **two
interchangeable storage engines**: a simple heap file and a disk-backed
B+Tree engine with primary-key clustering and secondary indexes.

This document describes how the system is put together: the pipeline a query
travels through, the on-disk formats, the page-based B+Tree internals, and
the conventions that tie the modules together.

---

## 1. Overview

MiniSQL is a single-process, single-database SQL shell. You type a
statement, it gets tokenized, parsed into a typed `operation` object, costed
by a query optimizer into an `AccessPath`, and handed to an executor that
picks one of ~25 concrete handler functions depending on the statement type
and the chosen access path.

Every table is **either** a heap table **or** a clustered (B+Tree) table —
decided once at `CREATE TABLE` time based on whether a `PRIMARY KEY` column
is declared:

| Table kind | Triggered by                          | Storage layout                                   |
|------------|----------------------------------------|---------------------------------------------------|
| Heap       | `CREATE TABLE` with **no** `PRIMARY KEY` | Flat file of fixed-width rows, append-only inserts, full-buffer rewrite on update/delete |
| Clustered  | `CREATE TABLE` with a `PRIMARY KEY`      | Disk-paged B+Tree keyed on the PK; optional secondary B+Tree indexes on `INDEX` columns |

There is no transaction manager, no write-ahead log, and no buffer/cache
pool — every statement opens the files it needs, does the work, and closes
them. Concurrency and crash recovery are out of scope.

---

## 2. System Pipeline

```
 ┌─────────────┐   raw text    ┌───────────────────┐
 │  main.cpp   │ ───────────►  │  query_processor   │  tokenizer()
 │  (REPL loop)│               │  (parser/)          │  command_router()
 └─────────────┘               └─────────┬──────────┘
                                          │ operation (typed AST-ish struct)
                                          ▼
                                ┌───────────────────┐
                                │  QueryOptimizer     │  optimise()
                                │  (optimiser/)        │
                                └─────────┬──────────┘
                                          │ AccessPath (chosen strategy)
                                          ▼
                                ┌───────────────────┐
                                │      executor        │  execute()
                                │  (executor/)          │  → 1-of-25 handlers
                                └─────────┬──────────┘
                                          │
                         ┌────────────────┴───────────────────┐
                         ▼                                    ▼
               ┌───────────────────┐                ┌─────────────────────┐
               │  Heap engine        │                │  Clustered engine      │
               │  database / buffer  │                │  BplusTree / Index /   │
               │  (storage/)          │                │  Disk_Manager           │
               └───────────────────┘                └─────────────────────┘
                         │                                    │
                         └──────────────┬─────────────────────┘
                                          ▼
                                ┌───────────────────┐
                                │  catalog / schema     │  metadata shared by
                                │  (storage/)             │  both engines
                                └───────────────────┘
```

`main.cpp` owns one long-lived instance each of `query_processor`,
`executor`, and `QueryOptimizer`, and re-runs the four-stage pipeline
(tokenize → parse/route → optimise → execute) for every line typed until
`EXIT`.

---

## 3. Module Map

| Folder         | Files                                              | Responsibility |
|-----------------|-----------------------------------------------------|----------------|
| `parser/`        | `parser.h/.cpp`                                      | Tokenizer, per-statement recursive parsers, the `operation` union-of-structs |
| `optimiser/`      | `QueryOptimiser.h/.cpp`                              | Picks an `AccessPath` (PK/SK point or range lookup, full scan, or heap) |
| `executor/`       | `executor.h/.cpp`                                    | ~25 statement×access-path handler functions; the only module that talks to *both* storage engines |
| `storage/`        | `core.h/.cpp`                                        | `catalog` (table registry) and `database` (heap-engine façade) |
|                 | `schema.h/.cpp`                                      | Column metadata, `.schema` file format, shared by both engines |
|                 | `buffer.h/.cpp`                                      | Heap engine's row (de)serialization, WHERE filtering, in-place update/delete |
|                 | `DataHandling.h/.cpp`                                | Shared row (de)serialization + pretty-printing used by the **clustered** engine |
|                 | `DiskManager.h/.cpp`                                 | Raw fixed-size page I/O, page allocation/free-list, page/meta headers |
|                 | `BPlusTree.h/.cpp`                                   | The on-disk B+Tree itself: search, insert+split, delete+merge/borrow, scans |
| `index/`          | `Index.h/.cpp`                                        | Thin wrapper over a `BplusTree` specialised for secondary indexes (composite key + PK lookups) |
| `utility/`        | `error.h`, `utils.h/.cpp`, `filter.h/.cpp`             | `DB_error`, datatypes, key comparators, the WHERE-clause expression tree |

---

## 4. The Query Lifecycle

1. **`read_query()`** (`utility/utils.cpp`) reads lines from stdin until it
   sees a `;`, respecting single-quoted strings so semicolons inside string
   literals don't end the statement early.
2. **`query_processor::tokenizer()`** splits the statement into tokens:
   identifiers/keywords, quoted string literals, `(`, `)`, `,`, and the
   comparison operators (`=`, `!=`, `>`, `<`, `>=`, `<=`).
3. **`query_processor::command_router()`** looks at `tokens[0]`
   (`SELECT` / `INSERT` / `CREATE` / `UPDATE` / `DELETE` / `SHOW` / `DROP`)
   and calls the matching `parser_*()` method, filling in one of five typed
   sub-structs (`create_operation`, `insert_operation`, `select_operation`,
   `update_operation`, `delete_operation`) inside a top-level `operation`
   object. Each parser is a small hand-rolled recursive scanner over the
   token array — there's no separate AST library; `WHERE` clauses are
   handed off to `where_clause::make_tree()` (see §10).
4. **`QueryOptimizer::optimise(operation&)`** inspects the statement type,
   loads the table's `schema`, and returns an `AccessPath` describing how
   the executor should read the table (see §8).
5. **`executor::execute(operation&, AccessPath&)`** dispatches to one
   concrete handler based on `(operation_type, AccessPath.type)` (see §9).
6. `main.cpp` times the whole cycle with `<chrono>` and prints
   `Query completed in N ms`.

---

## 5. Catalog & Schema Layer

These two file types are metadata shared by **both** storage engines.

### 5.1 Catalog (`<db>.catalog`)

One catalog file per database (in practice there is a single implicit
database, `"main"` — see §11). Binary, hand-packed:

```
[int32 table_count]
repeated table_count times:
  [uint8  name_len][name_len bytes name]
  [uint16 column_count]
  [uint8  is_clustered]   (0 = heap, 1 = clustered)
```

`catalog` (in `storage/core.h`) loads this fully into fixed-size C arrays
(`MAX_TABLES = 50`) and supports `add_table`, `remove_table`, `has_table`,
and `print_catalog` (used by `SHOW DATABASE`).

### 5.2 Schema (`<db>_<table>.schema`)

One file per table, loaded by `schema::load_schema()` into heap-allocated
parallel arrays (`column_name[]`, `dtypes[]`, `col_offset[]`, `col_index[]`).
Binary layout:

```
[uint8  num_of_cols]
[uint8  is_clustered]
[uint16 row_size]        total bytes of one row
[uint16 page_size]       always 4096
[uint8  table_name_len][table_name bytes]
repeated num_of_cols times:
  [uint8  col_name_len][col_name bytes]
  [uint8  datatype]      0=INT 1=TEXT 2=BOOL
  [uint8  col_index]     0=none 1=PRIMARY KEY 2=SECONDARY INDEX
  [uint16 col_size]      omitted for the last column (implied by row_size)
```

`schema` exposes the accessors the rest of the codebase relies on:
`getColumnOffset`, `getColumnSize`, `getColumnType`, `getColumnIndex` (name
→ index), `getColumnIndexType` (PK/SK/none), `getPkOffset`, and
`getIndexCount`.

Column offsets are computed once at load time and cached
(`col_offset[]`), so row (de)serialization anywhere in the codebase is just
pointer arithmetic + `memcpy`.

---

## 6. Storage Engine A — Heap Tables

Implemented by `database` (`storage/core.cpp`) and `buffer`
(`storage/buffer.cpp`).

- **File**: `<db>_<table>.tbl` — just concatenated fixed-width rows, no
  page headers, no index.
- **Insert**: `verify()` type-checks the incoming strings against the
  schema, `fill_buffer()` packs one or more rows into a byte buffer via
  `convert_and_write()` (INT → 4-byte little-endian, BOOL → 1 byte,
  TEXT → raw bytes zero-padded to the column width), then the buffer is
  appended to the file (`ios::app`).
- **Select**: `read_buffer()` loads the *entire* file into memory, then
  `print_buffer()` walks it row by row, evaluating the WHERE expression
  tree against each row (§10) and printing matches directly to `stdout`
  (space-separated columns, not the padded table format the clustered
  engine uses).
- **Update**: same full-file load, `update_buffer()` walks every row,
  re-evaluates the WHERE tree, and overwrites matching columns in place
  using the parsed `SET` clause (`SetClause`), then the whole buffer is
  written back with `ios::trunc`.
- **Delete**: `delete_from_buffer()` compacts the in-memory buffer by
  `memmove`-ing non-matching rows down over deleted ones, and the
  resulting (shorter) buffer is written back with `ios::trunc`. A
  `DELETE FROM t;` with no `WHERE` just truncates the file to zero bytes
  without ever reading it.

Every heap operation is **O(table size)**: there is no index and no partial
I/O — the whole table round-trips through memory on every write.

---

## 7. Storage Engine B — Clustered (B+Tree) Tables

This is the bulk of the codebase (`storage/DiskManager.*`,
`storage/BPlusTree.*`, `index/Index.*`, ~2000 lines combined) and
implements a real disk-based B+Tree, not an in-memory simulation.

### 7.1 Disk Manager & Page Format

`Disk_Manager` (`storage/DiskManager.h/.cpp`) is the only place that does
raw file I/O for this engine. Every page is a fixed **4096-byte** block;
`read_page(id, buf)` / `write_page(id, buf)` just `seekg`/`seekp` to
`id * 4096` and read/write the block. One file per B+Tree — the primary
table tree and each secondary index each get their own file (see §11 for
naming).

**Page 0 of every tree file is a meta page**, holding a `Table_Metadata`:

```
[int32 total_page_count]
[int32 root_page_id]
[int32 first_leaf_page_id]   head of the leaf linked list (for full scans)
[int32 free_page_head]       head of the free-page list (page reuse)
[int32 leafnode_order]       max rows per leaf page = (4096 - header) / row_size
[int32 internalnode_order]   max keys per internal page
```

Every other page starts with a `Header`:

```
[int32 page_type]     LEAFNODE=1, INTERNALNODE=0, FREEPAGE=2
[int32 page_id]
[int32 parent_id]
[int32 keys_count]
[int32 free_bytes]
[int32 nextleaf]      leaf pages only — doubly linked list for range scans
[int32 prevleaf]
```

`Disk_Manager::allocate_page()` first checks the free-list
(`free_page_head`); only if it's empty does it grow the file by one page.
`free_page()` pushes a freed page back onto that list (`init_free_page`)
instead of shrinking the file, so deleted B+Tree pages get recycled by
later inserts.

### 7.2 The B+Tree (`BplusTree` class)

`BplusTree` is deliberately generic: the same class implements **both**
the primary (row-storing) tree and secondary (index-only) trees, switched
by the `is_primary` constructor flag and by which comparator gets used:

- `compare_keys()` — plain key comparison (INT numeric or TEXT
  `memcmp`), used when `is_primary == true`.
- `compare_composite()` — compares the indexed value first, and on a tie
  falls back to comparing an appended 4-byte primary-key suffix. Used when
  `is_primary == false`, which is what lets a secondary index hold
  **duplicate** indexed values (e.g. many rows with the same `age`) while
  still keeping every leaf entry uniquely ordered.

**Leaf pages** store rows directly, back-to-back, sorted by key.
**Internal pages** store `[child_ptr][key][child_ptr][key]...[child_ptr]`
entries, `internalnode_order` of them max.

- **`search_leaf(key)`** — top-down descent from `root_page_id`,
  linear-scanning each internal page's keys (capped at depth 32 as a
  safety net against corrupt trees) to pick the child pointer, until it
  reaches a leaf.
- **`find_in_leaf(page, key)`** — binary search within a leaf/internal
  page. Returns a **negative encoded position** (`-(mid+1)`) if the key
  already exists, or a **non-negative insertion point** otherwise — this
  sign trick is used throughout the tree to distinguish "found" from
  "would insert here" without a second return value.
- **`insert_row()`** — finds the target leaf, shifts rows to make room
  (`memmove`), and writes the new row in sorted position. If the page
  doesn't have room (`free_bytes < row_size`), it builds a temporary
  oversized buffer, calls **`leaf_split()`** (right-biased 50/50 split
  that also relinks the leaf's `nextleaf`/`prevleaf` pointers), and
  propagates a separator key up into the parent via
  **`insert_parent()`** — which itself may recursively trigger
  **`parent_split()`** and, if the root itself splits,
  **`create_new_root()`** (this is how the tree grows in height).
- **`delete_row()`** — removes the row, compacts the leaf, and if the
  leaf drops below the minimum occupancy `(order+1)/2`, tries in order:
  **borrow from left sibling**, **borrow from right sibling**, **merge
  with left**, **merge with right** — the standard B+Tree rebalancing
  cascade, implemented explicitly (`borrow_left`, `borrow_right`,
  `merge_leaf`, and their internal-node counterparts
  `borrow_left_parent`/`borrow_right_parent`/`merge_parent`). A merge
  removes the separator key from the parent via `delete_parent()`, which
  can itself underflow and recurse upward, ultimately collapsing the root
  (`delete_root()`) if it's left with zero keys.
- **`update_parent()`** — when a leaf's *first* key changes (insert into
  position 0, or that row gets deleted), the separator key one level up
  needs to change too; this walks up the parent chain updating it.
- **Scans** — `scan_all` (full leaf-chain walk from `first_leaf_page_id`),
  `scan_forward`/`scan_backward` (seek to a key, then walk the leaf chain
  in the given direction — this is what a `>`/`<`/`>=`/`<=` predicate on
  an indexed column compiles down to), and `scan_point` (exact-key
  lookup). All four take a `std::function<void(const char*)>` callback,
  so the executor supplies a closure that filters/prints/collects rows
  without the tree ever materialising a result set itself.
- **`update_row()`** — used for in-place column updates on a *known* PK:
  finds the row by key and `memcpy`s the new column bytes directly into
  the page (primary keys themselves are treated as immutable — see the
  `PRIMARY KEY is immutable` check in `DataHandler::set_verify`).

### 7.3 The Index Wrapper (`Index` class)

`Index` owns a `BplusTree` constructed with `is_primary = false` and a key
size of `pk_size + index_size` — i.e. a secondary index's B+Tree key is
literally `[indexed_value][primary_key]` concatenated, which is what lets
`compare_composite` tie-break duplicates deterministically by PK.

- `insert_index`/`delete_index`/`update_index` — thin pass-throughs to
  the underlying tree's `insert_row`/`delete_row` (update = delete-old +
  insert-new).
- `search_lower_bound`/`search_upper_bound` — descend the tree to find the
  first leaf page that could contain a given indexed value (`<=` vs `<`
  comparisons at each internal level), used as the entry point for range
  scans.
- `find_all_pks` — point lookup: walks forward from the lower-bound leaf
  collecting every entry whose indexed value matches exactly, stopping the
  moment the value changes (duplicates are always contiguous because of
  how the composite key sorts).
- `find_all_pks_forward`/`find_all_pks_backward` — range lookup: same
  idea but keeps collecting for as long as the comparison (`<=` / `>=`)
  holds, walking the leaf `nextleaf`/`prevleaf` chain as needed.

Every one of these hands the caller **primary keys**, not rows — the
executor always does a second step, joining each PK back into the primary
tree with `tree.scan_point(...)` to fetch the actual row. This is a classic
index → heap (here, index → clustered-tree) join.

---

## 8. Query Optimizer — Access Path Selection

`QueryOptimizer::optimise()` (`optimiser/QueryOptimiser.cpp`) is a small
rule-based chooser, not a cost-based planner. It returns an `AccessPath`:

```cpp
enum AccessType {
    pk_point = 1,   // WHERE pk = <value>
    sk_point = 2,   // WHERE secondary_indexed_col = <value>
    pk_range = 3,   // WHERE pk > / >= / < / <= <value>
    sk_range = 4,   // WHERE secondary_indexed_col > / >= / < / <= <value>
    full_scan = 5,  // clustered table, no usable predicate
    no_scan = 6,    // CREATE/DROP on a clustered table
    heap = 7        // table is a heap table — bypass indexing entirely
};
```

The numeric values double as a **preference order** — lower is "better" (a
point lookup beats a range scan beats a full scan). The algorithm:

1. `CREATE`/`DROP` short-circuit immediately to `heap` or `no_scan` based
   on whether the statement declares a `PRIMARY KEY` (`is_heap()`).
   `SHOW` and syntactically invalid statements return an empty path.
2. Otherwise the table's schema is loaded. If the table isn't clustered,
   the path is just `heap` and the whole tree-walk below is skipped —
   heap tables never get index-based access.
3. For clustered tables, `traverse_tree()` does a **post-order walk of the
   WHERE expression tree** (§10) and, for every **leaf** comparison node,
   computes a candidate path number from the column's index type
   (`0`=none, `1`=PK, `2`=SK) and whether the operator is `=`
   (point) or a range operator:

   | Column index type | Operator | Candidate path |
   |---|---|---|
   | none, or operator is `!=` | any | `full_scan` (5) |
   | PRIMARY KEY | `=` | `pk_point` (1) |
   | PRIMARY KEY | range | `pk_range` (3) |
   | SECONDARY INDEX | `=` | `sk_point` (2) |
   | SECONDARY INDEX | range | `sk_range` (4) |

   Whichever leaf produces the **numerically smallest** path across the
   *entire* tree wins, and that leaf's column/operator/value populate the
   returned `AccessPath` (`col_name`, `search_value`, `dt`,
   `forward` = true for `>`/`>=`).

Because this picks a single "best" leaf without regard to whether it sits
under an `AND` or an `OR` in the expression tree, a query like
`WHERE pk = 5 OR unindexed_col = 'x'` will still choose `pk_point` and
only fetch the row for `pk = 5` — the executor always re-evaluates the
*full* WHERE tree against whatever rows the chosen access path retrieves
(see §9), so this is safe for `AND`-shaped predicates (the index path is
just a candidate-row filter, correctness is guaranteed by the re-check),
but it means `OR` across a non-indexed branch can under-fetch, since the
optimizer never falls back to a full scan just because one branch of an
`OR` is unindexed.

---

## 9. Executor — Dispatch & Operation Handlers

`executor::execute()` is a big `if/else` on `operation_type`, then a second
dispatch on `AccessPath.type`:

| Statement | `pk_point` (1) | `sk_point` (2) | `pk_range` (3) | `sk_range` (4) | `full_scan` (5) | `heap` (7) |
|---|---|---|---|---|---|---|
| SELECT | `execute_select_pklookup` | `execute_select_sklookup` | `execute_select_pkrange` | `execute_select_skrange` | `execute_select_fullscan` | `execute_select_heap` |
| UPDATE | `execute_update_pklookup` | `execute_update_sklookup` | `execute_update_pkrange` | `execute_update_skrange` | `execute_update_fullscan` | `execute_update_heap` |
| DELETE | `execute_delete_pklookup` | `execute_delete_sklookup` | `execute_delete_pkrange` | `execute_delete_skrange` | `execute_delete_fullscan` | `execute_delete_heap` |

CREATE/INSERT/DROP only branch on clustered-vs-heap
(`execute_create_cluster`/`execute_create_heap`, etc. — `no_scan` and
`full_scan` both just mean "not heap" for these three).

### 9.1 Shape common to every clustered handler

All ~16 clustered handlers follow the same skeleton:

1. Confirm the catalog exists and the table is registered
   (`catalog::db_exists` / `load_catalog` / `has_table`).
2. Load the table's `schema`.
3. Open the primary tree: `Disk_Manager dm_main; dm_main.open_file(table);
   BplusTree tree(dm_main, tmd_main, int32, sizeof(int), true);
   tree.open_tree();`
   — **the primary key is always opened as a 4-byte INT** at query time,
   regardless of what datatype the schema declares for it (`create_table`
   does honour the declared PK datatype/size when first *building* the
   tree, but every subsequent INSERT/SELECT/UPDATE/DELETE handler in
   `executor.cpp` hardcodes `int32, sizeof(int)` when re-opening it).
4. Open whichever secondary index `Disk_Manager`/`Index` pairs are
   relevant (either *all* of them, for scans/deletes that might touch any
   indexed column, or just the ones referenced by a `SET`/`WHERE` clause).
5. Run the access-path-specific scan (`scan_point`/`scan_forward`/
   `scan_backward`/`scan_all`, or an `Index::find_all_pks*` followed by a
   `scan_point` join), re-checking the full WHERE tree via
   `where_clause::evaluvate_tree()` on every candidate row before
   acting on it.
6. For UPDATE/INSERT/DELETE, mirror the change into every affected
   secondary index (`insert_index`/`update_index`/`delete_index`) so the
   indexes never drift out of sync with the primary tree.
7. Print a one-line confirmation (or `"No matches found:"` for an empty
   SELECT).

Row (de)serialization on this path goes through `DataHandler`
(`storage/DataHandling.cpp`) rather than `buffer` — `converter()` packs
typed strings into raw row bytes (and, in a second overload, splices an
indexed column's bytes together with a PK to build a secondary-index
entry), and `print_header`/`print_row`/`print_table_border` render results
as a padded ASCII table (unlike the heap engine's plain space-separated
output).

### 9.2 CREATE (clustered)

`execute_create_cluster()` registers the table in the catalog
(`is_clustered = 1`), writes the `.schema` file, creates the primary
`.tbl` file and calls `BplusTree::create_tree()` on it, then loops over
every column and, for each one marked as a secondary `INDEX`, creates a
separate index file and calls `Index::create_index()`.

### 9.3 INSERT (clustered)

`execute_insert_cluster()` verifies the incoming data
(`DataHandler::data_verify`), opens the primary tree and every secondary
index, then for each row: packs it (`converter`), inserts it into the
primary tree at the PK offset, and for every indexed column, builds a
`[value][pk]` composite entry and inserts it into that column's index.
Primary-key uniqueness is enforced by the B+Tree itself — `insert_row()`
throws `DB_error(ERR_RUNTIME, "Primary key is unique: ")` if
`find_in_leaf` reports the key already exists.

---

## 10. WHERE Clause Engine

WHERE clauses are parsed once (by `where_clause::make_tree()` in
`utility/filter.cpp`) into a binary expression tree of `ConditionNode`s,
independent of which storage engine ultimately evaluates it.

1. **Infix → postfix** via a small shunting-yard implementation
   (`precedence()` in `utility/utils.cpp` gives `AND`=2, `OR`=1, so `AND`
   binds tighter than `OR`), handling parenthesised sub-expressions with an
   operator stack.
2. **Postfix → tree**: a second pass pushes leaf comparison nodes
   (`column`, `operand`, `value`) and combinator nodes (`AND`/`OR`,
   popping their two children off a node stack) — standard postfix
   expression-tree construction.
3. **Evaluation** (`evaluvate_tree()`) recurses over the tree against one
   row buffer at a time. Leaf nodes look up the column's offset/size/type
   from the `schema`, decode the row's raw bytes for that column
   (INT/TEXT/BOOL), and compare against the literal via the templated
   `condition_evaluate<T>()` (`=`, `!=`, `<`, `>`, `<=`, `>=`).
   `AND`/`OR` nodes just combine their children's booleans. This same tree
   and the same evaluator run against both heap rows (`buffer::print_buffer`
   etc.) and clustered rows (every `execute_select/update/delete_*`
   handler) — it's the one piece of query logic shared unmodified by both
   engines.

`SetClause` (also in `filter.h`) is the analogous, much simpler structure
for `UPDATE ... SET col = val, col2 = val2`: just parallel arrays of column
names/values, parsed directly by `parser_update()` with no tree needed.

---

## 11. On-Disk Layout & File Naming

All data files live in `../data/` relative to the built binary (i.e. a
`data/` directory is expected as a sibling of wherever the executable
runs from — `bash_tool`/build tooling aside, this path is hardcoded
throughout `core.cpp` and `DiskManager.cpp`).

| File | Written by | Contents |
|---|---|---|
| `<db>.catalog` | `catalog` | Table registry (§5.1) |
| `<db>_<table>.schema` | `schema` | Column definitions (§5.2) — same format for heap and clustered tables |
| `<db>_<table>.tbl` | heap engine (`database`) | Raw concatenated rows, no paging |
| `main_<table>.tbl` | clustered engine (`Disk_Manager`) | Paged B+Tree file for the primary key |
| `main_<table>_<column>.tbl` | clustered engine (`Disk_Manager`) | Paged B+Tree file for one secondary index |

Two things worth noting about this layout:

- **There is effectively one database, always named `"main"`.** The
  `catalog`/`database` classes accept an arbitrary `db_name` parameter,
  but `executor.cpp` always constructs `database db("main")` and every
  clustered-engine call passes `"main"` explicitly, and
  `Disk_Manager::create_file`/`open_file` hardcode the `main_` file
  prefix regardless of any db name argument. The multi-database plumbing
  exists in the class signatures but isn't exercised by any parsed
  statement (there's no `USE <db>` or `CREATE DATABASE` in the grammar).
- Because a table is *either* heap *or* clustered (never both), the two
  engines never actually produce a colliding filename for the same table
  name, even though they'd compute the same `main_<table>.tbl` path.

---

## 12. Data Types & Row Encoding

```cpp
enum datatype { int32 = 0, text = 1, bool8 = 2 };
```

| Type | On-disk width | Encoding |
|---|---|---|
| `INT` | 4 bytes | native `int32_t`, `memcpy` in/out |
| `TEXT(n)` | `n` bytes (fixed, declared at `CREATE TABLE` time) | raw bytes, zero-padded/truncated at read (`find('\0')`) |
| `BOOL` | 1 byte | `0`/`1`, accepted on input as `"TRUE"`/`"FALSE"`/`"1"` |

Rows are stored as one contiguous byte block per table row, columns
back-to-back in declaration order, with **no alignment padding** —
`schema::col_offset[]` gives each column's exact byte offset, computed by
summing declared widths, so every reader/writer in the codebase can
`memcpy` directly against those offsets.

---

## 13. End-to-End Walkthrough

Tracing a small session through every layer:

```sql
CREATE TABLE employees (id INT KEY, age INT INDEX, name TEXT(20));
INSERT INTO employees VALUES (1, 30, 'Alice');
SELECT * FROM employees WHERE age = 30;
```

**`CREATE TABLE ...`**
1. `tokenizer()` splits it into tokens; `parser_create()` walks them,
   recognising `id INT KEY` → primary key column, `age INT INDEX` →
   secondary-indexed column, `name TEXT(20)` → 20-byte text column. Result:
   a `create_operation` with 3 columns, `column_index = {1, 2, 0}`.
2. `QueryOptimizer::optimise()` sees `CREATE` + a PK column present →
   `is_heap()` returns false → `AccessPath.type = no_scan`.
3. `execute()` routes to `execute_create_cluster()`: catalog gets an entry
   `employees | 3 cols | clustered`; `employees.schema` is written;
   `main_employees.tbl` is created and `BplusTree::create_tree()`
   initialises page 0 (meta) and page 1 (an empty root leaf); because
   `age` is marked `INDEX`, `main_employees_age.tbl` is also created and
   `Index::create_index()` initialises its own meta+root pages.

**`INSERT INTO ...`**
1. `parser_insert()` produces `column_data = ["1","30","Alice"]`.
2. Optimizer: INSERT on a clustered table always yields
   `AccessPath.type = full_scan` (a placeholder value — INSERT doesn't
   actually branch on access path, only on heap-vs-clustered).
3. `execute_insert_cluster()`: `DataHandler::data_verify` type-checks the
   three values; the primary tree and the `age` index are both opened;
   `converter()` packs the row into 28 raw bytes laid out as
   `[id:4][age:4][name:20]`; `tree.insert_row()` walks the (currently
   one-page) tree, finds insertion position 0 via `find_in_leaf`, and
   writes the row directly into the leaf (no split needed yet). A
   composite index entry `[age:4][id:4]` is built and inserted into the
   `age` index tree the same way.

**`SELECT * FROM employees WHERE age = 30;`**
1. `parser_select()` builds a one-leaf `ConditionNode` tree:
   `column=age, operand="=", value="30"`.
2. `QueryOptimizer::optimise()` loads the schema, sees the table is
   clustered, walks the WHERE tree: `age` has index type `2` (secondary)
   and the operator is `=` → candidate path `sk_point` (2), which becomes
   the chosen `AccessPath` (`col_name="age"`, `search_value="30"`,
   `dt=int32`).
3. `execute()` sees `operation_type == "SELECT"` and `acc_path.type == 2`
   → calls `execute_select_sklookup()`.
4. That handler opens the primary tree and the `age` index, converts
   `"30"` into a 4-byte search key, calls
   `Index::find_all_pks(key, int32, callback)` — which descends the `age`
   index to the lower-bound leaf and collects every PK whose `age` entry
   equals `30` (here, just `id=1`) — and for each PK returned, calls
   `tree.scan_point()` on the *primary* tree to fetch the full row, then
   re-evaluates the full WHERE tree (`age = 30`, trivially true here)
   before printing it through `DataHandler::print_row()` in a padded
   table with a header/border.

---

