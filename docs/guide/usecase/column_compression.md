# 列压缩功能说明

当前有针对行格式的压缩和针对数据页面的压缩，但是这两种压缩方式在处理一个表中的某些大字段和其他很多小字段，同时对小字段的读写很频繁，对大字段访问不频繁的情形中，它的读写访问时都会造成很多不必要的计算资源的浪费，因此需要列压缩功能来压缩那些访问不频繁的大字段，同时能够减少整行字段的存储空间，提高读写访问的效率。

例如：一张员工表：create table employee(id int, age int, gender boolean, other varchar(1000) primary key (id))，当对id,age,gender小字段访问比较频繁，而对other大字段的访问频率比较低时，可以将other列创建为压缩列。一般情况下，只有对other的读写才会触发对该列的压缩和解压，对其他列的访问并不会触发该列的压缩和解压。由此进一步降低了行数据存储的大小，使得我们在访问频繁的小字段能够更快，对存储空间比较大而访问频率比较低的字段，使得我们进一步够降低存储空间。

## 支持的数据类型
1. BLOB (包含 TINYBLOB , MEDIUMBLOB , LONGBLOB )
2. TEXT (包含 TINYTEXT , MEDIUMTEXT , LONGTEXT )
3. VARCHAR
4. VARBINARY

注意：其中LONGBLOB和LONGTEXT的长度最大只支持，相比官方String Type Storage Requirements支持的少一个字节。

## 支持的DDL语法类型

相对官方的[建表语法](https://dev.mysql.com/doc/refman/5.7/en/create-table.html), 其中的column_definition中的COLUMN_FORMAT定义有所变动。同时列压缩只支持Innodb存储引擎类型的表。

``` java
      column_definition:        data_type [NOT NULL | NULL] [DEFAULT default_value]          [AUTO_INCREMENT] [UNIQUE [KEY]] [[PRIMARY] KEY]          [COMMENT 'string']          [COLLATE collation_name]          [COLUMN_FORMAT {FIXED|DYNAMIC|DEFAULT}|COMPRESSED=[zlib]]  # COMPRESSED 压缩列关键字          [STORAGE {DISK|MEMORY}]          [reference_definition]
```

一个简单的示例如下：

``` java
CREATE TABLE t1(  id INT PRIMARY KEY,  b BLOB COMPRESSED);
```

此时省略了压缩算法，默认选择 ZLIB压缩算法 ，你也可以显示指定压缩算法关键字，目前只支持zlib压缩算法。

``` java
CREATE TABLE t1(  id INT PRIMARY KEY,  b BLOB COMPRESSED=zlib);
```

支持的DDL语法总结如下：

create table方面：
<table>
<tr>
<td rowspan="1" colSpan="1" >DDL</td>
<td rowspan="1" colSpan="1" >是否继承压缩属性</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >CREATE TABLE t2 LIKE t1;</td>
<td rowspan="1" colSpan="1" >Y</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >CREATE TABLE t2 SELECT * FROM t1;</td>
<td rowspan="1" colSpan="1" >Y</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >CREATE TABLE t2(a BLOB) SELECT * FROM t1;</td>
<td rowspan="1" colSpan="1" >N</td>
</tr>
</table>




alter table方面：

<table>
<tr>
<td rowspan="1" colSpan="1" >DDL</td>
<td rowspan="1" colSpan="1" >描述</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >ALTER TABLE t1 MODIFY COLUMN a BLOB;</td>
<td rowspan="1" colSpan="1" >将压缩列变为非压缩的</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >ALTER TABLE t1 MODIFY COLUMN a BLOB COMPRESSED;</td>
<td rowspan="1" colSpan="1" >将非压缩列变为压缩的</td>
</tr>
</table>




## 相关变量和状态说明

在带有压缩属性的列中，其数据存储有两种形式

- 非压缩格式，原数据保持不变，只在原数据前面添加一个字节的压缩头；
- 压缩格式，原数据处于压缩状态，同时在压缩数据前面添加一个字节的压缩头；


### 新增变量说明

<table>
<tr>
<td rowspan="1" colSpan="1" >name</td>
<td rowspan="1" colSpan="1" >dynamic</td>
<td rowspan="1" colSpan="1" >type</td>
<td rowspan="1" colSpan="1" >default</td>
<td rowspan="1" colSpan="1" >tencent variable</td>
<td rowspan="1" colSpan="1" >explanation</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >cdb_column_compression_enabled</td>
<td rowspan="1" colSpan="1" >Yes</td>
<td rowspan="1" colSpan="1" >bool</td>
<td rowspan="1" colSpan="1" >FALSE</td>
<td rowspan="1" colSpan="1" >Yes</td>
<td rowspan="1" colSpan="1" >列压缩的开关，关闭后，不允许建有压缩属性的表，已有压缩属性的表不受影响</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >innodb_column_compression_zlib_wrap</td>
<td rowspan="1" colSpan="1" >Yes</td>
<td rowspan="1" colSpan="1" >bool</td>
<td rowspan="1" colSpan="1" >TRUE</td>
<td rowspan="1" colSpan="1" >No</td>
<td rowspan="1" colSpan="1" >如果打开，将生成数据的zlib头和zlib尾并做adler32校验</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >innodb_column_compression_zlib_strategy</td>
<td rowspan="1" colSpan="1" >Yes</td>
<td rowspan="1" colSpan="1" >Integer</td>
<td rowspan="1" colSpan="1" >0</td>
<td rowspan="1" colSpan="1" >No</td>
<td rowspan="1" colSpan="1" >列压缩使用的压缩策略，最小值为：0，最大值为4，0～4分别和zlib中的压缩策略：Z_DEFAULT_STRATEGY,Z_FILTERED,Z_HUFFMAN_ONLY,Z_RLE,Z_FIXED是一一对应的。一般来说，对于文本数据，Z_DEFAULT_STRATEGY通常是最佳的，Z_RLE对于图像数据来说是最佳的。</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >innodb_column_compression_zlib_level</td>
<td rowspan="1" colSpan="1" >Yes</td>
<td rowspan="1" colSpan="1" >Integer</td>
<td rowspan="1" colSpan="1" >6</td>
<td rowspan="1" colSpan="1" >No</td>
<td rowspan="1" colSpan="1" >列压缩使用的压缩级别，最小值：0，最大值：9，0代表不压缩，该值越大代表压缩后的数据越小，但压缩时间也越长</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >innodb_column_compression_threshold</td>
<td rowspan="1" colSpan="1" >Yes</td>
<td rowspan="1" colSpan="1" >Integer</td>
<td rowspan="1" colSpan="1" >256</td>
<td rowspan="1" colSpan="1" >No</td>
<td rowspan="1" colSpan="1" >列压缩使用的压缩阈，最小值为：1，最大值为：0xffffffff，单位：字节。只有长度大于或等于该值数据才会被压缩，否则原数据保持不变，只是添加压缩头</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >innodb_column_compression_pct</td>
<td rowspan="1" colSpan="1" >Yes</td>
<td rowspan="1" colSpan="1" >Integer</td>
<td rowspan="1" colSpan="1" >100</td>
<td rowspan="1" colSpan="1" >No</td>
<td rowspan="1" colSpan="1" >列压缩使用的压缩率，最小值：1，最大值：100，单位：百分比。只有压缩后数据大小/压缩前数据大小低于该值时，数据才会被压缩，否则原数据保持不变，只是添加压缩头</td>
</tr>
</table>


### 新增状态说明

<table>
<tr>
<td rowspan="1" colSpan="1" >name</td>
<td rowspan="1" colSpan="1" >type</td>
<td rowspan="1" colSpan="1" >explanation</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >Innodb_column_compressed</td>
<td rowspan="1" colSpan="1" >Integer</td>
<td rowspan="1" colSpan="1" >列压缩的压缩次数，包括非压缩格式和压缩格式两种状态的压缩</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >Innodb_column_decompressed</td>
<td rowspan="1" colSpan="1" >Integer</td>
<td rowspan="1" colSpan="1" >列压缩的解压次数，包括非压缩格式和压缩格式两种状态的解压缩</td>
</tr>
</table>


### 新增错误说明
<table>
<tr>
<td rowspan="1" colSpan="1" >name</td>
<td rowspan="1" colSpan="1" >parameter</td>
<td rowspan="1" colSpan="1" >explanation</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >Compressed column '%-.192s' can't be used in key specification</td>
<td rowspan="1" colSpan="1" >指定压缩的列名</td>
<td rowspan="1" colSpan="1" >不能对有索引的列指定压缩属性</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >Unknown compression method: %s"</td>
<td rowspan="1" colSpan="1" >用户在DDL语句中指定的压缩算法名</td>
<td rowspan="1" colSpan="1" >在create table或者alter table时指定zlib之外非法的压缩算法</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >Compressed column '%-.192s' can't be used in column format specification</td>
<td rowspan="1" colSpan="1" >指定压缩的列名</td>
<td rowspan="1" colSpan="1" >在同一个列中，已经指定COLUMN_FORMAT属性就不能再指定压缩属性，其中COLUMN_FORMAT只在NDB中被使用</td>
</tr>

<tr>
<td rowspan="1" colSpan="1" >Alter table ... discard/import tablespace not support column compression</td>
<td rowspan="1" colSpan="1" >\</td>
<td rowspan="1" colSpan="1" >带有列压缩的表不能执行Alter table ... discard/import tablespace语句</td>
</tr>
</table>


## 性能

整体性能分为DDL和DML两方面：

DDL方面：

1. 列压缩对COPY算法的DDL有较大的性能影响；
2. 对于inplace的影响则取决于压缩后的数据量大小，如果采用压缩后，整体数据有降低，那么DDL的性能是有提升；反之，性能会有一定的降幅；
3. 对于instant来说，列压缩对该类型的DDL基本没有影响；

DML方面：

考虑最常见的压缩情形（压缩比：1:1.8），其插入和查询都有10%以内的提升，但对于更新则有10%以内的下降。这是因为在更新过程中，MySQL会先读出该行的值然后在写入该行更新之后的值，整个更新过程会触发一次解压和压缩而插入和查询只会进行一次压缩或者解压。


### 相关适配建议以及注意事项

#### 适配方面

1. binlog中列压缩相关的DDL语句会带有/*!50718 COMPRESSED */ 压缩关键字（与官方语法是不兼容的），因此主备的版本号必须都高于列压缩发行的内核版本，并且必须同时打开cdb_column_compression_enabled开关。
2. 建议在添加列压缩的表中最好满足以下几个条件：(1) 最好是有主键或者唯一索引；(2) 指定压缩属性的列读写都比较少；

#### 注意事项

1. 逻辑导出方面，逻辑导出时create table还是会附有Compressed相关的关键字。因此导入时在CDB内部是支持的。其他MySQL分支以及官方版本：
   - 官方版本号小于5.7.18，可以直接导入；
   - 官方版本号大于或等于5.7.18，需要在逻辑导出之后，去掉压缩关键字；
2. DTS导出其他云或是用户时，在binlog同步过程中可能会出现不兼容的问题，可以跳过带压缩关键字的DDL语句。
3. 物理备份方面，由于备份时其字段在innodb内部是已经压缩的状态，因此使用该备份的版本也必须是带有列压缩的版本。
