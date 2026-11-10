Study notes on how `AUTO_RANDOM` primary keys distribute writes and affect queries and index design.

Reference: [TiDB AUTO_RANDOM documentation (docs.pingcap.com)](https://docs.pingcap.com/tidb/stable/auto-random/).

# How it distributes writes

TiKV stores rows in key order. Regions cover contiguous key ranges, and each Region's Raft leader lives on one TiKV store. Sequential `AUTO_INCREMENT` inserts hit the last Region, concentrating writes on its leader.

For a signed `BIGINT`, the default `AUTO_RANDOM(5)` layout is:

```text
1 sign bit | 5 shard bits | 58 incremental bits
```

The shard bits come from a hash of the transaction's start timestamp. Across transactions, inserts spread over up to 32 key prefixes, which can occupy different Regions and stores. Rows in one transaction share a prefix, so a multi-row insert retains locality within that prefix.

Example IDs across transactions:

```text
1152921504606846977  -- shard prefix 00100, increment 1
3458764513820540930  -- shard prefix 01100, increment 2
576460752303423491   -- shard prefix 00010, increment 3
```

A new table normally starts with one Region. Pair `AUTO_RANDOM` with `PRE_SPLIT_REGIONS` to distribute early writes; `PRE_SPLIT_REGIONS = 4` creates 16 Regions.

# Ordering and scans

TiKV still sorts rows by their actual ID values. `WHERE id BETWEEN a AND b` remains a contiguous key-range scan, possibly spanning multiple Regions. What disappears is the relationship between ID order and insertion order:

- `ORDER BY id DESC LIMIT 50` does **not** return the latest 50 rows.
- `WHERE id > last_seen_id` is unsafe as a cursor for new arrivals: later inserts can have smaller IDs.
- Rows written in separate transactions may be scattered even if they were inserted together in time.

Small reads of recent or logically related rows can therefore touch more Regions and require more RPCs. Large scans can benefit from parallel coprocessor work across stores.

ID order is still useful for backfills, exports, and batch updates or deletes that only need to traverse the table:

```sql
SELECT * FROM orders WHERE id > ? ORDER BY id LIMIT 1000;
-- Repeat with the last ID returned.
```

Use a fixed snapshot for a consistent export. Without one, concurrent inserts can land behind the cursor and be missed. TiDB's non-transactional `BATCH ON id LIMIT n` DML also uses key ranges to divide work; it does not provide a snapshot across batches.

# Choosing keys and indexes

| Workload or existing key | Choice |
|---|---|
| Unique, well-distributed natural or app-generated key | Use it as the clustered PK. This avoids maintaining and looking up a separate unique secondary index. Examples: UUID as `BINARY(16)`, content hash as `BINARY(32)`, or a well-distributed external ID. |
| Low insert rate | `AUTO_INCREMENT` may be sufficient. |
| Heavy writes, surrogate PK, reads only by ID | `AUTO_RANDOM` without secondary indexes. |
| Heavy writes, surrogate PK, queries by business attributes or time | `AUTO_RANDOM` with secondary indexes matching those queries. |
| IDs must sort by time | Consider an app-generated scheme that balances ordering and write distribution. |

For application-controlled IDs that are poorly distributed, another option is a nonclustered PK with `SHARD_ROW_ID_BITS`. This scatters the hidden row IDs; the PK index can still have its own hotspot.

## With secondary indexes

Orders, messages, events, and audit logs often need an opaque row ID plus queries by business attributes:

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_RANDOM,
  user_id BIGINT,
  created_at DATETIME,
  amount DECIMAL(10,2),
  KEY idx_user_created (user_id, created_at)  -- A user's recent orders.
);
```

Leading with `user_id` distributes index writes across users. An index on `created_at` alone can create a sequential write hotspot; a hot index may need sharding or bucketing too.

## Without secondary indexes

A chat attachment table fits when another record stores the attachment ID and every read is a PK lookup:

```sql
CREATE TABLE attachments (
  id BIGINT PRIMARY KEY AUTO_RANDOM,
  storage_key VARCHAR(255),  -- Object path in S3/GCS.
  mime_type VARCHAR(64),
  size_bytes BIGINT,
  width INT,
  height INT,
  created_at DATETIME
) PRE_SPLIT_REGIONS = 4;
```

Upload the bytes to object storage, insert the metadata, retrieve `LAST_INSERT_ID()`, and store that ID in the message row. Later reads use:

```sql
SELECT storage_key, mime_type, width, height FROM attachments WHERE id = ?;
```

The clustered PK gives a direct row lookup and avoids secondary-index writes. If the table later needs queries such as "a user's uploads," add `user_id` and an appropriate index.

Other examples with the same access pattern:

- **Event or audit ingestion:** a durable sink consumed through TiCDC to Kafka or batch exports, without filtered queries against the table.
- **Payload side tables:** large JSON/text values fetched by a `payload_id` stored in an order or post.
- **Reports or exports:** the API returns a `report_id` for later retrieval through `GET /reports/{id}`.

# Explicit inserts

Normally, omit the PK and let TiDB generate it. Supplying it yourself is rejected unless `@@allow_auto_random_explicit_insert` is enabled (default: `0`). This option mainly supports migrations and restores that must preserve existing IDs and references.

```sql
CREATE TABLE t (id BIGINT PRIMARY KEY AUTO_RANDOM, v VARCHAR(10));

INSERT INTO t (v) VALUES ('a');  -- TiDB generates the ID.

SET @@SESSION.allow_auto_random_explicit_insert = 1;
INSERT INTO t (id, v) VALUES (12345, 'b');  -- Stored as supplied.
SET @@SESSION.allow_auto_random_explicit_insert = 0;
```

Explicit values have two risks:

- **Exhausting the allocator:** TiDB uses the supplied ID's incremental part to advance the allocation counter if necessary. With the default signed layout, these are the low 58 bits. A value such as `288230376151711700` puts the counter near its limit of `2^58 - 1`, leaving little room for generated IDs. Deleting the row does not lower the counter.
- **Restoring write hotspots:** TiDB does not add shard bits to supplied IDs. Sequential values such as `1, 2, 3` all have shard prefix `0`, concentrating writes in one key range.

Enable explicit inserts only for the migration session, then disable them. Check the imported IDs' low bits first; if they nearly exhaust the allocator, retain those IDs in a separate column and generate new PKs.

# Client handling

- **Return IDs as JSON strings.** Default `AUTO_RANDOM` values can exceed JavaScript's exact integer limit of `2^53 - 1` and be silently rounded when parsed as numbers.
- **Do not treat IDs as secrets.** They are not cryptographically random. Check authorization separately, or use a separate random token for secret links.
