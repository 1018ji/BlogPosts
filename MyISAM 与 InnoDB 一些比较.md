---
title: MyISAM 与 InnoDB 一些比较
date: 2018-04-23 17:48:07
categories: DB
tags:
  - MySQL
---
# MyISAM 与 InnoDB 一些比较

文档主要参考

MySQL 8.0

1. [MyISAM 参考](https://dev.mysql.com/doc/refman/8.0/en/myisam-storage-engine.html)
2. [InnoDB 参考](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)

| Feature |  InnoDB |  MyISAM |
|---|---|---|
| B-tree indexes | Yes | Yes |
| Backup/point-in-time recovery (Implemented in the server, rather than in the storage engine.) | Yes | Yes |
| Cluster database support | No | No |
| Clustered indexes | Yes | No |
| Compressed data | Yes | Yes (Compressed MyISAM tables are supported only when using the compressed row format. Tables using the compressed row format with MyISAM are read only.) |
| Data caches | Yes | No |
| Encrypted data (Implemented in the server via encryption functions. Data-at-rest tablespace encryption is available in MySQL 5.7 and later.) | Yes | Yes |
| Foreign key support | Yes | No |
| Full-text search indexes | Yes (InnoDB support for FULLTEXT indexes is available in MySQL 5.6 and later.) | Yes |
| Geospatial data type support | Yes | Yes |
| Geospatial indexing support | Yes (InnoDB support for geospatial indexing is available in MySQL 5.7 and later.) | Yes |
| Hash indexes | No (InnoDB utilizes hash indexes internally for its Adaptive Hash Index feature.) | No |
| Index caches | Yes | Yes |
| Locking granularity | Row | Table |
| MVCC | Yes | No |
| Query cache support | Yes | Yes |
| Replication support (Implemented in the server, rather than in the storage engine.) | Yes | Yes |
| Storage limits | 64TB | 256TB |
| T-tree indexes | No | No |
| Transactions | Yes | No |
| Update statistics for data dictionary | Yes | Yes |
