# Chapter 3 Storage and Retrieval

There is a big difference between storage engines that are optimized for transactional workloads and those that are optimized for analytics.

Database kinds:

1. traditional relational databases
1. NoSQL databases

Storage Engine Familes

1. log-structured
1. page-oriented

## 3.1 Data Structures That Power Your Database

```bash
#!/bin/bash
    db_set () {
        echo "$1,$2" >> database
}
    db_get () {
        grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

Our `db_set` function actually has pretty good performance for something that is so simple, because appending to a file is generally very efficient. Similarly to what `db_set` does, many databases internally use a log, which is an append-only data file. Real databases have more issues to deal with (such as concurrency control, reclaiming disk space so that the log doesn’t grow forever, and handling errors and partially written records), but the basic principle is the same.

In this book, log is used in the more general sense: an append-only sequence of records. It doesn’t have to be human-readable; it might be binary and intended only for other programs to read.

This is an important trade-off in storage systems: well-chosen indexes speed up read queries, but every index slows down writes.

### 3.1.1 Hash Indexes

### 3.1.2 SSTables and LSM-Trees

### 3.1.3 B-Trees

### 3.1.4 Comparing B-Trees and LSM-Trees

### 3.1.5 Other Indexing Structures

## 3.2 Transaction Processing or Analytics?


## 3.3 Column-Oriented Storage


## Summary



## References
<!-- references -->

