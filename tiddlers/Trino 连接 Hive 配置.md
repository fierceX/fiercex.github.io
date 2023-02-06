## 配置

需要的配置项的内容和值，都可以联系Hive/Hadoop运维人员获得

### 连接配置

在Trino项目下的`etc`文件夹下的`catalog`文件夹下进行配置

如果没有`catalog`文件夹，自己建一个

新建`.properties`后缀的配置文件，示例如下：

```properties

connector.name=hive-hadoop2

hive.metastore.uri=thrift://10.30.10.4:9083

hive.config.resources=core-site.xml,hdfs-site.xml

hive.metastore.authentication.type=KERBEROS

hive.metastore.service.principal=hive/manager01-fcy.hadoop.dztech.com@FCYEXAMPLE.COM

hive.metastore.client.principal=nlp_dev@FCYEXAMPLE.COM

hive.metastore.client.keytab=nlp_dev.keytab

hive.hdfs.authentication.type=KERBEROS

hive.hdfs.trino.principal=nlp_dev@FCYEXAMPLE.COM

hive.hdfs.trino.keytab=nlp_dev.keytab

```

需要修改的分别为：

1. `hive.config.resources`：hdfs地址配置文件

2. `hive.metastore.service.principal`：hive的节点登录信息

3. `hive.metastore.client.principal`,`hive.hdfs.trino.principal`：hive连接用户名

4. `hive.metastore.client.keytab`,`hive.hdfs.trino.keytab`：hive连接的认证文件

  

### 其他配置

**`krb5`认证参数**

在`etc`文件夹下的`jvm.config`增加`krb5`的认证配置文件

```properties

-Djava.security.krb5.conf=krb5.conf

```

**连接端口号**

在`etc`文件夹下的`config.properties`配置文件中，修改连接服务端的端口号

```properties

http-server.http.port=8081

```

## 连接

连接命令参考如下：

```bash

trino --server 10.30.12.12:8081 --catalog hive

```

其中需要注意的是，`catalog`参数，指定的是上面第一步配置的配置文件名称
通过配置多个配置文件，可以实现不同的配置文件对应不同的hive用户，实现权限隔离