# cassandra-example

## How data is stored in Cassandra

In the [Cassandra documentation](https://cassandra.apache.org/doc/latest/architecture/storage_engine.html) it is explained how data is stored on each Cassandra node.

### CommitLog

**Commitlogs** are an a*ppend only log* of all mutations local to a Cassandra node. Any data written to Cassandra will first be written to a commit log before being written to a memtable. This provides durability in the case of unexpected shutdown.

### Memtables

**Memtables** are _in-memory_ structures where Cassandra buffers writes. In general, there is one active memtable per table. Eventually, memtables are flushed onto disk and become immutable SSTables.

### SSTables

**SSTables** are the immutable data files that Cassandra uses for persisting data _on disk_.

As SSTables are _flushed_ to disk from Memtables or are streamed from other nodes, Cassandra triggers _compactions_ which combine multiple SSTables into one. Once the new SSTable has been written, the old SSTables can be removed.

Each SSTable is comprised of multiple components stored in separate files:
| File | Description |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| TOC.txt | A file that lists the components for the given SSTable. |
| Digest.crc32 | A file that consists of a checksum of the data file. |
| CompressionInfo.db | A file that contains meta data for the compression algorithm, if enabled. |
| Statistics.db | A file that holds statistical metadata about the SSTable. |
| Index.db | A file that contains the primary index data. |
| Summary.db | This file provides summary data of the primary index, e.g. index boundaries, and is supposed to be stored in memory. |
| Filter.db | This file embraces a data structure used to validate if row data exists in memory i.e. to minimize the access of data on disk. |
| Data.db | This file contains the actual data, i.e. the contents of rows. Note: All the other component files can be regenerated from the base data file. |

Within the _Data.db_ file, rows are organized by **partition**. These partitions are sorted in token order. Within a partition, rows are stored in the order of their **clustering keys**.

SSTables can be optionally compressed using block-based compression.

## Follow along example

As part of this example a local Cassandra instance will be used to show examples of how data is stored by Cassandra.

You will

- create a table
- insert data into a table
- update data into a table
- delete data from a table
- flush data on disk
- compact data
- check how data is stored on disk

### Create a local Cassandra instance

```
docker run --name cassandra-example -d cassandra
```

Open the following commands in two different consoles in order to follow along with the examples.

```
docker exec -it cassandra-example bash
docker exec -it cassandra-example cqlsh
```

### Initialise a table with a Primary Key

Run the follwing commands from the cqlsh console.

```
CREATE KEYSPACE IF NOT EXISTS test WITH REPLICATION = {'class' : 'SimpleStrategy','replication_factor' : 1};

use test;

CREATE TABLE IF NOT EXISTS example_pk (field1 int,field2 int,field3 int, PRIMARY KEY(field1));

INSERT INTO example_pk (field1, field2, field3) VALUES (1,2,3);
INSERT INTO example_pk (field1, field2, field3) VALUES (4,5,6);
INSERT INTO example_pk (field1, field2, field3) VALUES (7,8,9);
```

```
$ select * from example_pk ;

 field1 | field2 | field3
--------+--------+--------
      1 |      2 |      3
      4 |      5 |      6
      7 |      8 |      9

(3 rows)
```

Cassandra stores SSTables on disk in the `data` directory. Files are stored for keyspace. No new file created.

```
$ ll $CASSANDRA_HOME/data/data

total 32
drwxr-xr-x  8 cassandra cassandra 4096 Mar 13 16:45 ./
drwxrwxrwx  6 cassandra cassandra 4096 Mar 13 16:44 ../
drwxr-xr-x 26 cassandra cassandra 4096 Mar 13 16:44 system/
drwxr-xr-x  6 cassandra cassandra 4096 Mar 13 16:44 system_auth/
drwxr-xr-x  5 cassandra cassandra 4096 Mar 13 16:44 system_distributed/
drwxr-xr-x 12 cassandra cassandra 4096 Mar 13 16:44 system_schema/
drwxr-xr-x  4 cassandra cassandra 4096 Mar 13 16:44 system_traces/
drwxr-xr-x  3 cassandra cassandra 4096 Mar 13 16:45 test/
```

This is because Cassandra first writes data in memory and then it flushes them to disk after a certain threshold is reached.

To confirm that no data is stored on disk we can also use the [sstableutil](https://cassandra.apache.org/doc/latest/tools/sstable/sstableutil.html) utility.

```
$ sstableutil test example_pk

Listing files...
```

In order to manually persist one or more tables from the memtable to disk use the [nodetool flush](https://cassandra.apache.org/doc/latest/tools/nodetool/flush.html) utility.

```
nodetool flush -- test
```

Use again the `sstableutil` to confirm thata the data is persisted on disk.

```
$ sstableutil test example_pk

Listing files...
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-CompressionInfo.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Data.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Digest.crc32
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Filter.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Index.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Statistics.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Summary.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-TOC.txt
```

We will now focus on how the rows are stored on disk. We will look at the `Data.db` file that contains the actual data.
Run the [sstabledump](https://cassandra.apache.org/doc/latest/tools/sstable/sstabledump.html) utility.

```
$ cd /var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/

$ /opt/cassandra/tools/bin/sstabledump -d md-1-big-Data.db

[1]@0 Row[info=[ts=1615653905733986] ]:  | [field2=2 ts=1615653905733986], [field3=3 ts=1615653905733986]
[4]@33 Row[info=[ts=1615653905739080] ]:  | [field2=5 ts=1615653905739080], [field3=6 ts=1615653905739080]
[7]@67 Row[info=[ts=1615653905745454] ]:  | [field2=8 ts=1615653905745454], [field3=9 ts=1615653905745454]
```

### Insert a new row

SSTable are immutable files. When new rows are inserted, after a `flush` command a new SSTable is created.

```
INSERT INTO example_pk (field1, field2, field3) VALUES (2,3,4);
```

```
$ select * from example_pk ;

 field1 | field2 | field3
--------+--------+--------
      1 |      2 |      3
      2 |      3 |      4
      4 |      5 |      6
      7 |      8 |      9

(4 rows)
```

Flush data on disk

```
nodetool flush -- test
```

And this is what it generates:

```
$ sstableutil test example_pk

Listing files...
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-CompressionInfo.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Data.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Digest.crc32
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Filter.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Index.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Statistics.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-Summary.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-1-big-TOC.txt
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-2-big-CompressionInfo.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-2-big-Data.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-2-big-Digest.crc32
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-2-big-Filter.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-2-big-Index.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-2-big-Statistics.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-2-big-Summary.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-2-big-TOC.txt
```

You can use the following command to read from all the available SSTables.

```
$ for i in $(ls *Data.db); do /opt/cassandra/tools/bin/sstabledump -d $i; done

[1]@0 Row[info=[ts=1615653905733986] ]:  | [field2=2 ts=1615653905733986], [field3=3 ts=1615653905733986]
[4]@33 Row[info=[ts=1615653905739080] ]:  | [field2=5 ts=1615653905739080], [field3=6 ts=1615653905739080]
[7]@67 Row[info=[ts=1615653905745454] ]:  | [field2=8 ts=1615653905745454], [field3=9 ts=1615653905745454]
WARN  16:53:13,486 Only 16.689GiB free across all data volumes. Consider adding more capacity to your cluster or removing obsolete snapshots
[2]@0 Row[info=[ts=1615654191566237] ]:  | [field2=3 ts=1615654191566237], [field3=4 ts=1615654191566237]
```

### Compaction

[Compaction](https://cassandra.apache.org/doc/latest/operating/compaction/index.html) is about merging sstables, since partitions in sstables are sorted based on the hash of the partition key it is possible to efficiently merge separate sstables. Content of each partition is also sorted so each partition can be merged efficiently.

Picking the right compaction strategy for your workload will ensure the best performance for both querying and for compaction itself.

- **Size Tiered Compaction Strategy** is the the default compaction strategy. Useful as a fallback when other strategies don’t fit the workload. Most useful for non pure time series workloads with spinning disks, or when the I/O from LCS is too high.

- **Leveled Compaction Strategy (LCS)** is optimized for read heavy workloads, or workloads with lots of updates and deletes. It is not a good choice for immutable time series data.

- **Time Window Compaction Strategy** is designed for TTL’ed, mostly immutable time series data.

Let's see what happens when a compaction is triggered. Use the [nodetool compact](https://cassandra.apache.org/doc/latest/tools/nodetool/compact.html) utility to manually trigger a major compaction process.

```
nodetool compact
```

A new SSTable is created and the old SSTables are deleted.

```
$ sstableutil test example_pk

Listing files...
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-CompressionInfo.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Data.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Digest.crc32
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Filter.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Index.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Statistics.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Summary.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-TOC.txt
```

### Update

In Cassandra when inserting a row with the same Primary Key than an existing row, it overrides the existing value.

```
INSERT INTO example_pk (field1, field2, field3) VALUES (1,2,4);
```

```
$ select * from example_pk;

 field1 | field2 | field3
--------+--------+--------
      1 |      2 |      4
      2 |      3 |      4
      4 |      5 |      6
      7 |      8 |      9

(4 rows)
```

Let's see how it actually results on the data stored on disk.

```
nodetool flush -- test
```

```
$ sstableutil test example_pk

Listing files...
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-CompressionInfo.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Data.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Digest.crc32
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Filter.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Index.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Statistics.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-Summary.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-3-big-TOC.txt
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-4-big-CompressionInfo.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-4-big-Data.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-4-big-Digest.crc32
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-4-big-Filter.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-4-big-Index.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-4-big-Statistics.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-4-big-Summary.
```

```
$ for i in $(ls *Data.db); do /opt/cassandra/tools/bin/sstabledump -d $i; done

[1]@0 Row[info=[ts=1615653905733986] ]:  | [field2=2 ts=1615653905733986], [field3=3 ts=1615653905733986]
[2]@33 Row[info=[ts=1615654191566237] ]:  | [field2=3 ts=1615654191566237], [field3=4 ts=1615654191566237]
[4]@70 Row[info=[ts=1615653905739080] ]:  | [field2=5 ts=1615653905739080], [field3=6 ts=1615653905739080]
[7]@104 Row[info=[ts=1615653905745454] ]:  | [field2=8 ts=1615653905745454], [field3=9 ts=1615653905745454]
WARN  19:01:12,360 Only 16.684GiB free across all data volumes. Consider adding more capacity to your cluster or removing obsolete snapshots
[1]@0 Row[info=[ts=1615661975059697] ]:  | [field2=2 ts=1615661975059697], [field3=4 ts=1615661975059697]
```

As expected a new SSTable is generated and for the same Primary key we have 2 entries with different values.

The SSTables will be merged as a result of the compaction process and only one entry for each Primary Key will remain on disk.

```
nodetool compact
```

```
$ sstableutil test example_pk

Listing files...
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-5-big-CompressionInfo.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-5-big-Data.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-5-big-Digest.crc32
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-5-big-Filter.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-5-big-Index.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-5-big-Statistics.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-5-big-Summary.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-5-big-TOC.txt
```

```
$ for i in $(ls *Data.db); do /opt/cassandra/tools/bin/sstabledump -d $i; done

[1]@0 Row[info=[ts=1615661975059697] ]:  | [field2=2 ts=1615661975059697], [field3=4 ts=1615661975059697]
[2]@37 Row[info=[ts=1615654191566237] ]:  | [field2=3 ts=1615654191566237], [field3=4 ts=1615654191566237]
[4]@74 Row[info=[ts=1615653905739080] ]:  | [field2=5 ts=1615653905739080], [field3=6 ts=1615653905739080]
[7]@108 Row[info=[ts=1615653905745454] ]:  | [field2=8 ts=1615653905745454], [field3=9 ts=1615653905745454]
```

### Delete and Tombstones

When a delete request is received by Cassandra it does not actually remove the data from the underlying store. Instead it writes a special piece of data known as a tombstone. The [Tombstone](https://cassandra.apache.org/doc/latest/operating/compaction/index.html#tombstones-and-garbage-collection-gc-grace) represents the delete and causes all values which occurred before the tombstone to not appear in queries to the database. This approach is used instead of removing values because of the distributed nature of Cassandra.

Using Tombstones, the repair operation will correctly put the state of the system to what we expect for a record marked as deleted on all nodes. This does mean we will end up accruing Tombstones which will permanently accumulate disk space. To avoid keeping tombstones forever we have a parameter known as `gc_grace_seconds` for every table in Cassandra. The default value for `gc_grace_seconds` is 864000 which is equivalent to 10 days.

Delete a row from the database

```
DELETE FROM example_pk WHERE field1=1;
```

```
$ select * from example_pk;

 field1 | field2 | field3
--------+--------+--------
      2 |      3 |      4
      4 |      5 |      6
      7 |      8 |      9

(3 rows)
```

The entry is not visible anymore but a new SSTable is generated and it contains the Tombstones.

```
nodetool flush -- test
```

```
$ for i in $(ls *Data.db); do /opt/cassandra/tools/bin/sstabledump -d $i; done

[1]@0 Row[info=[ts=1615661975059697] ]:  | [field2=2 ts=1615661975059697], [field3=4 ts=1615661975059697]
[2]@37 Row[info=[ts=1615654191566237] ]:  | [field2=3 ts=1615654191566237], [field3=4 ts=1615654191566237]
[4]@74 Row[info=[ts=1615653905739080] ]:  | [field2=5 ts=1615653905739080], [field3=6 ts=1615653905739080]
[7]@108 Row[info=[ts=1615653905745454] ]:  | [field2=8 ts=1615653905745454], [field3=9 ts=1615653905745454]
WARN  19:04:07,859 Only 16.684GiB free across all data volumes. Consider adding more capacity to your cluster or removing obsolete snapshots
[1]@0 deletedAt=1615662206263871, localDeletion=1615662206
```

Even after the compaction process the Tombstones is not deleted.

```
nodetool compact
```

```
$ sstableutil test example_pk

Listing files...
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-7-big-CompressionInfo.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-7-big-Data.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-7-big-Digest.crc32
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-7-big-Filter.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-7-big-Index.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-7-big-Statistics.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-7-big-Summary.db
/var/lib/cassandra/data/test/example_pk-76bc7de0841b11eb80cf6d2c86545d91/md-7-big-TOC.txt
```

```
$ for i in $(ls *Data.db); do /opt/cassandra/tools/bin/sstabledump -d $i; done

[1]@0 deletedAt=1615662206263871, localDeletion=1615662206
[2]@19 Row[info=[ts=1615654191566237] ]:  | [field2=3 ts=1615654191566237], [field3=4 ts=1615654191566237]
[4]@56 Row[info=[ts=1615653905739080] ]:  | [field2=5 ts=1615653905739080], [field3=6 ts=1615653905739080]
[7]@89 Row[info=[ts=1615653905745454] ]:  | [field2=8 ts=1615653905745454], [field3=9 ts=1615653905745454]
```

Cassandra will fully drop those tombstones when a compaction triggers, only after `local_delete_time + gc_grace_seconds` as defined on the table the data belongs to.

## Components of the Cassandra data model
### Partition key

The **partition key** is responsible for distributing data among nodes. A partition key is the same as the primary key when the primary key consists of a single column.

Partition keys belong to a node. Cassandra is organized into a cluster of nodes, with each node having an equal part of the partition key hashes.

### Compound key
**Compound keys** include multiple columns in the primary key, but these additional columns do not necessarily affect the partition key. A partition key with multiple columns is known as a **composite key**.

### Clustering key
Clustering keys are responsible for sorting data within a partition. Each primary key column after the partition key is considered a clustering key. Clustering keys are sorted in ascending order by default.

### Composite key
Composite keys are partition keys that consist of multiple columns.
### A note about querying clustered composite keys
When issuing a CQL query, you must include all partition key columns, at a minimum. You can then apply an additional filter by adding each clustering key in the order in which the clustering keys appear.

## Example

```
CREATE TABLE IF NOT EXISTS example_primary_key (field1 int,field2 int,field3 int, PRIMARY KEY(field1));

INSERT INTO example_primary_key (field1, field2, field3) VALUES (1,2,3);
INSERT INTO example_primary_key (field1, field2, field3) VALUES (1,2,4);
INSERT INTO example_primary_key (field1, field2, field3) VALUES (1,3,4);

CREATE TABLE IF NOT EXISTS example_clustering_key (field1 int,field2 int,field3 int, PRIMARY KEY(field1, field2, field3));

INSERT INTO example_clustering_key (field1, field2, field3) VALUES (1,2,3);
INSERT INTO example_clustering_key (field1, field2, field3) VALUES (1,2,4);
INSERT INTO example_clustering_key (field1, field2, field3) VALUES (1,3,4);


CREATE TABLE IF NOT EXISTS example_composite_key (field1 int,field2 int,field3 int, PRIMARY KEY((field1, field2), field3));

INSERT INTO example_composite_key (field1, field2, field3) VALUES (1,2,3);
INSERT INTO example_composite_key (field1, field2, field3) VALUES (1,2,4);
INSERT INTO example_composite_key (field1, field2, field3) VALUES (1,3,4);
```

```
$ select * from example_primary_key;

 field1 | field2 | field3
--------+--------+--------
      1 |      3 |      4

(1 rows)

$ select * from example_clustering_key;

 field1 | field2 | field3
--------+--------+--------
      1 |      2 |      3
      1 |      2 |      4
      1 |      3 |      4

(3 rows)

$ select * from example_composite_key;

 field1 | field2 | field3
--------+--------+--------
      1 |      3 |      4
      1 |      2 |      3
      1 |      2 |      4

(3 rows)
```

```
$ select * from example_primary_key where field1=1;

 field1 | field2 | field3
--------+--------+--------
      1 |      3 |      4

(1 rows)

$ select * from example_clustering_key where field1=1;

 field1 | field2 | field3
--------+--------+--------
      1 |      2 |      3
      1 |      2 |      4
      1 |      3 |      4

(3 rows)

$ select * from example_composite_key where field1=1;
InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"
```


```
$ select * from example_primary_key where field1=1 and field2=3;
InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"
$ select * from example_clustering_key where field1=1 and field2=3;

 field1 | field2 | field3
--------+--------+--------
      1 |      3 |      4

(1 rows)

$ select * from example_composite_key where field1=1 and field2=3;

 field1 | field2 | field3
--------+--------+--------
      1 |      3 |      4

(1 rows)
```


```
$ select * from example_clustering_key where field1=1 and field3=4;
InvalidRequest: Error from server: code=2200 [Invalid query] message="PRIMARY KEY column "field3" cannot be restricted as preceding column "field2" is not restricted"

$ select * from example_composite_key where field1=1 and field3=4;
InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"
```




```
$ select * from example_clustering_key where field1=1 and field2=3 and field3=4;

 field1 | field2 | field3
--------+--------+--------
      1 |      3 |      4

(1 rows)

$ select * from example_composite_key where field1=1 and field2=3 and field3=4;

 field1 | field2 | field3
--------+--------+--------
      1 |      3 |      4

(1 rows)
```


```
$ select * from example_clustering_key where field1=1 and field2=2;

 field1 | field2 | field3
--------+--------+--------
      1 |      2 |      3
      1 |      2 |      4

(2 rows)

$ select * from example_composite_key where field1=1 and field2=2;

 field1 | field2 | field3
--------+--------+--------
      1 |      2 |      3
      1 |      2 |      4

(2 rows)
```


```
nodetool flush -- test
```

```
$ sstableutil test example_primary_key

Listing files...
/var/lib/cassandra/data/test/example_primary_key-22fc528085d211eb8df86d2c86545d91/md-1-big-CompressionInfo.db
/var/lib/cassandra/data/test/example_primary_key-22fc528085d211eb8df86d2c86545d91/md-1-big-Data.db
/var/lib/cassandra/data/test/example_primary_key-22fc528085d211eb8df86d2c86545d91/md-1-big-Digest.crc32
/var/lib/cassandra/data/test/example_primary_key-22fc528085d211eb8df86d2c86545d91/md-1-big-Filter.db
/var/lib/cassandra/data/test/example_primary_key-22fc528085d211eb8df86d2c86545d91/md-1-big-Index.db
/var/lib/cassandra/data/test/example_primary_key-22fc528085d211eb8df86d2c86545d91/md-1-big-Statistics.db
/var/lib/cassandra/data/test/example_primary_key-22fc528085d211eb8df86d2c86545d91/md-1-big-Summary.db
/var/lib/cassandra/data/test/example_primary_key-22fc528085d211eb8df86d2c86545d91/md-1-big-TOC.txt

$ /opt/cassandra/tools/bin/sstabledump /var/lib/cassandra/data/test/example_primary_key-22fc528085d211eb8df86d2c86545d91/md-1-big-Data.db -d

[1]@0 Row[info=[ts=1615842314268468] ]:  | [field2=3 ts=1615842314268468], [field3=4 ts=1615842314268468]

$ /opt/cassandra/tools/bin/sstabledump /var/lib/cassandra/data/test/example_primary_key-22fc528085d211eb8df86d2c86545d91/md-1-big-Data.db

[
  {
    "partition" : {
      "key" : [ "1" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 18,
        "liveness_info" : { "tstamp" : "2021-03-15T21:05:14.268468Z" },
        "cells" : [
          { "name" : "field2", "value" : 3 },
          { "name" : "field3", "value" : 4 }
        ]
      }
    ]
  }
]
```

```
$ sstableutil test example_clustering_key | grep Data | xargs /opt/cassandra/tools/bin/sstabledump -d


[1]@0 Row[info=[ts=1615842314386100] ]: 2, 3 |
[1]@31 Row[info=[ts=1615842314390177] ]: 2, 4 |
[1]@45 Row[info=[ts=1615842314394896] ]: 3, 4 |

$ sstableutil test example_clustering_key | grep Data | xargs /opt/cassandra/tools/bin/sstabledump

[
  {
    "partition" : {
      "key" : [ "1" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 18,
        "clustering" : [ 2, 3 ],
        "liveness_info" : { "tstamp" : "2021-03-15T21:05:14.386100Z" },
        "cells" : [ ]
      },
      {
        "type" : "row",
        "position" : 31,
        "clustering" : [ 2, 4 ],
        "liveness_info" : { "tstamp" : "2021-03-15T21:05:14.390177Z" },
        "cells" : [ ]
      },
      {
        "type" : "row",
        "position" : 45,
        "clustering" : [ 3, 4 ],
        "liveness_info" : { "tstamp" : "2021-03-15T21:05:14.394896Z" },
        "cells" : [ ]
      }
    ]
  }
]
```

```
$ sstableutil test example_composite_key | grep Data | xargs /opt/cassandra/tools/bin/sstabledump -d

[1:3]@0 Row[info=[ts=1615842315676373] ]: 4 |
[1:2]@40 Row[info=[ts=1615842314508722] ]: 3 |
[1:2]@77 Row[info=[ts=1615842314512551] ]: 4 |

$ sstableutil test example_composite_key | grep Data | xargs /opt/cassandra/tools/bin/sstabledump -d

[
  {
    "partition" : {
      "key" : [ "1", "3" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 28,
        "clustering" : [ 4 ],
        "liveness_info" : { "tstamp" : "2021-03-15T21:05:15.676373Z" },
        "cells" : [ ]
      }
    ]
  },
  {
    "partition" : {
      "key" : [ "1", "2" ],
      "position" : 40
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 68,
        "clustering" : [ 3 ],
        "liveness_info" : { "tstamp" : "2021-03-15T21:05:14.508722Z" },
        "cells" : [ ]
      },
      {
        "type" : "row",
        "position" : 77,
        "clustering" : [ 4 ],
        "liveness_info" : { "tstamp" : "2021-03-15T21:05:14.512551Z" },
        "cells" : [ ]
      }
    ]
  }
]
```
## Delete the local Cassandra instance

At the end of the example clean your local environment.

```
docker stop cassandra-example && docker rm cassandra-example
```


## References
[Designing a Cassandra Data Model](https://shermandigital.com/blog/designing-a-cassandra-data-model/)

[CQL & Data Structure](https://teddyma.gitbooks.io/learncassandra/content/model/cql_and_data_structure.html)

[How Cassandra Stores Data on Filesystem](https://saumitra.me/blog/how-cassandra-stores-data-on-filesystem/)

[sstabledump](https://cassandra.apache.org/doc/latest/tools/sstable/sstabledump.html)