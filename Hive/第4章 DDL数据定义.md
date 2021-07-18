# 1.库操作

## **1.1 创建数据库**

<span style='font-size:21px;'>**创建数据库sql语句**</span>

```mysql
CREATE DATABASE [IF NOT EXISTS] database_name
[COMMENT database_comment]
[LOCATION hdfs_path]
[WITH DBPROPERTIES (property_name=property_value, ...)];
```

创建一个数据库，数据库在HDFS上的默认存储路径是/user/hive/warehouse/*.db

```mysql
create database db_hive;
```

避免要创建的数据库已经存在错误，增加if not exists判断；`标准写法`

```mysql
create database db_hive;
#FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.DDLTask. Database db_hive already exists
create database if not exists db_hive;
```

创建一个数据库，指定数据库在HDFS上存放的位置

```mysql
create database db_hive location '/mario/hive/db_hive.db';
```

![20210714161903](https://typora-photograph.oss-cn-shanghai.aliyuncs.com/img/20210718131207.png)

## 1.2 查询数据库

### 1.2.1 显示数据库

显示数据库

```mysql
show databases;
```

过滤显示查询的数据库

```mysql
show databases like 'db_hive*';
```

### 1.2.2 查看数据库详情

显示数据库信息

```mysql
desc database db_hive;
```

显示数据库详细信息 `extended`

```mysql
desc database extended db_hive;
```

### 1.2.3 切换当前数据库

```mysql
use db_hive;
```

## 1.3 修改数据库

用户可以使用 ALTER DATABASE 命令为某个数据库的 DBPROPERTIES 设置键-值对属性值，来描述这个数据库的属性信息。

数据库的其他元数据信息都是不可更改的，包括数据库名和数据库所在的目录位置。

```mysql
alter database db_hive set dbproperties('createtime'='20170830');
desc database extended db_hive;
```

## 1.4 删除数据库

删除空数据库

```mysql
drop database db_hive;
```

如果删除的数据库不存在最好采用 if exists 判断数据库是否存在。

```mysql
drop database db_hive2;
#FAILED: SemanticException [Error 10072]: Database does not exist: db_hive2
drop database if exists db_hive;
```

如果数据库不为空，可以采用cascade命令，强制删除。

```mysql
drop database db_hive;
#FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.DDLTask. InvalidOperationException(message:Database db_hive is not empty. One or more tables exist.)
drop database db_hive cascade;
```

# 2.表操作

## 2.1 创建表

<span style='font-size:21px;'>**建表语句**</span>

```mysql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
[(col_name data_type [COMMENT col_comment], ...)] 
[COMMENT table_comment] 
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
[CLUSTERED BY (col_name, col_name, ...) 
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 
[ROW FORMAT row_format] 
[STORED AS file_format] 
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)]
[AS select_statement]
```

<span style='font-size:21px;'>**字段解释说明**</span>

1. **CREATE TABLE** 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 IF NOT EXISTS 选项来忽略这个异常。

2. **EXTERNAL**关键字可以让用户创建一个外部表，在建表的同时可以指定一个指向实际数据的路径（LOCATION），在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。

3. **COMMENT**：为表和列添加注释。

4. **PARTITIONED BY**创建分区表

5. **CLUSTERED BY**创建分桶表

6. **SORTED BY**不常用，对桶中的一个或多个列另外排序

7. **ROW FORMAT** 

   DELIMITED [FIELDS TERMINATED BY char] [COLLECTION ITEMS TERMINATED BY char] [MAP KEYS TERMINATED BY char] [LINES TERMINATED BY char] 

   用户在建表的时候可以自定义SerDe或者使用自带的SerDe。如果没有指定ROW FORMAT 或者ROW FORMAT DELIMITED，将会使用自带的SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的SerDe，Hive通过SerDe确定表的具体的列的数据。

   

   SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]

   SerDe是Serialize/Deserilize的简称， hive使用Serde进行行对象的序列与反序列化。

8. **STORED AS**指定存储文件类型：

   常用的存储文件类型：SEQUENCEFILE（二进制序列文件）、TEXTFILE（文本）、RCFILE（列式存储格式文件）

   如果文件数据是纯文本，可以使用STORED AS TEXTFILE。如果数据需要压缩，使用 STORED AS SEQUENCEFILE。

9. **LOCATION** ：指定表在HDFS上的存储位置。

10. **AS**：后跟查询语句，根据查询结果创建表。

11. **LIKE**允许用户复制现有的表结构，但是不复制数据。

### 2.1.1 管理表

**理论：**

默认创建的表都是所谓的管理表，有时也被称为内部表。因为这种表，Hive会（或多或少地）控制着数据的生命周期。Hive默认情况下会将这些表的数据存储在由配置项hive.metastore.warehouse.dir(例如，/user/hive/warehouse)所定义的目录的子目录下。	当我们删除一个管理表时，Hive也会删除这个表中数据。管理表不适合和其他工具共享数据。

**案例实操：**

普通创建表

```mysql
create table if not exists student(id int, name string)
row format delimited fields terminated by '\t'
stored as textfile
location '/user/hive/warehouse/student';
```

根据查询结果创建表（查询的结果会添加到新创建的表中）

```mysql
create table if not exists student2 as select id, name from student;
```

根据已经存在的表结构创建表

```mysql
create table if not exists student3 like student;
```

查询表的类型

```mysql
desc formatted student2;
```

### 2.1.2 外部表

**理论：**

因为表是外部表，所以Hive并非认为其完全拥有这份数据。删除该表并不会删除掉这份数据，不过描述表的元数据信息会被删除掉。

**管理表和外部表的使用场景：**

每天将收集到的网站日志定期流入HDFS文本文件。在外部表（原始日志表）的基础上做大量的统计分析，用到的中间表、结果表使用内部表存储，数据通过SELECT+INSERT进入内部表。

**案例实操：**分别创建部门和员工外部表，并向表中导入数据。

上传数据到HDFS

```shell
hdfs dfs -mkdir /student;
hdfs dfs -put /opt/module/datas/student.txt /student;
```

建表语句，创建外部表

```mysql
create external table stu_external(id int, name string) 
row format delimited fields terminated by '\t' 
location '/mario/hive/student';
```

查看创建的表

```mysql
select * from stu_external;
```

查看表格式化数据

```mysql
DESCRIBE FORMATTED stu_external;
#Table Type:  EXTERNAL_TABLE
```

删除外部表

```mysql
drop table stu_external;
```

外部表删除后，hdfs中的数据还在，但是metadata中stu_external的元数据已被删除

### 2.1.3 管理表与外部表的互相转换

查询表的类型

```mysql
desc formatted student2;
#Table Type:	MANAGED_TABLE
```

修改内部表student2为外部表

```mysql
alter table student2 set tblproperties('EXTERNAL'='TRUE');
```

查询表的类型

```mysql
desc formatted student2;
#Table Type:	EXTERNAL_TABLE
```

修改外部表 student2 为内部表

```mysql
alter table student2 set tblproperties('EXTERNAL'='FALSE');
```

查询表的类型

```mysql
desc formatted student2;
#Table Type:	MANAGED_TABLE
```

注意：('EXTERNAL'='TRUE')和('EXTERNAL'='FALSE')为固定写法，区分大小写！

## 2.2 修改表

### 2.2.1 重命名表

语法

```mysql
ALTER TABLE table_name RENAME TO new_table_name；
```

实操案例

```mysql
alter table dept_partition2 rename to dept_partition3;
```

### 2.2.2 增加，修改和删除表分区

详见分区表操作

### 2.2.3 增加，修改，替换列信息

**语法：**

更新列

```mysql
ALTER TABLE table_name CHANGE [COLUMN] col_old_name col_new_name column_type [COMMENT col_comment] [FIRST|AFTER column_name]
```

增加和替换列

```mysql
ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type [COMMENT col_comment], ...) 
```

注：ADD是代表新增一字段，字段位置在所有列后面 (partition列前) ，REPLACE则是表示替换表中所有字段。

**实操案例：**

查询表结构

```mysql
desc dept_partition;
```

添加列

```mysql
alter table dept_partition add columns(deptdesc string);
```

查询表结构

```mysql
desc dept_partition;
```

更新列

```mysql
alter table dept_partition change column deptdesc desc int;
```

查询表结构

```mysql
desc dept_partition;
```

替换列

```mysql
alter table dept_partition replace columns(deptno string, dname string, loc string);
```

查询表结构

```mysql
desc dept_partition;
```

## 2.3删除表

```mysql
drop table dept_partition;
```

