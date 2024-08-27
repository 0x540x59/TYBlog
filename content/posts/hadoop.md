---
ShowToc: true
date: "2024-08-27T09:26:50+08:00"
draft: false
tags:
- 大数据
- hadoop
title: Hadoop单机测试环境
---

有时需要测试一些大数据组件，只需要hadoop+hive基础环境，云厂商的EMR往往都是2~3个节点起步，有些还master，core，task各种角色节点必须齐备。不仅贵，集群改起配置来还麻烦。记录一下单机测试环境搭建。

# 系统环境

## JDK

```shell
curl -LO https://corretto.aws/downloads/latest/amazon-corretto-8-x64-linux-jdk.tar.gz

tar -xzvf amazon-corretto-8-x64-linux-jdk.tar.gz -C /usr/lib/jvm/
```

## 用户

```shell
useradd hadoop
su - hadoop

# 下面涉及到的文件目录权限都给hadoop用户
chown -R hadoop:hadoop /usr/local/hadoop
chown -R hadoop:hadoop /usr/local/hive
chown -R hadoop:hadoop /data/hadoop
```

## 主机信任

```shell
# 生成一个 SSH 密钥对
ssh-keygen -t rsa -P "" -f ~/.ssh/id_rsa
# 将公钥复制到授权密钥文件中
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
# 设置合适的权限
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
# 测试
ssh localhost date
```

## 环境变量

```shell
# /home/hadoop/.bashrc
export JAVA_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto/"
export HADOOP_HOME="/usr/local/hadoop"
export HADOOP_CONF_DIR="$HADOOP_HOME/etc/hadoop"
export HIVE_HOME="/usr/local/hive"
export HIVE_CONF_DIR="$HIVE_HOME/conf
export HIVE_AUX_JARS_PATH="$HIVE_HOME/lib"
export PATH="$JAVA_HOME/bin:$HADOOP_HOME/bin:$HIVE_HOME/bin:$PATH"
```
# hadoop-3.2.2

## 配置文件

```xml
<!-- /usr/local/hadoop/etc/hadoop/core-site.xml -->
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>

    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data/hadoop</value>
    </property>

    <property>
        <name>hadoop.proxyuser.root.hosts</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.root.groups</name>
        <value>*</value>
    </property>

    <property>
        <name>hadoop.proxyuser.hadoop.hosts</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.hadoop.groups</name>
        <value>*</value>
    </property>
</configuration>
```

## 格式化 HDFS

```shell
hdfs namenode -format
```

## 启动服务

```shell
start-dfs.sh
start-yarn.sh
```

# hive-3.1.2
## 配置文件

```shell
# /usr/local/hive/conf/hive-env.sh
export JAVA_HOME="/usr/lib/jvm/java-1.8.0-amazon-corretto/"
export HADOOP_HOME="/usr/local/hadoop"
export HADOOP_CONF_DIR="$HADOOP_HOME/etc/hadoop"
export HIVE_HOME="/usr/local/hive"
export HIVE_CONF_DIR="$HIVE_HOME/conf
export HIVE_AUX_JARS_PATH="$HIVE_HOME/lib"
```

```xml
<!--/usr/local/hive/conf/hive-site.xml-->
<configuration>
	<!-- 存储元数据mysql相关配置 -->
	<property>
	 <name>javax.jdo.option.ConnectionURL</name>
	 <value>jdbc:mysql://localhost:3306/hive_metastore?useSSL=false</value>
	</property>

	<property>
	 <name>javax.jdo.option.ConnectionDriverName</name>
	 <value>com.mysql.cj.jdbc.Driver</value>
	</property>

	<property>
	 <name>javax.jdo.option.ConnectionUserName</name>
	 <value>root</value>
	</property>

	<property>
	 <name>javax.jdo.option.ConnectionPassword</name>
	 <value>root</value>
	</property>

	<!-- H2S运行绑定host -->
	<property>
	    <name>hive.server2.thrift.bind.host</name>
	    <value>localhost</value>
	</property>

	<!-- 远程模式部署metastore metastore地址 -->
	<property>
	    <name>hive.metastore.uris</name>
	    <value>thrift://localhost:9083</value>
	</property>

	<!-- 关闭元数据存储授权  -->
	<property>
	    <name>hive.metastore.event.db.notification.api.auth</name>
	    <value>false</value>
	</property>

	<!-- 关闭元数据存储版本的验证 -->
	<property>
	    <name>hive.metastore.schema.verification</name>
	    <value>false</value>
	</property>
</configuration>
```

## MySQL JDBC驱动

```shell
curl -L https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.28/mysql-connector-java-8.0.28.jar -o /usr/local/hive/lib/mysql-connector-java-8.0.28.jar
```

## 初始化数据库

```shell
schematool -initSchema -dbType mysql
```

## 启动测试

```shell
# 启动metastore
hive --service metastore

# 启动hiveserver2，前台打印日志，方便调试。
hive --service hiveserver2 --hiveconf hive.root.logger=DEBUG,console

# hiveserver2启动时间较长，确保端口监听正常后再开始后面操作。
ss -tnlp | grep 9083
ss -tnlp | grep 10000
```

## beeline连接

```shell
beeline -u 'jdbc:hive2://localhost:10000' -n 'hadoop'
beeline> show databases;
beeline> create table employee (id int, name string, tel string);
beeline> show tables;
beeline> insert into employee values (1, 'jack', '13400008888');
beeline> select * from employee;
beeline> !q
```

## 已知问题

启动hive时报如下错误，是因为hive内依赖的**guava.jar**和hadoop内的**版本不一致**造成的。解决方案时，删除版本低的，拷贝高版本的，使其保持一致。

```shell
Exception in thread "main" java.lang.NoSuchMethodError: com.google.common.base.Preconditions.checkArgument(ZLjava/lang/String;Ljava/lang/Object;)V
```

```shell
find $HIVE_HOME/ -name "guava*.jar"
/usr/local/hive/lib/guava-19.0.jar

find $HADOOP_HOME/ -name "guava*.jar"
/usr/local/hadoop/share/hadoop/common/lib/guava-27.0-jre.jar
/usr/local/hadoop/share/hadoop/hdfs/lib/guava-27.0-jre.jar

rm -f /usr/local/hive/lib/guava-19.0.jar
cp /usr/local/hadoop/share/hadoop/common/lib/guava-27.0-jre.jar /usr/local/hive/lib/guava-19.0.jar
```

