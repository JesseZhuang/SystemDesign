## Definition

### What’s the Difference Between Columnar Database vs. Wide-column Database?

A Columnar data store will store each column separately on disk. A Wide-column database is a type of columnar database that supports a column family stored together on disk, not just a single column (stored by rows containing columns, same as relational).

### How Does a Wide-column Store Database Differ from a Relational Database?

A relational database management system (RDBMS) stores data in a table with rows that all span a number of columns. If one row needs an additional column, that column must be added to the entire table, with null or default values provided for all the other rows. If you need to query that RDBMS table for a value that isn’t indexed, the table scan to locate those values will be very slow.

Wide-column NoSQL databases still have the concept of rows, but reading or writing a row of data consists of reading or writing the individual columns. A column is only written if there’s a data element for it. Each data element can be referenced by the row key, but querying for a value is optimized like querying an index in a RDBMS, rather than a slow table scan.

## Examples

1. Apache [Cassandra](https://cassandra.apache.org/_/index.html)
1. Google [big table](https://cloud.google.com/bigtable/docs/overview), HBase compatible

## References

1. https://stackoverflow.com/questions/62010368/what-exactly-is-a-wide-column-store
1. https://stackoverflow.com/questions/63179561/wide-column-vs-column-family-vs-columnar-vs-column-oriented-db-definition
1. https://www.scylladb.com/glossary/wide-column-database/
1. 1. https://blog.logrocket.com/nosql-wide-column-stores-guide/ not really clear, take as a grain of salt
