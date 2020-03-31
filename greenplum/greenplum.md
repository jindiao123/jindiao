





# 上篇 基础篇

## 1、简介

### 1.1、OLTP与OLAP

**OLTP**（联机事务处理）系统也称为生产系统，他是事件驱动的，面向应用的，比如电子商务网站的交易系统就是一个典型的OLTP系统

**特点**

```
1、数据在系统中产生
2、基于交易的处理系统
3、没次交易牵涉的数据量很小
4、对响应时间要求非常高
5、用户数据非常庞大，只要是操作人员
6、数据库的各种操作只要基于索引进行
```

**OLAP**(联机分析处理)是基于数据仓库的信息分析处理过程，数据库出库的用户接口部分，OLAP系统是跨部门的，面向主题的

**特点**

```
1、本身不生产数据，其基础数据来源于生产系统中的操作数据
2、基于查询的分许系统
3、复杂查询经常使用多表关联、全表扫描，牵涉到的数据量往往十分庞大
4、相应时间与具体查询有很大关系
5、用户数量相对较小，期用户只要是业务人员与管理人员
6、由于业务问题不固定，数据库的各种操作不能完全基于索引进行
```

### 1.2、特性

1、支持海量数据存储和处理

2、高性价比

3、支持just in time BI

4、系统易用性

5、支持线性扩展

6、较好的并发支持及高可用性支持

7、支持mapreduce

8、数据库内部压缩



### 1.3、建表

```
指定分布建
create table test007(id int unique,name varchar(128)) distributed by(id);
create table test002 as select * from test007 distributed by(id);

默认分布建
create table test008 (like test007);
create table test001 as select * from test007;
不能指定分布建
select * into test003 from test002;
```

插入值：insert into test001 values(1,'jindiao'),(2,'tom'),(3,'li');

删除值：delete：在greenplum 3版本中，如果delete操作涉及子查询，并且子查询的结果还涉及数据重分布，这样的删除语句会报错，在greenplum 4中支持该操作对整张表执行建议采用truncate

truncate：直接删除表的物理文件，然后创建新的数据文件，比delete操作性能上有非常大的提升，当前如果有sql正在操作这张表，那么truncate会被锁住，直到上面的所有所释放

### 1.4、psql查询sql显示执行时间

```
开启：
\timing on

关闭：
\timing off
```

## 2、greenplum是实战

历史拉链

```
create table public.member_fatdt0(
member_id varchar(64),
phoneno varchar(20),
dw_beg_date	date,
dw_end_date date,
dtype char(1),
dw_status char(1),
dw_ins_date date
)with(appendonly=true,compresslevel=5)
distributed by(member_id)
PARTITION by range(dw_end_date)
(
PARTITION p20111201 start (date '2011-12-01') inclusive,
PARTITION p20111202 start (date '2011-12-02') inclusive,
PARTITION p20111203 start (date '2011-12-03') inclusive,
PARTITION p20111204 start (date '2011-12-04') inclusive,
PARTITION p20111205 start (date '2011-12-05') inclusive,
PARTITION p20111206 start (date '2011-12-06') inclusive,
PARTITION p20111207 start (date '2011-12-07') inclusive,
PARTITION p20111231 start (date '2011-12-31') inclusive
END (date '3001-01-01') EXCLUSIVE
);

member_id	varchar(64)  --会员id,
phoneno		varchar(20)  --电话号码,
dw_beg_date	date		  --生效日期,
dw_end_date date		  --失效日期,
dtype,		char(1)	  --类型（历史数据，当前数据）,
dw_status	char(1)		--数据操作类型（I,D,U）,
dw_ins_date date 		--数据仓库插入日期

```

### 2.1、数据分布

由于Greenplum是分布式的架构，为了充分体现分布式架构的优势，我们有必要了解数据是如何分散在各个数据节点上的，有必要了解数据倾斜对数据加载、数据分析、数据导出的影响。

#### 1、数据分散情况查看

我们来简单做个测试，首先，利用generate_series和repeat函数生成一些测试数据，代码如下：

```
create table tedt_distribute_1
as
select a as id,
round(random()) as flag,
repeat('a',1024) as value
from generate_series(1,50000)a;
```

500万数据分散在6个数据节点，利用下面这个SQL可以查询数据的分布情况

```
select gp_segment_id,count(*)
from tedt_distribute_1
group by 1;

 gp_segment_id | count 
---------------+-------
             3 | 12529
             2 | 12294
             0 | 12696
             1 | 12481
(4 rows)
```

**注意**

上述SQL中的group by 1，其中1代表select后面的第一个字段，即gp_segment_id。

#### 2、数据加载速度影响

接下来将通过实验来测试在分布键不同的情况下数据加载的速度。

##### （1）数据倾斜状态下的数据加载

1）测试数据准备，将测试数据导出:

```
copy test_group to '/home/gpadmin/data/test_group.dat'
with delimiter '|';
COPY 50000
```

2）建立测试表，以flag字段为分布键：

```
create table test_distribute_2 as select * 
from tedt_distribute_1 
limit 0 
distributed by(flag);

SELECT 0
```

3）执行数据导入：

```
[gpadmin@centos01 ~]$ time psql -h 192.168.158.111 -d testdb -c "copy test_distribute_2 from stdin with delimiter '|'" < /home/gpadmin/data/test_distribute.dat
COPY 50000

real    0m4.446s
user    0m0.063s
sys     0m0.271s
```

4）由于分布键flag取值只有0和1，因此数据只能分散到两个数据节点，如下：

```
select gp_segment_id,count(*) from test_distribute_2 group by 1;
 gp_segment_id | count 
---------------+-------
             0 | 24850
             2 | 25150
(2 rows)
```

5）由于数据分布在2和3节点，对应Primary Segment在dell3、Mirror节点dell4上，可通过以下SQL查询gp_segment_configuration获得：

```
select dbid, content,role,port,hostname from gp_segment_configuration
where content in(2,3) order by role;

 dbid | content | role | port  | hostname 
------+---------+------+-------+----------
    4 |       2 | m    | 40000 | centos03
    5 |       3 | m    | 40001 | centos03
    9 |       3 | p    | 50001 | centos02
    8 |       2 | p    | 50000 | centos02
(4 rows)
```

##### （2）数据分布均匀状态下的数据加载

1）建立测试表，以id字段为分布键：

```
create table test_distribute_3 as select * from tedt_distribute_1 limit 0 distributed by(id);
SELECT 0
```

2）执行数据导入：

```
[gpadmin@centos01 ~]$ time psql -h 192.168.158.111 -d testdb -c "copy test_distribute_3 from stdin with delimiter '|'" < /home/gpadmin/data/test_distribute.dat
COPY 50000

real    0m6.803s
user    0m0.057s
sys     0m0.051s
```

3）由于分布键id取值顺序分布，因此数据可均匀分散至所有数据节点，如下：

```
testdb=# select gp_segment_id,count(*) from test_distribute_3 group by 1;
 gp_segment_id | count 
---------------+-------
             3 | 12493
             2 | 12421
             0 | 12605
             1 | 12481
(4 rows)
```

#### 3、数据查询速度影响

（1）数据倾斜状态下的数据查询

```
testdb=# select gp_segment_id,count(*),max(length(value)) from test_distribute_2 group by 1;
 gp_segment_id | count | max  
---------------+-------+------
             0 | 24850 | 1024
             2 | 25150 | 1024
(2 rows)

Time: 1404.667 ms
```

（2）数据分布均匀状态下的数据查询

```
testdb=# select gp_segment_id,count(*),max(length(value)) from test_distribute_3 group by 1;
 gp_segment_id | count | max  
---------------+-------+------
             2 | 12421 | 1024
             3 | 12493 | 1024
             0 | 12605 | 1024
             1 | 12481 | 1024
(4 rows)

Time: 788.962 ms
```

### 2.2、数据压缩

#### 1、数据加载速度影响

建表语句如下：

```
testdb=# create table test_compress_1 as select * from tedt_distribute_1 distributed by(flag);
SELECT 50000
```

创建一个压缩表:

```
testdb=# create table test_compress_2 with(appendonly=true,compresslevel=5) as select * from tedt_distribute_1 distributed by(flag);
SELECT 50000
```

#### 2、数据查询速度影响

（1）普通表的数据查询

```
testdb=# select gp_segment_id,count(*),max(length(value)) from test_compress_1 group by 1;
 gp_segment_id | count | max  
---------------+-------+------
             2 | 25150 | 1024
             0 | 24850 | 1024
(2 rows)

Time: 106.383 ms
```

（2）压缩表的数据查询

```
testdb=# select gp_segment_id,count(*),max(length(value)) from test_compress_2 group by 1;
 gp_segment_id | count | max  
---------------+-------+------
             0 | 24850 | 1024
             2 | 25150 | 1024
(2 rows)

Time: 171.553 ms
```

### 2.3、索引

Greenplum支持B-tree、bitmap、函数索引等，在这里我们简单介绍一下B-tree索引：

```
testdb=# create table test_index_1 as select * from tedt_distribute_1;
NOTICE:  Table doesn't have 'DISTRIBUTED BY' clause. Creating a NULL policy entry.
SELECT 50000
Time: 1951.272 ms
testdb=# select id,flag from test_index_1 where id = 100;
 id  | flag 
-----+------
 100 |    0
(1 row)

Time: 75.741 ms
```

接下来我们在flag字段上创建bitmap索引：

```
testdb=# create index test_index_1_idx on test_index_1 (id);
CREATE INDEX
Time: 403.960 ms
```

再次查看执行计划，采用了索引扫描，如下所示。

```
testdb=# explain select id,flag from test_index_1 where id = 100;
                                         QUERY PLAN                                         
--------------------------------------------------------------------------------------------
 Gather Motion 4:1  (slice1; segments: 4)  (cost=0.00..6.00 rows=1 width=12)
   ->  Index Scan using test_index_1_idx on test_index_1  (cost=0.00..6.00 rows=1 width=12)
         Index Cond: (id = 100)
 Optimizer: Pivotal Optimizer (GPORCA) version 3.65.0
(4 rows)

Time: 39.938 ms
testdb=# select id,flag from test_index_1 where id = 100;
 id  | flag 
-----+------
 100 |    0
(1 row)

Time: 12.526 ms
```

建好索引后，再次执行上面的查询语句，有索引的情况下，用了12毫秒，相比未创建索引时403毫秒，有了质的提升。另外，表关联字段上的索引和appen-only压缩表上的索引都能带来较大的性能提升，虽然在数据库应用中，索引的应用场景不多，但是读者仍然可以结合实际的场景来运用索引。

# 中篇 进阶篇

## 3、数据字典讲解

Greenplum是基于PostgreSQL开发的，所以其大部分数据字典是一样的。这些数据字段会根据其特性做一些修改，并不是完全一样的，如pg_class、pg_attribute等。

Greenplum也有自己的一些数据字典，这些数据字典一般是以gp_开头的。在本章中，我们还将介绍一些gp_tookit的工具箱。Greenplum是一个分布式数据库，每个节点都是一个数据库，除了一些集群配置信息的数据字典外，其他所有数据字典master和segment都有，而且这些数据字典的信息与master的大部分是一致的。

### 3.1、oid无处不在

跟PostgreSQL一样，大部分的数据字典都是以oid关联的，oid是一种特殊的数据类型。在PG/GP中，oid都是递增的，每一个表空间、表、索引、数据文件名、函数、约束等都对应有一个唯一标识的oid。oid是全局递增的，可以把oid想象成一个递增的序列（SEQUENCE）。

通过下面的语句可以找到数据字典中带隐藏字段oid的所有表，这些表的oid增加都是共享一个序列的。当然，还有其他的表也是共享这些序列的，比如pg_class的relfilenode。

```
testdb=# select attrelid::regclass,attname
testdb-# from pg_attribute a,pg_class b
testdb-# where a.attrelid=b.oid
testdb-# and b.relnamespace=11
testdb-# and atttypid=26
testdb-# and b.relstorage='h'
testdb-# and attname='oid'
testdb-# and b.relname not like '%index';
        attrelid         | attname 
-------------------------+---------
 pg_proc                 | oid
 pg_type                 | oid
 pg_class                | oid
 pg_attrdef              | oid
 pg_constraint           | oid
 pg_operator             | oid
 pg_opfamily             | oid
 pg_opclass              | oid
 pg_am                   | oid
 pg_amop                 | oid
 pg_amproc               | oid
 pg_language             | oid
 pg_largeobject_metadata | oid
 pg_rewrite              | oid
 pg_trigger              | oid
 pg_event_trigger        | oid
 pg_cast                 | oid
 pg_enum                 | oid
 pg_namespace            | oid
 pg_conversion           | oid
 pg_database             | oid
 pg_tablespace           | oid
 pg_authid               | oid
 pg_ts_config            | oid
 pg_ts_dict              | oid
 pg_ts_parser            | oid
 pg_ts_template          | oid
 pg_extension            | oid
 pg_foreign_data_wrapper | oid
 pg_foreign_server       | oid
 pg_user_mapping         | oid
 pg_default_acl          | oid
 pg_collation            | oid
 pg_resqueue             | oid
 pg_resqueuecapability   | oid
 pg_resourcetype         | oid
 pg_resgroup             | oid
 pg_resgroupcapability   | oid
 pg_extprotocol          | oid
 pg_partition            | oid
 pg_partition_rule       | oid
 pg_compression          | oid
(42 rows)

Time: 45.763 ms
```

将oid转换成这些数据类型，可以简化很多的操作。

![image-20200219153305965](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219153305965.png)

最常用的是regclass，它关联数据字典的oid，使用方法如下：

```
testdb=# select 1259::regclass;
 regclass 
----------
 pg_class
(1 row)

Time: 15.880 ms

# 1259是pg_class对应的oid，这个是默认的，每一个PostgreSQL都是如此。

testdb=# select oid,relname from pg_class where oid='pg_class' ::regclass;
 oid  | relname  
------+----------
 1259 | pg_class
(1 row)

Time: 11.789 ms
```

其他几个类型也是一样的用法，如regproc（regprocedure）是与pg_proc（保存普通函数的命令）关联的。regoper（regoperator）是与pg_operator（操作符）的oid关联的。

```
testdb=# select oid::regoper,oid::regoperator,oid,oprname from pg_operator limit 1;
     oid      |        oid        | oid | oprname 
--------------+-------------------+-----+---------
 pg_catalog.= | =(integer,bigint) |  15 | =
(1 row)

Time: 12.838 ms
```

下面我们先介绍一下Greenplum保存集群配置信息的数据字典。

### 3.2、数据库集群信息

Greenplum的集群配置信息在master上面，这些配置信息对集群管理非常重要，通过这些配置信息可以了解整个集群的状况，可以得知是否有节点失败，通过修改这些配置可以实现集群的扩容等操作。

#### 1、Gp_configuration和gp_segment_configuration

在Greenplum 3.x版本中，集群的配置信息记录在gp_configuration中。表gp_configuration中的字段含义如表4-2所示。

![image-20200219154526957](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219154526957.png)

在Greenplum 4.x版本中，由于引入了文件空间（filespace）的概念，一个节点的数据目录可以是多个，因此将gp_configuration拆分成两个表，gp_segment_configuration（如表4-3所示）和pg_filespace_entry。Greenplum4.x引入了基于文件的数据同步策略，所以也相应地增加了几个数据字典来体现这一个特性。

![image-20200219154747953](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219154747953.png)

```
testdb=# select * from gp_segment_configuration;
 dbid | content | role | preferred_role | mode | status | port  | hostname | address  |                 datadir                 
------+---------+------+----------------+------+--------+-------+----------+----------+-----------------------------------------
    1 |      -1 | p    | p              | n    | u      |  5432 | centos01 | centos01 | /data/greenplum/gpmaster/gpseg-1
    2 |       0 | m    | p              | n    | d      | 40000 | centos02 | centos02 | /data/greenplum/gpdatap/gpdatap1/gpseg0
    6 |       0 | p    | m              | n    | u      | 50000 | centos03 | centos03 | /data/greenplum/gpdatam/gpdatam1/gpseg0
    3 |       1 | m    | p              | n    | d      | 40001 | centos02 | centos02 | /data/greenplum/gpdatap/gpdatap2/gpseg1
    7 |       1 | p    | m              | n    | u      | 50001 | centos03 | centos03 | /data/greenplum/gpdatam/gpdatam2/gpseg1
    4 |       2 | m    | p              | n    | d      | 40000 | centos03 | centos03 | /data/greenplum/gpdatap/gpdatap1/gpseg2
    8 |       2 | p    | m              | n    | u      | 50000 | centos02 | centos02 | /data/greenplum/gpdatam/gpdatam1/gpseg2
    5 |       3 | m    | p              | n    | d      | 40001 | centos03 | centos03 | /data/greenplum/gpdatap/gpdatap2/gpseg3
    9 |       3 | p    | m              | n    | u      | 50001 | centos02 | centos02 | /data/greenplum/gpdatam/gpdatam2/gpseg3
(9 rows)
```

这两张表是在pg_global表空间下面的，是全局的，同一个集群中所有数据库共用的信息。

#### 2、Gp_id

在Greenplum 3.x中，每一个子节点都有gp_id表，这个表记录该节点在集群中的配置信息

![image-20200219155731750](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219155731750.png)

在Greenplum 4.x中，这个gp_id表已经废弃掉，所有子节点的gp_id数据都是一模一样的：

```
testdb=# select * from gp_id;
  gpname   | numsegments | dbid | content 
-----------+-------------+------+---------
 Greenplum |          -1 |   -1 |      -1
(1 row)
```

Greenplum中没有获取hostname的函数，我们可以通过python来创建一个函数（关于自定义函数如何创建

```
在使用plpythonu函数，要先创建语言
testdb=#create language plpythonu;

testdb=# create or replace function public.hostname()
testdb-# returns text
testdb-# as $$
testdb$# import socket
testdb$# return socket.gethostname()
testdb$# $$ language plpythonu;
CREATE FUNCTION
Time: 35.696 ms

testdb=# select hostname();
 hostname 
----------
 centos01
(1 row)

Time: 1.863 ms

```

要获取子节点的信息，还需要借助一个函数—gp_dist_random。对于pg_catalog中的数据字典表，我们都是在主节点上查询的，不会查询子节点上的数据字典，如果想在主节点上查询子节点的数据字典，可以利用gp_dist_random函数。看下面的例子我们就清楚gp_dist_random函数是怎么使用的。

```
testdb=# select gp_segment_id,count(1) from gp_dist_random('pg_class') group by 1 order by 1;
 gp_segment_id | count 
---------------+-------
             0 |   459
             1 |   459
             2 |   459
             3 |   459
(4 rows)

Time: 48.053 ms

# 这样我们就可以获取每个数据字典的大小。
```

同样的，要查看子节点的hostname，就必须从每个子节点都取出一条数据。刚好，gp_id在每个子节点上都只有一条数据，所以，我们可以使用下面的语句进行查询。

```
testdb=# select hostname() from gp_dist_random('gp_id');
 hostname 
----------
 centos03
 centos03
 centos02
 centos02
(4 rows)

Time: 136.568 ms
```

#### 3、Gp_configuration_history

当数据节点失败的时候，GP MASTER通过心跳检测机制检测出Segment失败，就会触发主、备数据节点切换的动作，每一个动作都会记录在gp_configuration_history表中。gp_configuration_history表结构如表4-5所示。

![image-20200219162835196](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219162835196.png)

当数据库发生切换的时候，我们可以通过gp_configuration_history表来了解数据库切换的原因，以及发生切换的时间。

#### 4、pg_filespace_entry

在Greenplum 4.x中，引入了文件空间（filespace）的概念，一个数据库的数据节点可以有多个数据目录，所以数据目录的字段信息从gp_configuration中抽离出来，保存在pg_filespace_entry表中。pg_filespace_entry表结构如表4-6所示。

![image-20200219163045130](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219163045130.png)

### 3.3、常用数据字典

#### 1、pg_class

pg_class可以说是数据字典最重要的一个表了，它保存着所有表、视图、序列、索引的原数据信息，每一个DDL/DML操作都必须跟这个表发生联系，表4-7就是其表结构。

![image-20200219163905115](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219163905115.png)

![image-20200219163942591](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219163942591.png)

```
select * from gp_dist_random('pg_class');
select * from pg_class;
```

权限控制对于一个完善的数据库是必不可少的，对于表、视图来说，pg_class中有一个字段relacl用于保存了权限信息，如下：

```
select relacl from pg_class where relname='tedt_distribute_1';
```

查relacl这个字段有点不方便，不过利用数据库中的很多函数可以方便一些，如下：

```
testdb=# \df *privilege*
                                            List of functions
   Schema   |                Name                | Result data type |    Argument data types     |  Type  
------------+------------------------------------+------------------+----------------------------+--------
 pg_catalog | has_any_column_privilege           | boolean          | name, oid, text            | normal
 pg_catalog | has_any_column_privilege           | boolean          | name, text, text           | normal
 pg_catalog | has_any_column_privilege           | boolean          | oid, oid, text             | normal
 pg_catalog | has_any_column_privilege           | boolean          | oid, text                  | normal
 pg_catalog | has_any_column_privilege           | boolean          | oid, text, text            | normal
 pg_catalog | has_any_column_privilege           | boolean          | text, text                 | normal
 pg_catalog | has_column_privilege               | boolean          | name, oid, smallint, text  | normal
 pg_catalog | has_column_privilege               | boolean          | name, oid, text, text      | normal
 pg_catalog | has_column_privilege               | boolean          | name, text, smallint, text | normal
 pg_catalog | has_column_privilege               | boolean          | name, text, text, text     | normal
 pg_catalog | has_column_privilege               | boolean          | oid, oid, smallint, text   | normal
 ....
```

示例：查询tpchuser用户是否具有访问public.tedt_distribute_1表的select权限。结果为't'表示有这个权限，结果为'f'表示没有这个权限。

```
testdb=# select has_table_privilege('tpchuser','tedt_distribute_1','select');
 has_table_privilege 
---------------------
 t
(1 row)

Time: 19.110 ms
```

#### 2、pg_attribute

![image-20200219165841766](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219165841766.png)

![image-20200219165858481](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219165858481.png)

查看pg_attribute，我们会发现，同一个表在pg_attribute中的记录数会比实际表的字段数多，这是因为表中有很多的隐藏字段，这些隐藏字段如表4-9所示。

![image-20200219165942172](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219165942172.png)

#### 3、gp_distribution_policy

表的分布键保存在gp_distribution_policy表中

![image-20200219170104573](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219170104573.png)

```
testdb=# select * from gp_distribution_policy;
 localoid | policytype | numsegments | distkey | distclass 
----------+------------+-------------+---------+-----------
    16452 | p          |           4 |         | 
    16458 | p          |           4 | 2       | 10022
    16464 | p          |           4 | 1       | 10027
    16470 | p          |           4 | 2       | 10022
    16476 | p          |           4 | 2       | 10022
    16486 | p          |           4 |         | 
(6 rows)

Time: 11.860 ms
```

#### 4、pg_statistic和pg_stats

数据库中表的统计信息保存在pg_statistic中，表中的记录是由ANALYZE创建的，并且随后被查询规划器使用。注意所有统计信息天生都是近似的数值。

这里提到的表上面有一个视图pg_stats，可以方便我们查看pg_statistic的内容。这个视图的数据比pg_statistic好理解，其结构如表4-11所示。

![image-20200219170910011](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219170910011.png)

### 3.4、分区表信息

#### 1、如何实现分区表

分区的意思是把逻辑上的一个大表分割成物理上的几块。Greenplum中分区表的实现基本上是与PostgreSQL中实现的原理一样，都是通过表继承、规则、约束来实现的。

#### 2、pg_partition

一个表是否是分区表保存在pg_partition中，如果一个表是分区表（不包括子分区），则对应有一条记录在这个数据字典中

![image-20200219172647469](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219172647469.png)

如果想查询一个表是否是分区表，只要将pg_partition与pg_class关联，然后执行count即可，如果这个表中有数据，则为分区表，否则不是分区表。

```
testdb=# select count(*) from pg_partition where parrelid='public.tedt_distribute_1'::regclass;
 count 
-------
     0
(1 row)

Time: 3.943 ms
```

#### 3、pg_partition_rule

分区表的分区规则保存在pg_partition_rule中。在这个表中，我们可以找到一个分区表对应的子表有哪些及分区规则等信息

![image-20200219172951040](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219172951040.png)

#### 4、分区表的创建

##### 范围分区（range）

根据分区字段的值范围区间来分区，每一个分区就是一个子表。

一个按日期范围分区的表使用单个date或者timestamp列作为分区键列。如果需要，用户可以使用同一个分区键列来创建子分区，例如按月分区然后按日建子分区。请考虑使用最细的粒度分区。例如，对于一个用日期分区的表，用户可以按日分区并且得到365个每日的分区，而不是先按年分区然后按月建子分区再然后按日建子分区。一种多级设计可能会减少查询规划时间，但是一种平面的分区设计运行得更快。

用户可以通过给出一个START值、一个END值以及一个定义分区增量值的子句让Greenplum数据库自动产生分区。默认情况下，START值总是被包括在内而END值总是被排除在外。例如：

```
testdb=# CREATE TABLE sales (id int, date date, amt decimal(10,2))
testdb-# DISTRIBUTED BY (id)
testdb-# PARTITION BY RANGE (date)
testdb-# ( START (date '2020-01-01') INCLUSIVE
testdb(#    END (date '2020-01-15') EXCLUSIVE
testdb(#    EVERY (INTERVAL '1 day') );

NOTICE:  CREATE TABLE will create partition "sales_1_prt_1" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_2" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_3" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_4" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_5" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_6" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_7" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_8" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_9" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_10" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_11" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_12" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_13" for table "sales"
NOTICE:  CREATE TABLE will create partition "sales_1_prt_14" for table "sales"
CREATE TABLE
Time: 674.040 ms
```

##### 快速分区（every）

根据选定的范围，跨越基数，快速分区每一个子表。

```
testdb=# CREATE TABLE rank (id int, rank int, year int, gender 
testdb(# char(1), count int)
testdb-# DISTRIBUTED BY (id)
testdb-# PARTITION BY RANGE (year)
testdb-# ( START (2016) END (2020) EVERY (1), 
testdb(#   DEFAULT PARTITION extra );

NOTICE:  CREATE TABLE will create partition "rank_1_prt_extra" for table "rank"
NOTICE:  CREATE TABLE will create partition "rank_1_prt_2" for table "rank"
NOTICE:  CREATE TABLE will create partition "rank_1_prt_3" for table "rank"
NOTICE:  CREATE TABLE will create partition "rank_1_prt_4" for table "rank"
NOTICE:  CREATE TABLE will create partition "rank_1_prt_5" for table "rank"
CREATE TABLE
```

**every：指定跨越基数。**

##### 列表分区（list）

根据值的分组，相同的数据归类到一组，也就一个分区中。
一个按列表分区的表可以使用任意允许等值比较的数据类型列作为它的分区键列。一个列表分区也可以用一个多列（组合）分区键，反之一个范围分区只允许单一列作为分区键。对于列表分区，用户必须为每一个用户想要创建的分区（列表值）声明一个分区说明。例如：

```
testdb=# CREATE TABLE rank (id int, rank int, year int, gender 
testdb(# char(1), count int ) 
testdb-# DISTRIBUTED BY (id)
testdb-# PARTITION BY LIST (gender)
testdb-# ( PARTITION girls VALUES ('F'), 
testdb(#   PARTITION boys VALUES ('M'), 
testdb(#   DEFAULT PARTITION other );

NOTICE:  CREATE TABLE will create partition "rank_1_prt_girls" for table "rank"
NOTICE:  CREATE TABLE will create partition "rank_1_prt_boys" for table "rank"
NOTICE:  CREATE TABLE will create partition "rank_1_prt_other" for table "rank"
ERROR:  relation "rank" already exists
```

##### 定义多级分区

用户可以用分区的子分区创建一种多级分区设计。使用一个子分区模板可以确保每一个分区都有相同的子分区设计，包括用户后来增加的分区。例如：

```
CREATE TABLE sales (trans_id int, date date, amount 
decimal(9,2), region text) 
DISTRIBUTED BY (trans_id)
PARTITION BY RANGE (date)
SUBPARTITION BY LIST (region)
SUBPARTITION TEMPLATE
( SUBPARTITION usa VALUES ('usa'), 
  SUBPARTITION asia VALUES ('asia'), 
  SUBPARTITION europe VALUES ('europe'), 
  DEFAULT SUBPARTITION other_regions)
(START (date '2011-01-01') INCLUSIVE
   END (date '2012-01-01') EXCLUSIVE
   EVERY (INTERVAL '1 month'), 
   DEFAULT PARTITION outlying_dates );
```

下面的例子展示了一个三级分区设计，其中 sales表被按照year分区，然后按照 month分区，再然后按照region分区。SUBPARTITION TEMPLATE子句保证每一个年度的分区都有相同的子分区结构。这个例子在该层次的每一个级别上都声明了一个DEFAULT分区。

```
CREATE TABLE p3_sales (id int, year int, month int, day int, 
region text)
DISTRIBUTED BY (id)
PARTITION BY RANGE (year)
    SUBPARTITION BY RANGE (month)
       SUBPARTITION TEMPLATE (
        START (1) END (13) EVERY (1), 
        DEFAULT SUBPARTITION other_months )
           SUBPARTITION BY LIST (region)
             SUBPARTITION TEMPLATE 
             (SUBPARTITION usa VALUES ('usa'),
               SUBPARTITION europe VALUES ('europe'),
               SUBPARTITION asia VALUES ('asia'),
               DEFAULT SUBPARTITION other_regions )
( START (2002) END (2012) EVERY (1), 
  DEFAULT PARTITION outlying_years );
```

##### 对一个现有的表进行分区

表只能在创建时被分区。如果用户有一个表想要分区，用户必须创建一个分过区的表，把原始表的数据载入到新表，再删除原始表并且把分过区的表重命名为原始表的名称。用户还必须重新授权表上的权限。例如：

```
CREATE TABLE sales2 (LIKE sales) 
PARTITION BY RANGE (date)
( START (date 2016-01-01') INCLUSIVE
   END (date '2017-01-01') EXCLUSIVE
   EVERY (INTERVAL '1 month') );
INSERT INTO sales2 SELECT * FROM sales;
DROP TABLE sales;
ALTER TABLE sales2 RENAME TO sales;
GRANT ALL PRIVILEGES ON sales TO admin;
GRANT SELECT ON sales TO guest;
```

#### 5、自定义类型以及类型转换

在Greenplum中，我们经常使用cast函数或::type进行类型转换，究竟哪两种类型之间是可以转换的，哪两种类型之间不能转换，转换的规则是什么，这些都在pg_cast中定义了。

![image-20200219174855482](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219174855482.png)

```
testdb=# select castfunc::regprocedure from pg_cast where castsource='text'::regtype and casttarget ='date'::regtype;
 castfunc 
----------
(0 rows)

testdb=# select '20200202'::date;
    date    
------------
 2020-02-02
(1 row)

testdb=# select date('20200203');
    date    
------------
 2020-02-03
(1 row)
```

可以看出，cast('20110302' as date)和'20110302'::date其实都调用了date('20110302')函数进行类型转换。

#### 6、主、备节点同步的相关数据字典

在Greenplum 4.x中，分别由表4-15中的5张数据字典表来保存基于数据文件的备份信息。这些数据字典都是用于在主节点与备节点间基于文件备份的同步信息。

![image-20200219180358864](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200219180358864.png)

在这几张表中，数据量最大、最重要的表应该是gp_persistent_relation_node和gp_relation_node。这些数据字典在每一个节点中都有，如果主节点和备节点处于完全同步的状态，则主节点和备节点对应的这几张数据字典表的内容应该是一模一样的。

#### 7、数据字典应用示例

##### 1）获取表的字段信息

表名放在pg_class中，schema名放在pg_namespace中，字段信息放在pg_attribute中。一般关联这3张表：

```
select a.attname,pg_catalog.format_type(a.atttypid, a.atttypmod) as date_type
from pg_catalog.pg_attribute a,
(
select c.oid
from pg_catalog.pg_class c
left join pg_catalog.pg_namespace n
on n.oid = c.relnamespace
where c.relname = 'pg_class'
and n.nspname = 'pg_catalog'
) b
where a.attrelid = b.oid
and a.attnum > 0
and not a.attisdropped order by a.attnum;

    attname     | date_type 
----------------+-----------
 relname        | name
 relnamespace   | oid
 reltype        | oid
 reloftype      | oid
 relowner       | oid
 relam          | oid
 relfilenode    | oid
 reltablespace  | oid
 relpages       | integer
 reltuples      | real
 relallvisible  | integer
 reltoastrelid  | oid
 relhasindex    | boolean
 relisshared    | boolean
 relpersistence | "char"
 relkind        | "char"
.....
```

使用regclass就会简化很多：

```
select a.attname,pg_catalog.format_type(a.atttypid, a.atttypmod) as data_type
from pg_catalog.pg_attribute a
where a.attrelid = 'pg_catalog.pg_class'::regclass
and a.attnum > 0
and not a.attisdropped order by a.attnum;
    attname     | data_type 
----------------+-----------
 relname        | name
 relnamespace   | oid
 reltype        | oid
 reloftype      | oid
 relowner       | oid
 relam          | oid
 relfilenode    | oid
 reltablespace  | oid
 relpages       | integer
 reltuples      | real
 relallvisible  | integer
 reltoastrelid  | oid
 relhasindex    | boolean
 relisshared    | boolean
 relpersistence | "char"
.......
```

其实regclass就是一个类型，oid或text到regclass有一个类型转换，与多表关联不一样。在多数据字典表关联的情况下，如果表不存在，会返回空记录，不会报错，如果采用了regclass，则会报错，所以在不确定表是否存在的情况下，慎用regclass。

##### 2）获取表的分布键

gp_distribution_policy是记录分布键信息的数据字典，localoid与pg_class的oid关联。attrnums是一个数组，记录字段的attnum，与pg_attribute中的attnum关联。

```
testdb=# create table cxfa2(a int, b int, c int, d int) distributed by (c, a);
CREATE TABLE
testdb=# select * from gp_distribution_policy where localoid = 'cxfa2'::regclass;
 localoid | policytype | numsegments | distkey |  distclass  
----------+------------+-------------+---------+-------------
    16582 | p          |           4 | 3 1     | 10027 10027
(1 row)
```

这样就可以关联pg_attribute来获取分布键了：

##### 3）获取一个视图的定义

在数据库中，有一个函数（pg_get_viewdef），可以直接获取视图的定义，函数的使用方法如下：

```
testdb=# \df pg_get_viewdef
                               List of functions
   Schema   |      Name      | Result data type | Argument data types |  Type  
------------+----------------+------------------+---------------------+--------
 pg_catalog | pg_get_viewdef | text             | oid                 | normal
 pg_catalog | pg_get_viewdef | text             | oid, boolean        | normal
 pg_catalog | pg_get_viewdef | text             | oid, integer        | normal
 pg_catalog | pg_get_viewdef | text             | text                | normal
 pg_catalog | pg_get_viewdef | text             | text, boolean       | normal
(5 rows)
```

使用这个系统函数可以获取视图的定义，可以传入表的oid或表名，第二个参数表示是否格式化输出，默认不格式化输出。

```
testdb=# create table cxfa( a int) distributed by (a);
CREATE TABLE
testdb=# create view v_cxfa as select * from cxfa;
CREATE VIEW
testdb=# select pg_get_viewdef('v_cxfa', true);
 pg_get_viewdef 
----------------
  SELECT cxfa.a+
    FROM cxfa;
(1 row)
```

其实这个函数是获取数据字典pg_rewrite（存储为表和视图定义的重写规则），将规则重新还原出SQL语句展现给我们。可以通过下面语句去查询数据库保存的重写规则，图4-1是一个简单视图的规则定义。

```
testdb=# select ev_action from pg_rewrite where ev_class='v_cxfa'::regclass;

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 ({QUERY :commandType 1 :querySource 0 :canSetTag true :utilityStmt <> :resultRelation 0 :hasAggs false :hasWindowFuncs false :hasSubLinks false :hasDynamicFunctions false :hasFuncsWithExecRe
strictions false :hasDistinctOn false :hasRecursive false :hasModifyingCTE false :hasForUpdate false :cteList <> :rtable ({RTE :alias {ALIAS :aliasname old :colnames <>} :eref {ALIAS :aliasna
me old :colnames ("a")} :rtekind 0 :relid 16588 :relkind v :lateral false :inh false :inFromCl false :requiredPerms 0 :checkAsUser 0 :selectedCols (b) :modifiedCols (b) :forceDistRandom false
 :securityQuals <>} {RTE :alias {ALIAS :aliasname new :colnames <>} :eref {ALIAS :aliasname new :colnames ("a")} :rtekind 0 :relid 16588 :relkind v :lateral false :inh false :inFromCl false :
requiredPerms 0 :checkAsUser 0 :selectedCols (b) :modifiedCols (b) :forceDistRandom false :securityQuals <>} {RTE :alias <> :eref {ALIAS :aliasname cxfa :colnames ("a")} :rtekind 0 :relid 165
85 :relkind r :lateral false :inh true :inFromCl true :requiredPerms 2 :checkAsUser 0 :selectedCols (b 10) :modifiedCols (b) :forceDistRandom false :securityQuals <>}) :jointree {FROMEXPR :fr
omlist ({RANGETBLREF :rtindex 3}) :quals <>} :targetList ({TARGETENTRY :expr {VAR :varno 3 :varattno 1 :vartype 23 :vartypmod -1 :varcollid 0 :varlevelsup 0 :varnoold 3 :varoattno 1 :location
 29} :resno 1 :resname a :ressortgroupref 0 :resorigtbl 16585 :resorigcol 1 :resjunk false}) :withCheckOptions <> :returningList <> :groupClause <> :havingQual <> :windowClause <> :distinctCl
ause <> :sortClause <> :scatterClause <> :isTableValueSelect false :limitOffset <> :limitCount <> :rowMarks <> :setOperations <> :constraintDeps <> :parentStmtType false})
(1 row)
```

##### 4）查询comment（备注信息）

comment信息是放在表pg_description中的。pg_description表结构如表4-16所示。

![image-20200220100508368](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200220100508368.png)

查询表上的comment信息：

```
testdb=# select coalesce(description,'') as comment from pg_description
testdb-# where
testdb-# objoid='cxfa'::regclass and objsubid=0;
 comment 
---------
(0 rows)
```

查询表中字段的comment信息：

```
testdb=# select b.attname as columname, coalesce(a.description,'') as comment
testdb-# from pg_catalog.pg_description a,pg_catalog.pg_attribute b
testdb-# where objoid='cxfa'::regclass
testdb-# and a.objoid=b.attrelid
testdb-# and a.objsubid=b.attnum;
 columname | comment 
-----------+---------
(0 rows)
```

##### 5）获取数据库建表语句

获取建表语句，这个函数在日常使用和将表迁移到另外一个数据库中是非常有用的。

get_create_sql是用于获取表和视图的DDL语句，不支持外部表，建表语句包括以下内容：

a、字段信息。

b、索引信息。

c、分区信息（主要考虑到性能，目前只支持单一分区键，一层分区）。

d、comment信息。

e、distributed key。

f、是否压缩、列存储、appendonly。

g、只有一个参数，tablename为schemaname.tablename，输出为一个text文本。

```
CREATE or replace FUNCTION public.get_create_sql(tablename text)
      RETURNS text
AS $$
      try:                                                                                                      
            table_name = tablename.lower().split('.')[1]
            table_schema = tablename.lower().split('.')[0]                                                
      except (IndexError):                                    
            return 'Please in put "tableschema.table_name" ' 
#获取表的oid
      get_table_oid = " select oid,reloptions,relkind from pg_class where oid='%s'::regclass"%(tablename)
      try:
            rv_oid = plpy.execute(get_table_oid, 5)
            if not rv_oid: 
                  return 'Did not find any relation named "'+ tablename +'".'
      except (Error):                                   
            return 'Did not find any relation named "'+ tablename +'".'
      table_oid = rv_oid[0]['oid']
      rv_reloptions = rv_oid[0]['reloptions']
      rv_relkind=rv_oid[0]['relkind']
      create_sql="";
      table_kind='table';
#如果该表不是一般表或视图，则报错
      if rv_relkind!='r' and rv_relkind!='v':
            plpy.error('%s is not table or view'%(tablename));
      elif rv_relkind=='v':
            get_view_def="select pg_get_viewdef(%s,'t') as viewdef;"%(table_oid)
            rv_viewdef=plpy.execute(get_view_def);
            create_sql = 'create view %s as \n'%(tablename)
            create_sql += rv_viewdef[0]['viewdef']+'\n';            
            table_kind='view'
      else:
      #get column name and column type –获取字段名、字段类型和默认值
            get_columns = "SELECT a.attname, pg_catalog.format_type(a.atttypid, a.atttypmod),\
                                  (SELECT substring(pg_catalog.pg_get_expr(d.adbin, d.adrelid) for 128) \
                                     FROM pg_catalog.pg_attrdef d WHERE d.adrelid = a.attrelid AND d.adnum = a.attnum AND a.atthasdef) as default,\
                                  a.attnotnull as isnull \
                             FROM pg_catalog.pg_attribute a \
                            WHERE a.attrelid = %s AND a.attnum > 0 AND NOT a.attisdropped \
                         ORDER BY a.attnum;" %(table_oid);
            rv_columns = plpy.execute(get_columns)
      #get distributed key  --获取分布键
            get_table_distribution1 = "SELECT attrnums FROM pg_catalog.gp_distribution_policy t WHERE localoid = '" + table_oid + "' "
            rv_distribution1 = plpy.execute(get_table_distribution1, 500)
            rv_distribution2 = ''
            if rv_distribution1 and rv_distribution1[0]['attrnums']:
                  get_table_distribution2 = "SELECT attname FROM pg_attribute WHERE attrelid = '" + table_oid + "' AND attnum in (" + str(rv_distribution1[0]['attrnums']).strip('{').strip('}').strip('[').strip(']') + ")"
                  rv_distribution2 = plpy.execute(get_table_distribution2, 500)
      #get index define
            create_sql = 'create table %s(\n'%(tablename)
            get_index = "select pg_get_indexdef(indexrelid) AS indexdef from pg_index where indrelid=%s"%(table_oid);
            rv_index =      plpy.execute(get_index);
      #get partition info —获取分区信息
            get_parinfo1 = "select attname as columnname from pg_attribute where attnum = (select paratts[0] from pg_partition where parrelid=%s) and attrelid=%s;"%(table_oid,table_oid);
            get_parinfo2 ="""
            SELECT pp.parrelid,pr1.parchildrelid,
                      CASE
                          WHEN pp.parkind = 'h'::"char" THEN 'hash'::text
                          WHEN pp.parkind = 'r'::"char" THEN 'range'::text
                          WHEN pp.parkind = 'l'::"char" THEN 'list'::text
                          ELSE NULL::text
                      END AS partitiontype,
                      pg_get_partition_rule_def(pr1.oid, true) AS partitionboundary
             FROM  pg_partition pp, pg_partition_rule pr1  
             WHERE pp.paristemplate = false AND pp.parrelid = %s AND pr1.paroid = pp.oid 
             order by pr1.parname;
            """%(table_oid);
            v_par_parent = plpy.execute(get_parinfo1);
            v_par_info = plpy.execute(get_parinfo2);
            max_column_len = 10
            max_type_len = 4
            max_modifiers_len = 4
            max_default_len=4
            for i in rv_columns:
                  if i['attname']: 
                        if max_column_len < i['attname'].__len__():  max_column_len = i['attname'].__len__()
                  if i['format_type']: 
                        if max_type_len < i['format_type'].__len__(): max_type_len = i['format_type'].__len__()
                  if i['default']: 
                        if max_type_len < i['default'].__len__(): max_default_len = i['default'].__len__()
            first = True
            #拼接字段内容
            for i in rv_columns:
                  if first==True:
                        split_char=' ';
                        first=False
                  else :
                        split_char=',';
                  if i['attname']: 
                        create_sql += " " + split_char + i['attname'].ljust(max_column_len +6) + ''
                  else:                                                                                     
                        create_sql += "" + split_char + ' '.ljust(max_column_len + 6) 
                  if i['format_type']:                                          
                        create_sql += ' ' + i['format_type'].ljust(max_type_len + 2) 
                  else:                                                                                     
                        create_sql += ' ' + ' '.ljust(max_type_len + 2) 
                  if i['isnull'] and i['isnull']:                                          
                        create_sql += ' ' + ' not null '.ljust(8) 
                  if i['default']:                                          
                        create_sql += ' default ' + i['default'].ljust(max_default_len + 6)
                  create_sql += "\n"
            create_sql += ")"            
            #拼接with语句的内容
            if rv_reloptions:
                  create_sql+="with("+str(rv_reloptions).strip('{').strip('}').strip('[').strip(']') +")\n"
            if rv_distribution2:
                  create_sql += 'Distributed by ('
                  for i in rv_distribution2:
                        create_sql +=      i['attname'] + ','
                  create_sql = create_sql.strip(',') + ')'
            elif rv_distribution1:
                  create_sql += 'Distributed randomly\n'
            if v_par_parent:
                  partitiontype = v_par_info[0]['partitiontype'];
                  create_sql+='\nPARTITION BY ' + partitiontype + "("+v_par_parent[0]['columnname']+")\n(\n";
                  for i in v_par_info:
                        create_sql+="      "+i['partitionboundary']+',\n';
                  create_sql = create_sql.strip(',\n');
                  create_sql+="\n)"
            create_sql += ";\n\n"
            #拼接索引信息
            for i in rv_index:
                  create_sql += i['indexdef']+';\n'
      #get comment，获取comment信息
      get_table_comment="select 'comment on %s %s is '''|| COALESCE (description,'')||'''' as comment from pg_description where objoid=%s and objsubid=0;"%(table_kind,tablename,table_oid)
      get_column_comment="select 'comment on column %s.'||b.attname ||' is ''' || COALESCE(a.description,'') ||''' ' as comment from pg_catalog.pg_description a,pg_catalog.pg_attribute b where objoid=%s and a.objoid=b.attrelid and a.objsubid=b.attnum;"%(tablename,table_oid)
      rv_table_comment=plpy.execute(get_table_comment);
      rv_column_comment=plpy.execute(get_column_comment);   
      for i in rv_table_comment:
            create_sql += i['comment']+';\n'
      for i in rv_column_comment:
            create_sql += i['comment']+';\n'
      return create_sql;
$$ LANGUAGE plpythonu;
```



## 4、执行计划

### 4.1、入门

#### 4.1.1什么事执行计划

执行计划就是sql的运行步骤

#### 4.1.2、查看执行计划

explain 【analyze】【verbose】statement

explain 查看执行计划

analyze 执行命令并显示实际运行时间

verbose 显示规划树完整的内部表现形式，而不仅是一个摘要

statement 执行的sql语句

### 4.2、执行计划概述

#### 4.2.1、架构

shareNothing架构特点：

每层的数据完全不共享

每个segment只有一部分数据

每一个节点都通过网络连接在一起

#### 4.2.2　重分布与广播

关联数据在不同节点上，对于普通关系型数据库来说，是无法进行连接的。关联的数据需要通过网络流入到一个节点中进行计算，这样就需要发生数据迁移。数据迁移有广播和重分布两种。

图5-2所示很好地展示了Greenplum中重分布数据的实现。

在图5-2中，两个Segment分别进行计算，但由于其中一张表的关联键与分布键不一致，需要关联的数据不在同一个节点上，所以在SLICE1上需要将其中一个表进行重分布，可理解为在每个节点之间互相交换数据。

关于广播与重分布，Greenplum有一个很重要的概念：Slice（切片）。每一个广播或重分布会产生一个切片，每一个切片在每个数据节点上都会对应发起一个进程来处理该Slice负责的数据，上一层负责该Slice的进程会读取下级Slice广播或重分布的数据，然后进行相应的计算。

**注意** 由于在每个Segment上每一个Slice都会发起一个进程来处理，所以在SQL中要严格控制切片的个数，如果重分布或者广播太多，应适当将SQL拆分，避免由于进程太多给数据库或者是机器带来太多的负担。进程太多也比较容易导致SQL失败。

#### 4.2.2、Greenplum Master的工作

Master在SQL的执行过程中承担着很多重要的工作，主要如下：

·执行计划解析及分发。

·将子节点的数据汇集在一起。

·将所有Segment的有序数据进行归并操作（归并排序）。

·聚合函数在Master上进行最后的计算。

·需要有唯一的序列的功能（如开窗函数不带partiton by字句）。

### 4.3　Greenplum执行计划中的术语

#### 4.3.1　数据扫描方式

（1）Seq Scan：顺序扫描

顺序扫描在数据库中，是最常见，也是最简单的一种方式，就是将一个数据文件从头到尾读取一次，这种方式非常符合磁盘的读写特性，顺序读写，吞吐很高。对于分析性的语句，顺序扫描基本上是对全表的所有数据进行分析计算，因此这一种方式非常有效。在数据仓库中，绝大部分都是这种扫描方式，在Greenplum中结合压缩表一起使用，可以减少磁盘IO的损耗。

（2）Index Scan：索引扫描

索引扫描是通过索引来定位数据的，一般对数据进行特定的筛选，筛选后的数据量比较小（对于整个表而言）。使用索引进行筛选，必须事先在筛选的字段上建立索引，查询时先通过索引文件定位到实际数据在数据文件中的位置，再返回数据。

对于磁盘而言，索引扫描都是随机IO，对于查询小数据量而言，速度很快。

（3）Bitmap Heap Scan：位图堆表扫描

当索引定位到的数据在整表中占比较大的时候，通过索引定位到的数据会使用位图的方式对索引字段进行位图堆表扫描，以确定结果数据的准确。对于数据仓库应用而言，很少用这种扫描方式。

通过参数enable_seqscan禁止顺序扫描：set enable_seqscan  = off; 

（4）Tid Scan：通过隐藏字段ctid扫描

ctid是PostgreSQL中标记数据位置的字段，通过这个字段来查找数据，速度非常快，类似于Oracle的rowid。Greenplum是一个分布式数据库，每一个子节点都是一个PostgreSQL数据库，每一个子节点都单独维护自己的一套ctid字段。

就是说，如果想确定到具体一行数据，还必须通过制定另外一个隐藏字段（gp_segment_id）来确定取哪一个数据库的ctid值。

```
Select * from test1 where ctid='(1,1)' and gp_segment_id=1;
```

（5）Subquery Scan'*SELECT*'：子查询扫描

只要SQL中有子查询，需要对子查询的结果做顺序扫描，就会进行子查询扫描。

（6）Function Scan：函数扫描

数据库中有一些函数的返回值是一个结果集，当数据库从这个结果集中取出数据的时候，就会用到这个Function Scan，顺序获取函数返回的结果集（这是函数扫描方式，不属于表扫描方式

#### 4.3.2　分布式执行

（1）Gather Motion（N：1）

聚合操作，在Master上将子节点所有的数据聚合起来。一般的聚合规则是：哪一个子节点的数据先返回到Master上就将该节点的数据先放在MASTER上。

（2）Broadcast Motion（N：N）

广播，将每个Segment上某一个表的数据全部发送给所有Segment。这样每一个Segment都相当于有一份全量数据，广播基本只会出现在两边关联的时候，相关内容再选择广播或者重分布，5.7节中有详细的介绍。

（3）Redistribute Motion（N：N）

当需要做跨库关联或者聚合的时候，当数据不能满足广播的条件，或者广播的消耗过大时，Greenplum就会选择重分布数据，即数据按照新的分布键（关联键）重新打散到每个Segment上，重分布一般在以下三种情况下会发生：

·关联：将每个Segment的数据根据关联键重新计算hash值，并根据Greenplum的路由算法路由到目标子节点中，使关联时属于同一个关联键的数据都在同一个Segment上。

·Group By：当表需要Group By，但是Group By的字段不是分布键时，为了使Group By的字段在同一个库中，Greenplum会分两个Group By操作来执行，首先，在单库上执行一个Group By操作，从而减少需要重分布的数据量；然后将结果数据按照Group By字段重分布，之后再做聚合获得最终结果。

·开窗函数：跟Group By类似，开窗函数（Window Function）的实现也需要将数据重分布到每个节点上进行计算，不过其实现比Group By更复杂一些。

（4）切片（Slice）

Greenplm在实现分布式执行计划的时候，需要将SQL拆分成多个切片（Slice），每一个Slice其实是单库执行的一部分SQL，上面描述的每一个motion都会导致Greenplum多一个Slice操作，而每一个Slice操作子节点都会发起一个进程来处理数据。

#### 4.3.3　两种聚合方式

（1）HashAggregate

对于Hash聚合来说，数据库会根据Group By字段后面的值计算Hash值，并根据前面使用的聚合函数在内存中维护对应的列表，然后数据库会通过这个列表来实现聚合操作，效率相对较高。

（2）GroupAggregate

对于普通聚合函数，使用Group聚合，其原理是先将表中的数据按照Group By的字段排序，这样同一个Group By的值就在一起，只需要对排好序的数据进行一次全扫描就可以得到聚合的结果了。

#### 4.3.4　关联

1.Hash Join

Hash Join（Hash关联）是一种很高效的关联方式，简单地说，其实现原理就是将一张关联表按照关联键在内存中建立哈希表，在关联的时候通过哈希的方式来处理。学过数据结构的读者应该都清楚，哈希表是一种非常高效的数据结构。

2.Hash Left Join

通过Hash Join的方式来实现左连接，在执行计划中的体现就是Hash Left Join：

3.NestedLoop

NestedLoop关联是最简单，也是最低效的关联方式，但是在有些情况下，不得不使用NestedLoop，例如笛卡儿积：

Merge Join也是两表关联中比较常见的关联方式，这种关联方式需要将两张表按照关联键进行排序，然后按照归并排序的方式将数据进行关联，效率比Hash Join差。

下面的例子先通过设置两个参数来强制执行计划，采取的是Merge Join方式：

```
testDB=# set enable_hashjoin =off;
SET
testDB=# set enable_mergejoin =on;
SET
testDB=# explain select * from test1 a join test2 b on a.id=b.id;     
```

5.Merge Full Join

如果关联使用的是full outer join，则执行计划使用的就是Merge Full Join。在Greenplum中其他的关联方式都无法进行全关联。

```
testDB=# explain select * from test1 a full outer join test2 b on a.id=b.id;
```

6.Hash EXISTS Join

关联子查询exist之类的SQL会被改写成inner join，如果SQL被改写了，则会出现Hash EXISTS Join。

```
testDB=# explain select * from test1 a where exists(select 1 from test2 b where a.id=b.id);
```

#### 4.3.5　SQL消耗

在每个SQL的执行计划中，每一步都会有（cost=0.01..0.05 rows=3 width=150）这3项表示SQL的消耗，后面会介绍消耗具体的计算方法，这里先介绍这3个字段的含义。

（1）Cost

以数据库自定义的消耗单位，通过统计信息来估计SQL的消耗。具体消耗的单位可以参考PostgreSQL的官方文档：http://www.pgsqldb.org/pgsqldoc-8.1c/runtime-config-query.html

（2）Rows

根据统计信息估计SQL返回结果集的行数。

（3）Width

返回结果集每一行的长度，这个长度值是根据pg_statistic表中的统计信息来计算的。

建表

```
create table test2(
 id   integer
,col1 numeric
,col2 numeric
,col3 numeric
,col4 numeric
,col5 numeric
,col6 numeric
,col7 numeric
,col8 numeric
,col9 numeric
,col11 varchar(100)
,col12 varchar(100)
,col13 varchar(100)
,col14 varchar(100)
)distributed by(id);
```

插入数据

```
insert into test2 
select generate_series(1,10000),
       (random()*200)::int,
       (random()*800)::int,
       (random()*1600)::int,
       (random()*3200)::int,
       (random()*6400)::int,
       (random()*12800)::int,
       (random()*40000)::int,
       (random()*100000)::int,
       (random()*1000000)::int,
       'hello',
       'welcome',
       'haha',
       'chen';
```

1.开窗函数

对于如下的SQL：

------

```
explain select 
row_number()over(partition by offer_type order by join_from)
,row_number()over(partition by member_id order by gmt_create) 
from offer;
```

### 4.4　案例

#### 4.4.1　关联键强制类型转换，导致重分布

两表的关联键id的类型都是一样的，都是integer类型。如果强制将两个integer类型转换成其他类型，会导致两个表都要重分布。

正常关联的执行计划如下：

```
testdb=# explain select * from test1 a join test2 b on a.id=b.id;  
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Gather Motion 4:1  (slice1; segments: 4)  (cost=0.00..874.99 rows=10000 width=158)
   ->  Hash Join  (cost=0.00..869.70 rows=2500 width=158)
         Hash Cond: (test1.id = test2.id)
         ->  Seq Scan on test1  (cost=0.00..431.15 rows=2500 width=79)
         ->  Hash  (cost=431.15..431.15 rows=2500 width=79)
               ->  Seq Scan on test2  (cost=0.00..431.15 rows=2500 width=79)
 Optimizer: Pivotal Optimizer (GPORCA) version 3.86.0
(7 rows)
```

强制将两个表的执行计划转换成numeric之后的执行计划：

```
testdb=#  explain select * from test1 a join test2 b on a.id::numeric=b.id::numeric; 
                                                QUERY PLAN                                                
----------------------------------------------------------------------------------------------------------
 Gather Motion 4:1  (slice3; segments: 4)  (cost=0.00..876.23 rows=10000 width=158)
   ->  Hash Join  (cost=0.00..870.94 rows=2500 width=158)
         Hash Cond: ((test1.id)::numeric = (test2.id)::numeric)
         ->  Redistribute Motion 4:4  (slice1; segments: 4)  (cost=0.00..432.14 rows=2500 width=79)
               Hash Key: (test1.id)::numeric
               ->  Seq Scan on test1  (cost=0.00..431.15 rows=2500 width=79)
         ->  Hash  (cost=432.14..432.14 rows=2500 width=79)
               ->  Redistribute Motion 4:4  (slice2; segments: 4)  (cost=0.00..432.14 rows=2500 width=79)
                     Hash Key: (test2.id)::numeric
                     ->  Seq Scan on test2  (cost=0.00..431.15 rows=2500 width=79)
 Optimizer: Pivotal Optimizer (GPORCA) version 3.86.0
(11 rows)

```

可以看出，由于两个表刚开始的时候都是按照integer的类型进行分布的，但是关联的时候强制将类型转换成numeric类型，由于integer与numeric的hash值是不一样的，所以数据需要重分布到新的节点进行关联。

#### 4.4.2　统计信息过期

一般的解决办法就是将表重新使用analyze分析一下，重新收集统计信息。或者使用vacuum full analyze对表中的空洞进行回收，从而提高性能。

#### 4.4.3　执行计划出错

有时候统计信息是正确的，但是由于信息不够全面，或者执行的优化器还不够精准，可能会使对结果集大小的估计有很大的偏差。

例如，在SQL中加入了一个无用的条件：id::integer&0=0；

```
testdb=# select count(1) from test1 ;
 count 
-------
 10000
(1 row)

testdb=# select count(1) from test1 where id::integer&0=0;
 count 
-------
 10000
(1 row)
```

以上两个SQL的数据量是一样的，但是执行计划看起来却有很大的区别：

```
testdb=# explain select * from test1 ;
                                    QUERY PLAN                                     
-----------------------------------------------------------------------------------
 Gather Motion 4:1  (slice1; segments: 4)  (cost=0.00..434.16 rows=10000 width=79)
   ->  Seq Scan on test1  (cost=0.00..431.15 rows=2500 width=79)
 Optimizer: Pivotal Optimizer (GPORCA) version 3.86.0
(3 rows)

testdb=# 
testdb=# explain select * from test1 where id::integer&0=0;
                                    QUERY PLAN                                    
----------------------------------------------------------------------------------
 Gather Motion 4:1  (slice1; segments: 4)  (cost=0.00..432.44 rows=4000 width=79)
   ->  Seq Scan on test1  (cost=0.00..431.38 rows=1000 width=79)
         Filter: ((id & 0) = 0)
 Optimizer: Pivotal Optimizer (GPORCA) version 3.86.0
(4 rows)
```

从cost值可以看出，数据库对结果集的估计，一个是4000，一个是10000，相差将近2倍。如果数据库通过这个估计值去判断表是进行广播还是重分布，就有可能会将大表广播，数据量太大，会导致内存占用过高、关联的性能变差，进而会导致执行太慢，或者因为内存不足而导致SQL报错。

#### 4.4.4　分布键选择不恰当

分布键选择不当一般有两种情况：

·随便选择一个字段作为分布键，导致关联的时候需要重分布一个表来关联。

·分布键导致数据分布不均，SQL都卡在一个Segment上进行计算。

对于第一种情况，我们可以通过查询执行计划来得知。当执行计划出现Redistribute Motion或Broadcast Motion时，我们就知道重新分布了数据，这个时候就要留意分布键选择是否有误，进而导致多余的重分布，比如一个表用了字段id来分布，另外一个表通过id和name两个字段来分布，然后通过id来进行关联，这个时候也会导致数据重分布。

第二种情况就比较麻烦，因为在执行计划中，我们看不出SQL有什么问题，往往要到SQL执行非常慢的时候才意识到有问题。在数据分布不均中，有一个特例，就是空值，这是一个比较常见的问题。

**下面将介绍几个方法来判断表是否分布不均。**

1）每个表都有一个隐藏字段gp_segment_id，表示数据是在哪个Segment上的，我们可以对这个字段进行Group By来查看每个节点的数据量。

---

```
testdb=#  select gp_segment_id,count(1) from test1 group by 1 order by 1 ;
 gp_segment_id | count 
---------------+-------
             0 |  2524
             1 |  2544
             2 |  2489
             3 |  2443
(4 rows)
```

2）对于appendonly表，我们还可以通过get_ao_distribution函数来获取数据分布的信息。

------

```
testDB=# select * from get_ao_distribution('test01') order by 1;
 segmentid | tupcount 
-----------+----------
         0 |     3948
         1 |     3576
         2 |     5448
         3 |     4020
         4 |     4740
         5 |     3396
(6 rows)
```

#### 4.4.5　计算distinct

1)第一种是将全部数据按照使用distinct那个字段排序，然后执行一个unique操作去掉重复的数据

```
tpchdb=# explain analyze select distinct l_quantity from lineitem;
                                                                      QUERY PLAN                                                                      
------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather Motion 4:1  (slice2; segments: 4)  (cost=0.00..745.93 rows=50 width=5) (actual time=738.721..738.879 rows=50 loops=1)
   ->  GroupAggregate  (cost=0.00..745.93 rows=13 width=5) (actual time=737.334..737.350 rows=22 loops=1)
         Group Key: l_quantity
         ->  Sort  (cost=0.00..745.93 rows=13 width=5) (actual time=737.321..737.324 rows=88 loops=1)
               Sort Key: l_quantity
               Sort Method:  quicksort  Memory: 132kB
               ->  Redistribute Motion 4:4  (slice1; segments: 4)  (cost=0.00..745.93 rows=13 width=5) (actual time=599.933..737.017 rows=88 loops=1)
                     Hash Key: l_quantity
                     ->  HashAggregate  (cost=0.00..745.93 rows=13 width=5) (actual time=703.443..703.465 rows=50 loops=1)
                           Group Key: l_quantity
                           Extra Text: (seg0)   Hash chain length 1.9 avg, 4 max, using 26 of 32 buckets; total 0 expansions.
 
                           ->  Seq Scan on lineitem  (cost=0.00..549.00 rows=1500304 width=5) (actual time=4.524..417.533 rows=1501915 loops=1)
 Planning time: 3.284 ms
   (slice0)    Executor memory: 6224K bytes.
   (slice1)    Executor memory: 6372K bytes avg x 4 workers, 6372K bytes max (seg0).
   (slice2)    Executor memory: 92K bytes avg x 4 workers, 92K bytes max (seg0).  Work_mem: 65K bytes max.
 Memory used:  128000kB
 Optimizer: Pivotal Optimizer (GPORCA) version 3.86.0
 Execution time: 757.730 ms
(20 rows)
```

2）第二种是按照使用distinct哪个字段来计算hash值，然后放到一个hash数组中，同样的值会得到相同的hash值，从而实现去重的功能

```
tpchdb=# explain analyze select l_quantity from lineitem group by l_quantity;
                                                                       QUERY PLAN                                                                       
--------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather Motion 4:1  (slice2; segments: 4)  (cost=0.00..745.93 rows=50 width=5) (actual time=1126.999..1127.008 rows=50 loops=1)
   ->  GroupAggregate  (cost=0.00..745.93 rows=13 width=5) (actual time=1125.789..1125.817 rows=22 loops=1)
         Group Key: l_quantity
         ->  Sort  (cost=0.00..745.93 rows=13 width=5) (actual time=1125.770..1125.775 rows=88 loops=1)
               Sort Key: l_quantity
               Sort Method:  quicksort  Memory: 132kB
               ->  Redistribute Motion 4:4  (slice1; segments: 4)  (cost=0.00..745.93 rows=13 width=5) (actual time=1078.352..1125.414 rows=88 loops=1)
                     Hash Key: l_quantity
                     ->  HashAggregate  (cost=0.00..745.93 rows=13 width=5) (actual time=1123.857..1123.877 rows=50 loops=1)
                           Group Key: l_quantity
                           Extra Text: (seg0)   Hash chain length 1.9 avg, 4 max, using 26 of 32 buckets; total 0 expansions.
 
                           ->  Seq Scan on lineitem  (cost=0.00..549.00 rows=1500304 width=5) (actual time=3.029..140.956 rows=1501915 loops=1)
 Planning time: 3.608 ms
   (slice0)    Executor memory: 6224K bytes.
   (slice1)    Executor memory: 6372K bytes avg x 4 workers, 6372K bytes max (seg0).
   (slice2)    Executor memory: 92K bytes avg x 4 workers, 92K bytes max (seg0).  Work_mem: 65K bytes max.
 Memory used:  128000kB
 Optimizer: Pivotal Optimizer (GPORCA) version 3.86.0
 Execution time: 1146.088 ms
(20 rows)
```

**在Greenplum 4.3版本中，distinct跟group by两种方式都采用了HashAggregate这种方式，性能上就区别不大了。**

#### 4.4.6　union与union all

注意，如果使用union，会进行去重。在Greenplum中，如果不是分布键，去重的就要涉及数据的重分布，而在Greenplum中则更加特殊，因为这个去重是以整行数据为分布键的，这样分布键很长，一般Union的结果会插入到另外一张表中，又会造成一次数据重分布，效率会较差。

```
testdb=# create table t01 
testdb-# as select * from test_group 
testdb-# distributed by(id); 
SELECT 100000
testdb=# create table t02 
testdb-# as select * from test_group 
testdb-# distributed by(id); 
SELECT 100000
testdb=# 
testdb=# create table t03 
testdb-# as select * from test_group limit 0
testdb-# distributed by(id); 
SELECT 0
testdb=# explain insert into t03 select * from t01 union select * from t02;
                                                                              QUERY PLAN                                                                               
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Insert  (cost=0.00..22849.30 rows=50000 width=79)
   ->  Result  (cost=0.00..974.30 rows=50000 width=112)
         ->  HashAggregate  (cost=0.00..968.70 rows=50000 width=79)
               Group Key: t01.id, t01.col1, t01.col2, t01.col3, t01.col4, t01.col5, t01.col6, t01.col7, t01.col8, t01.col9, t01.col11, t01.col12, t01.col13, t01.col14
               ->  Append  (cost=0.00..876.29 rows=50000 width=79)
                     ->  Seq Scan on t01  (cost=0.00..432.50 rows=25000 width=79)
                     ->  Seq Scan on t02  (cost=0.00..432.50 rows=25000 width=79)
 Optimizer: Pivotal Optimizer (GPORCA) version 3.86.0
(8 rows)
```

使用Union All可能会造成不必要的数据重分布

```
testdb=# create view view01 
testdb-# as 
testdb-# select * from t01 
testdb-# union all 
testdb-# select * from t02;
CREATE VIEW
testdb=# explain insert into t03 
testdb-# select * from view01;
                                 QUERY PLAN                                 
----------------------------------------------------------------------------
 Insert  (cost=0.00..22756.89 rows=50000 width=79)
   ->  Result  (cost=0.00..881.89 rows=50000 width=112)
         ->  Append  (cost=0.00..876.29 rows=50000 width=79)
               ->  Seq Scan on t01  (cost=0.00..432.50 rows=25000 width=79)
               ->  Seq Scan on t02  (cost=0.00..432.50 rows=25000 width=79)
 Optimizer: Pivotal Optimizer (GPORCA) version 3.86.0
(6 rows)
```

表t01和t02的分布键都是id，在使用union all之后，分布键也应该还是id，t03表的分布键也是一样的，当向t03表插入数据的时候，却发生了数据重分布。因此在使用union all的时候要小心这个坑，对于这段SQL，可以去掉union all，分别将两个表的数据单独插入到t03表中，以避免不必要的数据重分布。

#### 4.4.7　子查询not in

```
testdb=# explain analyze select * from test_group where col1 not in (select col2 from test_group);
                                                                       QUERY PLAN                                                                        
---------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather Motion 4:1  (slice2; segments: 4)  (cost=0.00..916.02 rows=40000 width=79) (actual time=299.554..409.905 rows=100000 loops=1)
   ->  Hash Left Anti Semi (Not-In) Join  (cost=0.00..905.45 rows=10000 width=79) (actual time=298.291..320.580 rows=25111 loops=1)
         Hash Cond: (test_group.col1 = test_group_1.col2)
         Extra Text: (seg0)   Hash chain length 100000.0 avg, 100000 max, using 1 of 524288 buckets.
         ->  Seq Scan on test_group  (cost=0.00..432.50 rows=25000 width=79) (actual time=0.017..5.265 rows=25111 loops=1)
         ->  Hash  (cost=439.61..439.61 rows=100000 width=5) (actual time=145.257..145.257 rows=100000 loops=1)
               ->  Broadcast Motion 4:4  (slice1; segments: 4)  (cost=0.00..439.61 rows=100000 width=5) (actual time=0.085..113.505 rows=100000 loops=1)
                     ->  Seq Scan on test_group test_group_1  (cost=0.00..432.50 rows=25000 width=5) (actual time=0.013..5.146 rows=25111 loops=1)
 Planning time: 30.539 ms
   (slice0)    Executor memory: 151K bytes.
   (slice1)    Executor memory: 58K bytes avg x 4 workers, 58K bytes max (seg0).
   (slice2)    Executor memory: 12504K bytes avg x 4 workers, 12504K bytes max (seg0).  Work_mem: 3125K bytes max.
 Memory used:  128000kB
 Optimizer: Pivotal Optimizer (GPORCA) version 3.86.0
 Execution time: 442.691 ms
(15 rows)
```

无论我们怎么调整控制执行计划的参数，对于这种not in的SQL，数据库都无法选择其他的执行计划。这样，SQL都使用笛卡儿积来执行，效率极差。为了避免这种极差的执行计划，我们只能通过改写SQL来实现这种not in的语法。我们使用left join去重后的表关联来实现一样的效果，执行计划如下：

```
testdb=# explain analyze select * from test_group a left join(select col2 from test_group group by col2) b on a.col1=b.col2 where b.col2 is null;
                                                                                   QUERY PLAN                                                                                    
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Gather Motion 4:1  (slice3; segments: 4)  (cost=0.00..913.77 rows=100001 width=84) (actual time=12.624..154.200 rows=100000 loops=1)
   ->  Result  (cost=0.00..885.67 rows=25001 width=84) (actual time=12.218..39.234 rows=25111 loops=1)
         Filter: (test_group_1.col2 IS NULL)
         ->  Hash Left Join  (cost=0.00..884.85 rows=25001 width=84) (actual time=12.216..33.298 rows=25111 loops=1)
               Hash Cond: (test_group.col1 = test_group_1.col2)
               Extra Text: (seg0)   Hash chain length 1.0 avg, 1 max, using 1 of 262144 buckets.Hash chain length 1.0 avg, 1 max, using 1 of 32 buckets; total 0 expansions.
 
               ->  Seq Scan on test_group  (cost=0.00..432.50 rows=25000 width=79) (actual time=0.019..5.390 rows=25111 loops=1)
               ->  Hash  (cost=435.78..435.78 rows=1 width=5) (actual time=11.314..11.314 rows=1 loops=1)
                     ->  Broadcast Motion 4:4  (slice2; segments: 4)  (cost=0.00..435.78 rows=1 width=5) (actual time=9.819..11.287 rows=1 loops=1)
                           ->  GroupAggregate  (cost=0.00..435.78 rows=1 width=5) (actual time=9.136..9.136 rows=1 loops=1)
                                 Group Key: test_group_1.col2
                                 ->  Sort  (cost=0.00..435.78 rows=1 width=5) (actual time=9.124..9.124 rows=4 loops=1)
                                       Sort Key: test_group_1.col2
                                       Sort Method:  quicksort  Memory: 132kB
                                       ->  Redistribute Motion 4:4  (slice1; segments: 4)  (cost=0.00..435.78 rows=1 width=5) (actual time=8.137..8.836 rows=4 loops=1)
                                             Hash Key: test_group_1.col2
                                             ->  HashAggregate  (cost=0.00..435.78 rows=1 width=5) (actual time=7.757..7.776 rows=1 loops=1)
                                                   Group Key: test_group_1.col2
                                                   Extra Text: (seg0)   Hash chain length 1.0 avg, 1 max, using 1 of 32 buckets; total 0 expansions.
 
                                                   ->  Seq Scan on test_group test_group_1  (cost=0.00..432.50 rows=25000 width=5) (actual time=0.016..3.371 rows=25111 loops=1)
 Planning time: 5.356 ms
   (slice0)    Executor memory: 151K bytes.
   (slice1)    Executor memory: 180K bytes avg x 4 workers, 180K bytes max (seg0).
   (slice2)    Executor memory: 92K bytes avg x 4 workers, 92K bytes max (seg0).  Work_mem: 65K bytes max.
   (slice3)    Executor memory: 2216K bytes avg x 4 workers, 2216K bytes max (seg0).  Work_mem: 1K bytes max.
 Memory used:  128000kB
 Optimizer: Pivotal Optimizer (GPORCA) version 3.86.0
 Execution time: 191.602 ms
(30 rows)
```

从SQL中的cost我们看出，这两个实现一样功能的SQL在性能上有极大的差异

```
testdb=# select count(1) from (select * from test_group where col1 not in (select col2 from test_group))t;
 count  
--------
 100000
(1 row)

Time: 136.794 ms
testdb=# select count(1) from (select * from test_group a left join(select col2 from test_group group by col2) b on a.col1=b.col2 where b.col2 is null)t;
 count  
--------
 100000
(1 row)

Time: 34.546 ms
```

#### 4.4.8　聚合函数太多导致内存不足

在Greenplum 4.1数据库中，SQL进行很多的聚合运算时，有时候会报如下的错误：

------

```
Error 7 (ERROR:  Unexpected internal error: Segment process received signal SIGSEGV (postgres.c:3360)  (seg43 slice1 sdw19-4:30003 pid=26345) (cdbdisp.c:1457))
```

------

这段SQL其实就是占用内存太多，进程被操作系统发出信号干扰导致的报错。

查看执行计划，发现是HashAggregate搞的鬼。一般来说，数据库会根据统计信息来选择HashAggregate或GroupAggregate，但是有可能统计信息不够详细或SQL太复杂而选错执行计划。

一般遇到这种问题，有两种方法：

1）拆分成多个SQL来执行，减少HashAggregate使用的内存。

2）在执行SQL之前，先执行enable_hashagg=off；将HashAggregate参数关掉。强制不采用HashAggregate这种聚合方式，则数据库会采用GroupAggregate，虽然增加了排序的代价，但是内存使用量是可控的，建议用这种方式，比较简单。

下次如果再遇到这种内存不足的报错，并且SQL中有很多的聚合函数，建议采用这两种方法改脚本重新运行。

# 中下篇　Greenplum高级应用

本章将介绍一些Greenplum的高级特性，主要是与其他关系型数据库有区别的地方。通过本章的介绍，读者会对Greenplum中针对数据仓库OLAP类型的一些优化有一个更加深入的了解。

当今的数据处理大致可以分成两大类：联机事务处理OLTP（On-Line Transaction Processing）、联机分析处理OLAP（On-Line Analytical Processing）。OLTP是传统的关系型数据库的主要应用，主要是基本的、日常的事务处理，例如银行交易。OLAP是数据仓库系统的主要应用，支持复杂的分析操作，侧重决策支持，并且提供直观易懂的查询结果。表6-1列出了OLTP与OLAP之间的比较。

![image-20200311155816081](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200311155816081.png)

看完了上面OLTP和OLAP在应用上的比较，现在让我们来看一下它们在硬件的使用特性上有哪些明显的区别，如表6-2所示。

![image-20200311155903441](C:\Users\LENOVO\AppData\Roaming\Typora\typora-user-images\image-20200311155903441.png)

### 5.1　Appendonly表与压缩表

对于数据仓库应用来说，由于数据量大，而且每次的分析几乎都是全表扫描，因此磁盘无论在存储上还是在吞吐上，都是一个瓶颈，为了缓解这个问题，Greenplum引入了Appendonly表和压缩表的概念。

#### 5.1.1　应用场景及语法介绍

压缩表必须是Appendonly表。Appendonly表顾名思义，就是只能不断追加的表，不能进行更新和删除。

（1）压缩表的应用场景

1）业务上不需要对表进行更新和删除操作，用truncate+insert就可以实现业务逻辑。

2）访问表的时候基本上是全表扫描，不需要在表上建立索引。

3）不能经常对表进行加字段或修改字段类型，对Appendonly表加字段比普通表慢很多。

（2）语法介绍

建表的时候加上with（appendonly=true）就可以指定表是Appendonly表。如果需要建压缩表，则加上with（appendonly=true，compresslevel=5），其中compresslevel是压缩率，取值为1~9，一般选择5已经足够了。

#### 5.1.2　压缩表的性能差异

​	普通表与压缩表在性能上的差距，由于测试的都是IO密集型的SQL，因此性能差异基本上是与压缩率成正比的。压缩率是跟实际数据类型相关的，在我们的系统中，大部分的表压缩率大概是在3~4之间。

​	一张简单的商品表，总数据量500万条，常见的数据类型有date、numeric、integer、varchar、text等，普通表与压缩表性能比较结果如表6-3所示。

![image-20200311162718309](F:\Typora\resources\app\Docs\img\image-20200311162718309.png)

#### 5.1.3　Appendonly表特性

​	对于每一张Appendonly表，创建的时候都附带创建一张pg_aoseg模式下的表，表名一般是pg_aoseg_+表的oid，这张表中保存了这张Appendonly表的一些原数据信息

**1）首先建一张Appendonly的表：**

```
testdb=# create table test_appendonly with (appendonly=true , compresslevel=5) 
testdb-# as 
testdb-# select generate_series(0,1000) a,'helloworld'::varchar(50) b 
testdb-# distributed by (a);
SELECT 1001
Time: 406.498 ms
```

**2）查一下表的oid：**

```
testdb=# select oid from pg_class where relname='test_appendonly';
  oid  
-------
 25349
(1 row)
testdb=# select oid,oid::regclass from pg_class where relname='test_appendonly' or relname like '%25349%';
  oid  |                oid                
-------+-----------------------------------
 25349 | test_appendonly
 25351 | pg_aoseg.pg_aoseg_25349
 25353 | pg_aoseg.pg_aovisimap_25349
 25355 | pg_aoseg.pg_aovisimap_25349_index
(4 rows)
```

**3）aoseg表的字段信息：**

```
testdb=# \d pg_aoseg.pg_aoseg_25349
?o? "pg_aoseg.pg_aoseg_25349"
     Column      |   Type   
-----------------+----------
 segno           | integer
 eof             | bigint
 tupcount        | bigint
 varblockcount   | bigint
 eofuncompressed | bigint
 modcount        | bigint
 formatversion   | smallint
 state           | smallint
```

**4）这个表的每个字段的含义在Greenplum的文档中没有**

![image-20200311163624882](F:\Typora\resources\app\Docs\img\image-20200311163624882.png)

**5）get_ao_compression_ratio：查询压缩表的压缩率，与使用gp_dist_random查询底层pg_aoseg前缀的表的结果是一样的。**

```
testdb=# select * from get_ao_compression_ratio('test_appendonly');
 get_ao_compression_ratio 
--------------------------
                     5.95
(1 row)

Time: 378.326 ms
testdb=# select sum(eofuncompressed)/sum(eof) as compression_ratio from gp_dist_random('pg_aoseg.pg_aoseg_25349');
 compression_ratio  
--------------------
 5.9454545454545455
(1 row)

Time: 33.798 ms
```

**6）get_ao_distribution：查询每个子节点的数据量。**

```
testdb=# select * from get_ao_distribution('test_appendonly') order by segmentid;
 segmentid | tupcount 
-----------+----------
         0 |      257
         1 |      252
         2 |      252
         3 |      240
(4 rows)
testdb=# select gp_segment_id,tupcount from gp_dist_random('pg_aoseg.pg_aoseg_25349') order by gp_segment_id;
 gp_segment_id | tupcount 
---------------+----------
             0 |      257
             1 |      252
             2 |      252
             3 |      240
(4 rows)

Time: 40.588 ms
```

****

**注意:** Greenplum 4.3之后对Appendonly表进行了优化，以前的Appendonly表只能插入数据，不能够更新和删除数据。Greenplum4.3之后，将Appendonly表改成了Append-Optimized表，用法跟原来的是一样的，建表的时候还是制定appendonly=true。Append-Optimized表可以更新和删除数据。与Heap表一样，Greenplum在Update数据的时候，其实都是将原有的数据标志位删除，然后新增了一条数据，所以当Append-Optimized经过一定时间的更新之后，需要使用vacuum命令对其空间进行回收，用法与Heap表的一样。

### 5.2　列存储

Appendonly表还有一种特殊的形式，那就是列存储。关于列存储的相关介绍，读者可以参考网络上的一些资料。列存储主要是针对数据库优化的一种数据存储类型。

#### 5.2.1　应用场景

列存储一般适用于宽表（即字段非常多的表）。在使用列存储时，同一个字段的数据都连续保存在一个物理文件中，所以列存储的压缩率比普通压缩表的压缩率要高很多，另外在多数字段中筛选其中几个字段时，需要扫描的数据量很小，扫描速度比较快。因此，列存储尤其适合在宽表中对部分字段进行筛选的场景。

#### 5.2.2　数据文件存储特性

列存储在物理上存储，一个列对应一个数据文件，文件名是原文件名加上.n（n是一个数字）的形式来表示的，第一个字段，n=1，第二个字段n=129，以后每增加一个字段，n都会增加128。

每一个数据文件最大是1GB，如果一个字段在一个Segment中的数据量超过了1GB，就要重新生成一个新的文件，每个字段中间剩余的这127个数字，就是预留给字段内容很大，单个文件1GB放不下，拆分成多个文件时，可以使用预留的127个数字为数据文件编号，如xxxx.1是第一个字段的数据文件，如果这个文件达到了1GB，则产生xxxx.2数据文件，继续保存第一个字段的数据。

从这个物理的存储特性就可以很明显地看到列存储的缺点，那就是会产生大量的小文件，假设一个集群中有30个节点，我们建了一张有100个字段宽表的，假设这张表是按照每天分区的一张分布表，保留了一年的数据，那么将会产生的文件数是：

30×100×365=1095000

#### 5.2.3　如何使用列存储

列存储的表必须是appendonly表，创建一个列存储的表只需要在创建表的时候，在with子句中加入appendonly=true，ORIENTATION=column即可。一般appendonly表会结合压缩一起使用，在with子句中增加compresslevel=5启用压缩，如下：

```
testdb=# create table test_column_ao (
testdb(#        id bigint
testdb(#       ,name varchar(128)
testdb(#       ,value varchar(128)) 
testdb-# with (appendonly=true,ORIENTATION=column,compresslevel=5) 
testdb-# distributed by (id);
CREATE TABLE
```

#### 5.2.4　性能比较

#### 测试数据1

纯随机数据，总共有121个字段，1个int字段作为ID，其他字段都是字符串类型，字符串字段的内容都是随机的，值是由md5（random()）生成的，数据非常均匀且随机。数据量为500000，表的压缩属性都是compresslevel=5。比较结果如表6-6所示。

![image-20200318094501176](F:\Typora\resources\app\Docs\img\image-20200318094501176.png)

①完全随机的数据压缩率比较低，使用两种方式的压缩结果差不多。

②当只查询表的数据量的时候，普通压缩表需要全表扫描，但是列存储只需要查询其中一个字段，速度非常快。

③随着查询字段的增加，普通压缩表查询的速度基本上变动不大，因为扫描的数据量没有变化。而列存储则随着查询的字段增加，消耗时间增长明显。

④在测试的时候瓶颈在于磁盘的IO，这是因为测试集群磁盘性能较差。在CPU消耗上，列存储的消耗略大于普通表的消耗。

#### 测试数据2

这是一个实际业务场景的表，主要是保存商品描述信息的表，字段数为98个，数据类型有date、text、varchar、int、numeric，数据量为3100万，表的压缩属性都是compresslevel=5。

![image-20200318094613770](F:\Typora\resources\app\Docs\img\image-20200318094613770.png)

采用这两种数据方式加载数据时都使用外部表，加载时的瓶颈在于文件服务器的网卡，所以使用这两种方式进行数据加载的速度都差不多。

通过表6-7的结果可以看出，测试的结果与测试数据1的结果大致相同，但是还有一些区别，主要是：

①测试的数据是真实的数据，每个字段的内容都会有些相似，当每个字段的数据比较相似时，使用列存储，这样在压缩文件时，可以获得更高的压缩率，节省磁盘的空间占用。

②瓶颈还是磁盘IO，由于列存储有更好的压缩率，所以在查询全部字段的情况下，列存储消耗的时间还是比普通压缩表小很多的，不像测试数据1，由于压缩率差不多，查询所有字段的性能也差不多。

### 5.3　外部表高级应用

#### 6.3.1　外部表实现原理

gpfdist也可以看做一个http服务，当我们启动gpfdist的时候，可以用wget命令去下载gpfdist的文件，将创建外部表命令时使用的url地址中的gpfdist换成http即可，如：

```
wget http://10.20.151.59:8081/inc/1.dat -O 2.dat
```

当查询外部表时，所有的Segment都会连上gpfdist，然后gpfdist将数据随机分发给每个节点。

![image-20200318114828971](F:\Typora\resources\app\Docs\img\image-20200318114828971.png)

**1.gpfdist的工作流程**

1）启动gpfdist，并在Master上建表。表建好之后并没有任何的数据流动，只是定义好了外部表的原数据信息。

2）将外部表插入到一张Greenplum的物理表中，开始导入数据。

3）Segment根据建表时定义的gpfdist url个数，启动相同的并发到gpfdist获取数据，其中每个Segment节点都会连接到gpfdist上获取数据。

4）gpfdist收到Segment的连接并要接收数据时开始读取文件，顺序读取文件，然后将文件拆分成多个块，随机抛给Segment。

5）由于gpfdist并不知道数据库中有多少个Segment，数据是按照哪个分布键拆分的，因此数据是随机发送到每个Segment上的，数据到达Segment的时间基本上是随机的，所以外部表可以看成是一张随机分布的表，将数据插入到物理表的时候，需要进行一次重分布。

6）为了提高性能，数据读取与重分布是同时进行的，当数据重分布完成后，整个数据导入流程结束。

**2.gpfdist最主要的功能**

①负载均衡：每个Segment分配到的数据都是随机的，所以每个节点的负载都非常均衡。

②并发读取，性能高：每台Segment都同时通过网卡到文件服务器获取数据，并发读取，从而获取了比较高的性能。相对于copy命令，数据要通过Master流入，使用外部表就消除了Master这个单点问题。

**3.如何提高gpfdist性能？**

（1）文件服务器

一般文件服务器比较容易出现瓶颈，因为所有的Segment都连接到这个节点上获取数据，所以如果文件服务器是单机的，那么就很容易在磁盘IO和网卡上出现瓶颈。

（2）Segment节点

相对来说，Segment的机器数一般会比文件服务器多很多，而且网卡性能也比较好，因此Segment一般不容易出现瓶颈。当Segment出现瓶颈的时候，可能不只是数据导入出现瓶颈，应该是整个数据库的性能都已经出现了瓶颈。

对于文件服务器，当磁盘IO出现瓶颈的时候，我们可以使用磁盘阵列来提高磁盘的吞吐能力，如果条件允许，还可以采用分布式文件系统来提高整体的性能，例如MooseFS等。对于网卡出现瓶颈，第一种方法是更换网卡为万兆网卡，但是这种方法还需要各环节的网络环境满足条件，如交换机支持等，成本较高，第二种方法就是通过多网卡机制来保证。

如图6-3所示，就是多网卡的一个例子，我们将文件拆分成多个文件，并启动多个gpfdist分别读取不同的文件，然后将gpfdist绑定到不同的网卡上以提高性能。

![image-20200318142214566](F:\Typora\resources\app\Docs\img\image-20200318142214566.png)

在创建表的时候指定多个gpfdist的地址，例如：

------

```
CREATE EXTERNAL TABLE table_name 
      ( column_name data_type [, ...] | LIKE other_table )
      LOCATION ('gpfdist://filehostip1[:port1]/file_pattern1',
      'gpfdist://filehostip2[:port2]/file_pattern2',
      'gpfdist://filehostip3[:port3]/file_pattern3',
      'gpfdist://filehostip4[:port4]/file_pattern4' )
      ......
```

可以在postgresql.conf中修改参数gp_external_max_segs，默认是64，这个参数用于控制同时有多少个Segment连接到gpfdist上。

#### 5.3.2　可写外部表

在语法上可写外部表与普通外部表主要有两个地方不一样：

```
CREATE WRITABLE EXTERNAL TABLE table_name
    ( column_name data_type [, ...] | LIKE other_table )
      LOCATION ('gpfdist://outputhost[:port]/filename' [, ...])
          |    ('gphdfs://hdfs_host[:port]/path')
      FORMAT 'TEXT' 
               [( [DELIMITER [AS] 'delimiter']
               [NULL [AS] 'null string']
               [ESCAPE [AS] 'escape' | 'OFF'] )]
           | 'CSV'
               [([QUOTE [AS] 'quote'] 
               [DELIMITER [AS] 'delimiter']
               [NULL [AS] 'null string']
               [FORCE QUOTE column [, ...]] ]
               [ESCAPE [AS] 'escape'] )]
    [ ENCODING 'write_encoding' ]
    [ DISTRIBUTED BY (column, [ ... ] ) | DISTRIBUTED RANDOMLY ]
```

①可写外部表没有错误表，不能指定允许有多少行数据错误。因为外部表的数据一般是从数据库的一张表中导出的，格式肯定是正确的，一般不会有异常数据。

②可写外部表在建表时可以指定分布键，如果不指定分布键，默认为随机分布（distributed randomly）。

使用可写外部表的注意事项：

①可写外部表不能中断（truncate）。当我们将数据写入可写外部表时，如果可写外部表中途断开了，要想重新运行必须手动将原有的文件删除，否则那一部分数据会重复。

②可写外部表指定的分布键应该与原表的分布键一致，避免多余的数据重分布。

③gpfdist必须使用Greenplum 4.x之后的版本。可写外部表是Greenplum 4.x之后加入的新功能，在其他机器上启动gpfdist，只需要将$GPHOME/bin/gpfdist复制过去即可。



## 6　Greenplum架构介绍

### 6.1、并行和分布式计算

#### 6.1.1、并行计算

并行计算（Parallel computing）一般是指许多指令同时进行的计算模式。相对于串行计算，并行计算可以划分成时间并行和空间并行。时间并行即流水线技术，空间并行使用多个处理器执行并发计算，当前研究的主要是空间的并行问题。空间上的并行导致两类并行机器的产生，即单指令流多数据流（SIMD）和多指令流多数据流（MIMD）。MIMD类的机器又可分为常见的五类：并行向量处理机（PVP）、对称多处理机（SMP）、大规模并行处理机（MPP）、工作站机群（COW）、分布式共享存储处理机（DSM）。我们简单看一下SMP、MPP。

（1）SMP

所谓对称多处理器，是指服务器中多个CPU对称工作，无主次或从属关系，各CPU共享相同的物理内存，每个CPU访问内存中的任何地址所需时间是相同的，因此SMP也被称为一致存储器访问结构（Uniform Memory Access，UMA）。对SMP服务器进行扩展的方式包括增加内存、使用更快的CPU、增加CPU、扩充I/O（槽口数与总线数）以及添加更多的外部设备（通常是磁盘存储）。SMP服务器的主要特征是共享，系统中所有资源（CPU、内存、I/O等）都是共享的。也正是由于这种特征，导致了SMP服务器的主要问题，那就是它的扩展能力非常有限。对于SMP服务器而言，每一个共享的环节都可能造成SMP服务器扩展时的瓶颈，而最受限制的则是内存。由于每个CPU必须通过相同的内存总线访问相同的内存资源，因此随着CPU数量的增加，内存访问冲突将迅速增加，最终会造成CPU资源的浪费，使CPU性能的有效性大大降低。实验证明，SMP服务器CPU利用率最好的情况是CPU为2~4个。

（2）MPP

MPP由多个SMP服务器通过一定的节点互联网络进行连接，协同工作，完成相同的任务，从用户的角度来看是一个服务器系统。其基本特征是由多个SMP服务器（每个SMP服务器被称做节点）通过节点互联网络连接而成，每个节点只访问自己的本地资源（内存、存储等），是一种完全无共享（Shared-Nothing）结构，因而扩展能力最好，理论上其扩展无限制。在MPP系统中，每个SMP节点也可以运行自己的操作系统、数据库等。但和NUMA不同的是，它不存在异地内存访问的问题。换言之，每个节点内的CPU不能访问另一个节点的内存。节点之间的信息交互是通过节点互联网络实现的，这个过程一般称为数据重分配（Data Redistribution）。但是MPP服务器需要一种复杂的机制来调度和平衡各个节点的负载和并行处理过程。目前一些基于MPP技术的服务器往往通过系统级软件（如数据库）来屏蔽这种复杂性。

#### 6.1.2、分布式计算

分布式系统（distributed system）是建立在网络之上的软件系统。分布式系统具有高度的内聚性和透明性。因此，网络和分布式系统之间的区别更多的在于高层软件（特别是操作系统），而不是硬件。

内聚性是指每一个数据库分布节点高度自治，有本地的数据库管理系统。透明性是指每一个数据库分布节点对用户的应用来说都是透明的，看不出是本地还是远程。在分布式数据库系统中，用户感觉不到数据是分布的，即用户无须知道关系是否分割、有无副本、数据存于哪个站点以及事务在哪个站点上执行等。

分布式计算就是研究如何把一个需要非常巨大的计算能力才能解决的问题分成许多小的部分，然后把这些部分分配给许多计算机进行处理，最后把这些计算结果综合起来得到最终的结果。

### 6.2　并行数据库

并行数据库要求尽可能并行执行所有的数据库操作，从而在整体上提高数据库系统的性能。根据所在计算机的处理器（Processor）、内存（Memory）及存储设备（Storage）的相互关系，并行数据库可以归纳为三种基本的体系结构（这也是并行计算的三种基本体系结构），即共享内存结构（Shared-Memory）、共享磁盘结构（Shared-Disk）和无共享资源结构（Shared-Nothing）。

（1）Shared-Memory结构

该结构包括多个处理器、一个全局共享的内存（主存储器）和多个磁盘存储，各个处理器通过高速通信网络（Interconnection Network）与共享内存连接，并均可直接访问系统中的一个、多个或全部的磁盘存储，在系统中，所有的内存和磁盘存储均由多个处理器共享。在并行数据库领域，Shared-Memory结构很少被使用。

（2）Shared-Disk结构

该结构由多个具有独立内存（主存储器）的处理器和多个磁盘存储构成，各个处理器相互之间没有任何直接的信息和数据的交换，多个处理器和磁盘存储由高速通信网络连接，每个处理器都可以读写全部的磁盘存储。Shared-Disk结构的典型代表是Oracle集群。

（3）Shared-Nothing结构

该结构由多个完全独立的处理节点构成，每个处理节点具有自己独立的处理器、独立的内存（主存储器）和独立的磁盘存储，多个处理节点在处理器由高速通信网络连接，系统中的各个处理器使用自己的内存独立地处理自己的数据。

在这种结构中，每一个处理节点就是一个小型的数据库系统，多个节点一起构成整个的分布式的并行数据库系统。由于每个处理器使用自己的资源处理自己的数据，不存在内存和磁盘的争用，从而提高了整体性能。另外这种结构具有优良的可扩展性，只需增加额外的处理节点，就可以以接近线性的比例增加系统的处理能力。Shared-Nothing结构的典型代表是Teradata、Vertica、Greenplum、Aster Data、IBM DB2和MySQL的集群也使用了这种结构。

### 6.3　Greenplum架构分析

Greenplum是一种基于PostgreSQL（开源数据库）的分布式数据库，其采用的Shared-Nothing架构（MPP）、主机、操作系统、内存、存储都是自我控制的，不存在共享。Greenplum架构主要由Master Host、Segment Host、Interconnect三大部分组成，如图7-1所示。

![image-20200323154648915](F:\Typora\resources\app\Docs\img\image-20200323154648915.png)

（1）Master Host

Master Host是Greenplum数据库系统的入口，它接受客户端的连接请求、负责权限认证、处理SQL命令（SQL的解析并形成执行计划）、分发执行计划、汇总Segment的执行结果、返回执行结果给客户端。由于Greenplum数据库是基于PostgresSQL数据库的，终端用户通过Master同数据库交互就如同操作一个普通的PostgresSQL数据库。用户可以使用PostgresSQL或者JDBC、ODBC等应用程序接口连接数据库。Greenplum Master不存储业务数据，仅存储数据字典。

（2）Segment Host

Segment Host负责业务数据的存储和存取、用户查询SQL的执行。Greenplum数据库的性能由一组Segment服务中最慢的Segment决定，因此要确保基本的运行Greenplum数据的硬件与操作系统在同一个性能级别，同样建议在Greenplum数据系统中的所有的Segment机器有一样的资源与配置。

（3）Interconnect

Interconnect是Greenplum数据库的网络层，在每个Segment中起到一个IPC的作用（Inter-Process Communication）。Greenplum数据库推荐使用标准的千兆以太网交换机来做Interconnect。在默认情况下，Interconnect使用的是UDP协议来进行传输，因为在Greenplum的软件当中，没有其他包去检查和验证UDP，所以UDP协议在可靠性上等同于TCP协议，并且超过了TCP的性能和可扩展性，而且使用TCP协议会有一个限制，最大只能使用1000个Segment实例。

### 6.4　冗余与故障切换

Greenplum数据库配置了镜像节点之后，当主节点不可用时会自动切换至镜像节点，集群仍然保持可用状态。当主节点恢复并启动之后，主节点会自动恢复期间的变更。只要Master不能连接上Segment实例时，就会在系统表中将此实例标识为不可用，并用镜像节点来代替。当然如果在配置服务器集群时，没有开启镜像功能，任何一个Segment实例不可用，整个集群将变得不可用，大大降低集群可用性。镜像节点一般需要和主节点位于不同服务器上，可以为Master节点配置镜像，确保系统的变更信息不会丢失，提升系统的健壮性，另外，我们还需要从网络配置上确保节点之间的网络交互的高可用。

### 6.5　数据分布及负载均衡

Greenplum是一个分布式数据库系统，故其所有的业务数据都是物理存放在集群的所有Segment实例数据库上。这些看似独立的PostgreSQL数据库通过网络相互连接，并和Master节点协同构成整个数据库集群。在学习和使用Greenplum数据库时，我们必须理解数据在集群中是如何存放的。

![image-20200323160852956](F:\Typora\resources\app\Docs\img\image-20200323160852956.png)

在Greenplum数据库中所有表都是分布式的，所以每一张表都会被切片，每个Segment实例数据库会存放相应的数据片段。切片规则可由用户定义，可选的方案有根据用户对每一张表指定的Hash Key进行Hash分片或者选择随机分片。图7-4中的业务场景在Greenplum数据库中的存放规则如图7-5所示。sale、customer、product、vendor四张表的数据都会切片存放在所有的Segment上，当我们需要进行数据分析时，所有Segment实例同时工作，由于每个Segment只需要计算一部分数据，所以计算效率将会大大提升。这正是Greenplum数据库分布式计算提升性能的关键所在。

![image-20200323160943194](F:\Typora\resources\app\Docs\img\image-20200323160943194.png)

（1）Hash Distribution

当选择Hash Distribution策略时，可指定表的一列或者多列组合。Greenplum会根据指定的Hash Key列计算每一行数据对应的Hash值，并映射至相应的Segment实例。当选择的Hash Key列的值唯一时，数据将会均匀地分散至所有Segment实例。Greenplum数据库默认会采用Hash Distribution，如果创建表时未指定Distributed Key，则会选择Primary Key作为Distributed Key，如果Primary Key也不存在，就会选择表的第一列作为Distributed Key。

（2）Random Distribution

当选择Random Distribution时，数据将会随机分配至Segment，相同值的数据行不一定会分发至同一个Segment。虽然Random Distribution策略可以确保数据平均分散至所有Segment，但是在进行表关联分析时，仍然会按照关联键重分布数据，所以Random Distribution策略通常不是一个明智的选择（除非你的SQL只有对单表进行全局的聚合操作，即没有group by或者join等需要数据重分布的操作，此时这种分布模式可以避免数据倾斜，而且性能更高）。

### 6.6　跨库关联

Greenplum数据库将表数据分散至所有Segment实例，当需要进行表关联分析时，由于各个表的Distributed Key不同，相同值的行数据可能分布在不同服务器的不同Segment实例，因此不可避免需要在不同Segment间移动数据才能完成Join操作。跨库关联也正是分布式数据库的难点之一。

#### （1）Join操作的两个表的Distributed Key即Join Key

由于Join Key即为两个表的Distributed Key，故两个表关联的行本身就在本地数据库（即同一个Segment实例），直接关联即可。在这种情况下，性能也是最佳的。我们在进行模型设计时，尽可能将经常关联的字段且唯一的字段设置为Distributed Key，

#### （2）Join操作的两个表中的一个Distributed Key与Join Key相同

由于其中一个表的Join Key和Distributed Key不一致，故两个表关联的行不在同一个数据库中，便无法完成Join操作。在这种情况下就不可避免地需要数据跨节点移动，将关联的行组织在同一个Segment实例，最终完成Join操作。Greenplum可以选择两种方式将关联的行组织在同一个Segment中，其中一个方式是将Join Key和Distributed Key不一致的表按照关联字段重分布（即Redistribute Motion），另一种方式是可以将Join Key和Distributed Key不一致的表在每个Segment广播（即Broadcast Motion），也就是每个Segment都复制一份全量，

#### （3）Join操作的两个表的Distributed Key和Join Key都不同

由于两个表的Join Key和Distributed Key都不一致，故两个表关联的行不在同一个数据库中，便无法完成Join操作。同样在这种情况下，一种方式将两个表都按照关联字段重分布（即Redistribute Motion），另一种方式可以将其中一个表在每个Segment广播（即Broadcast Motion），也就是每个Segment都复制一份全量，

### 6.7　分布式事务

分布式事务处理是指一个事务可能涉及多个数据库操作，而分布式的关键在于两阶段提交（Two Phase Commit，2PC）。两阶段提交用于确保所有分布式事务能够同时提交或者回滚，以便数据库能够处于一致性状态。

分布式事务处理的关键是必须有一种方法可以知道事务在任何地方所做的所有动作，提交或回滚事务必须产生一致的结果（全部提交或全部回滚），所以就需要有一个事务协调器来负责处理每一个数据库的事务。在Greenplum中，Master就充当了这样一个角色。

两阶段提交顾名思义把事务提交分成两个阶段：

第一阶段，Master向每个Segment发出“准备提交”请求，数据库收到请求后执行相同的数据修改和日志记录等处理，处理完成后只是把事务的状态改成“可以提交”，然后把执行的结果返回给Master。

第二阶段，Master收到回应后进入第二阶段，如果所有Segment都返回成功，那么Master向所有的Segment发出“确认提交”请求，数据库服务器把事务的“可以提交”状态改为“提交完成”状态，这个事务就算正常完成了

如果在第一阶段内有Segment执行发生了错误，Master收不到Segment回应或者Segment返回失败，则认为事务失败，回撤事务，Segment收到Rollback的命令，即将当前事务全部回滚

两阶段提交并不能保证数据一定会恢复到一致性状态。例如，当Master向Segment发出提交命令的时候，在提交过程中，有一个Segment失败了，但是其他Segment已经提交成功了，那么这个事务是不能再次回滚的，这样就会造成不一致的情况。

两阶段提交的核心思想就是将可能发生不一致的时间降低到最小，因为提交过程对数据库来说应该是一个瞬间完成的动作，而且发生错误的概率极小，危险期比较短。

查看事务：

```
testdb=# select * from gp_distributed_xacts;
 distributed_xid | distributed_id | state | gp_session_id | xmin_distributed_snapshot
-----------------+----------------+-------+---------------+---------------------------
               0 |                | None  |            -1 |                         0
               0 |                | None  |            16 |                         9
               0 |                | None  |             1 |                         0
(3 rows)

testdb=# begin;
BEGIN
testdb=# select * from gp_distributed_xacts;
 distributed_xid |    distributed_id     |       state        | gp_session_id | xmin_distributed_snapshot
-----------------+-----------------------+--------------------+---------------+---------------------------
               0 |                       | None               |            -1 |                         0
             216 | 1584935076-0000000216 | Active Distributed |            16 |                         9
               0 |                       | None               |             1 |                         0
(3 rows)

testdb=# create table test_trans(a int) distributed by(a);
CREATE TABLE
testdb=# end;
COMMIT
```

再看一下每个Segment的状态

```
testdb=# select dbid,distributed_xid,distributed_id,status from gp_dist_random('gp_distributed_log') where distributed_id = '1584935076-0000000216';
 dbid | distributed_xid |    distributed_id     |  status
------+-----------------+-----------------------+-----------
    2 |             216 | 1584935076-0000000216 | Committed
    3 |             216 | 1584935076-0000000216 | Committed
    4 |             216 | 1584935076-0000000216 | Committed
    5 |             216 | 1584935076-0000000216 | Committed
(4 rows)
```

# 下篇：管理篇

## 7　Greenplum线上环境部署

### 7.1　服务器硬件选型

数据库服务器硬件选型应该遵循以下几个原则。

（1）高性能原则

保证所选购的服务器，不仅能够满足现有应用的需要，而且能够满足一定时期内业务量增长的需要。对于数据库而言，数据库性能依赖于硬件的性能和各种硬件资源的均衡，CPU、内存、磁盘、网络这几个关键组件在系统中都很关键，如果过分突出某一方面硬件资源则会造成大量的资源浪费，而任何一个硬件性能差都会成为系统的瓶颈，故在硬件选择的时候，需要根据应用需求在预算中做到平衡。

（2）可靠性原则

不仅要考虑服务器单个节点的可靠性或稳定性，而且要考虑服务器与相关辅助系统之间连接的整体可靠性。

（3）可扩展性原则

需要服务器能够在相应时间根据业务发展的需要对其自身进行相应的升级，如CPU型号升级、内存扩大、硬盘扩大、更换网卡等。

（4）可管理性原则

需要服务器的软硬件对标准的管理系统提供支持。

#### 7.1.1　CPU

目前，在做CPU选择方案时，主要考虑以下两个方面。

·Intel还是AMD

·更快的处理器还是更多的处理器

Intel作为单核运算速度方面的领导者，其CPU及相关组件相对较贵，而AMD在多核处理技术方面表现优异，仍不失为一个较好的选择。

如果仅有较少进程运行在单个处理器上，则可以考虑配备更快的CPU。相反，如果大量并发进程运行在所有处理器上，则考虑配备更多的核。

一台服务器上CPU的数量决定部署到机器上的Greenplum数据库Segment实例的数量，比如一个CPU配置一个主Segment。

#### 7.1.2　内存

内存是计算机中重要的部件之一，它是与CPU进行沟通的桥梁。计算机中所有程序的运行都是在内存中进行的，因此内存的性能对计算机的影响非常大。内存的选择总体上来说是越大越好，不过当内存足以容纳所有业务数据时，则必须提升CPU性能才能获得性能的提升。

在Greenplum中，内存主要用于在SQL执行过程中保存中间结果（如排序、HashJoin、数据广播等），如果内存不够，Greenplum会选择使用磁盘来缓存数据，这样就会大大降低SQL执行的性能。

另外，对于处理像数据库这样的海量数据，并且内存远小于数据存储的情况，如果内存已经够用，通过增加内存来获取性能的提升则非常细微。对于Greenplum这种数据库系统，磁盘的吞吐量决定着Greenplum的性能，磁盘吞吐越高性能越好。

#### 7.1.3　磁盘及硬盘接口

（1）SATA

全称Serial Advanced Technology Attachment（串行高级技术附件，一种基于行业标准的串行硬件驱动器接口），是由Intel、IBM、Dell、APT、Maxtor和Seagate公司共同提出的硬盘接口规范。

（2）SAS

全称Serial Attached SCSI（串行连接SCSI），缩写为SAS，是新一代的SCSI技术。和现在流行的Serial ATA（SATA）硬盘相同，都采用串行技术以获得更高的传输速度，并通过缩短连接线改善内部空间等。

（3）SSD

全称Solid State Disk，即固态硬盘。目前的硬盘（ATA或SATA）都是磁碟型的，数据就储存在磁碟扇区里，固态硬盘数据就储存在芯片里。

在选择硬盘和硬盘控制器时，确认选择的硬盘控制器能够包括硬盘带宽之和。假如你有20个70Mb/s内部带宽的硬盘，为了获得硬盘最佳性能，需要最少支持1.4Gb/s的硬盘控制器。

要提升服务器I/O吞吐量、可用性及存储容量，常见的方法是做RAID，即独立冗余磁盘阵列（Redundant Array of Independent Disk，RAID）。几种常见的RAID技术如下。

（1）RAID 0

从严格意义上说，RAID 0不是RAID，因为它没有数据冗余和校验。RAID 0技术只是实现了条带化，具有很高的数据传输率，最高的存储利用率，但是RAID中硬盘数越多，安全性越低。

（2）RAID 1

通常称为RAID镜像。RAID 1主要是通过数据镜像实现数据冗余，在两对分离的磁盘上产生互为备份的数据，因此RAID 1具有很高的安全性。但是RAID 1空间利用率低，磁盘控制器负载大，因此只有当系统需要极高的可靠性时，才选择RAID 1。

（3）RAID 1+0

RAID 0+1至少需要4块硬盘才可以实现，不过它综合了RAID 0和RAID 1的特点，独立磁盘配置成RAID 0，两套完整的RAID 0互相镜像。它的读写性能出色，安全性也较高。但是，构建RAID 0+1阵列的成本投入大，数据空间利用率只有50%，因此还不能称为经济高效的方案。

（4）RAID 5

RAID 5是目前应用最广泛的RAID技术。各块独立硬盘进行条带化分割，相同的条带区进行奇偶校验（异或运算），校验数据平均分布在每块硬盘上。RAID 5具有数据安全、读写速度快、空间利用率高等优点，应用非常广泛。但不足之处是，如果1块硬盘出现故障，整个系统的性能将大大降低。

#### 7.1.4　网络

Greenplum数据库互联的性能与Segment上网卡的网络负载有关，所以Greenplum服务器一般由一组带有多个网卡的硬件组成。为了达到最好性能，Greenplum建议为Segment机器上的每一个主Segment配置一个千兆网卡，或者配置每台机器都有万兆网卡。

如果Greenplum数据库网络集群中有多个网络交换机，那么交换机之间均衡地分配子网较为理想，比如每个机器上的网卡1和2用其中一个交换机，网卡3和4使用另一个交换机。

### 7.2　服务器系统参数调整

对于不同的应用场景，每一种硬件都有不同的参数配置，通过参数调整，可以极大地提高读写性能。例如，对于OLTP数据库而言，关注的是磁盘的随机读写，提高单次读写的性能；而对于OLAP数据库而言，最关注的是磁盘的顺序读写，重点在于提高每次磁盘读写的吞吐量。

（1）共享内存

Greenplum只有在操作系统中内存大小配置适当的时候才能正常工作。大部分操作系统的共享内存设置太小，不适合Greenplum的场景，因为这样可以避免进程因为内存占用过高被操作系统停止。

（2）网络

Greenplum的数据分布在各个节点上，因此在计算过程中经常需要将数据移动到其他节点上进行计算，这时，合理的网络配置就显得格外的重要。

（3）系统对用户的限制

操作系统在默认情况下会对用户进行一些资源的限制，以避免某个用户占用太多资源导致其他用户资源不可用。对于数据库来说，只会有一个操作系统用户，这些限制都必须取消掉，例如Greenplum会同时打开很多文件句柄，而操作系统默认的文件句柄数一般很小。

### 7.4　数据库参数介绍

数据库优化主要从两个方面着手。一方面是提升CPU、内存、磁盘、网络等集群服务器的硬件配置，另一方面是优化提交到数据库的语句。在这里我们简单介绍一下影响数据库性能的参数及其他常用的配置参数。

（1）shared_buffers

数据距离CPU越近效率就越高，而离CPU由近到远的主要设备有寄存器、CPU cache、RAM、Disk Drives等。CPU的寄存器和cache是没办法直接优化的，为了避免磁盘访问，只能尽可能将更多有用信息存放在RAM中。Greenplum数据库的RAM主要用于存放如下信息。

·执行程序。

·程序数据和堆栈。

·postgreSQL shared buffer cache。

·kernel disk buffer cache。

·kernel。

因此最大化地保持数据库信息在内存中而不影响其他区域才是最佳的调优方式，但这常常不是一件容易的事情。

PostgreSQL并非直接在磁盘上进行数据修改，而是将数据读入shared buffercache，进而PostgreSQL后台进程修改cache中的数据块，最终再写回磁盘。后台进程如果在cache中找到相关数据，则直接进行操作，如果没找到，则需要从kernel disk buffer cache或者磁盘中读入。PostgreSQL默认的shared buffer较小，将此cache调大则可降低昂贵的磁盘访问。但是前面提到，修改此参数时一定要避免swap发生，因为内存不仅仅用于shared buffer cache。刚开始可以设置一个较小的值，比如总内存的15%，然后逐渐增加，过程中监控性能提升和swap的情况。

（2）effective_cache_size

设置优化器假设磁盘高速缓存的大小用于查询语句的执行计划判断，主要用于判断使用索引的成本，此参数越大越有机会选择索引扫描，越小越倾向于选择顺序扫描，此参数只会影响执行计划的选择。

（3）work_mem

当PostgreSQL对大表进行排序时，数据库会按照此参数指定大小进行分片排序，将中间结果存放在临时文件中，这些中间结果的临时文件最终会再次合并排序，所以增加此参数可以减少临时文件个数进而提升排序效率。当然如果设置过大，会导致swap的发生，所以设置此参数时仍然需要谨慎。同样刚开始仍可设置为总内存的5%。

（4）temp_buffers

temp_buffers即临时缓冲区，用于数据库访问临时表数据，Greenplum默认值为1M。可以在单独的session中对该参数进行设置，在访问比较大的临时表时，对性能提升有很大帮助。

（5）client_encoding

设置客户端字符集，默认和数据库encoding相同。

（6）client_min_messages

控制发送至客户端的信息级别，每个级别包括更低级别的消息，越是低的消息级别发送至客户端的信息越少。例如，warning级别包括warning、error、fatal、panic等级别的信息，而panic则只包括panic级别的信息。此参数主要用于错误调试。

（7）cpu_index_tuple_cost

设置执行计划评估每一个索引行扫描的CPU成本。同类参数还包括cpu_operator_cost、cpu_tuple_cost、cursor_tuple_fraction。

（8）debug_assertions

打开各种断言检查，这是调试助手。如果遇到了奇怪的问题或数据库崩溃，那么可以将这个参数打开，便于分析错误原因。

（9）debug_print_parse

当需要查看查询语句的分析树时，可以设置开启此参数，默认为off。

（10）debug_print_plan

当需要查看查询语句的执行计划时，可以设置开启此参数，默认为off。同类参数包括debug_print_prelim_plan、debug_print_rewritten、debug_print_slice_table。

（11）default_tablespace

指定创建对象时的默认表空间。

（12）dynamic_library_path

在创建函数或者加载命令时，如果未指定目录，将会从这个路径搜索需要的文件。

（13）enable_bitmapscan

表示是否允许位图索引扫描，类似的参数还有enable_groupagg、enable_hashagg、enable_hashjoin、enable_indexscan、enable_mergejoin、enable_nestloop、enable_seqscan、enable_sort、enable_tidscan。这些参数主要用于控制执行计划。

（14）gp_autostats_mode

指定触发自动搜集统计信息的条件。当此值为on_no_stats时，create table as select会自动搜集统计信息，如果insert和copy操作的表没有统计信息，也会自动触发统计信息搜集。当此值为on_change时，如果变化量超过gp_autostats_on_change_threshold参数设置的值，会自动触发统计信息搜集。此参数还可设为none值，即不自动触发统计信息搜集。

（15）gp_enable_gpperfmon

要使用Greenplum Performance Monitor工具，必须开启此参数。

（16）gp_external_max_segs

设置外部表数据扫描可用segments数目。

（17）gp_fts_probe_interval

设置ftsprobe进程对segment failure检查的间隔时间。

（18）gp_fts_probe_threadcount

设置ftsprobe线程数，此参数建议大于等于每台服务器segments的数目。

（19）gp_fts_probe_timeout

设置ftsprobe进程用于标识segment down的连接Segment的超时时间。

（20）gp_hashjoin_tuples_per_bucket

此参数越小，hash tables越大，可提升join性能，相关参数还有gp_interconnect_hash_multiplier、gp_interconnect_queue_depth。

（21）gp_interconnect_setup_timeout

此参数在负载较大的集群中，应该设置较大的值。

（22）gp_interconnect_type

可选值为TCP、UDP，用于设置连接协议。TCP最大只允许1000个节点实例。

（23）gp_log_format

设置服务器日志文件格式。可选值为csv和text。

（24）gp_max_databases

设置服务器允许的最大数据库数，相关参数还有gp_max_filespaces、gp_max_packet_size、gp_max_tablespaces

（25）gp_resqueue_memory_policy

此参数允许none和auto这2个值，当设置为none时，和Greenplum 4.1版本以前的策略一致；而设置为auto时，查询内存使用受statement_mem和资源队列的内存限制，而work_mem、max_work_mem和maintenance_work_mem这三个参数将失效。

（26）gp_resqueue_priority_cpucores_per_se gment

指定每个Segment可用的CPU单元

（27）gp_segment_connect_timeout

设置网络连接超时时间。

（28）gp_set_proc_affinity

设置进程是否绑定至CPU。

（29）gp_set_read_only

设置数据库是否允许写。

（30）gp_vmem_idle_resource_timeout

设置数据库会话超时时间，超过此参数值会话将释放系统资源（比如shared memory）。此参数越小，集群并发度支持越高。

（31）gp_vmem_protect_limit

设置服务器中postgres进程可用的总内存，建议设置为（X*physical_memory）/primary_segments，其中X可设置为1.0和1.5之间的数字，当X=1.5时容易引发swap，但是会减少因内存不足而失败的查询数。

（32）log_min_duration_statement

当查询语句执行时间超过此值时，将会记录日志，相关参数参考log_开头的所有参数。

（33）maintenance_work_mem

设置用于维护的操作可用的内存数，比如vacuum、create index等操作将受到这个参数的影响。

（34）max_appendonly_tables

最大可并发处理的appendonly表的数目。

（35）max_connections

最大连接数，Segment建议设置成Master的5~10倍。

（36）max_statement_mem

设置单个查询语句的最大内存限制，相关参数是max_work_mem。

（37）random_page_cost

设置随机扫描的执行计划评估成本，此值越大越倾向于选择顺序扫描，越小越倾向于选择索引扫描。

（38）search_path

设置未指定schema时，按照这个顺序选择对象的schema。

（39）statement_timeout

设置语句终止执行的超时时间，0表示永不终止。

更多参数请详细参考PostgreSQL及Greenplum文档。



## 第8章　数据库管理

### 8.1　用户及权限管理

在Greenplum中：

1）一个Database下可以有多个Schema，一个Schema只属于一个Database。Schema在Greenplum中也叫做Namespace，不同Database之间的Schema没有关系，可以重名。

2）Language在使用前必须创建，一个语言只属于一个Database。

3）Table、View、Sequence、Function必须属于一个Schema。

4）一个FileSpace可以有多个TableSpace，一个TableSpace只属于一个FileSpace，FileSpace与Role没有关系。

5）TableSpace与Table是一对多的关系，一个Schema下的表可以分布在多个TableSpace下。

6）在图9-1中，除了FileSpace之外，其他的权限管理都是通过Role来实现，在这些层次结构中，用户必须对上一层有访问权限，才能够访问该层的内容。

7）Group与Role是一样的概念，在Greenplum中，使用Role就可以了，虽然Group在语法上还可以用，实际上已经被废弃了。

#### 8.1.2　Grant语法

与普通数据库一样，赋权通过Grant来实现，Greenplum中的权限控制相比其他数据库会更细一点。

创建数据库语法为：

------

```
testDB=# \h create role
Command:     CREATE ROLE
Description: define a new database role
Syntax:
CREATE ROLE name [[WITH] option [ ... ]]
where option can be:
      SUPERUSER | NOSUPERUSER
    | CREATEDB | NOCREATEDB
    | CREATEROLE | NOCREATEROLE
    | CREATEEXTTABLE | NOCREATEEXTTABLE 
      [ ( attribute='value'[, ...] ) ]
           where attributes and values are:
           type='readable'|'writable'
           protocol='gpfdist'|'http'|'gphdfs'
    | INHERIT | NOINHERIT
    | LOGIN | NOLOGIN
    | CONNECTION LIMIT connlimit
    | [ ENCRYPTED | UNENCRYPTED ] PASSWORD 'password'
    | VALID UNTIL 'timestamp' 
    | IN ROLE rolename [, ...]
    | ROLE rolename [, ...]
    | ADMIN rolename [, ...]
| RESOURCE QUEUE queue_name
```

------

从语法上看，参数配置主要有以下几种，配置可以叠加使用。

1）超级用户（SUPERUSER|NOSUPERUSER）：最高用户权限，不受资源队列控制，拥有所有的权限，可以对数据库进行任何操作，一般只有DBA可以拥有这个权限。

2）创建数据库权限（CREATEDB|NOCREATEDB）。

3）创建用户权限（CREATEROLE|NOCREATEROLE）：这个用户可以创建其他用户。

4）登录权限（LOGIN|NOLOGIN）：可以指定该用户登录的连接数控制。

5）创建外部表权限（CREATEEXTTABLE|NOCREATEEXTTABLE）：属性配置中也可以对外部表有更细的权限控制，如只读、可写外部表权限等。

6）用户继承（INHERIT|NOINHERIT）：子用户可以拥有父用户的所有权限。

7）资源队列控制（RESOURCE QUEUE）：这个在9.3节有更详细的描述。

8）密码控制（ENCRYPTED|UNENCRYPTED）：还可以指定密码以及失效时间。

下面是一个创建用户的例子，testrole1这个用户可以有登录权限，以及创建数据库与创建用户的权限：

------

```
testDB=# create role testrole1 createdb createrole  login ;
NOTICE:  resource queue required -- using default resource queue "pg_default"
CREATE ROLE
```

------

Testrole2这个用户的密码为test，有效期只到“2013-12-21：00：00：00”，连接数限制为5，并且继承了testrole1的所有权限。

------

```
testDB=# create role testrole2 password 'test' valid until '2013-12-21 00:00:00' connection limit 5 inherit in role testrole1 login ;
NOTICE:  resource queue required -- using default resource queue "pg_default"
CREATE ROLE
```

------

赋权命令Grant，在语法上与其他数据库类似，其基本语法结构是：

------

```
GRANT权限类型ON Relation（如表，视图，函数，Schema等）TO用户或用户组;
```

### 8.2　资源队列及并发控制

1.内存

如果一个资源队列中限制了最大使用内存是2000MB，同时设置了同时执行的SQL数为10个，那么每一个SQL最多使用的内存就是200MB。同时，每个SQL消耗的内存，不能大于statement_mem参数中设置的内存大小。当一个SQL运行的时候，这个内存大小就会被分配出来，直到SQL执行结束后才释放。

2.CPU

CPU优先级管理，每一个资源队列中，都有一个对应的CPU优先级。CPU的优先级有三个等级。

·adhoc，低优先级。

·reporting，高优先级。

·executive，最高优先级。

当系统中有新的SQL进入的时候，各个SQL消耗CPU的资源会根据其优先级重新评估

3.语法介绍

并不是所有的SQL都会被限制在资源队列中，在默认情况下，SELECT、SELECT INTO、CREATE TABLE AS SELECT和DECLARE CURSOR会被限制在队列中。如果将参数resource_select_only设置成off，那么INSERT、UPDATE、DELETE语句也会被限制在队列中。

下面介绍如何创建资源队列，以及如何使用资源队列，语法如下：

------

```
CREATE RESOURCE QUEUE name WITH (queue_attribute=value [, ... ]) 
where queue_attribute is:
   ACTIVE_STATEMENTS=integer
        [ MAX_COST=float [COST_OVERCOMMIT={TRUE|FALSE}] ]
        [ MIN_COST=float ]
        [ PRIORITY={MIN|LOW|MEDIUM|HIGH|MAX} ]
        [ MEMORY_LIMIT='memory_units' ]
|  MAX_COST=float [ COST_OVERCOMMIT={TRUE|FALSE} ] 
        [ ACTIVE_STATEMENTS=integer ]
        [ MIN_COST=float ]
        [ PRIORITY={MIN|LOW|MEDIUM|HIGH|MAX} ]
        [ MEMORY_LIMIT='memory_units' ]
```

------

·创建一个队列只有限制最大的活动SQL数：

------

```
testDB=# CREATE RESOURCE QUEUE adhoc WITH (ACTIVE_STATEMENTS=3);
CREATE QUEUE
```

------

·创建一个队列加上内存限制：

------

```
testDB=# CREATE RESOURCE QUEUE myqueue WITH (ACTIVE_STATEMENTS=20, MEMORY_LIMIT='200MB');
CREATE QUEUE
```

如果想对一个SQL进行特殊处理，增加其运行时的内存，那么可以设置statement_mem参数，将它调大，例如：

------

```
=> SET statement_mem='2GB';
=> SELECT * FROM my_big_table WHERE column='value' ORDER BY id;
=> RESET statement_mem;
```

------

·设置最大的cost值（cost值如何计算，请参考第5章）：

------

```
CREATE RESOURCE QUEUE webuser WITH (MAX_COST=10000.0);
```

------

当cost值超出资源限制时，会报错：

------

```
ERROR:  statement requires more resources than resource queue allows
```

------

在默认情况下COST_OVERCOMMIT参数为false，这个时候，只要cost值超过MAX_COST的SQL都会报错。如果COST_OVERCOMMIT设置为true，在当前资源队列中，没有其他的SQL在运行的时候，这个超过MAX_COST的SQL也会被执行。

有一个与MAX_COST对应的MIN_COST，如果一个SQL的cost值小于MIN_COST，那么这个SQL不管资源是什么情况，都会马上被执行。

·设置CPU优先级：

CPU优先级有5个级别：MIN|LOW|MEDIUM|HIGH|MAX，可以根据不同的需求选择：

------

```
=# ALTER RESOURCE QUEUE adhoc WITH (PRIORITY=LOW);
=# ALTER RESOURCE QUEUE reporting WITH (PRIORITY=HIGH);
```

------

Greenplum中提供了很多表和视图用于查看资源队列的情况。查看配置情况：

------

```
testDB=# select * from pg_resqueue_attributes;
  rsqname   |      resname      | ressetting | restypid 
------------+-------------------+------------+----------
 adhoc      | active_statements | 3          |        1
 adhoc      | max_cost          | -1         |        2
 adhoc      | min_cost          | 0          |        3
 adhoc      | cost_overcommit   | 0          |        4
 adhoc      | priority          | medium     |        5
 adhoc      | memory_limit      | -1         |        6
```

------

查看现有的资源队列使用情况：

------

```
testDB=# select * from pg_resqueue_status;
  rsqname   | rsqcountlimit | rsqcountvalue | rsqcostlimit | rsqcostvalue | rsqwaiters | rsqholders 
------------+---------------+---------------+--------------+--------------+------------+------------
 adhoc      |             3 |             0 |           -1 |              |          0 |          0
 pg_default |            20 |             0 |           -1 |              |          0 |          0
(2 rows)
```

------

在gp_toolkit中，还有几个视图可用于查看资源队列的使用情况：

------

```
testDB=# \dv gp_toolkit.gp_resq*
                         List of relations
   Schema   |            Name            | Type |  Owner  | Storage 
------------+----------------------------+------+---------+---------
 gp_toolkit | gp_resq_activity           | view | gpadmin | none
 gp_toolkit | gp_resq_activity_by_queue  | view | gpadmin | none
 gp_toolkit | gp_resq_priority_backend   | view | gpadmin | none
 gp_toolkit | gp_resq_priority_statement | view | gpadmin | none
 gp_toolkit | gp_resq_role               | view | gpadmin | none
 gp_toolkit | gp_resqueue_status         | view | gpadmin | none
(6 rows)
```

------

创建/修改用户指定资源队列：

------

```
testDB=# create role aquery RESOURCE QUEUE adhoc;
testDB=# alter role etl resource queue adhoc;
```

------

修改资源队列的语法如下，只有超级用户才可以修改资源组，ALTER RESOURCE QUEUE的语法介绍很明显，这里就不多介绍。修改资源队列语法如下：

------

```
ALTER RESOURCE QUEUE name WITH ( queue_attribute=value [, ... ] ) 
where queue_attribute is:
   ACTIVE_STATEMENTS=integer
   MEMORY_LIMIT='memory_units'
   MAX_COST=float
   COST_OVERCOMMIT={TRUE|FALSE}
   MIN_COST=float
   PRIORITY={MIN|LOW|MEDIUM|HIGH|MAX}
```

### 8.3　Greenplum锁机制

reenplum是一个分布式的数据库，其对应的锁机制肯定比普通的数据库还要复杂一些，因为需要统一每一个节点的锁，这个锁的控制是在Master上执行的。

![image-20200324114817687](F:\Typora\resources\app\Docs\img\image-20200324114817687.png)

这几种锁的冲突

![image-20200324114941150](F:\Typora\resources\app\Docs\img\image-20200324114941150.png)

通过lock命令可以显式地将表锁住，语法如下：

------

```
Command:     LOCK
Description: lock a table
Syntax:
LOCK [ TABLE ] name [, ...] [ IN lockmode MODE ] [ NOWAIT ]
where lockmode is one of:
    ACCESS SHARE | ROW SHARE | ROW EXCLUSIVE | SHARE UPDATE EXCLUSIVE
    | SHARE | SHARE ROW EXCLUSIVE | EXCLUSIVE | ACCESS EXCLUSIVE
```

### 8.4　数据目录结构

·base是数据目录，每个数据库在这个目录下，会有一个对应的文件夹。

·global是每一个数据库公用的数据目录。

·gpperfmon监控数据库性能时，存放监控数据的地方。

·pg_changetracking是Segment之间主备同步用到的一些原数据信息保存的地方。

·pg_clog是记录数据库事务信息的地方，保存了每一个事务id的状态，这个非常重要，不能丢失，一旦丢失，整个数据库就基本上不可用了。

·pg_log是数据库的日志信息。

·pg_twophase是二阶段提交的事务信息（关于二阶段提交的内容可参阅第7章中的介绍）

·pg_xlog是数据库重写日志保存的地方，其中每个文件固定大小为64MB，并不断重复使用。

·gp_dbid记录这个数据库的dbid以及它对应的mirror节点的dbid。

·pg_hba.conf是访问权限控制文件。

·pg_ident.conf是Ident映射文件。

·PG_VERSION是PostgreSQL的版本号。

·postgresql.conf是参数配置文件。

·postmaster.opts是启动该数据库的pg_ctl命令。

·postmaster.pid是该数据库的进程号和数据目录信息。

其中数据目录base下面的文件夹结构为：

------

```
#ls
1  10890  10891  16992  285346
```

------

其中，一个文件夹代表一个数据库，文件夹的名字就是数据库的oid，可以通过pg_database查询其对应关系：

------

```
testDB=# select oid,datname from pg_database order by oid;
  oid   |  datname  
--------+-----------
      1 | template1
  10890 | template0
  10891 | postgres
  16992 | testDB
285346 | gpperfmon
(6 rows)
```

### 8.4　数据文件存储分布

（1）表

最普通的堆表只有一个数据文件，如果表中有大字段，那么这个表会多两个数据文件，分别是toast表和toast表索引。

```
testdb=# create table test_varchar2037(id int,values varchar(2037)) distributed by (id);
CREATE TABLE
testdb=# create table test_varchar2036(id int,values varchar(2036)) distributed by (id);
CREATE TABLE
testdb=# create table test_text(id int,values text) distributed by (id);
CREATE TABLE
testdb=# select oid,relname,reltoastrelid from pg_class where relname like 'test_varchar%' or relname = 'test_text';
  oid  |     relname      | reltoastrelid
-------+------------------+---------------
 33532 | test_varchar2037 |         33535
 33538 | test_varchar2036 |             0
 33541 | test_text        |         33544
(3 rows)
```

可以看出临界点varchar的大小就是2036，当超过2036时，数据库就会为该表创建一个toast表，text类型本来就是保存大文本字段的，所以一定有toast表。

如果原来表中最大的一个字段是varchar（2036），通过修改表结构将其改为varchar（2037），那么原来没有toast表的也会增加toast表。

```
testdb=# alter table test_varchar2036  alter  values type varchar(2037);
ALTER TABLE
testdb=# select oid,relname,reltoastrelid from pg_class where relname like 'test_varchar%' or relname = 'test_text';
  oid  |     relname      | reltoastrelid
-------+------------------+---------------
 33532 | test_varchar2037 |         33535
 33541 | test_text        |         33544
 33538 | test_varchar2036 |         33547
(3 rows)
```

默认toast表的名字为pg_toast_+原表的relfilenode，索引为pg_toast_+原表的relfilenode+_index，如下（33541是表test_text的oid）：

```
testdb=# select oid,relname from pg_class where relname like '%33541%';
  oid  |       relname
-------+----------------------
 33544 | pg_toast_33541
 33546 | pg_toast_33541_index
(2 rows)
```

（2）索引

索引文件只有一个，可以通过索引名在pg_class中查找。索引在创建的时候就分配了32KB的存储空间，等到这32KB用完了才开始扩大。

（3）序列

序列（Sequence）与索引一样，也只有一个数据文件，在pg_class中对应一条记录，relfilenode字段就是文件名。

如果一个表中有字段是serial类型的，即一个递增序列，那么这个表会自动创建一个序列，也就会多一个数据文件。

### 9.1　监控Segment是否正常

1）检查gp_segment_configuration以确定是否有Segment处于down的状态，或者查看gp_configuration_history以观察最近数据库是否发生了切换。

------

```
select * from gp_segment_configuration where status='d' or mode<>'s';
```

Segment已经卡住了，但是Master没有感知到Segment失败，这个时候，首先就是要监控当前运行的SQL是否有超过很长时间的。其次就是要在Greenplum中建立一张心跳表，这张心跳表至少要在每个Segment都有一条记录，然后不断去更新表中所有的记录，当发现这个SQL超过一定时间都没有执行完，就要发出报警。

如何保证每个Segment都有一条记录？下面介绍一个方法：首先创建一张临时表，其中的数据量比节点数大很多，保证分布键的数据比较散，确保每个Segment都有数据，然后在每个节点上取一条数据插入新的一张表中，这样就能够保证每一个Segment都至少有一条数据，具体的SQL如下：

①创建临时表，插入10000条数据

------

```
create table xdual_temp 
as 
select generate_series(1,10000) id
distributed by (id);
```

------

②建立心跳表，2个字段，第二个字段是timestamp类型的，每次心跳检测数据会被更新

------

```
create table xdual(id int,update_time timestamp(0))
distributed by(id);
```

------

③往心跳表的每个Segment中插入一条数据

------

```
insert into xdual(id,update_time)
select id,now() from 
   (select id,row_number() over(partition by gp_segment_id order by id) rn  
      from xdual_temp
   )t
where rn=1;
```

------

效果如下：

------

```
testDB=# select gp_segment_id,* from xdual order by 1;
 gp_segment_id | id |     update_time     
---------------+----+---------------------
             0 |  1 | 2012-08-12 19:40:43
             1 |  4 | 2012-08-12 19:40:43
             2 |  9 | 2012-08-12 19:40:43
             3 |  2 | 2012-08-12 19:40:43
             4 |  3 | 2012-08-12 19:40:43
             5 |  8 | 2012-08-12 19:40:43
(6 rows)
```









