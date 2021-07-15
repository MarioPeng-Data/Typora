# 1.Hive安装地址

http://hive.apache.org/

# 2.Mysql安装

## 2.1安装包准备

（1）卸载自带的Mysql-libs（如果之前安装过mysql，要全都卸载掉）

```shell
rpm -qa | grep -i -E mysql\|mariadb | xargs -n1 sudo rpm -e --nodeps
```

（2）将安装包和JDBC驱动上传到/opt/software，共计6个

```shell
mysql-community-common-5.7.29-1.el7.x86_64.rpm 
mysql-community-libs-5.7.29-1.el7.x86_64.rpm 
mysql-community-libs-compat-5.7.29-1.el7.x86_64.rpm 
mysql-community-client-5.7.29-1.el7.x86_64.rpm 
mysql-community-server-5.7.29-1.el7.x86_64.rpm 
mysql-connector-java-5.1.48.jar
```

## 2.2安装Mysql

```shell
#安装mysql依赖 
sudo rpm -ivh mysql-community-common-5.7.29-1.el7.x86_64.rpm 
sudo rpm -ivh mysql-community-libs-5.7.29-1.el7.x86_64.rpm 
sudo rpm -ivh mysql-community-libs-compat-5.7.29-1.el7.x86_64.rpm 
#安装mysql-client 
sudo rpm -ivh mysql-community-client-5.7.29-1.el7.x86_64.rpm 
#安装mysql-server 
sudo rpm -ivh mysql-community-server-5.7.29-1.el7.x86_64.rpm 
#启动mysql 
sudo systemctl start mysqld 
#查看mysql密码 
sudo cat /var/log/mysqld.log | grep password
```

注意：安装MySQL的rpm包是 需要按顺序安装

## 2.3配置MySQL

配置只要是root用户+密码，在任何主机上都能登录MySQL数据库。

```shell
#用刚刚查到的密码进入mysql（如果报错，给密码加单引号） 
mysql -uroot -p’password’ 
#设置复杂密码(由于mysql密码策略，此密码必须足够复杂) 
set password=password("Qs23=zs32"); 
#更改mysql密码策略 
set global validate_password_length=4; 
set global validate_password_policy=0; 
#设置简单好记的密码 
set password=password("000000"); 
#进入msyql库 
use mysql 
#查询user表 
select user, host from user; 
#修改user表，把Host表内容修改为% 
update user set host="%" where user="root"; 
#刷新 
flush privileges; 
#退出 
quit; 
#设置一下开机启动。 
vim /etc/rc.local 
#添加 
service mysqld start
```

扩展：rpm安装默认目录

（1）数据文件：/var/lib/mysql/

（2）配置文件模板：/usr/share/mysql

（3）mysql客户端工具目录：/usr/bin

（4）日志目录：/var/log/

（5）pid，sock文件目录：/tmp/

# 3.Hive安装部署

```shell
#把apache-hive-3.1.2-bin.tar.gz上传到linux的/opt/software目录下 
#解压apache-hive-3.1.2-bin.tar.gz到/opt/module/目录下面 
tar -zxvf apache-hive-3.1.2-bin.tar.gz -C /opt/module/ 
#修改apache-hive-3.1.2-bin.tar.gz的名称为hive-3.1.2 
mv /opt/module/apache-hive-3.1.2-bin /opt/module/hive-3.1.2 
#修改/etc/profile.d/my_env.sh，添加环境变量 
sudo vim /etc/profile 
#添加内容 
#HIVE_HOME export 
HIVE_HOME=/opt/module/hive-3.1.2 
export PATH=$PATH:$HIVE_HOME/bin 
#重启Xshell对话框使环境变量生效 
#解决日志Jar包冲突 
mv $HIVE_HOME/lib/log4j-slf4j-impl-2.10.0.jar $HIVE_HOME/lib/log4j-slf4j-impl-2.10.0.bak
```

# 4.Hive元数据配置到Mysql中

## 4.1拷贝驱动

```shell
#将MySQL的JDBC驱动拷贝到Hive的lib目录下 
mv mysql-connector-java-5.1.48.jar $HIVE_HOME/lib
```

## 4.2配置Metastore到MySql

```shell
#在$HIVE_HOME/conf目录下新建hive-site.xml文件 
vim $HIVE_HOME/conf/hive-site.xml 
#添加如下内容
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://master:3306/metastore?useSSL=false</value>
    </property>
 
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
 
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>
 
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>000000</value>
    </property>
 
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>
 
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
 
    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://master:9083</value>
    </property>
 
    <property>
        <name>hive.server2.thrift.port</name>
        <value>10000</value>
    </property>
 
    <property>
        <name>hive.server2.thrift.bind.host</name>
        <value>master</value>
    </property>
 
    <property>
        <name>hive.metastore.event.db.notification.api.auth</name>
        <value>false</value>
    </property>
 
</configuration>
```

# 5.安装Tez引擎

## 5.1Tez简介

Tez是一个Hive的运行引擎，性能优于MR。为什么优于MR呢？看下图。

<img src="https://gitee.com/peng-bo19951013/Picture/raw/master/20210715214610.png" alt="img" style="zoom: 50%;" />

用Hive直接编写MR程序，假设有四个有依赖关系的MR作业，上图中，绿色是Reduce Task，云状表示写屏蔽，需要将中间结果持久化写到HDFS。

Tez可以将多个有依赖的作业转换为一个作业，这样只需写一次HDFS，且中间节点较少，从而大大提升作业的计算性能。

## 5.2安装Tez

```shell
#将tez安装包拷贝到集群，解压tar包
tar -zxvf apache-tez-0.9.2-bin.tar.gz -C /opt/module/
#重命名
mv /opt/module/apache-tez-0.9.2-bin /opt/module/tez-0.9.2
#上传tez依赖到HDFS
hdfs dfs -mkdir /tez
hdfs dfs -put /opt/module/tez-0.9.2/share/tez.tar.gz /tez
#新建tez-site.xml
vim $HADOOP_HOME/etc/hadoop/tez-site.xml
##添加如下内容：
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<property>
	<name>tez.lib.uris</name>
    <value>${fs.defaultFS}/tez/tez.tar.gz</value>
</property>
<property>
     <name>tez.use.cluster.hadoop-libs</name>
     <value>false</value>
</property>
<property>
     <name>tez.history.logging.service.class</name>
     <value>org.apache.tez.dag.history.logging.ats.ATSHistoryLoggingService</value>
</property>
</configuration>
#修改Hadoop环境变量
##编辑hadoop-env.sh
vim $HADOOP_HOME/etc/hadoop/hadoop-env.sh

##添加Tez的Jar包相关信息
export TEZ_CONF_DIR=$HADOOP_HOME/etc/hadoop
export TEZ_JARS=/opt/module/tez-0.9.2
export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:${TEZ_CONF_DIR}:${TEZ_JARS}/*:${TEZ_JARS}/lib/*

#修改Hive的计算引擎
vim $HIVE_HOME/conf/hive-site.xml
##添加
<property>
    <name>hive.execution.engine</name>
    <value>tez</value>
</property>
#解决日志Jar包冲突
rm /opt/module/tez-0.9.2/lib/slf4j-log4j12-1.7.10.jar
```

# 6.启动Hive

## 6.1初始化元数据库

```shell
#登陆MySQL
mysql -uroot -p000000
#新建Hive元数据库
create database metastore;
quit;
#初始化Hive元数据库
/opt/module/hive-3.1.2/bin/schematool -initSchema -dbType mysql -verbose
```

## 6.2 启动metastore和hiveserver2

```shell
#Hive 2.x以上版本，要先启动这两个服务，否则会报错：
FAILED: HiveException java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
#编写hive服务启动脚本
vim $HIVE_HOME/bin/hiveservices.sh
##内容如下：
#!/bin/bash
HIVE_LOG_DIR=$HIVE_HOME/logs
 
mkdir -p $HIVE_LOG_DIR
 
#检查进程是否运行正常，参数1为进程名，参数2为进程端口
function check_process()
{
    pid=$(ps -ef 2>/dev/null | grep -v grep | grep -i $1 | awk '{print $2}')
    ppid=$(netstat -nltp 2>/dev/null | grep $2 | awk '{print $7}' | cut -d '/' -f 1)
    echo $pid
    [[ "$pid" =~ "$ppid" ]] && [ "$ppid" ] && return 0 || return 1
}
 
function hive_start()
{
    metapid=$(check_process HiveMetastore 9083)
    cmd="nohup hive --service metastore >$HIVE_LOG_DIR/metastore.log 2>&1 &"
    cmd=$cmd" sleep 4; hdfs dfsadmin -safemode wait >/dev/null 2>&1"
    [ -z "$metapid" ] && eval $cmd || echo "Metastroe服务已启动"
    server2pid=$(check_process HiveServer2 10000)
    cmd="nohup hive --service hiveserver2 >$HIVE_LOG_DIR/hiveServer2.log 2>&1 &"
    [ -z "$server2pid" ] && eval $cmd || echo "HiveServer2服务已启动"
}
 
function hive_stop()
{
    metapid=$(check_process HiveMetastore 9083)
    [ "$metapid" ] && kill $metapid || echo "Metastore服务未启动"
    server2pid=$(check_process HiveServer2 10000)
    [ "$server2pid" ] && kill $server2pid || echo "HiveServer2服务未启动"
}
 
case $1 in
"start")
    hive_start
    ;;
"stop")
    hive_stop
    ;;
"restart")
    hive_stop
    sleep 2
    hive_start
    ;;
"status")
    check_process HiveMetastore 9083 >/dev/null && echo "Metastore服务运行正常" || echo "Metastore服务运行异常"
    check_process HiveServer2 10000 >/dev/null && echo "HiveServer2服务运行正常" || echo "HiveServer2服务运行异常"
    ;;
*)
    echo Invalid Args!
    echo 'Usage: '$(basename $0)' start|stop|restart|status'
    ;;
esac
#添加执行权限
chmod +x $HIVE_HOME/bin/hiveservices.sh
#启动Hive后台服务
hiveservices.sh start
```

## 6.3Hive JDBC访问

```shell
#启动beeline客户端
beeline -u jdbc:hive2://slave1:10000 -n mario
#看到如下界面
Connecting to jdbc:hive2://hadoop102:10000
Connected to: Apache Hive (version 3.1.2)
Driver: Hive JDBC (version 3.1.2)
Transaction isolation: TRANSACTION_REPEATABLE_READ
Beeline version 3.1.2 by Apache Hive
0: jdbc:hive2://hadoop102:10000>
#测试
create table student(id int, value string);
insert into table student values(1004,"zhangliu");
select count(*) from student ;
```

