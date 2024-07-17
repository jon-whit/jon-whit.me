---
title: "Building a Key/Value Database: Log-Structured Merge Tree (LSM)"
date: 2024-07-16
categories: ["golang", "databases", "lsm", "storage"]
---
## Overview
For a long time now I've wanted to learn how to build my own database and gain a deeper understanding of the internal mechanics of modern database technologies. Lots of modern distributed databases today are built ontop of distributed key/value storage engines. For example, [TiDB](https://www.pingcap.com/tidb/) and [CockroachDB](https://www.cockroachlabs.com/) are just but two examples of modern SQL database technologies that are built ontop of a highly scalable and performant key/value storage engine. [TiKV](https://tikv.org/) is the key/value engine that powers TiDB while CockroachdDB is powered by [CockroachDB's distributed key/value layer](https://www.cockroachlabs.com/glossary/distributed-db/key-value-kv-layer/). Other databases such as [etcd](https://etcd.io/), Redis, Memcached, etc.. are also direct examples of distributed key/value stores. Needless to say, key/value stores are everywhere today, and they form the basis for so much of our distributed persistence solutions in modern platforms.

Over the coming months I'll be writing about my research learnings and experience building a key/value database in Go and I will provide Github code samples along the way so you can follow along and learn with me. 

## Key/Value (KV) Databases (from an API perspective)
The most basic form of a KV database API allows for basic `Set`, `Get`, and `Delete` operations on keys and values. Something of the form:

```go
Get(key string) ([]byte, error)
Set(key, value string) error
Delete(key string) error
```

A lot of KV APIs also allow for efficient `Range` queries over a range of keys starting with a certain prefix and up to some ending key prefix, for example:

```go
Range(startKeyPrefix, endKeyPrefix string) (KeyValueIterator, error)

type KeyValueIterator interface {
    Next() ([]byte, error)
    Close() error
}
```

Implementations of these primitives (Get, Set, Delete, Range) are typically just short-hand wrappers around a transactional API where the operation begins by starting a transaction, applying the operation, and then committing the result or rolling it back.

More advanced API implementations explicitly provide the ability to manage transactions and allow for reading and writing data with different [levels of isolation](https://en.wikipedia.org/wiki/Isolation_(database_systems)) promises/guarantees. For example, to do a Set followed by a Get in the same transaction it would look like:

```go
...

txn := db.NewTransaction()
defer txn.Rollback() // auto-rollback if Commit fails, otherwise no side-effect

txn.SetLevel(kvdb.Serializable)

if err := txn.Set("foo", "bar"); err != nil {
    ... // handle error
}

val, err := txn.Get("foo")
if err != nil {
    ... // handle error   
}

fmt.Println(string(val)) // Outputs 'bar'

if err := txn.Commit(); err != nil {
    ... // handle error
}
```


The ANSI/ISO SQL standard defines 4 different isolation levels, including (in order of most strict to least strict):

* **Serializable**
* **Repeatable reads**
* **Read committed**
* **Read uncommitted**

Data is often read/written/modified concurrently across multiple transactions in database systems, and different isolation levels ensure different properties of concurrent execution of these transactions. Isolation refers to the property that ensures certain invariants are met with respect to concurrent access to data in the database. There are different kinds of phenomena that can occur as a result of concurrent access (both reads and writes) to data, including:

* **Dirty Reads** - a dirty read occurs when one transaction uses the value(s) of a data item that has been updated by another (concurrent) transaction that has not yet committed that data item yet. The mutation is not yet committed and yet the reader was able to view the contents.

```go
...

txn1 := db.NewTransaction()
defer txn1.Rollback()

txn2 := db.NewTransaction()
defer txn2.Rollback()

if err := txn1.Set("foo", "bar"); err != nil { // txn1 sets a key
    ... // handle error
}

val, err := txn2.Get("foo") // txn2 gets the value for key
if err != nil {
    ... // handle error
}

// If 'bar' is printed it's a dirty read, because txn1 hasn't committed yet!
fmt.Println(string(val))

if err := txn1.Commit(); err != nil {
    ... // handle error
}

if err := txn2.Commit(); err != nil {
    ... // handle error
}
```

* **Non-repeatable Reads**
* **Phantom Reads**

The phenomena enumerated above may manifest themselves with different isolation levels. For example, the Serializable isolation level avoids all of the phenomena above and is the strictest isolation setting. The isolation level *Repeated reads* avoids the phenomena up to and including non-repeatable reads, but it does not protect from phantom reads. So on and so forth.. For more information on these phenemona please see the Wiki page on [Isolation](https://en.wikipedia.org/wiki/Isolation_(database_systems)).


Isolation is implemented through concurrency control mechanisms. [Multiversion Concurrency Control (or MVCC)](https://en.wikipedia.org/wiki/Multiversion_concurrency_control) is a form of concurrency control that is a non-locking method commonly used in database implementations which is based upon the idea that when data needs to be written or modified in some way, the data will not be overwritten but instead will be appended as a new "version" of that data item. When reading data under the MVCC paradigm, the reader can choose the "version" or snapshot to read the data contents at without being blocked by concurrent writers (assuming the snapshot is already committed). In a future article in this series we'll jump into MVCC in more depth, but just know that for now we achieve different concurrency behaviors through isolation and MVCC is one way to implement concurrency control.

## KV Database Datastructures

### B Tree
B Tree is generally better for read performance as compared to an LSM Tree, but the tradeoff is potentially more latent write performance. Insertions can cause rebalancing of a whole subtree which can lead to more latent writes as compared to writes to an LSM Tree which is a direct append to an in-memory log.

### B+ Tree

### LSM Tree

Writes to an LSM Tree involve an immediate append to a memtable, which is an in-memory append-only log of writes that are pending a flush to the downstream SST persistence layer. Since mutations are just an immediate append to an in-memory structure, writes to an LSM are generally more efficient than B Tree variants, and with more clever caching and disk avoidance tactics the read performance of LSM Trees can generally be just as good as B Tree variants.

(todo: add a diagram here)

### LSM Tree (with value log)
The novelty here is based on the idea that you can get better SSD performance through improving locality and write amplification as compared to standard LSM trees.

DGraph BadgerDB design https://dgraph.io/docs/badger/design/

See https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf for more research on this.

## Summary
Throughout the remainder of this "Building a Key/Value Database" series we'll focus on the design and implementation of an LSM tree implementation specifically. If you'd like to jump ahead to a specific article, please feel free, but otherwise we'll be covering each article in succession because they largely build on one another:

* LSM (Part 1): Skiplists and Memtables
* LSM (Part 2): Sorted String Tables (SSTs), Merge Mechanics, and Bloom Filters
* LSM (Part 3): Log Compaction, Persistence, and the Commit Log (WAL)
* LSM (Part 4): Isolation Levels, MVCC, and Snapshots


#### Things to Research Prior to Posting
* Write amplification