---
ShowToc: true
date: "2024-08-27T13:44:48+08:00"
draft: false
title: PrestoDB开启Ranger权限控制
---

# 说明

2.4.0之后编译好的二进制包可以从以下链接直接下载。

https://repo.maven.apache.org/maven2/org/apache/ranger/ranger-distro/

需要用到3个组件
* ranger-admin
* ranger-usersync
* ranger-hive-plugin

1. 可能会问为什么不是ranger-presto-plugin呢？因为prestodb没有单独的Ranger插件，注意是prestodb，这里涉及到一些presto的发展历史，不了解的话可能一头雾水。
>Facebook开发的用来和Hive配合的及席查询引擎(竞品：Impala 、Kylin、SparkSQL)，随后开源，Facebook公司对Presto有很强的控制欲，想要控制它的下一步发展方向，甚至空降项目经理，Presto创世人及其团队集体离职，从此Presto分为PrestoSQL(创始人) & PrestoDB(Facebook控制)，随后 Presto被Facebook注册商标™️了，PrestoSQL无奈只能改为Trino（Dec 27, 2020），[rebranding PrestoSQL as Trino](https://trino.io/blog/2020/12/27/announcing-trino.html)。

2. 可能还有疑问，[ranger官方仓库](https://github.com/apache/ranger.git)里面不是有`plugin-trino`和`plugin-presto`吗？`plugin-trino`是trino插件的话，`plugin-presto`不应该就是prestodb插件吗？
>并不是，从它的pom.xml依赖里面就能看出来，它是prestosql，也就是trino改名之前的版本。

```xml
<presto.version>333</presto.version>
...
<dependency>
	<groupId>io.prestosql</groupId>
	<artifactId>presto-spi</artifactId>
	<version>${presto.version}</version>
</dependency>
<dependency>
	<groupId>io.prestosql</groupId>
	<artifactId>presto-jdbc</artifactId>
	<version>${presto.version}</version>
</dependency>
```

3. 难道就没有人尝试支持一下吗？
> 有人提了一个PR，2019年，至今也没有merge，甚至都没有人review。
> [Adding support for prestodb plugin on ranger by vasveena · Pull Request #49 · apache/ranger (github.com)](https://github.com/apache/ranger/pull/49)

4. 那你为啥一定用prestodb呢，换trino不行吗？
> 引入的早，那个时候还没有trino。trino出来的时候考虑过换，被某架构师否决，说trino看起来代码质量不行。现在两个分支已经相去甚远，迁移的话可能得测试对生产数据有没有影响。

5. 那prestodb有解吗？
>prestodb也有人提issue，官方就是说不支持catalog级别的ranger权限控制，但在0.278之后，支持Ranger-Based Authorization for the Hive connector。
>[prestodb ranger integration error · Issue #17388 · prestodb/presto (github.com)](https://github.com/prestodb/presto/issues/17388)

也就是说只支持hive这一种catalog的权限控制，因为数仓里主要的数据都是在hive里面，这解决了大部分问题，比没有要好。使用方式见如下文档，这也解释为什么要用ranger-hive-plugin。
[Hive Security Configuration - Presto 0.288 Documentation (prestodb.io)](https://prestodb.io/docs/current/connector/hive-security.html#hive-ranger-based-authorization)

其实阿里云这个文档指出了这个路径，很简洁。我相当于是把所有的坑全趟一遍，才了解了来龙去脉，说起来都是泪。
[如何配置Presto启用Ranger权限控制_开源大数据平台 E-MapReduce(EMR)-阿里云帮助中心 (aliyun.com)](https://help.aliyun.com/zh/emr/emr-on-ecs/user-guide/configuring-presto-enable-ranger-permission-control?spm=a2c4g.11186623.0.0.44a01a8eq4aKuh)
[如何集成Hive到Ranger并配置权限_开源大数据平台 E-MapReduce(EMR)-阿里云帮助中心 (aliyun.com)](https://help.aliyun.com/zh/emr/emr-on-ecs/user-guide/enable-hive-in-ranger-and-configure-related-permissions?spm=a2c4g.11186623.0.0.e4141a8eJUJUOH)

# hadoop+hive

[hadoop+hive单机测试环境测试环境]({{< ref "hadoop" >}}  )

# ranger

## elasticsearch
```shell
docker run -d --name elasticsearch --net esnetwork -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.14.2

# 测试
curl -v http://localhost:9200

# 创建索引
curl -X PUT "localhost:9200/ranger-audit" -H 'Content-Type: application/json' -d '{ "settings": { "number_of_shards": 1, "number_of_replicas": 0 } }'
```

## ranger-admin
配置文件`/usr/local/ranger-admin/install.properties`
```properties
# https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.28/mysql-connector-java-8.0.28.jar
SQL_CONNECTOR_JAR=/usr/local/ranger-admin/mysql-connector-java-8.0.28.jar

db_root_user=root
db_root_password=root
db_host=localhost:3306

db_name=ranger
db_user=root
db_password=root

# 审计日志存储位置配置，如果配置为solr,那么solr的配置就会生效，如果配置为elasticsearch,那么es的配置就会生效
audit_store=elasticsearch

# * audit_solr_url Elasticsearch Host(s). E.g. 127.0.0.1
audit_elasticsearch_urls=localhost
audit_elasticsearch_port=9200
audit_elasticsearch_protocol=http
audit_elasticsearch_user=
audit_elasticsearch_password=
audit_elasticsearch_index=ranger-audit
audit_elasticsearch_bootstrap_enabled=true

unix_user=root
unix_user_pwd=...
unix_group=root
```

创建并初始化ranger数据库
```sql
create database ranger default character set utf8;
```

初始化
```shell
su root
./setup.sh
```

启动ranger-admin进程

```shell
sudo ranger-admin start
```

启动成功后，访问http://localhost:6080，用户名和密码均为`admin`

## ranger-usersync
配置文件`/usr/local/ranger-usersync/install.properties`
```properties
POLICY_MGR_URL=http://localhost:6080

unix_user=root
unix_group=root

# 同步间隔, 默认5分钟
SYNC_INTERVAL =
SYNC_SOURCE = ldap
SYNC_LDAP_URL = ldap://freeipa.irisdt.cn:389
SYNC_LDAP_BIND_DN = uid=ldapsearch,cn=users,cn=accounts,dc=irisdt,dc=cn
SYNC_LDAP_BIND_PASSWORD = xxxxxx
SYNC_LDAP_SEARCH_BASE = cn=accounts,dc=irisdt,dc=cn
SYNC_LDAP_USER_SEARCH_BASE = cn=users,cn=accounts,dc=irisdt,dc=cn
SYNC_GROUP_SEARCH_ENABLED=true
SYNC_GROUP_USER_MAP_SYNC_ENABLED=true
SYNC_GROUP_SEARCH_BASE=cn=groups,cn=accounts,dc=irisdt,dc=cn
```
初始化
```shell
su root
./setup.sh
```
配置文件`/etc/ranger/usersync/conf/ranger-ugsync-site.xml`
```xml
<property>
	<name>ranger.usersync.enabled</name>
	<value>true</value>
</property>
```
启动服务
```shell
sudo ranger-usersync start
```

## ranger-hive-plugin
配置文件`/usr/local/ranger-hive-plugin/install.properties`
```properties
POLICY_MGR_URL=http://localhost:6080
REPOSITORY_NAME=hivedev
COMPONENT_INSTALL_DIR_NAME=/usr/local/hive

# 审计相关
XAAUDIT.ELASTICSEARCH.ENABLE=true
XAAUDIT.ELASTICSEARCH.URL=localhost
XAAUDIT.ELASTICSEARCH.USER=NONE
XAAUDIT.ELASTICSEARCH.PASSWORD=NONE
XAAUDIT.ELASTICSEARCH.INDEX=ranger-audit
XAAUDIT.ELASTICSEARCH.PORT=9200
XAAUDIT.ELASTICSEARCH.PROTOCOL=http

CUSTOM_USER=hadoop
CUSTOM_GROUP=hadoop
```
初始化
```shell
./enable-hive-plugin.sh
```

启用`ranger-hive-plugin`之后，重启hiveserver2，发现启动不了，反复尝试连接zookeeper。

## zookeeper-3.6.4
配置文件`/usr/local/zookeeper/conf/zoo.cfg`
```
# 其它的配置使用默认值
# 启动的时候要监听8080端口，如果跟别的服务冲突了，就把admin页面关掉。
zookeeper.admin.enableServer=false
```
启动
```shell
zkServer.sh start
```
# ranger hive plugin
开启`ranger-hive-plugin`之后，再来测试hive看看权限控制是否生效了。在ranger-admin中给配置policy，只允许hadoop用户访问所有资源。

```shell
beeline -u 'jdbc:hive2://localhost:10000' -n 'alice'
0: jdbc:hive2://localhost:10000> show tables;
Error: Error while compiling statement: FAILED: HiveAccessControlException Permission denied: user [alice] does not have [USE] privilege on [default] (state=42000,code=40000)

beeline -u 'jdbc:hive2://localhost:10000' -n 'hadoop'
0: jdbc:hive2://localhost:10000> show tables;
+-----------+
| tab_name  |
+-----------+
| employee  |
+-----------+
1 row selected (0.184 seconds)
```

# presto
配置文件`/opt/presto-server/etc/catalog/hive.properties`
```properties
hive.metastore.uri=thrift://10.120.3.3:9083

hive.security=ranger
hive.ranger.rest-endpoint=http://10.120.3.3:6080
hive.ranger.policy.hive-servicename=hivedev
hive.ranger.service.basic-auth-username=admin
hive.ranger.service.basic-auth-password=admin
```
重启后测试
```shell
presto --catalog hive --user alice
presto> use default;
USE
presto:default> show tables;
Query 20240813_075745_00008_254uj failed: Access Denied: Cannot select from columns [table_schema, table_name] in table or view tables: Access denied - User [ alice ] does not have [SELECT] privilege on all mentioned columns of [ information_schema/tables ]

presto --catalog hive --user hadoop
presto> use default;
USE
presto:default> show tables;
  Table
----------
 employee
(1 row)

Query 20240813_075836_00010_254uj, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0:00 [1 rows, 25B] [4 rows/s, 112B/s]
```
# 注意事项

Hive开启Ranger权限控制后，HiveServer2服务会加载Ranger Hive plugin，仅在通过HiveServer2提交SQL作业时需要进行权限校验，其他方式访问Hive将不会触发权限校验。
- 支持权限校验的访问方式
    - 通过Beeline客户端访问HiveServer2。
    - 通过JDBC URL连接HiveServer2。
- 不支持权限校验的访问方式
    - 通过Hive客户端直接连接Metastore。
    - 通过Hive-Client API直接连接Metastore。

# 参考文档

[Apache Ranger入门与进阶 (yangc.top)](http://www.yangc.top/archives/apache-ranger)

[Row-level filtering and column-masking using Apache Ranger policies in Apache Hive - Ranger - Apache Software Foundation](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65868896)
