---
title: "Cassandra CQL3文档"
subtitle: ""
excerpt: "Cassandra CQL3文档"
date: 2021-01-14
published: true 
tags:
    - 技术
    - 数据库
categories: [ tech ]
---

摘抄翻译自[官网文档](https://cassandra.apache.org/doc/old/CQL-3.0.html)。部分为机翻润色。

<!--more-->

## CQL 语法

### 约定

本文档的语言规则描述使用以下这种 [BNF](http://en.wikipedia.org/wiki/Backus–Naur_Form) 式的标注：

```
<start> ::= TERMINAL <non-terminal1> <non-terminal1>
```

- 非终止符将具有<尖括号>。
- 作为 BNF 的附加快捷符号，我们将使用传统正则表达式的（`?`, `+` 和 `*`）表示给定符号是可选的或者是可重复。我们还将允许括号对符号进行分组， `[<characters>]`  表示  `<characters>` 中的任意一个。

### 标识符与关键词

- 对于没被双引号括住的词汇，CQL 对大小写不区分，反之区分。
- 标识符的正则是`[a-zA-Z][a-zA-Z0-9_]*`，但对于被括住的词汇没有限制。同时，如果在被括住的词汇中出现了某些特殊名称，可能会造成工作异常，应尽量避免。一般来说，不加引号的标识符应该是首选的。如果使用加引号的标识符，强烈建议避免使用方括号括起来的名称（如`"[applied]"`）和任何看起来像函数调用的名称（如`"f(x)"`）。

### 常量

- 字符串：使用单引号括住。注意与双引号标识符进行区分。
- 整型：`'-'?[0-9]+`
- 浮点型：`'-'?[0-9]+('.'[0-9]*)?([eE][+-]?[0-9+])?`。此外 `NaN` 和 `Infinity` 也是浮点型常量。
- 布尔型： `true` or `false` ，不区分大小写
- UUID：`hex{8}-hex{4}-hex{4}-hex{4}-hex{12}`
- blob型：`0[xX](hex)+`

### 语句

以下为后文将会使用的非终止符定义：

```
<identifier> ::= any quoted or unquoted identifier, excluding reserved keywords
 <tablename> ::= (<identifier> '.')? <identifier>

    <string> ::= a string constant
   <integer> ::= an integer constant
     <float> ::= a float constant
    <number> ::= <integer> | <float>
      <uuid> ::= a uuid constant
   <boolean> ::= a boolean constant
       <hex> ::= a blob constant

  <constant> ::= <string>
               | <number>
               | <uuid>
               | <boolean>
               | <hex>
  <variable> ::= '?'    // 匿名变量
               | ':' <identifier>    // 命名变量
      <term> ::= <constant>
               | <collection-literal>
               | <variable>
               | <function> '(' (<term> (',' <term>)*)? ')'

  <collection-literal> ::= <map-literal>
                         | <set-literal>
                         | <list-literal>
         <map-literal> ::= '{' ( <term> ':' <term> ( ',' <term> ':' <term> )* )? '}'
         <set-literal> ::= '{' ( <term> ( ',' <term> )* )? '}'
        <list-literal> ::= '[' ( <term> ( ',' <term> )* )? ']'

    <function> ::= <ident>

  <properties> ::= <property> (AND <property>)*
    <property> ::= <identifier> '=' ( <identifier> | <constant> | <map-literal> )
```

对于字符串语句，有两种语法：一是单引号括起来，二是使用双美元符号括起来。后者允许字符串内包括单引号，一般使用场景是在函数中使用。例子：

```
  'some string value'

  $$double-dollar string can contain single ' quotes$$
```

## 数据定义

### CREATE KEYSPACE 创建键空间

*Syntax:*

```
<create-keyspace-stmt> ::= CREATE KEYSPACE (IF NOT EXISTS)? <identifier> WITH <properties
```

*Sample:*

```CQL
CREATE KEYSPACE Excelsior
           WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 3};

CREATE KEYSPACE Excalibur
           WITH replication = {'class': 'NetworkTopologyStrategy', 'DC1' : 1, 'DC2' : 3}
            AND durable_writes = false;
```

顶级的键空间，可以理解为RDB中的“库”，定义了一个副本策略（*replication strategy*）和一些表的配置。

| name             | kind     | 必需  | default | description           |
| ---------------- | -------- | --- | ------- | --------------------- |
| `replication`    | *map*    | yes |         | 键空间的副本策略和选项           |
| `durable_writes` | *simple* | no  | true    | 是否对该键空间上的更新使用commit日志 |

`replication` 参数至少必须包含一个 `class` 子选项，定义要使用的副本策略。Cassandra 默认支持以下几种：

- `'SimpleStrategy'`: 只定义整个集群的复制系数（*replication factor*）的简单策略。其唯一且必需的参数只有 `'replication_factor'` ，定义了复制系数的大小。
- `'NetworkTopologyStrategy'`: 允许给各个数据中心设置不同的复制系数。
- `'OldNetworkTopologyStrategy'`: 旧的副本策略，应避免使用，并使用 `'NetworkTopologyStrategy' ` 代替。

### USE 指定键空间

*Syntax:*

```
<use-stmt> ::= USE <identifier>
```

*Sample:*

```CQL
USE myApp;
```

### ALTER KEYSPACE 修改键空间

*Syntax:*

```
<create-keyspace-stmt> ::= ALTER KEYSPACE <identifier> WITH <properties>
```

*Sample:*

```CQL
ALTER KEYSPACE Excelsior
          WITH replication = {'class': 'SimpleStrategy', 'replication_factor' : 4};
```

支持的 `<properties>` 与 `CREATE KEYSPACE` 相同。

### DROP KEYSPACE 删除键空间

*Syntax:*

```
<drop-keyspace-stmt> ::= DROP KEYSPACE ( IF EXISTS )? <identifier>
```

*Sample:*

```CQL
DROP KEYSPACE myApp;
```

`DROP KEYSPACE` 将会立刻完成，不可逆地删除掉一个键空间，包括键空间下的所有的键族和这些键族中的数据。

如果键空间不存在，且没有加上 `IF EXISTS`，执行将会报错；加上则为无操作。

### CREATE TABLE 创建表

*Syntax:*

```
<create-table-stmt> ::= CREATE ( TABLE | COLUMNFAMILY ) ( IF NOT EXISTS )? <tablename>
                          '(' <column-definition> ( ',' <column-definition> )* ')'
                          ( WITH <option> ( AND <option>)* )?

<column-definition> ::= <identifier> <type> ( STATIC )? ( PRIMARY KEY )?
                      | PRIMARY KEY '(' <partition-key> ( ',' <identifier> )* ')'

<partition-key> ::= <identifier>
                  | '(' <identifier> (',' <identifier> )* ')'

<option> ::= <property>
           | COMPACT STORAGE
           | CLUSTERING ORDER
```

*Sample:*

```CQL
CREATE TABLE monkeySpecies (
    species text PRIMARY KEY,
    common_name text,
    population varint,
    average_size int
) WITH comment='Important biological records'
   AND read_repair_chance = 1.0;

CREATE TABLE timeline (
    userid uuid,
    posted_month int,
    posted_time uuid,
    body text,
    posted_by text,
    PRIMARY KEY (userid, posted_month, posted_time)
) WITH compaction = { 'class' : 'LeveledCompactionStrategy' };
```

出于历史原因，`CREATE COLUMNFAMILY` 为 `CREATE TABLE` 的别名。

#### `<tablename>`

与键空间命名相同规则，最大32位的字母数字标识符。如果建表时未提供键空间，则在当前的键空间下创建；否则在指定的键空间下创建，但不会切换当前键空间。

#### `<column-definition>`

在 CQL 中，如果 `PRIMARY KEY` 由多栏组成，那么其顺序非常重要。 `PRIMARY KEY` 中的第一栏被称为分区键（*partition key*），共享相同分区键的所有行（实际上甚至跨表）都存储在同一个物理节点上。并且，对共享同一个分区键的行的插入、更新、删除操作，是原子且隔离的。分区键同样允许由多栏共同构成，使用括号括住即可。

`PRIMARY KEY`中的其他栏则被称为聚集键（*clustering columns*）。在一个给定的物理节点上，一个给定分区键的行是按照聚集键的顺序存储的，这使得按照该集群顺序检索行特别高效(参见SELECT)。

#### `STATIC` columns

`STATIC` 静态栏会使同一个分区键的每一行共享同一个数据。

```CQL
CREATE TABLE test (
    pk int,
    t int,
    v text,
    s text static,
    PRIMARY KEY (pk, t)
);
INSERT INTO test(pk, t, v, s) VALUES (0, 0, 'val0', 'static0');
INSERT INTO test(pk, t, v, s) VALUES (0, 1, 'val1', 'static1');
SELECT * FROM test WHERE pk=0 AND t=0;

// 结果：s='static1'
```

对于不同的分区键，则不会共享同样的数据。

如果使用静态栏，则会存在以下这些限制：

- 使用 `COMPACT STORAGE` 选项的表不支持
- 对于没有聚集键的表，不能拥有静态栏（因为此时不会存在共享分区键的不同行）
- 只有非 `PRIMARY KEY` 栏才能是静态的

#### `<option>`

 `COMPACT STORAGE`：主要是向后兼容对CQL3以前的定义。这项属性会使存储更加紧凑，但也失去了拓展性和灵活性。最显著的一点是，紧凑存储的表不能拥有静态栏，而且一定需要有且只能有1个聚集键。因此除非兼容考虑一般不推荐使用。

`CLUSTERING ORDER`：允许定义行在硬盘中的排序。注意这项设置会影响 `SELECT` 的 `ORDER BY` 选项。

以下表格显示了其他支持的 `<property>`：

| option                       | kind     | default     | description                               |
| ---------------------------- | -------- | ----------- | ----------------------------------------- |
| `comment`                    | *simple* | none        | 注释。                                       |
| `read_repair_chance`         | *simple* | 0.1         | 为读取修复而查询额外节点（例如超过一致性级别所需的节点）的概率。          |
| `dclocal_read_repair_chance` | *simple* | 0           | 为读取修复而查询属于同一数据中心的额外节点（例如多于一致性级别所需的节点）的概率。 |
| `gc_grace_seconds`           | *simple* | 864000      | 垃圾收集墓碑（删除标记）之前等待的时间。                      |
| `bloom_filter_fp_chance`     | *simple* | 0.00075     | SSTable布隆过滤器的目标误判率。将参考此值调整布隆过滤器的大小。       |
| `default_time_to_live`       | *simple* | 0           | 一张表的默认TTL秒数。                              |
| `compaction`                 | *map*    | *see below* | 压实选项，见下文。                                 |
| `compression`                | *map*    | *see below* | 压缩选项，见下文。                                 |
| `caching`                    | *map*    | *see below* | 缓存选项，见下文。                                 |

#### Compaction options

至少需要定义 `class` 子选项，定义了要使用的压实策略。默认支持`'SizeTieredCompactionStrategy'`, `'LeveledCompactionStrategy'` 和 `'DateTieredCompactionStrategy'`。可以通过将完整类名指定为字符串常量来提供自定义策略。其余的子选项取决于所选的类。默认类支持的子选项包括：

| option                           | supported compaction strategy | default      | description                                                                                                                                                                                                                                                                                                                        |
| -------------------------------- | ----------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                        | *all*                         | true         | A boolean denoting whether compaction should be enabled or not.                                                                                                                                                                                                                                                                    |
| `tombstone_threshold`            | *all*                         | 0.2          | A ratio such that if a sstable has more than this ratio of gcable tombstones over all contained columns, the sstable will be compacted (with no other sstables) for the purpose of purging those tombstones.                                                                                                                       |
| `tombstone_compaction_interval`  | *all*                         | 1 day        | The minimum time to wait after an sstable creation time before considering it for “tombstone compaction”, where “tombstone compaction” is the compaction triggered if the sstable has more gcable tombstones than `tombstone_threshold`.                                                                                           |
| `unchecked_tombstone_compaction` | *all*                         | false        | Setting this to true enables more aggressive tombstone compactions – single sstable tombstone compactions will run without checking how likely it is that they will be successful.                                                                                                                                                 |
| `min_sstable_size`               | SizeTieredCompactionStrategy  | 50MB         | The size tiered strategy groups SSTables to compact in buckets. A bucket groups SSTables that differs from less than 50% in size. However, for small sizes, this would result in a bucketing that is too fine grained. `min_sstable_size` defines a size threshold (in bytes) below which all SSTables belong to one unique bucket |
| `min_threshold`                  | SizeTieredCompactionStrategy  | 4            | Minimum number of SSTables needed to start a minor compaction.                                                                                                                                                                                                                                                                     |
| `max_threshold`                  | SizeTieredCompactionStrategy  | 32           | Maximum number of SSTables processed by one minor compaction.                                                                                                                                                                                                                                                                      |
| `bucket_low`                     | SizeTieredCompactionStrategy  | 0.5          | Size tiered consider sstables to be within the same bucket if their size is within [average_size * `bucket_low`, average_size * `bucket_high` ] (i.e the default groups sstable whose sizes diverges by at most 50%)                                                                                                               |
| `bucket_high`                    | SizeTieredCompactionStrategy  | 1.5          | Size tiered consider sstables to be within the same bucket if their size is within [average_size * `bucket_low`, average_size * `bucket_high` ] (i.e the default groups sstable whose sizes diverges by at most 50%).                                                                                                              |
| `sstable_size_in_mb`             | LeveledCompactionStrategy     | 5MB          | The target size (in MB) for sstables in the leveled strategy. Note that while sstable sizes should stay less or equal to `sstable_size_in_mb`, it is possible to exceptionally have a larger sstable as during compaction, data for a given partition key are never split into 2 sstables                                          |
| `timestamp_resolution`           | DateTieredCompactionStrategy  | MICROSECONDS | The timestamp resolution used when inserting data, could be MILLISECONDS, MICROSECONDS etc (should be understandable by Java TimeUnit) - don’t change this unless you do mutations with USING TIMESTAMP (or equivalent directly in the client)                                                                                     |
| `base_time_seconds`              | DateTieredCompactionStrategy  | 60           | The base size of the time windows.                                                                                                                                                                                                                                                                                                 |
| `max_sstable_age_days`           | DateTieredCompactionStrategy  | 365          | SSTables only containing data that is older than this will never be compacted.                                                                                                                                                                                                                                                     |

#### Compression options

有以下这些子选项：

| option               | default       | description                                                                                                                                                                                                                                                                                                                                                                                                         |
| -------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `class`              | LZ4Compressor | The compression algorithm to use. Default compressor are: LZ4Compressor, SnappyCompressor and DeflateCompressor. Use `'enabled' : false` to disable compression. Custom compressor can be provided by specifying the full class name as a [string constant](https://cassandra.apache.org/doc/old/CQL-3.0.html#constants).                                                                                           |
| `enabled`            | true          | By default compression is enabled. To disable it, set `enabled` to `false`                                                                                                                                                                                                                                                                                                                                          |
| `chunk_length_in_kb` | 64KB          | On disk SSTables are compressed by block (to allow random reads). This defines the size (in KB) of said block. Bigger values may improve the compression rate, but increases the minimum size of data to be read from disk for a read                                                                                                                                                                               |
| `crc_check_chance`   | 1.0           | When compression is enabled, each compressed block includes a checksum of that block for the purpose of detecting disk bitrot and avoiding the propagation of corruption to other replica. This option defines the probability with which those checksums are checked during read. By default they are always checked. Set to 0 to disable checksum checking and to 0.5 for instance to check them every other read |

#### Caching options

| option               | default | description                                                                                                                                                                                                                                                      |
| -------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `keys`               | ALL     | Whether to cache keys (“key cache”) for this table. Valid values are: `ALL` and `NONE`.                                                                                                                                                                          |
| `rows_per_partition` | NONE    | The amount of rows to cache per partition (“row cache”). If an integer `n` is specified, the first `n` queried rows of a partition will be cached. Other possible options are `ALL`, to cache all rows of a queried partition, or `NONE` to disable row caching. |

#### 其他注意事项

- 在插入/更新一行数据时，并非所有栏都需要被定义，并且没有的栏不会占据硬盘的空间。此外，添加一列新栏是一个常数级的操作，因此在建表时不需要去预料猜测未来的用法。

### ALTER TABLE 修改表

*Syntax:*

```
<alter-table-stmt> ::= ALTER (TABLE | COLUMNFAMILY) <tablename> <instruction>

<instruction> ::= ALTER <identifier> TYPE <type>
                | ADD   <identifier> <type>
                | DROP  <identifier>
                | WITH  <option> ( AND <option> )*
```

*Sample:*

```CQL
ALTER TABLE addamsFamily
ALTER lastKnownLocation TYPE uuid;

ALTER TABLE addamsFamily
ADD gravesite varchar;

ALTER TABLE addamsFamily
WITH comment = 'A most excellent and useful column family'
 AND read_repair_chance = 0.2;
```

 `<instruction>` 的不同语句的功能：

- `ALTER`：更新某一栏的类型。注意，不能修改聚集键的类型和位于二级索引下的列。更新类型时不会进行类型校验，但最好不要将一种类型转换为其不兼容的类型，除非此栏中没有数据，不然会导致 CQL 驱动、工具混乱。

- `ADD`：添加一列新栏。不能与已存在的冲突；使用 `COMPACT STORAGE` 选项的表无法添加。

- `DROP`：移除某一栏。删除的列将立即在查询中不可用，将来也不会包含在压缩的sstable中。如果一列被读取，查询将不会返回在最后删除该列之前写入的值。使用 `COMPACT STORAGE` 选项的表无法删除列。

- `WITH`：更新表的选项。支持的选项与建表语句相同，但是 `COMPACT STORAGE` 除外。需要注意的是，设置任何压实子选项都会删除以前所有的压实选项，因此如果想保留它们，就需要重新指定所有的子选项。压缩子选项集也一样。

### DROP TABLE 删除表

*Syntax:*

```
<drop-table-stmt> ::= DROP TABLE ( IF EXISTS )? <tablename>
```

*Sample:*

```CQL
DROP TABLE worldSeriesAttendees;
```

### TRUNCATE 清空表

*Syntax:*

```
<truncate-stmt> ::= TRUNCATE ( TABLE | COLUMNFAMILY )? <tablename>
```

*Sample:*

```CQL
TRUNCATE superImportantData;
```

### CREATE INDEX 创建二级索引

*Syntax:*

```
<create-index-stmt> ::= CREATE ( CUSTOM )? INDEX ( IF NOT EXISTS )? ( <indexname> )?
                            ON <tablename> '(' <index-identifier> ')'
                            ( USING <string> ( WITH OPTIONS = <map-literal> )? )?

<index-identifier> ::= <identifier>
                     | keys( <identifier> )
```

*Sample:*

```CQL
CREATE INDEX userIndex ON NerdMovies (user);
CREATE INDEX ON Mutants (abilityId);
CREATE INDEX ON users (keys(favs));
CREATE CUSTOM INDEX ON users (email) USING 'path.to.the.IndexClass';
CREATE CUSTOM INDEX ON users (email) USING 'path.to.the.IndexClass' WITH OPTIONS = {'storage': '/mnt/ssd/indexes/'};
```

如果需要，可以在ON关键字之前指定索引本身的名称。如果该列的数据已经存在，则将对其进行异步索引。创建索引之后，列的新数据将在插入时自动建立索引。

在map列上创建索引时，可以对键或值建立索引。如果列标识符位于 `keys() ` 函数内，则索引将位于map键上，从而允许在 WHERE 子句中使用 `CONTAINS KEY`。否则，索引将位于map值上。

### DROP INDEX 删除索引

*Syntax:*

```
<drop-index-stmt> ::= DROP INDEX ( IF EXISTS )? ( <keyspace> '.' )? <identifier>
```

*Sample:*

```CQL
DROP INDEX userIndex;

DROP INDEX userkeyspace.address_index;
```

### MATERIALIZED VIEW 物化视图

此处暂略相关内容。

### CREATE TYPE 创建类型

*Syntax:*

```
<create-type-stmt> ::= CREATE TYPE ( IF NOT EXISTS )? <typename>
                         '(' <field-definition> ( ',' <field-definition> )* ')'

<typename> ::= ( <keyspace-name> '.' )? <identifier>

<field-definition> ::= <identifier> <type>
```

*Sample:*

```CQL
CREATE TYPE address (
    street_name text,
    street_number int,
    city text,
    state text,
    zip int
)

CREATE TYPE work_and_home_addresses (
    home_address address,
    work_address address
)
```

创建自定义类型，每个类型都是一组明确命名、明确类型的字段集合。字段类型可以是任何有效的类型，包括集合和其他已存在的用户自定义类型。

#### `<typename>`

有效的类型名是标识符。不能使用现有的CQL类型名和保留类型名。

如果只提供类型名，则使用当前键空间创建类型（参考 `USE`）。如果以存在的键空间名称作为前缀，则该类型将在指定的键空间中创建，而不是在当前键空间中创建。

### ALTER TYPE 修改类型

*Syntax:*

```
<alter-type-stmt> ::= ALTER TYPE <typename> <instruction>

<instruction> ::= ALTER <field-name> TYPE <type>
                | ADD <field-name> <type>
                | RENAME <field-name> TO <field-name> ( AND <field-name> TO <field-name> )*
```

*Sample:*

```CQL
ALTER TYPE address ALTER zip TYPE varint

ALTER TYPE address ADD country text

ALTER TYPE address RENAME zip TO zipcode AND street_name TO street
```

允许添加新字段、重命名现有字段或更改现有字段的类型。更改列的类型时，新类型必须与旧类型兼容。

### DROP TYPE 删除类型

*Syntax:*

```
<drop-type-stmt> ::= DROP TYPE ( IF EXISTS )? <typename>
```

立即、不可逆地删除一个类型。

### CREATE TRIGGER 创建触发器

*Syntax:*

```
<create-trigger-stmt> ::= CREATE TRIGGER ( IF NOT EXISTS )? ( <triggername> )?
                            ON <tablename> 
                            USING <string>
```

*Sample:*

```CQL
CREATE TRIGGER myTrigger ON myTable USING 'org.apache.cassandra.triggers.InvertedIndex';
```

触发器的实际逻辑可以用任何 JVM 语言编写，并且存放在数据库之外。将触发器代码放在Cassandra安装目录的lib/triggers 子目录中，在集群启动期间将会加载，并存在于集群的各个节点上。在表上定义的触发器会在请求的DML 语句触发之前触发，从而确保了事务的原子性。

### DROP TRIGGER 删除触发器

*Syntax:*

```
<drop-trigger-stmt> ::= DROP TRIGGER ( IF EXISTS )? ( <triggername> )?
                            ON <tablename>
```

*Sample:*

```CQL
DROP TRIGGER myTrigger ON myTable;
```

### CREATE FUNCTION 创建函数

*Syntax:*

```
<create-function-stmt> ::= CREATE ( OR REPLACE )? 
                            FUNCTION ( IF NOT EXISTS )?
                            ( <keyspace> '.' )? <function-name>
                            '(' <arg-name> <arg-type> ( ',' <arg-name> <arg-type> )* ')'
                            ( CALLED | RETURNS NULL ) ON NULL INPUT
                            RETURNS <type>
                            LANGUAGE <language>
                            AS <body>
```

*Sample:*

```CQL
CREATE OR REPLACE FUNCTION somefunction
    ( somearg int, anotherarg text, complexarg frozen<someUDT>, listarg list<bigint> )
    RETURNS NULL ON NULL INPUT
    RETURNS text
    LANGUAGE java
    AS $$
       // some Java code
    $$;
CREATE FUNCTION akeyspace.fname IF NOT EXISTS
    ( someArg int )
    CALLED ON NULL INPUT
    RETURNS text
    LANGUAGE java
    AS $$
       // some Java code
    $$;
```

暂略。

### DROP FUNCTION 删除函数

*Syntax:*

```
<drop-function-stmt> ::= DROP FUNCTION ( IF EXISTS )?
                         ( <keyspace> '.' )? <function-name>
                         ( '(' <arg-type> ( ',' <arg-type> )* ')' )?
```

*Sample:*

```CQL
DROP FUNCTION myfunction;
DROP FUNCTION mykeyspace.afunction;
DROP FUNCTION afunction ( int );
DROP FUNCTION afunction ( text );
```

暂略。

### CREATE AGGREGATE 创建聚合函数

*Syntax:*

```
<create-aggregate-stmt> ::= CREATE ( OR REPLACE )? 
                            AGGREGATE ( IF NOT EXISTS )?
                            ( <keyspace> '.' )? <aggregate-name>
                            '(' <arg-type> ( ',' <arg-type> )* ')'
                            SFUNC <state-functionname>
                            STYPE <state-type>
                            ( FINALFUNC <final-functionname> )?
                            ( INITCOND <init-cond> )?
```

*Sample:*

```CQL
CREATE AGGREGATE myaggregate ( val text )
  SFUNC myaggregate_state
  STYPE text
  FINALFUNC myaggregate_final
  INITCOND 'foo';
```

可以创建或替换用户自定义聚合函数。

暂略。

### DROP AGGREGATE 删除聚合函数

*Syntax:*

```
<drop-aggregate-stmt> ::= DROP AGGREGATE ( IF EXISTS )?
                         ( <keyspace> '.' )? <aggregate-name>
                         ( '(' <arg-type> ( ',' <arg-type> )* ')' )?
```

*Sample:*

```CQL
DROP AGGREGATE myAggregate;
DROP AGGREGATE myKeyspace.anAggregate;
DROP AGGREGATE someAggregate ( int );
DROP AGGREGATE someAggregate ( text );
```

暂略。

## 数据操作

### INSERT 插入数据

*Syntax:*

```
<insertStatement> ::= INSERT INTO <tablename>
                      ( ( <name-list> VALUES <value-list> )
                      | ( JSON <string> ))
                      ( IF NOT EXISTS )?
                      ( USING <option> ( AND <option> )* )?

<names-list> ::= '(' <identifier> ( ',' <identifier> )* ')'

<value-list> ::= '(' <term-or-literal> ( ',' <term-or-literal> )* ')'

<term-or-literal> ::= <term>
                    | <collection-literal>

<option> ::= TIMESTAMP <integer>
           | TTL <integer>
```

*Sample:*

```CQL
INSERT INTO NerdMovies (movie, director, main_actor, year)
                VALUES ('Serenity', 'Joss Whedon', 'Nathan Fillion', 2005)
USING TTL 86400;

INSERT INTO NerdMovies JSON '{"movie": "Serenity", "director": "Joss Whedon", "year": 2005}'
```

- 插入数据必须指定所有的主键
- 当使用 `VALUES` 语法时，必须提供要插入的数据列表；当使用 `JSON` 语法时，这是可选的。
- 与 SQL 不同，`INSERT` 默认不会检查此行是否已经存在。如果主键不存在，插入将会新增；否则则更新。此外，也无法知道实际上是进行了创建还是更新。
- 但是，可以使用 `IF NOT EXISTS` 条件，仅在不存在相应行的情况下进行插入。但请注意，如果使用`IF NOT EXISTS`，将会导致不可忽视的性能损失（内部会使用 Paxos），所以应该谨慎使用。
- `INSERT `的所有更新都是原子且独立地应用的。
- `INSERT` 不支持计数器，而 `UPDATE` 支持。
- 可用的 `<option>` 选项请参考 `UPDATE` ，`<collection-literal>` 参考 `collections `。

### UPDATE 更新数据

*Syntax:*

```
<update-stmt> ::= UPDATE <tablename>
                  ( USING <option> ( AND <option> )* )?
                  SET <assignment> ( ',' <assignment> )*
                  WHERE <where-clause>
                  ( IF <condition> ( AND condition )* )?

<assignment> ::= <identifier> '=' <term>
               | <identifier> '=' <identifier> ('+' | '-') (<int-term> | <set-literal> | <list-literal>)
               | <identifier> '=' <identifier> '+' <map-literal>
               | <identifier> '[' <term> ']' '=' <term>

<condition> ::= <identifier> <op> <term>
              | <identifier> IN (<variable> | '(' ( <term> ( ',' <term> )* )? ')')
              | <identifier> '[' <term> ']' <op> <term>
              | <identifier> '[' <term> ']' IN <term>

<op> ::= '<' | '<=' | '=' | '!=' | '>=' | '>'

<where-clause> ::= <relation> ( AND <relation> )*

<relation> ::= <identifier> '=' <term>
             | '(' <identifier> (',' <identifier>)* ')' '=' <term-tuple>
             | <identifier> IN '(' ( <term> ( ',' <term>)* )? ')'
             | <identifier> IN <variable>
             | '(' <identifier> (',' <identifier>)* ')' IN '(' ( <term-tuple> ( ',' <term-tuple>)* )? ')'
             | '(' <identifier> (',' <identifier>)* ')' IN <variable>

<option> ::= TIMESTAMP <integer>
           | TTL <integer>
```

*Sample:*

```CQL
UPDATE NerdMovies USING TTL 400
SET director = 'Joss Whedon',
    main_actor = 'Nathan Fillion',
    year = 2005
WHERE movie = 'Serenity';

UPDATE UserActions SET total = total + 2 WHERE user = B70DE1D0-9908-4AE3-BE34-5573E5B09F14 AND action = 'click';
```

- 可以更新指定一行的一或多列数据。
- `<where-clause>` 必须包含所有的主键。
- 与 SQL 不同，`UPDATE` 默认不会检查此行是否已经存在。如果主键不存在，将会新增行；否则则更新。此外，也无法知道实际上是进行了创建还是更新。
- 但是可以使用 `IF` 在列上使用条件来决定是否要更新。但这会导致不可忽视的性能损失（内部会使用 Paxos），所以应该谨慎使用。
- 同一个分区键内的所有更新都是原子且独立地应用的。
- `<assignment>` 的 `c=c+3` 形式用于递增/递减计数器。“=” 前后的标识符必须相同（计数器只支持递增/递减，不支持赋值特定值）。
- `<assignment>`  中的 `id = id + <collection-literal>` 和 `id[value1] = value2` 用于集合，具体请参考相关章节。

#### `<options>`

 `UPDATE` 和 `INSERT`  支持以下设置：

- `TIMESTAMP`：设置操作的时间戳。如果没有指定，协调器将使用语句执行开始时的时间（微秒）作为时间戳。这一般是合适的默认值。
- `TTL`：对插入的数据指定生存时间（秒）。如果设置了该值，在指定的时间之后，插入的值将自动从数据库中删除。需要注意的是 TTL 作用的是值而不是列本身，因此此列的任何后续更新都会刷新 TTL 为更新时指定的 TTL 值。TTL 为 0 或负数则代表无 TTL（默认情况），值不会过期。

### DELETE 删除数据

*Syntax:*

```
<delete-stmt> ::= DELETE ( <selection> ( ',' <selection> )* )?
                  FROM <tablename>
                  ( USING TIMESTAMP <integer>)?
                  WHERE <where-clause>
                  ( IF ( EXISTS | ( <condition> ( AND <condition> )*) ) )?

<selection> ::= <identifier> ( '[' <term> ']' )?

<where-clause> ::= <relation> ( AND <relation> )*

<relation> ::= <identifier> <op> <term>
             | '(' <identifier> (',' <identifier>)* ')' <op> <term-tuple>
             | <identifier> IN '(' ( <term> ( ',' <term>)* )? ')'
             | <identifier> IN <variable>
             | '(' <identifier> (',' <identifier>)* ')' IN '(' ( <term-tuple> ( ',' <term-tuple>)* )? ')'
             | '(' <identifier> (',' <identifier>)* ')' IN <variable>

<op> ::= '=' | '<' | '>' | '<=' | '>='

<condition> ::= <identifier> (<op> | '!=') <term>
              | <identifier> IN (<variable> | '(' ( <term> ( ',' <term> )* )? ')')
              | <identifier> '[' <term> ']' (<op> | '!=') <term>
              | <identifier> '[' <term> ']' IN <term>
```

*Sample:*

```CQL
DELETE FROM NerdMovies USING TIMESTAMP 1240003134 WHERE movie = 'Serenity';

DELETE phone FROM Users WHERE userid IN (C73DE1D3-AF08-40F3-B124-3FF3E5109F22, B70DE1D0-9908-4AE3-BE34-5573E5B09F14);
```

- 使用 `IN` 语法和范围操作符（如 >=）可以批量删除
- 支持`TIMESTAMP`选项，其语义与`UPDATE`语句相同。
- 同一个分区键内的所有删除都是原子且独立地应用的。
- 使用 `IF` 语法会导致不可忽视的性能损失，类似 `INSERT` 和 `UPDATE`。

### BATCH 批处理数据

*Syntax:*

```
<batch-stmt> ::= BEGIN ( UNLOGGED | COUNTER ) BATCH
                 ( USING <option> ( AND <option> )* )?
                    <modification-stmt> ( ';' <modification-stmt> )*
                 APPLY BATCH

<modification-stmt> ::= <insert-stmt>
                      | <update-stmt>
                      | <delete-stmt>

<option> ::= TIMESTAMP <integer>
```

*Sample:*

```CQL
BEGIN BATCH
  INSERT INTO users (userid, password, name) VALUES ('user2', 'ch@ngem3b', 'second user');
  UPDATE users SET password = 'ps22dhds' WHERE userid = 'user3';
  INSERT INTO users (userid, password) VALUES ('user4', 'ch@ngem3c');
  DELETE name FROM users WHERE userid = 'user1';
APPLY BATCH;
```

 `BATCH` 可以将多条语句封装成一条语句执行，主要是以下作用：

- 节省网络成本
- 一条 `BATCH` 中的所有对同一个分区键的更新是独立执行的
- 默认情况下，batch 里的所有操作都是以 `LOGGED` 执行的，以确保所有更改最终都被完成（或没有完成）。具体参考 ` UNLOGGED`。

注意：

- `BATCH` 只能包含 `UPDATE`, `INSERT` 和 `DELETE` 语句。
- 批处理并**不是** SQL 事务的完全类似物。
- 如果没有为每个操作指定时间戳，那么所有操作都将应用相同的时间戳。时间戳与 Cassandra 的冲突解决策略相关，详情请参考时间戳相关章节。

#### `UNLOGGED`

当批处理跨越多个分区时，其原子性会带来性能损失。如果想规避这种损失，可以使用 `UNLOGGED` 选项跳过 batchlog。代价是如果批处理失败了，可能会使补丁只作用了一部分。

#### `COUNTER`

批处理计数器更新。与 Cassandra 中的其他更新不同，计数器更新不是幂等的。

#### `<option>`

`BATCH` 也支持 `TIMESTAMP` 选项，和 `UPDATE` 的类似。如果 `BATCH` 使用了 `TIMESTAMP` ，那么内部的语句就不能再使用 `TIMESTAMP` 。

## 查询

### SELECT

*Syntax:*

```
<select-stmt> ::= SELECT ( JSON )? <select-clause>
                  FROM <tablename>
                  ( WHERE <where-clause> )?
                  ( ORDER BY <order-by> )?
                  ( LIMIT <integer> )?
                  ( ALLOW FILTERING )?

<select-clause> ::= DISTINCT? <selection-list>
                  | COUNT '(' ( '*' | '1' ) ')' (AS <identifier>)?

<selection-list> ::= <selector> (AS <identifier>)? ( ',' <selector> (AS <identifier>)? )*
                   | '*'

<selector> ::= <identifier>
             | WRITETIME '(' <identifier> ')'
             | TTL '(' <identifier> ')'
             | <function> '(' (<selector> (',' <selector>)*)? ')'

<where-clause> ::= <relation> ( AND <relation> )*

<relation> ::= <identifier> <op> <term>
             | '(' <identifier> (',' <identifier>)* ')' <op> <term-tuple>
             | <identifier> IN '(' ( <term> ( ',' <term>)* )? ')'
             | '(' <identifier> (',' <identifier>)* ')' IN '(' ( <term-tuple> ( ',' <term-tuple>)* )? ')'
             | TOKEN '(' <identifier> ( ',' <identifer>)* ')' <op> <term>

<op> ::= '=' | '<' | '>' | '<=' | '>=' | CONTAINS | CONTAINS KEY
<order-by> ::= <ordering> ( ',' <odering> )*
<ordering> ::= <identifer> ( ASC | DESC )?
<term-tuple> ::= '(' <term> (',' <term>)* ')'
```

*Sample:*

```CQL
SELECT name, occupation FROM users WHERE userid IN (199, 200, 207);

SELECT JSON name, occupation FROM users WHERE userid = 199;

SELECT name AS user_name, occupation AS user_occupation FROM users;

SELECT time, value
FROM events
WHERE event_type = 'myEvent'
  AND time > '2011-02-03'
  AND time <= '2012-01-01'

SELECT COUNT(*) FROM users;

SELECT COUNT(*) AS user_count FROM users;
```

返回符合查询条件的结果集。如果使用了 `JSON` 关键字，那么结果将只会包括一栏名为 “json” 的列。具体请参考 `SELECT JSON`。

#### `<select-clause>`

 `<select-clause>` 决定了哪些列需要在结果集中被返回，其为逗号分隔的列表，或代表全部列的通配符（`*`）。

一个 `<selector>` 可以是一个列名，也可以是一个包含了一或多个  @@s 的 `<function>`。允许的函数与  `<term>`  相同，具体请参考函数相关章节。除了这些通用函数以外，还支持 `WRITETIME` 函数选择此栏被插入时的时间戳，类似地还有 `TTL` 函数对应其生存时间。

任何 `<selector>` 都可以使用 `AS` 关键字标注别名。请注意， `<where-clause>` 和 `<order-by>` 子句应该使用它们的原始名称而非它们的别名。

 `COUNT(1)`  是 `COUNT(*)`  的别名。

#### `<where-clause>`

 `<where-clause>` 指明了哪些行一定会被查询到。其由列上的关系组成，并且这些列为主键的一部分，或存在一个二级索引。

并非所有关系都允许在查询中使用。比如不支持在分区键上使用“不等于”关系（但是可以参考下文的 `TOKEN` 方法对分区键执行不等于查询）。以及，对于一个给定的分区键，其聚集键仅允许选择一组连续行的关系。比如对于以下表：

```CQL
CREATE TABLE posts (
    userid text,
    blog_title text,
    posted_at timestamp,
    entry_title text,
    content text,
    category int,
    PRIMARY KEY (userid, blog_title, posted_at)
)
```

这一句查询是可行的：

```CQL
SELECT entry_title, content FROM posts WHERE userid='john doe' AND blog_title='John''s Blog' AND posted_at >= '2012-01-01' AND posted_at < '2012-01-31'
```

但以下这一句不可行，因为其没有选择一组连续的行（假设没有设置二级索引）：

```CQL
// Needs a blog_title to be set to select ranges of posted_at
SELECT entry_title, content FROM posts WHERE userid='john doe' AND posted_at >= '2012-01-01' AND posted_at < '2012-01-31'
```

在指定关系时，`TOKEN` 函数可以被用于分区键的查询。在这种情况下，将根据分区键的 token 而不是它们的值来进行选择。注意，键的 token 依赖于被使用的分区器，尤其是， RandomPartitioner 不会产生有意义的顺序。同时还要注意，排序分区器总是根据字节来进行 token 值的排序，所以即便分区键的值是 int 型，实际上却是`token(-1) > token(0)`。比如：

```CQL
SELECT * FROM posts WHERE token(userid) > token('tom') AND token(userid) < token('bob')
```

并且，`IN` 关系只允许使用在分区键的最后一列和完整主键的最后一列上。

可以对聚集键使用元组标识进行组合查询，比如：

```CQL
SELECT * FROM posts WHERE userid='john doe' AND (blog_title, posted_at) > ('John''s Blog', '2012-01-01')
```

这会返回排序在 `blog_tile` = `John's Blog` 且  `posted_at`  = `2012-01-01` 之后的行。事实上，只要这些行的r `blog_title > 'John''s Blog'`，即便其 `post_at <= '2012-01-01'` ，结果依然会返回。如果不想出现这种结果，那应该使用以下这种：

```CQL
SELECT * FROM posts WHERE userid='john doe' AND blog_title > 'John''s Blog' AND posted_at > '2012-01-01'
```

元组标识同样可以用于聚集键的 `IN` 子句中：

```CQL
SELECT * FROM posts WHERE userid='john doe' AND (blog_title, posted_at) IN (('John''s Blog', '2012-01-01), ('Extreme Chess', '2014-06-01'))
```

 `CONTAINS` 操作符只作用在集合类的列中。为 map 的情况下， `CONTAINS` 指向 map 的值；若想指向 map 的键，应该使用 `CONTAINS KEY` 。

#### `<order-by>`

 `ORDER BY` 选项可以指定返回结果的顺序。`ASC` 升序，`DESC` 降序，默认升序。依赖于表的 ` CLUSTERING ORDER ` 配置，存在以下限制：

- 如果表定义时没有任何特定的 ` CLUSTERING ORDER ` ，允许的排序是聚集键的顺序或其倒序。
- 否则，只允许  `CLUSTERING ORDER` 中设置的顺序或其倒序。

#### `LIMIT`

限制一条 `SELECT` 语句返回的行数。

#### `ALLOW FILTERING`

默认情况下，CQL 只允许那些不会引起服务端 “filtering” 操作的查询，例如我们知道会在结果集中读取返回全部记录的查询。因为可以预测到这些 “non filtering” 查询的性能，其所消耗的时间将与所要返回的数据量成正比（可以使用 `LIMIT` 控制）。

 `ALLOW FILTERING` 选项将显式地允许（某些）需要 filtering 的查询。请注意，如上所述，使用 `ALLOW FILTERING`  可能会导致不可预测的性能状况，比如即使只是选择少数记录的查询，其性能也可能收到集群数据总量的影响。

举个例子，下面的表格包含了用户的出生年份（有二级索引）和居住国家：

```CQL
CREATE TABLE users (
    username text PRIMARY KEY,
    firstname text,
    lastname text,
    birth_year int,
    country text
)

CREATE INDEX ON users(birth_year);
```

以下这些查询是可行的：

```CQL
SELECT * FROM users;
SELECT firstname, lastname FROM users WHERE birth_year = 1981;
```

Cassandra 保证这些查询的性能与返回的数据量成比例。实际上，如果没有1981年出生的用户，那么第二个查询的性能并不会依赖于数据库中存储的用户数据（至少不会是直接依赖。考虑到二级索引的实现，该查询可能仍然依赖于集群中的节点数量，而节点数量间接依赖于存储的数据量。）。当然，这两个查询实际上都可能返回非常大的结果集，但是都可以通过添加一个 `LIMIT` 参数来控制返回的数据量。

不过下面这种就会被拒绝执行：

```CQL
SELECT firstname, lastname FROM users WHERE birth_year = 1981 AND country = 'FR';
```

因为 Cassandra 不能保证它不会扫描大量数据，即使实际的结果量其实很小。一般来说，即使只有少数人真正来自法国，它依然会扫描所有1981年出生用户的索引条目。但如果你确实知道你在做什么，那么可以加上 `ALLOW FILTERING` 让查询变得合法：

```CQL
SELECT firstname, lastname FROM users WHERE birth_year = 1981 AND country = 'FR' ALLOW FILTERING;
```

## 数据库角色/权限控制

暂时接触不上，以后再补充。

## 数据类型

CQL 支持丰富的类型来定义表中的列，其中包括集合类型和用户自定义类型（通过继承 Cassandra 提供。

```
<type> ::= <native-type>
         | <collection-type>
         | <tuple-type>
         | <string>       // 用于用户自定义类型，其 JAVA 类的全限定名

<native-type> ::= ascii
                | bigint
                | blob
                | boolean
                | counter
                | date
                | decimal
                | double
                | float
                | inet
                | int
                | smallint
                | text
                | time
                | timestamp
                | timeuuid
                | tinyint
                | uuid
                | varchar
                | varint

<collection-type> ::= list '<' <native-type> '>'
                    | set  '<' <native-type> '>'
                    | map  '<' <native-type> ',' <native-type> '>'
<tuple-type> ::= tuple '<' <type> (',' <type>)* '>'
```

注意原始类型是关键词并且也是不区分大小写的。不过它们并不是保留字。

下表给出了原始类型的附加信息以及其支持的常量类型：

| type        | constants supported | description                                                                                                                                                                                        |
| ----------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ascii`     | strings             | ASCII character string                                                                                                                                                                             |
| `bigint`    | integers            | 64-bit signed long                                                                                                                                                                                 |
| `blob`      | blobs               | Arbitrary bytes (no validation)                                                                                                                                                                    |
| `boolean`   | booleans            | true or false                                                                                                                                                                                      |
| `counter`   | integers            | Counter column (64-bit signed value). See [Counters](https://cassandra.apache.org/doc/old/CQL-3.0.html#counters) for details                                                                       |
| `date`      | integers, strings   | A date (with no corresponding time value). See [Working with dates](https://cassandra.apache.org/doc/old/CQL-3.0.html#usingdates) below for more information.                                      |
| `decimal`   | integers, floats    | Variable-precision decimal                                                                                                                                                                         |
| `double`    | integers            | 64-bit IEEE-754 floating point                                                                                                                                                                     |
| `float`     | integers, floats    | 32-bit IEEE-754 floating point                                                                                                                                                                     |
| `inet`      | strings             | An IP address. It can be either 4 bytes long (IPv4) or 16 bytes long (IPv6). There is no `inet` constant, IP address should be inputed as strings                                                  |
| `int`       | integers            | 32-bit signed int                                                                                                                                                                                  |
| `smallint`  | integers            | 16-bit signed int                                                                                                                                                                                  |
| `text`      | strings             | UTF8 encoded string                                                                                                                                                                                |
| `time`      | integers, strings   | A time with nanosecond precision. See [Working with time](https://cassandra.apache.org/doc/old/CQL-3.0.html#usingtime) below for more information.                                                 |
| `timestamp` | integers, strings   | A timestamp. Strings constant are allow to input timestamps as dates, see [Working with timestamps](https://cassandra.apache.org/doc/old/CQL-3.0.html#usingtimestamps) below for more information. |
| `timeuuid`  | uuids               | Type 1 UUID. This is generally used as a “conflict-free” timestamp. Also see the [functions on Timeuuid](https://cassandra.apache.org/doc/old/CQL-3.0.html#timeuuidFun)                            |
| `tinyint`   | integers            | 8-bit signed int                                                                                                                                                                                   |
| `uuid`      | uuids               | Type 1 or type 4 UUID                                                                                                                                                                              |
| `varchar`   | strings             | UTF8 encoded string                                                                                                                                                                                |
| `varint`    | integers            | Arbitrary-precision integer                                                                                                                                                                        |

### 关于时间戳 timestamps

 `timestamp` 类型由一个代表 January 1 1970 at 00:00:00 GMT. 以来经过的毫秒数的64位有符号整数编码。

时间戳可以直接使用长整数在 CQL 中使用，也可以使用以下 任意 ISO 8601 格式的字符串使用：

- `2011-02-03 04:05+0000`
- `2011-02-03 04:05:00+0000`
- `2011-02-03 04:05:00.000+0000`
- `2011-02-03T04:05+0000`
- `2011-02-03T04:05:00+0000`
- `2011-02-03T04:05:00.000+0000`

以上 `+0000` 用于指定时区，可以省略，此时使用的是 Cassandra 节点所配置的时区。

- `2011-02-03 04:05`
- `2011-02-03 04:05:00`
- `2011-02-03 04:05:00.000`
- `2011-02-03T04:05`
- `2011-02-03T04:05:00`
- `2011-02-03T04:05:00.000`

如果日期是唯一重要的部分，时间也可以省略，此时的时间将被设定为 `00:00:00`。

- `2011-02-03`
- `2011-02-03+0000`

### 关于日期 dates

 `date` 类型由一个32位无符号整数编码，代表 January 1st, 1970 以来的天数。

CQL 中日期可以使用一个无符号整数也可以使用以下格式的字符串：

- `2014-01-01`

### 关于时间 time

 `time` 类型由一个64位有符号整数编码，代表从半夜以来的纳秒数。

- `08:12:54`
- `08:12:54.123`
- `08:12:54.123456`
- `08:12:54.123456789`

### Counters 计数器

 `counter` 类型用于定义计数器列，由一个64位有符号整数编码，支持递增和递减两种操作（参照 `UPDATE`）。

计数器不能被直接赋值，在发生第一次递增或递减操作之前计数器不会存在，在发生第一次操作时其初始值会被当作是0。计数器列支持被删除，但存在一些限制。（具体待补充，跳转链接过期了）

计数器类型存在以下限制：

- 不能被当作主键
- 一张包含计数器类型的表只能包含计数器。换句话说，主键以外的列要不全是计数器类型，要不全都不是计数器类型。

### 关于集合 collections

#### 注意点

集合主要用于存储、规范化相对小规模的数据。如果集合的数据量会无边界地增长，那么集合就不再合适了，应该使用包含聚集键的特定表。具体地说，集合存在以下限制：

- 集合总是被完整地读取，读取一个集合在内部不会被分页
- 集合不能有多于 65535 个元素。虽然可以插入大于 65535 个元素，但无法读取超过 65535 的第一个元素（参考 [CASSANDRA-5428](https://issues.apache.org/jira/browse/CASSANDRA-5428)）
- 对于 set 和 map 类型，插入操作在内部不会触发写前读（read-before-write），而 list 的一些操作会。因此推荐尽可能优先使用 list。

#### Maps

 `map` 是一组键值对，且键是唯一的。此外，map 在内部是按键排序的，并且总是按照这个顺序返回。创建 map 的例子：

```CQL
CREATE TABLE users (
    id text PRIMARY KEY,
    given text,
    surname text,
    favs map<text, text>   // A map of text keys, and text values
)
```

写入 map 数据使用的是 JSON 式的语法。如果要使用 `INSERT` 写入一条数据，则将整个 map 指定为 json 风格的关联数组（注意，这种会替换掉整个 map）：

```CQL
// Inserting (or Updating)
INSERT INTO users (id, given, surname, favs)
           VALUES ('jsmith', 'John', 'Smith', { 'fruit' : 'apple', 'band' : 'Beatles' })
```

可以使用 `UPDATE` 更新 map 中的部分数据或添加新数据：

```CQL
// Updating (or inserting)
UPDATE users SET favs['author'] = 'Ed Poe' WHERE id = 'jsmith'
UPDATE users SET favs = favs +  { 'movie' : 'Cassablanca' } WHERE id = 'jsmith'
```

`TTL` 可以用于 `INSERT` 和 `UPDATE`，但都只对新插入/更新的值有效。比如：

```CQL
// Updating (or inserting)
UPDATE users USING TTL 10 SET favs['color'] = 'green' WHERE id = 'jsmith'
```

TTL 只作用在  `{ 'color' : 'green' }`  中，map 的其他值则不会受到影响。

删除一个 map 记录：

```CQL
DELETE favs['author'] FROM users WHERE id = 'jsmith'
```

#### Sets

Set 是唯一值的类型化集合，由其中的值进行排序。创建一个 set：

```CQL
CREATE TABLE images (
    name text PRIMARY KEY,
    owner text,
    date timestamp,
    tags set<text>
);
```

写入一个 set：

```CQL
INSERT INTO images (name, owner, date, tags)
            VALUES ('cat.jpg', 'jsmith', 'now', { 'kitten', 'cat', 'pet' });
```

注意：`INSERT` 总是会替换掉整个 set。

可以通过 `UPDATE` 在现有 set 中添加/删除新 set 值来添加和删除集合的值。

```CQL
UPDATE images SET tags = tags + { 'cute', 'cuddly' } WHERE name = 'cat.jpg';
UPDATE images SET tags = tags - { 'lame' } WHERE name = 'cat.jpg';
```

跟 map 一样，TTL 只作用在新插入/更新的值上。

#### Lists

List 是一种非唯一值的类型化集合，其中元素按 list 中的位置排序。创建一个 list：

```CQL
CREATE TABLE plays (
    id text PRIMARY KEY,
    game text,
    players int,
    scores list<int>
)
```

需要注意，list 存在一些限制和需要考虑的性能约束，比起 list 应尽可能优先使用 set。

写入 list ：

```CQL
INSERT INTO plays (id, game, players, scores)
           VALUES ('123-afde', 'quake', 3, [17, 4, 2]);
```

注意：`INSERT` 总是会替换掉整个 list。

在已存在 list 的头部或尾部插入部分新的数据：

```CQL
UPDATE plays SET players = 5, scores = scores + [ 14, 21 ] WHERE id = '123-afde';
UPDATE plays SET players = 5, scores = [ 12 ] + scores WHERE id = '123-afde';
```

注意：添加操作并非幂等的，因此如果采用失败重试的策略，有可能会往 list 中重复插入相同的值。

List 还提供了以下操作：根据元素在 list 中的位置设置、删除元素；删除列表中给定值的所有出现项。然而，与所有其他集合操作不同，这三种操作会在更新之前触发内部读取，导致性能通常较慢。

```CQL
UPDATE plays SET scores[1] = 7 WHERE id = '123-afde';                // sets the 2nd element of scores to 7 (raises an error is scores has less than 2 elements)
DELETE scores[1] FROM plays WHERE id = '123-afde';                   // deletes the 2nd element of scores (raises an error is scores has less than 2 elements)
UPDATE plays SET scores = scores - [ 12, 21 ] WHERE id = '123-afde'; // removes all occurrences of 12 and 21 from scores
```

与 map 相同，TTL 仅作用在新插入/更新的值上。

## 函数

CQL3 区分了内置函数（也称本地函数 native functions）和用户自定义函数。CQL3 包括了以下几种本地函数：

### Token

`token` 函数用于计算一个给定分区键的 token 值。token 函数的准确签名取决于相关表和集群所使用的分区器。

token 的参数类型取决于分区键列的类型，返回类型取决于所使用的分区器：

- Murmur3Partitioner： `bigint`
- RandomPartitioner： `varint`
- ByteOrderedPartitioner： `blob`

举个例子，在使用默认 Murmur3Partitioner 分区器的集群中，如果表被定义为：

```CQL
CREATE TABLE users (
    userid text PRIMARY KEY,
    username text,
    ...
)
```

token 函数则将接受单独一个 `text` 类型的入参，然后返回的类型为 `bigint`。

### Uuid

`uuid` 函数不需要入参，生成一个随机的 Version 4 UUID 用于 `INSERT` 或 `SET` 语句中。

### Timeuuid functions

#### `now`

`now` 函数不需要入参，会在执行语句时，在协调器节点上生成一个唯一的 timeuuid。注意虽然这个函数对于插入有用，但对于 `WHERE` 子句来说几乎毫无意义，比如：

```CQL
SELECT * FROM myTable WHERE t = now()
```

在设计上这条查询永远不会返回任何结果，因为 `now()` 生成的值是唯一的。

#### `minTimeuuid` 与 `maxTimeuuid`

`minTimeuuid`函数接收一个 `timestamp` 类型的入参 t （可以是长整数也可以是字符串），然后返回一个假造的 `timeuuid` ，对应此时间戳 t 可能的最小值 `timeuuid`。`maxTimeuuid` 则相反对应其最大值。例如：

```CQL
SELECT * FROM myTable WHERE t > maxTimeuuid('2013-01-01 00:05+0000') AND t < minTimeuuid('2013-02-02 10:00+0000')
```

这条会选择所有  `timeuuid` 列 t 中严格地比 ‘2013-01-01 00:05+0000’  晚但比 ‘2013-02-02 10:00+0000’ 早的所有行。请注意， `t >= maxTimeuuid('2013-01-01 00:05+0000')` 依然不会选择精确地在 ‘2013-01-01 00:05+0000’  时生成的 `timeuuid` ，本质上等同于  `t > maxTimeuuid('2013-01-01 00:05+0000')`。

注意：我们称 `minTimeuuid` 和 `maxTimeuuid` 生成的值为假的 UUID，是因为它们并没有遵循 [RFC 4122](http://www.ietf.org/rfc/rfc4122.txt) 的基于时间的UUID生成进程。特别是这两个函数返回的值并不是唯一的，因此你应该只能使用这些函数进行类似上文例子的查询。不要在插入数据时使用这些函数。

### 时间转换函数

有一系列函数用于转换  `timeuuid`、 `timestamp`  或 `date` 类型值为另一种  `native`  类型。

| function name     | input type  | description                                                 |
| ----------------- | ----------- | ----------------------------------------------------------- |
| `toDate`          | `timeuuid`  | Converts the `timeuuid` argument into a `date` type         |
| `toDate`          | `timestamp` | Converts the `timestamp` argument into a `date` type        |
| `toTimestamp`     | `timeuuid`  | Converts the `timeuuid` argument into a `timestamp` type    |
| `toTimestamp`     | `date`      | Converts the `date` argument into a `timestamp` type        |
| `toUnixTimestamp` | `timeuuid`  | Converts the `timeuuid` argument into a `bigInt` raw value  |
| `toUnixTimestamp` | `timestamp` | Converts the `timestamp` argument into a `bigInt` raw value |
| `toUnixTimestamp` | `date`      | Converts the `date` argument into a `bigInt` raw value      |
| `dateOf`          | `timeuuid`  | Similar to `toTimestamp(timeuuid)` (DEPRECATED)             |
| `unixTimestampOf` | `timeuuid`  | Similar to `toUnixTimestamp(timeuuid)` (DEPRECATED)         |

### Blob转换函数

对于所有 CQL3 所支持的本地类型（除了`blob`)，`typeAsBlob` 函数接受一个 `type` 参数并返回一个 `blob` 值。相反地，函数 `blobAsType` 接受一个 64 位 `blob` 参数，将其转换为一个 `bigint` 值。例如， `bigintAsBlob(3)` 返回 `0x0000000000000003` ， `blobAsBigint(0x0000000000000003)` 返回 `3`.

## 聚合函数

聚合函数从每一行中接受值，最终对整个集合返回一个值。如果是 `normal` 列、`scalar ` 函数、 `UDT`  字段、 `writetime` 或 `ttl` 一起在聚合函数中被选择，其返回值都会变成其匹配的第一行的值。

CQL3区分了内置聚合（本地聚合）和用户自定义聚合，内置聚合函数有以下几种：

### Count

用于计数返回的条数。

```CQL
SELECT COUNT(*) FROM plays;
SELECT COUNT(1) FROM plays;
```

也可以用于计数给定列的非空值：

```CQL
SELECT COUNT(scores) FROM plays;
```

### Max 和 Min

计算给定列在返回数据中的最大值、最小值。

```CQL
SELECT MIN(players), MAX(players) FROM plays WHERE game = 'quake';
```

### Sum

计算累加值。

```CQL
SELECT SUM(players) FROM plays;
```

### Avg

计算平均值。

```CQL
SELECT AVG(players) FROM plays;
```

## 用户自定义函数/聚合

暂略。

## JSON 支持

Cassandra 2.2 开始引入了 `SELECT` 和 `INSERT` 语句的 JSON 支持。这种支持并没有从根本上改变 CQL API，只是提供了一种简便的方法去与 JSON 文档协作。

### SELECT JSON

`SELECT` 语句可以使用 `JSON ` 关键字，以单 `JSON` 编码 map 的形式返回每一行数据。其他都与一般的 `SELECT` 语句表现相同。

结果中 map 的键名与查询时的栏名相同，比如  "`SELECT JSON a, ttl(b) FROM ...`"  会返回的 json 键名是 `a` 和 `ttl(b)`。不过这里存在一种特例，为了与 `INSERT JSON` 的表现保持同步，包含大写字符、区分大小写的列名将用双引号括起来。如  "`SELECT JSON myColumn FROM ...`"  会返回的键为  `"\"myColumn\""` 。

map 的值会被表示为 JSON编码（参考下文）。

### INSERT JSON

`INSERT` 语句可以使用 `JSON` 关键字将一个 JSON 编码 map 当作一行数据插入。JSON map 的格式通常应该与同一个表上的 `SELECT JSON` 语句返回的格式相同。对于区分大小写的列名，应使用双引号括住。比如：

```
INSERT INTO mytable JSON '{"\"myKey\"": 0, "value": 0}'
```

JSON map 中省略的任何列都将默认为 `NULL` 值（将导致一个墓碑被创建）。

### Cassandra 数据类型的 JSON 编码

Cassandra 会尽可能地以原生 JSON 表现去展现或接收数据。Cassandra 对于所有单字段类型接受匹配 CQL 文本格式地字符串表现。比如浮点数、整型、UUID、date，都可以使用 CQL 文本字符串来表现。而对于复合类型，如 集合、元组、用户自定义类型等，则一定会被表现为原生 JSON 集合（map 或 list）或者 JSON 编码的集合字符串表现。

下表展示了 Cassandra 可以接受的 `INSERT JSON` 值（包括 `fromJson()` 的参数），以及 Cassandra 在使用 `SELECT JSON` 语句和 `fromJson()` 函数会返回的数据格式。（PS：后面那个应该是 `toJson()` 吧）

| type        | formats accepted       | return format | notes                                                                                                                                                                                                                                                   |
| ----------- | ---------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ascii`     | string                 | string        | Uses JSON’s `\u` character escape                                                                                                                                                                                                                       |
| `bigint`    | integer, string        | integer       | String must be valid 64 bit integer                                                                                                                                                                                                                     |
| `blob`      | string                 | string        | String should be 0x followed by an even number of hex digits                                                                                                                                                                                            |
| `boolean`   | boolean, string        | boolean       | String must be “true” or "false"                                                                                                                                                                                                                        |
| `date`      | string                 | string        | Date in format `YYYY-MM-DD`, timezone UTC                                                                                                                                                                                                               |
| `decimal`   | integer, float, string | float         | May exceed 32 or 64-bit IEEE-754 floating point precision in client-side decoder                                                                                                                                                                        |
| `double`    | integer, float, string | float         | String must be valid integer or float                                                                                                                                                                                                                   |
| `float`     | integer, float, string | float         | String must be valid integer or float                                                                                                                                                                                                                   |
| `inet`      | string                 | string        | IPv4 or IPv6 address                                                                                                                                                                                                                                    |
| `int`       | integer, string        | integer       | String must be valid 32 bit integer                                                                                                                                                                                                                     |
| `list`      | list, string           | list          | Uses JSON’s native list representation                                                                                                                                                                                                                  |
| `map`       | map, string            | map           | Uses JSON’s native map representation                                                                                                                                                                                                                   |
| `smallint`  | integer, string        | integer       | String must be valid 16 bit integer                                                                                                                                                                                                                     |
| `set`       | list, string           | list          | Uses JSON’s native list representation                                                                                                                                                                                                                  |
| `text`      | string                 | string        | Uses JSON’s `\u` character escape                                                                                                                                                                                                                       |
| `time`      | string                 | string        | Time of day in format `HH-MM-SS[.fffffffff]`                                                                                                                                                                                                            |
| `timestamp` | integer, string        | string        | A timestamp. Strings constant are allow to input timestamps as dates, see [Working with dates](https://cassandra.apache.org/doc/old/CQL-3.0.html#usingdates) below for more information. Datestamps with format `YYYY-MM-DD HH:MM:SS.SSS` are returned. |
| `timeuuid`  | string                 | string        | Type 1 UUID. See [Constants](https://cassandra.apache.org/doc/old/CQL-3.0.html#constants) for the UUID format                                                                                                                                           |
| `tinyint`   | integer, string        | integer       | String must be valid 8 bit integer                                                                                                                                                                                                                      |
| `tuple`     | list, string           | list          | Uses JSON’s native list representation                                                                                                                                                                                                                  |
| `UDT`       | map, string            | map           | Uses JSON’s native map representation with field names as keys                                                                                                                                                                                          |
| `uuid`      | string                 | string        | See [Constants](https://cassandra.apache.org/doc/old/CQL-3.0.html#constants) for the UUID format                                                                                                                                                        |
| `varchar`   | string                 | string        | Uses JSON’s `\u` character escape                                                                                                                                                                                                                       |
| `varint`    | integer, string        | integer       | Variable length; may overflow 32 or 64 bit integers in client-side decoder                                                                                                                                                                              |

### fromJson() 函数

 `fromJson()` 与 `INSERT JSON` 类似，但只会返回一列的值。它只能用于 `INSERT` 的 `VALUES` 子句中，或者作为 `UPDATE`、 `DELETE` 或 `SELECT` 语句中的其中一列值。比如，它不能用于 `SELECT` 语句中的 `<select-clause>`子句。

### toJson() 函数

 `toJson()` 与 `SELECT JSON` 类似，但只会返回一列的值。其只应该被用于 `SELECT` 语句的 `<select-clause>`子句当中。

## 附录A：CQL 关键字

CQL 的关键字区分保留和非保留的。保留关键字不可以作为标识符，但可以使用双引号括起来用作标识符。而非保留关键字只在特定的上下文中具有特定的含义，否则可以用作标识符。

| 关键字            | 是否保留？ |
| -------------- | ----- |
| `ADD`          | yes   |
| `AGGREGATE`    | no    |
| `ALL`          | no    |
| `ALLOW`        | yes   |
| `ALTER`        | yes   |
| `AND`          | yes   |
| `APPLY`        | yes   |
| `AS`           | no    |
| `ASC`          | yes   |
| `ASCII`        | no    |
| `AUTHORIZE`    | yes   |
| `BATCH`        | yes   |
| `BEGIN`        | yes   |
| `BIGINT`       | no    |
| `BLOB`         | no    |
| `BOOLEAN`      | no    |
| `BY`           | yes   |
| `CALLED`       | no    |
| `CLUSTERING`   | no    |
| `COLUMNFAMILY` | yes   |
| `COMPACT`      | no    |
| `CONTAINS`     | no    |
| `COUNT`        | no    |
| `COUNTER`      | no    |
| `CREATE`       | yes   |
| `CUSTOM`       | no    |
| `DATE`         | no    |
| `DECIMAL`      | no    |
| `DELETE`       | yes   |
| `DESC`         | yes   |
| `DESCRIBE`     | yes   |
| `DISTINCT`     | no    |
| `DOUBLE`       | no    |
| `DROP`         | yes   |
| `ENTRIES`      | yes   |
| `EXECUTE`      | yes   |
| `EXISTS`       | no    |
| `FILTERING`    | no    |
| `FINALFUNC`    | no    |
| `FLOAT`        | no    |
| `FROM`         | yes   |
| `FROZEN`       | no    |
| `FULL`         | yes   |
| `FUNCTION`     | no    |
| `FUNCTIONS`    | no    |
| `GRANT`        | yes   |
| `IF`           | yes   |
| `IN`           | yes   |
| `INDEX`        | yes   |
| `INET`         | no    |
| `INFINITY`     | yes   |
| `INITCOND`     | no    |
| `INPUT`        | no    |
| `INSERT`       | yes   |
| `INT`          | no    |
| `INTO`         | yes   |
| `JSON`         | no    |
| `KEY`          | no    |
| `KEYS`         | no    |
| `KEYSPACE`     | yes   |
| `KEYSPACES`    | no    |
| `LANGUAGE`     | no    |
| `LIMIT`        | yes   |
| `LIST`         | no    |
| `LOGIN`        | no    |
| `MAP`          | no    |
| `MODIFY`       | yes   |
| `NAN`          | yes   |
| `NOLOGIN`      | no    |
| `NORECURSIVE`  | yes   |
| `NOSUPERUSER`  | no    |
| `NOT`          | yes   |
| `NULL`         | yes   |
| `OF`           | yes   |
| `ON`           | yes   |
| `OPTIONS`      | no    |
| `OR`           | yes   |
| `ORDER`        | yes   |
| `PASSWORD`     | no    |
| `PERMISSION`   | no    |
| `PERMISSIONS`  | no    |
| `PRIMARY`      | yes   |
| `RENAME`       | yes   |
| `REPLACE`      | yes   |
| `RETURNS`      | no    |
| `REVOKE`       | yes   |
| `ROLE`         | no    |
| `ROLES`        | no    |
| `SCHEMA`       | yes   |
| `SELECT`       | yes   |
| `SET`          | yes   |
| `SFUNC`        | no    |
| `SMALLINT`     | no    |
| `STATIC`       | no    |
| `STORAGE`      | no    |
| `STYPE`        | no    |
| `SUPERUSER`    | no    |
| `TABLE`        | yes   |
| `TEXT`         | no    |
| `TIME`         | no    |
| `TIMESTAMP`    | no    |
| `TIMEUUID`     | no    |
| `TINYINT`      | no    |
| `TO`           | yes   |
| `TOKEN`        | yes   |
| `TRIGGER`      | no    |
| `TRUNCATE`     | yes   |
| `TTL`          | no    |
| `TUPLE`        | no    |
| `TYPE`         | no    |
| `UNLOGGED`     | yes   |
| `UPDATE`       | yes   |
| `USE`          | yes   |
| `USER`         | no    |
| `USERS`        | no    |
| `USING`        | yes   |
| `UUID`         | no    |
| `VALUES`       | no    |
| `VARCHAR`      | no    |
| `VARINT`       | no    |
| `WHERE`        | yes   |
| `WITH`         | yes   |
| `WRITETIME`    | no    |

## 附录B：CQL 保留类型

以下类型暂时还没有被 CQL 实际使用，但它们被保留以供未来可能的使用需要。用户自定义类型的名字不应该与保留类型相同。

| type        |
| ----------- |
| `bitstring` |
| `byte`      |
| `complex`   |
| `date`      |
| `enum`      |
| `interval`  |
| `macaddr`   |
