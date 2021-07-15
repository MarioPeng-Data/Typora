# 1.什么是Hive

Hive是由Facebook开源，用来解决海量结构化日志数据统计

Hive是基于Hadoop的一个**数据仓库工具**，可以将**结构化的数据文件映射为一张表**，并提供**类SQL**查询功能。

本质：将HQL转化为MapReduce程序:

![img](https://gitee.com/peng-bo19951013/Picture/raw/master/20210715211237.png)

（1）Hive处理的数据存储在HDFS上 

（2）Hive分析引擎默认为MapReduce 

（3）执行程序运行在Yarn上

# 2.Hive的优缺点

**优点：**

- 操作接口采用了类SQL语法，减少了学习成本
- 用于处理海量数据
- 支持用户自定义函数

**缺点:**

- 效率比较低
- 延迟比较高
- 对于小数据量处理没有优势

# 3.Hive架构原理

<img src="https://gitee.com/peng-bo19951013/Picture/raw/master/20210715211434.png" alt="img" style="zoom: 50%;" />

Hive通过给用户提供一系列接口，接受用户的指令（HQL），使自己的Driver，结合元数据（Metastore），将这些指令翻译成MapReduce程序，将程序提交到Yarn上执行

**Client（用户的接口）**

- CLI（hive shell）、JDBC、ODBC、WEBUI

**Driver（驱动器）**

- 解析器——将SQL字符串抽象成语法树
- 编译器——将语法树生成逻辑执行计划
- 优化器——对逻辑计划进行优化
- 执行器——将优化过的逻辑计划转化为物理计划，实际上就是MapReduce

**Metastore**

- 元数据：表名，所属的数据库，表的拥有者，字段，分区，表的类型，数据存储位置等等
- 默认存储在Derby数据库中，实际开发中都使用关系型数据库存储，例如MySQL

# 4.Hive和数据库比较

（1）查询语言——语法上有一些区别，Hive使用的是类SQL查询语言

（2）数据存储位置——Hive是基于hadoop的，数据是存储在HDFS上的，而关系型数据库是将数据存储在本地文件中

（3）数据更新——Hive是基于hadoop的，数据是在HDFS上，HDFS只支持追加和覆盖，不支持修改，所以Hive对Update支持的不好，而关系型数据库是经常会进行修改的

（4）执行引擎——Hive使用MapReduce引擎，关系型数据库有自己的引擎

（5）可扩展性——Hive是基于hadoop的，因此Hive的扩展性和hadoop的扩展性是一致的，2009年已经支持4000台节点，而关系型数据库中，Oracle最多也就支持100多台

