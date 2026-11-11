# Clustered vs nonclustered primary keys

A **clustered table** stores each row under its primary key (PK), so the PK is also the row key. A **nonclustered table** stores rows under a hidden `_tidb_rowid`; the PK is a separate unique index that points to each row.

TiKV stores rows as key-value pairs. The two layouts look like this:

```text
Clustered:     t{tableID}_r{pk}          → [all column values]
Nonclustered:  t{tableID}_r{_tidb_rowid} → [all column values]
               t{tableID}_i{indexID}_{pk} → _tidb_rowid
```

Choose the layout when creating the table:

```sql
CREATE TABLE a (id BIGINT PRIMARY KEY CLUSTERED, v INT);
CREATE TABLE b (id VARCHAR(36) PRIMARY KEY NONCLUSTERED, v INT);
```

If you omit the keyword, `tidb_enable_clustered_index` determines the layout. A table without a PK uses a hidden `_tidb_rowid`. You cannot switch an existing table between layouts with `ALTER TABLE`; create a new table and copy the data. MySQL InnoDB always clusters rows by the PK.

## Storage trade-offs

| | Clustered | Nonclustered |
|---|---|---|
| PK lookup | One row-key lookup | Read the PK index, then the row |
| Insert | Write the row (plus secondary indexes) | Write the row and PK index (plus secondary indexes) |
| PK range scan | Read rows in PK order | Scan the PK index and fetch rows |
| Row pointer in each secondary index | PK value | 8-byte `_tidb_rowid` |
| Potential hotspot | Sequential PK values | Sequential `_tidb_rowid` values |

## Choosing a layout

**Clustered is usually the best fit** when the PK is well distributed, such as a UUID stored as `BINARY(16)` or a hash, or when you use `AUTO_RANDOM`. PK lookups and range scans read the rows directly. For example, a high-write event table can use `AUTO_RANDOM` IDs and fetch events by ID. A sequential clustered PK can create a write hotspot.

Choose **nonclustered** when:

- The PK is wide and the table has several secondary indexes. For example, a legacy `customers` table with a wide external customer code as its PK and indexes on email and status stores an 8-byte row ID in each secondary index instead of the wide PK.
- The PK is sequential but cannot use `AUTO_RANDOM`, such as an externally assigned ordered string. `SHARD_ROW_ID_BITS` can distribute hidden row IDs.
- You need to drop or replace the PK. A nonclustered PK is a separate index; a clustered PK is the row key and requires rebuilding the table to change.

Some feature support depends on the TiDB version. Check the [clustered index documentation](https://docs.pingcap.com/tidb/stable/clustered-indexes/) for current restrictions. To inspect a table's layout, use `SHOW CREATE TABLE` or query `INFORMATION_SCHEMA.TABLES.TIDB_PK_TYPE`.

## Relationship to AUTO_RANDOM and SHARD_ROW_ID_BITS

`AUTO_RANDOM` requires a clustered `BIGINT` PK: its shard bits distribute the row key. With a nonclustered PK, rows still use sequential hidden row IDs, so randomizing the separate PK index does not prevent a row-write hotspot.

`SHARD_ROW_ID_BITS` distributes hidden row IDs in nonclustered tables. Use it when the PK cannot itself serve as a well-distributed row key. See [AUTO_RANDOM](auto-random.md) for details.
