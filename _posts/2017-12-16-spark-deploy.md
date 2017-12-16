# Spark运行模式与部署

## 一、运行模式总览

![](https://raw.githubusercontent.com/tina437213/images/master/img/spark%20deploy.jpg)

## 二、各种模式的部署实践

### 2.1 local模式

#### Prerequisites

```
Linux: SUSE Linux (hadoop1 192.168.1.57)
JDK: 1.8
```

#### 2.1.1 local单机模式

注：如果不需要用到hdfs，此模式下无需装hadoop，与本地文件（file:///）交互即可。

1）下载spark安装包，本文采用[spark-2.1.1-bin-hadoop2.7.tgz](https://archive.apache.org/dist/spark/spark-2.1.1/spark-2.1.1-bin-hadoop2.7.tgz) 。

2）上传到自定义目录，如/usr/local/share，解压。(不需要root用户，普通用户也可安装，对应选择有权限的目录即可)

spark local模式有2种运行方式：交互运行模式和run脚本模式。

**<u>交互运行模式</u>**

进入bin目录，运行./spark-shell，即进入local单机模式，看到如下界面

```
hadoop1:/usr/local/share/spark-2.1.1-bin-hadoop2.7/bin # ./spark-shell 
Spark context Web UI available at http://192.168.1.57:4040
Spark context available as 'sc' (master = local[*], app id = local-1509088161365).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.1.1
      /_/
         
Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_144)
Type in expressions to have them evaluated.
Type :help for more information.
scala> 
```

```
./spark-shell   # 默认效果等同于master=local[*]
./spark-shell --master local 	 # 本地以1个worker线程运行(例如非并行的情况).
./spark-shell --master local[2]  # 本地以K worker 线程 (理想情况下, K设置为你机器的CPU核数).
./spark-shell --master local[*]  # 本地以本机同样核数的线程运行.
```

**<u>run脚本模式</u>**

```
hadoop1:~ # cd /usr/local/share/spark-2.1.1-bin-hadoop2.7/bin
hadoop1:/usr/local/share/spark-2.1.1-bin-hadoop2.7/bin # ./run-example SparkPi
```

观察屏幕打印信息中包含：

```
Pi is roughly 3.1411357056785283
```

***【问题：这些执行任务的线程，到底是共享在什么进程中呢？】***

查看提交代码脚本，以上两种方式实际都会调用来提交命令，

``` 
# spark-shell
exec "${SPARK_HOME}"/bin/spark-submit --class org.apache.spark.repl.Main --name "Spark shell" "$@"   
# run-example
exec "${SPARK_HOME}"/bin/spark-submit run-example "$@" 
```

```
hadoop1:~ # jps
42361 Jps
42203 SparkSubmit
```

这个SparkSubmit进程又当爹、又当妈，既是客户提交任务的Client进程、又是Spark的driver程序、还充当着Spark执行Task的Executor角色。（如下图所示：driver的web ui）

![](https://raw.githubusercontent.com/tina437213/images/master/img/spark-submit.jpg)



#### 2.1.2 local-cluster伪分布式(单机模拟集群)

这种运行模式，和Local[N]很像，不同的是，它会在单机启动多个进程来模拟集群下的分布式场景，而不像Local[N]这种多个线程只能在一个进程下委屈求全的共享资源。通常也是用来验证开发出来的应用程序逻辑上有没有问题，或者想使用Spark的计算框架而没有太多资源。

用法是：提交应用程序时使用local-cluster[x,y,z]参数：x代表要生成的executor数，y和z分别代表每个executor所拥有的core和memory数。

```
hadoop1:/usr/local/share/spark-2.1.1-bin-hadoop2.7/bin # ./spark-shell --master "local-cluster[2, 3, 1024]"
Spark context Web UI available at http://192.168.1.57:4040
Spark context available as 'sc' (master = local-cluster[2, 3, 1024], app id = app-20171027220146-0000).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.1.1
      /_/
         
Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_144)
Type in expressions to have them evaluated.
Type :help for more information.

scala> 

 同理./spark-submit --master "local-cluster[2, 3, 1024]"
```

上面这条命令代表会使用2个executor进程，每个进程分配3个core和1G的内存，来运行应用程序。可以看到，在程序执行过程中，会生成如下几个进程：

```
hadoop1:/usr/local/share/spark-2.1.1-bin-hadoop2.7/bin # jps
112274 SparkSubmit
112405 CoarseGrainedExecutorBackend
112404 CoarseGrainedExecutorBackend
129432 Jps
```

SparkSubmit依然充当全能角色，又是Client进程，又是driver程序，还有点资源管理的作用。生成的两个CoarseGrainedExecutorBackend，就是用来并发执行程序的进程。它们使用的资源如下：

![](https://raw.githubusercontent.com/tina437213/images/master/img/spark-local-cluster.jpg)

### 2.2 集群模式

#### 2.2.1 spark standalone

**Standalone**模式需要将Spark复制到集群中的每个节点，然后分别启动每个节点即可；Spark Standalone模式的集群由Master与Worker节点组成，程序通过与Master节点交互申请资源，Worker节点启动Executor运行；

如下图所示，本文启用了3台虚拟机，hadoop1充当Master角色，hadoop2和hadoop3充当worker角色。

![](https://raw.githubusercontent.com/tina437213/images/master/img/spark-standalone-deploy.jpg)

**具体部署步骤：**（第2，3步同local模式）

1）3台机器配置ssh免密码登录(注意需要先配置3台机器的hosts)

安装Openssh server

```
sudo apt-get install openssh-server
```

在3台机器上都生成私钥和公钥

```
ssh-keygen -t rsa   #一路回车,默认会将公钥放在/root/.ssh
```

将公钥复制为authorized_keys文件，此时使用ssh连接本机就不需要输入密码了

```
cd /root/.ssh
cp id_rsa.pub authorized_keys
```

配置三台机器互相之间的ssh免密码登录(以hadoop1为例)

```
ssh-copy-id -i hadoop2  #将本机的公钥拷贝到指定机器hadoop2的authorized_keys文件中
ssh-copy-id -i hadoop3
```

测试（分别在3台机器上ssh 连接另外2台机器，均能免密码登录，以hadoop1为例）

```
ssh hadoop2
ssh hadoop3
```

以下先在hadoop1机器上操作，然后再把安装包拷贝至hadoop2，hadoop3

2） 下载spark安装包，本文采用[spark-2.1.1-bin-hadoop2.7.tgz](https://archive.apache.org/dist/spark/spark-2.1.1/spark-2.1.1-bin-hadoop2.7.tgz) 。

3）上传到自定义目录，如/usr/local/share，解压。(不需要root用户，普通用户也可安装，对应选择有权限的目录即可)

4）配置环境变量

配置spark-env.sh

```
# cd /usr/local/share/spark-2.1.1-bin-hadoop2.7/conf
# mv spark-env.sh.template  spark-env.sh
## 添加以下几行
# vi spark-env.sh
export JAVA_HOME=/usr/jdk64/jdk1.8.0_144
export SPARK_HOME=/usr/local/share/spark-2.1.1-bin-hadoop2.7
export SPARK_MASTER_IP=192.168.1.57
export SPARK_WORKER_MEMORY=1g
```

配置slaves

```
# mv slaves.template  slaves
## 将localhost改为2台worker的地址
# vi slaves
hadoop2
hadoop3
```

将配置好的安装包拷贝至另外2台机器

```
## 将spark部署包scp到hadoop2,hadoop3上
# cd /usr/local/share
# scp spark-2.1.1-bin-hadoop2.7 root@hadoop2:/usr/local/share
# scp spark-2.1.1-bin-hadoop2.7 root@hadoop3:/usr/local/share

## 为使用方便，通常我们会设置spark_home环境变量(hadoop1,hadoop2,hadoop3同理配置)
# vi /etc/profile
export SPARK_HOME=/usr/local/share/spark-2.1.1-bin-hadoop2.7
export PATH=$SPARK_HOME/bin:$PATH
```

启动集群服务

```
## 启动服务
# cd /usr/local/share/spark-2.1.1-bin-hadoop2.7/sbin
# ./start-all.sh
starting org.apache.spark.deploy.master.Master, logging to /usr/local/share/spark-2.1.1-bin-hadoop2.7/logs/spark-root-org.apache.spark.deploy.master.Master-1-hadoop1.out
hadoop2: starting org.apache.spark.deploy.worker.Worker, logging to /usr/local/share/spark-2.1.1-bin-hadoop2.7/logs/spark-root-org.apache.spark.deploy.worker.Worker-1-hadoop2.out
hadoop3: starting org.apache.spark.deploy.worker.Worker, logging to /usr/local/share/spark-2.1.1-bin-hadoop2.7/logs/spark-root-org.apache.spark.deploy.worker.Worker-1-hadoop3.out
## 查看启动的服务
# jps
44490 Master

## 在hadoop2，hadoop3上查看
# jps
61316 Worker
```

5） 打开spark web页面

在浏览器输入 http://hadoop1:8080，能看到spark集群的相关信息。注意：spark master web ui 默认端口为8080，当系统有其它程序也在使用该接口时，启动master时也不会报错，spark自己会改用其它端口，自动端口号加1，但为了可以控制到指定的端口，我们可以自行设置，修改方法：

```
# vi start-master.sh 
SPARK_MASTER_WEBUI_PORT=8081 (原8080)
## 重启服务
# ./stop-all.sh
# ./start-all.sh
```

打开http://hadoop1:8081/，可看到如下页面

![](https://raw.githubusercontent.com/tina437213/images/master/img/spark-master-8081.jpg)

至此，spark standalone模式部署完毕。

#### 2.2.2 spark on yarn

通常企业中采用的模式为：一套hadoop集群（含yarn），各个应用在自己的客户端机器上向集群提交spark作业（yarn-cluster or yarn-client）。所以此模式的部署工作拆解为以下3步：

1）hadoop集群部署（注意集群中并不需要部署spark）

详见【hadoop集群部署】

2）集群外客户端机器上部署hadoop客户端

详见【hadoop集群外客户端部署】

3）集群外客户端机器上部署spark客户端

```
A.安装包部署
	下载spark安装包，本文采用spark-2.1.1-bin-hadoop2.7.tgz。
	上传到自定义目录，如/usr/local/share，解压。
B.修改配置文件
    修改spark-env.sh
    export JAVA_HOME=/usr/jdk64/jdk1.8.0_144
    export SPARK_HOME=/usr/local/share/spark-2.1.1-bin-hadoop2.7
    export HADOOP_HOME=/usr/hdp/current/hadoop-client
    export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
C.设置环境变量
    vi /etc/profile (或者~/.bashrc)
    export SPARK_HOME=/usr/local/share/spark-2.1.1-bin-hadoop2.7
    export PATH=$SPARK_HOME/bin:$PATH
```

补充：【启动history server服务】(作业运行结束后，通过HistoryServer来分析作业运行情况)

#### 2.2.3 spark on mesos

因为实际中较少被使用到，略。