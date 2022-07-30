# Docker Hadoop Workbench

fork 的 git: https://github.com/bambrow/docker-hadoop-workbench

原作者写的MD: https://bambrow.com/20210625-docker-hadoop-1/
## 介绍
fork git 基于 [Docker Compose](https://docs.docker.com/compose/) ，搭建的集群包含以下部分：

- Hadoop
- Hive
- Spark

参考了 [Big Data Europe](https://github.com/big-data-europe) 的一些工作。项目中所使用的 Docker 镜像可能会被更新，可以参看他们的 [Docker Hub](https://hub.docker.com/u/bde2020/) 以获取最新镜像。

本项目所依赖的版本号如下：
```
Client:
 Version:           20.10.2
Server: Docker Engine - Community
 Engine:
  Version:          20.10.6
docker-compose version 1.29.1, build c34c88b2
```

## 快速开始
直接克隆项目并运行集群：

```
./start.sh
```

可以修改 start.sh 与 stop.sh 文件里的 DOCKER_COMPOSE_FILE 变量以使用其他版本的 YAML 文件。


## 集群内容

- Namenode: http://localhost:9870/dfshealth.html#tab-overview
- Datanode: http://localhost:9864/
- ResourceManager: http://localhost:8088/cluster
- NodeManager: http://localhost:8042/node
- HistoryServer: http://localhost:8188/applicationhistory
- HiveServer2: http://localhost:10002/
- Spark Master: http://localhost:8080/
- Spark Worker: http://localhost:8081/
- Spark Job WebUI: http://localhost:4040/ (只有当spark任务在`spark-master`上且是运行着才起作用)
- Presto WebUI: http://localhost:8090/
- Spark History Server：http://localhost:18080/

## 账密
hive 元数据我改成了mysql:

root 520521

hive hive

## 连接

使用 `hdfs dfs` 连接到 `hdfs://localhost:9000/` (请先在本机安装 [Hadoop](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html)):

```
hdfs dfs -ls hdfs://localhost:9000/
```

可以使用 Beeline 连接到 HiveServer2 (请先在本机安装  [Hive](https://cwiki.apache.org/confluence/display/Hive/AdminManual+Installation) ):

```
beeline -u jdbc:hive2://localhost:10000/default -n hive -p hive
```

可以使用 `spark-shell` 通过 thrift 协议连接到 Hive Metastore (请先在本机安装  [Spark](https://spark.apache.org/downloads.html) ):

```
$ spark-shell

Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 3.1.2
      /_/

Using Scala version 2.12.10 (OpenJDK 64-Bit Server VM, Java 11.0.11)

scala> :paste
// Entering paste mode (ctrl-D to finish)

import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder.master("local")
              .config("hive.metastore.uris", "thrift://localhost:9083")
              .enableHiveSupport.appName("thrift-test").getOrCreate

spark.sql("show databases").show


// Exiting paste mode, now interpreting.

+---------+
|namespace|
+---------+
|  default|
+---------+

import org.apache.spark.sql.SparkSession
spark: org.apache.spark.sql.SparkSession = org.apache.spark.sql.SparkSession@1223467f
```

可以使用 [Presto CLI](https://prestodb.io/docs/current/installation/cli.html) 连接 Presto 并且读取 Hive 的数据：

```bash
wget https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.255/presto-cli-0.255-executable.jar
mv presto-cli-0.255-executable.jar presto
chmod +x presto
./presto --server localhost:8090 --catalog hive --schema default
```

## 运行 MapReduce 任务 `WordCount`

这部分基于 [Big Data Europe's Hadoop Docker](https://github.com/big-data-europe/docker-hadoop) 的项目里的运行示例。

首先我们运行一个辅助容器 `hadoop-base` :
```bash
docker run -d --network hadoop --env-file hadoop.env --name hadoop-base bde2020/hadoop-base:2.0.0-hadoop3.2.1-java8 tail -f /dev/null
```

接下来运行以下命令以准备数据并启动 MapReduce 任务：
```bash
docker exec -it hadoop-base hdfs dfs -mkdir -p /input/
docker exec -it hadoop-base hdfs dfs -copyFromLocal -f /opt/hadoop-3.2.1/README.txt /input/
docker exec -it hadoop-base mkdir jars
docker cp jars/WordCount.jar hadoop-base:jars/WordCount.jar
docker exec -it hadoop-base /bin/bash 
hadoop jar jars/WordCount.jar WordCount /input /output
```

接下来，你可以通过以下链接看到任务状态：

[http://localhost:8088/cluster/apps](http://localhost:8088/cluster/apps)

[http://localhost:8188/applicationhistory](http://localhost:8188/applicationhistory) (运行结束后).

当任务运行完成，运行以下命令查看结果：

```bash
hdfs dfs -cat /output/*
```

最后你可以使用 `exit` 退出该容器。


## 运行 Hive 任务

请首先确定 `hadoop-base` 正在运行中。关于如何启动此辅助容器，请参看上一节。接下来准备数据：

```bash
docker exec -it hadoop-base hdfs dfs -mkdir -p /test/
docker exec -it hadoop-base mkdir test
docker cp data hadoop-base:test/data
docker exec -it hadoop-base /bin/bash
hdfs dfs -put test/data/* /test/
hdfs dfs -ls /test
exit
```

然后新建 Hive 表：

```bash
docker cp scripts/hive-beers.q hive-server:hive-beers.q
docker exec -it hive-server /bin/bash
cd /
hive -f hive-beers.q
exit
```

接下来你就可以使用 Beeline 访问到这些数据了：

```
beeline -u jdbc:hive2://localhost:10000/test -n hive -p hive

0: jdbc:hive2://localhost:10000/test> select count(*) from beers;
```

同样，你可以通过以下链接看到任务状态：

[http://localhost:8088/cluster/apps](http://localhost:8088/cluster/apps)

[http://localhost:8188/applicationhistory](http://localhost:8188/applicationhistory) (运行结束后)

## 运行 Spark Shell

在进行这一步前，请先参看前面两个章节以准备 Hive 数据并创建表格。然后运行以下命令：


```
docker exec -it spark-master spark/bin/spark-shell
```

进入 Spark Shell 后，你可以直接通过先前创建的 Hive 表进行操作：

```
scala> spark.sql("show databases").show
+---------+
|namespace|
+---------+
|  default|
|     test|
+---------+

scala> val df = spark.sql("select * from test.beers")
df: org.apache.spark.sql.DataFrame = [id: int, brewery_id: int ... 11 more fields]

scala> df.count
res0: Long = 7822
```

你可以在以下两个地址看到你的 Spark Shell 会话：

http://localhost:8080/ 

http://localhost:4040/jobs/ (运行时)

```
如果你在运行 `spark-shell` 的时候遇到了以下警告:
WARN TaskSchedulerImpl: Initial job has not accepted any resources; check your cluster UI to ensure that workers are registered and have sufficient resources```
该警告显示没有资源可以去运行你的任务，并提醒你去检查 worker 是否都已经被注册，而且拥有足够多的资源。此时你需要使用 docker logs -f spark-master 检查一下 spark-master 的日志。不出意外的话，你会看到下面的内容：

WARN Master: Got heartbeat from unregistered worker worker-20210622022950-xxx.xx.xx.xx-xxxxx. This worker was never registered, so ignoring the heartbeat.
这是在提示你有一个 worker 没有被注册，所以忽略了它的心跳。该 worker 没有被注册的原因很多，很可能是之前电脑被休眠过，导致 worker 掉线。这时你可以使用 docker-compose restart spark-worker 重启 spark-worker，重启完成后，该 worker 就会被自动注册。

同样，如果要运行 spark-sql，可以使用这个命令：docker exec -it spark-master spark/bin/spark-sql。
```
## 运行 Spark Submit 任务

```
bash
docker exec -it spark-master /spark/bin/spark-submit --class org.apache.spark.examples.SparkPi /spark/examples/jars/spark-examples_2.12-3.1.1.jar 100
```

你可以在以下两个地址看到你的 Spark Pi 任务：

http://localhost:8080/

http://localhost:4040/jobs/ (运行时)


## 设置列表

以下列举了容器内部的一些设置所在的位置。后面的以 `conf` 结尾的是它们在 `hadoop.env` 中的代号。你可以参考 `hadoop.env` 文件做额外的设置。

- `namenode`:
  - `/etc/hadoop/core-site.xml` CORE_CONF
  - `/etc/hadoop/hdfs-site.xml` HDFS_CONF
  - `/etc/hadoop/yarn-site.xml` YARN_CONF
  - `/etc/hadoop/httpfs-site.xml` HTTPFS_CONF
  - `/etc/hadoop/kms-site.xml` KMS_CONF
  - `/etc/hadoop/mapred-site.xml` MAPRED_CONF
- `hive-server`:
  - `/opt/hive/hive-site.xml` HIVE_CONF
