# Part II<!-- omit in toc -->

- [System Research](#system-research)
	- [1. Storage Engines](#1-storage-engines)
	- [2. Index Condition Pushdown](#2-index-condition-pushdown)
	- [3.](#3)
	- [4.](#4)
	- [5.](#5)
- [Performance Experiment Design 1](#performance-experiment-design-1)
	- [Performance Issue Test](#performance-issue-test)
	- [Datasets](#datasets)
	- [Queries](#queries)
	- [Parameters Set/Varied](#parameters-setvaried)
	- [Results Expected](#results-expected)
- [Performance Experiment Design 2](#performance-experiment-design-2)
	- [Performance Issue Test](#performance-issue-test-1)
	- [Datasets](#datasets-1)
	- [Queries](#queries-1)
	- [Prameters Set/Varied](#prameters-setvaried)
	- [Results Expected](#results-expected-1)
- [Performance Experiment Design 3](#performance-experiment-design-3)
	- [Performance Issue Test](#performance-issue-test-2)
	- [Datasets](#datasets-2)
	- [Queries](#queries-2)
	- [Parameters Set/Varied](#parameters-setvaried-1)
	- [Results Expected](#results-expected-2)
- [Performance Experiment Design 4](#performance-experiment-design-4)
	- [Performance Issue Test](#performance-issue-test-3)
	- [Datasets](#datasets-3)
	- [Queries](#queries-3)
	- [Parameters Set/Varied](#parameters-setvaried-2)
	- [Results Expected](#results-expected-3)
- [Lessons Learned/Issues Encountered](#lessons-learnedissues-encountered)
##  System Research

### 1. Storage Engines
According to the official docs, "the choice of a transactional storage engine such as InnoDB or a nontransactional one such as MyISAM can be very important for performance and scalability." By default, the storage engine for new tables is InnoDB. It claims that InnoDB tables "outperform the simpler MyISAM tables, especially for a busy database." MyISAM has a small footprint and does table-level locking which reduces performances (relative to InnoDB) so "it is often used in read-only or read-mostly workloads in Web and data warehousing configurations." InnoDB tables arrange data using primary key. Each table's primary key is used to create a clustered index so that I/O is reduced when when accessing data sequentially. InnoDB tables also use "Adaptive Hash Indexing" to make lookups of frequently accessed rows faster (which we will benchmark as well). The documentation also states you can create and drop indexes and perform other DDL operations with much less impact on performance and availability." InnoDB tables were "designed for CPU efficiency and maximum performance when processing large data volumes".

Truncated Table 16.1 from MySQL docs: Storage Engines Features 
| Feature | MyISAM | InnoDB |
| :--- | :---: | :---: |
| B-tree indexes | Yes | Yes |
| Clustered indexes | No | Yes |
| Data caches | No | Yes |
| Foreign key support | No | Yes |
| Hash indexes | No | No |
| Index caches | Yes | Yes |
| Locking granularity | Table | Row |
| Storage limits | 256TB | 64TB |
| Transactions | No | Yes |

### 2. Index Condition Pushdown
The most frequently used storage engine is InnoDB - when a primary-key is specified, rows are inserted ordered according to the primary-key and a clustered index is created so that entire rows can be read in quickly. _Index Condition Pushdown_ (ICP) is an optimization setting that can be toggled when a secondary index is created. If it is off, "the storage engine traverses the (secondary) index to locate rows" and "returns them to the MySQL server which evaluates the WHERE condition for the rows." If enabled, and the WHERE condition in the query only needs the secondary indexed column, then it can push it to the storage engine which evaluates the condition using the index entry "and only if this is satisfied is the row read from the table." This is supposed to make queries that use the secondary indexed column along with other columns to be retrieved more performant.

### 3.

### 4.

### 5.

## Performance Experiment Design 1
### Performance Issue Test
Comparing read and write performance of two storage engines - InnoDB and MyISAM.
### Datasets 
We'll use ONEMTUP and execute 'read-only' and 'write-only' queries using parallel connections that simulates 4 concurrent users.
- User 1: read then write
- User 2: write then read
- User 3: read then write
- User 4: write then read 

### Queries
Four concurrent read/write processes using 10% selection with no index:
```
SELECT count(*) FROM ONEMTUP
WHERE tenPercent = 0

UPDATE ONEMTUP
SET string4 = 'x' where tenPercent = 0
```
Get `query_id` from `SHOW PROFILES;`  
Get query time from `SHOW PROFILE FOR QUERY <id>`

### Parameters Set/Varied
Enable profiling to see execution time: `SET profiling = 1;`  
Disable `AUTOCOMMIT`  
The command `SHOW ENGINES` tells us that the default engine is InnoDB, so to use other engines - we must specify explicitly. We'll create two tables: `CREATE TABLE ... ENGINE=InnoDB` and `CREATE TABLE ... ENGINE=MYISAM`  
To verify:
```
SELECT table_name, table_type, engine
FROM information_schema.tables
WHERE table_schema = 'database_name'
ORDER BY table_name;
```

### Results Expected
InnoDB seems to have many of the features we learned about or used in Postgres. In InnoDB, clustered indexes are implemented when specifying the primary-key whereas MyISAM only has unclustered indexes. Having it means the data is stored in primary-key sorted order which would decrease I/O when rows are processed sequentially. Although, in these tests, the biggest difference would be the result of InnoDB's row-level locking vs. MyISAM's table-level locking. InnoDB should perform magnitudes better than MyISAM.


## Performance Experiment Design 2

### Performance Issue Test
Comparing Index Condition Pushdown enabled/disabled in an InnoDB table with a secondary index. A `WHERE` condition that uses the secondary index, and a second `WHERE` condition that can't use the index by itself. The 'adaptive hash index' must be disabled so it doesn't cache the tuple (since we will run the query multiple times).

### Datasets
TENMTUP with its original primary-key and a clustered index on the `stringu1` column.

### Queries
`CREATE INDEX design2_index ON TENMTUP(unique1, stringu1)`
To verify index: `SHOW INDEXES FROM TENMTUP;`
```
SELECT count(*) FROM TENMTUP
WHERE fiftyPercent = 0 
AND stringu1 like '%SHDA%'
AND string4 like 'OOOO%';
```
Get `query_id` from `SHOW PROFILES;`  
Get query time from `SHOW PROFILE FOR QUERY <id>`

### Prameters Set/Varied
Disable `innodb_adaptive_hash_index`  
Enable profiling to see execution time: `SET profiling = 1;`  
Disable `AUTOCOMMIT`

Disabled run: `SET optimizer_switch = 'index_condition_pushdown=off';`  
Enabled run: `SET optimizer_switch = 'index_condition_pushdown=on';`

### Results Expected
When enabled, the query will get a `stringu1 like '%SHDA%'` tuple from the index - for each, read the full row, then check the `string4 like 'OOOO%` condition, then return those rows.
When disabled, the query will use the secondary index to find `stringu1 like '%SHDA%'` and retreive full rows back to the server, then filter through them to find `string4 like 'OOOO%'`
Reading only 1 column instead of 16 should mean the enabled option performs better.

## Performance Experiment Design 3
### Performance Issue Test
### Datasets
### Queries
### Parameters Set/Varied
### Results Expected

## Performance Experiment Design 4
### Performance Issue Test
### Datasets
### Queries
### Parameters Set/Varied
### Results Expected
## Lessons Learned/Issues Encountered
- The performance_schema engine and table would given a lot more information about execution time but only for systems N1 with 8 or more processors or with very high memory on GCP. We can't afford it. Instead, we have to use the deprecated 'show profiles' which has similar information.
- Profiling is set to 'off' with each new connection, so we have to turn it on every time we start a session.
- We should turn off autocommit to avoid unnecessary I/O when issuing large numbers of consecutive INSERT, UPDATE, or DELETE statements. We got some transaction semantics practice and saw how commiting should be grouped otherwise queries in loops take longer.
- While trying to insert a one-million row table, our script would time-out. We changed various timeout settings and packet sizes in GCP but still had problems, so we used GCP's import CSV from a bucket option.   
![Import from Bucket](./screenshots/bucket_import.png)