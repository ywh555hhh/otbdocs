# Parallel Copy DDL

##  简介

尽管Instant DDL和Parallel Inplace DDL已经使得很多DDL操作的性能得到极大的提升，但是并不能覆盖所有的DDL操作。例如常见的修改列类型的操作，如果修改的列类型是char和int之间，需要重构表数据，不能够通过Instant算法执行，需要通过Copy算法来执行。

如前文所述，Copy DDL是从Server层执行的，因此，几乎所有DDL操作都可以指定Copy算法执行。因此，优化Copy DDL是十分必要的。

##  设计方案

Copy DDL操作的原理简单来说就是：

1. create DDL操作后的表结构；
2. 查询原表数据；
3. 插入数据到新表；
4. 重命名新表

代码执行路径如下：

``` java
mysql_alter_table
	if (use_inplace) {
      mysql_inplace_alter_table() //如果能使用inplace alter table，走inplace方式
	}
	// 否则走copy方式
	ha_create_table();
	lock_tables();
	copy_data_between_tables();
	wait_while_table_is_used();
	mysql_rename_table();
```

Copy DDL执行过程中关键的函数是copy_data_between_tables，该函数会在Server层构建一个TableScanIterator，逐行从Innodb层的原表中读上来数据并且调用ha_write_rows插入到新建的临时表中去。这个函数比较耗时，尤其在表数据量比较庞大的时候。

考虑在server层构建并行的TableScanIterator和m_prebuilt的复杂度和实现代价较高，我们将copy_data_between_tables函数下推到innodb层执行，innodb层使用Parallel Reader框架，并行读取原表的数据，并插入到新建的临时表中去。

该方案的难点在于，需要在innodb层做数据行格式的转换，如果是新增列，需要将所有行对应的位置添加default值，如果是修改列，需要将对应列的数据转换成修改后的类型的值格式。

当涉及到不同类型的转换（例如char(10)转int，int转decimal）时，目前innodb层并没有实现相关的功能，这里我们利用原本server层提供的接口，在innodb层构建源表和目标表之间的Copy_field，调用invoke_do_copy实现不同类型之间的转换，如果转换出错，返回对应的error code。这样做的风险较小可控。

##  性能

使用sysbench测试，sbtest表，5亿条数据，测试修改列类型这一操作在不同线程下所需的时间：

``` sql
#单线程
MySQL [sbtest1_500000000]> alter table sbtest1 change column k k char(10);
Query OK, 500000000 rows affected (1 hour 33 min 17.29 sec)
Records: 500000000  Duplicates: 0  Warnings: 0

#4线程 2.34x
MySQL [sbtest1_500000000]> alter table sbtest1 change column k k int;
Query OK, 500000000 rows affected (39 min 53.94 sec)
Records: 0  Duplicates: 0  Warnings: 0

#16线程 5.47x
MySQL [sbtest1_500000000]> alter table sbtest1 change column k k char(10);
Query OK, 500000000 rows affected (17 min 3.52 sec)
Records: 0  Duplicates: 0  Warnings: 0

#64 线程 7.06x
MySQL [sbtest1_500000000]> alter table sbtest1 change column k k int;
Query OK, 500000000 rows affected (13 min 13.38 sec)
Records: 0  Duplicates: 0  Warnings: 0
```



##  使用方式及限制

新增session级别的参数：txsql_parallel_copy_ddl，用于控制本session的copy ddl操作是否使用parallel copy ddl优化。

并发线程数与innodb_parallel_read_threads参数相关

限制：

以下类型的表暂不支持Parallel Copy DDL:

1. 非innodb表
2. 临时表
3. alter table order by，对行顺序有要求，需要单线程去做
4. 新增auto increment列的表
5. 含有外键约束的表
6. 含有multi value函数索引的表