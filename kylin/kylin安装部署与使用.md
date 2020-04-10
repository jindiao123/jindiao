# kylin安装部署与使用

## 1、部署环境

| 环境      | 版本            |
| --------- | --------------- |
| Linux系统 | centos-7.6.1810 |
| ambari    | ambari-2.6.2.2  |
| kylin     | kylin-2.6.5     |
| nginx     | nginx-1.17.9    |
| pcre      | pcre-8.43       |
|           | 1.10.3-10       |

## 2、单机部署

### 2.1、安装包下载

kylin安装包：https://mirror.bit.edu.cn/apache/kylin/apache-kylin-2.6.5/apache-kylin-2.6.5-bin-hbase1x.tar.gz

:a: 提示：#后面的命令表示使用root用户执行的命令，$后面的命令代表使用kylin用户执行的命令。

### 2.2、创建kylin用户

:a:注意：由于yarn会把任务发送到所有节点上，所有所有节点上都要创建kylin用户​

```
# groupadd kylin
# useradd  -g kylin kylin
# passwd kylin
```

### 2.3、创建安装目录并赋予用户对目录有读写权限：

```
mkdir /opt/kylin
chown -R kylin:kylin /opt/kylin/
```

### 2.4、解压安装包：

```
$ tar -zxf /opt/kylin/apache-kylin-2.6.5-bin-hbase1x.tar.gz -C /opt/kylin
```

### 2.5、创建软连接：

```
$ ln -s apache-kylin-2.6.5-bin-hbase1x/ kylin
```

### 2.6、配置环境变量：

```
$ vim ~/.bash_profile 

末尾添加：
export KYLIN_HOME=/opt/kylin/apache-kylin-2.6.5-bin-hbase1x

PATH=$PATH:$KYLIN_HOME/bin

让环境变量生效：
$ source ~/.bash_profile 
```

### 2.7、在kerberos中创建kylin用户并

```
# kadmin.local													进入kerberos命令行

kadmin.local:  addprinc kylin	
WARNING: no policy specified for kylin@LW.COM; defaulting to no policy
Enter password for principal "kylin@LW.COM": 					输入密码
Re-enter password for principal "kylin@LW.COM": 				确认密码
Principal "kylin@LW.COM" created.
kadmin.local:  xst -norandkey -k kylin.keytab kylin@LW.COM		生成keytabs文件

# scp /etc/security/keytabs/kylin.keytab bdnode12:/etc/security/keytabs/
														将密钥文件拷贝到kylin所部署的节点上
```

### 2.8、给kylin赋予hive和hbase权限

```
//进入ranger UI 页面，给kylin添加访问hbase和hive的权限

$ kinit -kt /etc/security/keytabs/kylin.keytab kylin		登录kerberos的kylin用户

验证hbase权限成功：
$ hbase shell
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 1.1.2.2.6.5.0-292, r897822d4dd5956ca186974c10382e9094683fa29, Fri May 11 08:00:59 UTC 2018
hbase(main):001:0> list
TABLE                                                          
wifi_terminal_track.20191013                                                    
wifi_terminal_track.20191101                                        
wifi_terminal_track.20191106                                       
wifi_terminal_track.20191107                                        
4 row(s) in 0.8630 seconds

验证hive权限成功
$ hive
log4j:WARN No such property [maxFileSize] in org.apache.log4j.DailyRollingFileAppender.
Logging initialized using configuration in file:/etc/hive/2.6.5.0-292/0/hive-log4j.properties
hive> show databases;
OK
default
es
Time taken: 3.849 seconds, Fetched: 2 row(s)

```

### 2.9、起用服务

```
$ check-env.sh  					查看是否配置正确

$ kylin.sh start					启动成功
.....................................
A new Kylin instance is started by kylin. To stop it, run 'kylin.sh stop'
Check the log at /opt/kylin/apache-kylin-2.6.5-bin-hbase1x/logs/kylin.log
Web UI is at http://bdnode12:7070/kylin

访问UI页面：ip:7070/kylin
```

### 2.10、停用服务

```
$ kylin.sh stop
Stopping Kylin: 569
Kylin with pid 569 has been stopped
```

## 3、集群部署

### 3.1、节点分配

| 节点     | 属性  |
| -------- | ----- |
| bdnode12 | all   |
| bdnode13 | query |
| bdnode15 | query |
| bdnode16 | query |
| bdnode17 | query |
| bdnode18 | query |



### 3.2、安装负载均衡器

为了将查询请求发送给集群而非单个节点，可以部署一个负载均衡器，如 [Nginx](http://nginx.org/en/)， [F5](https://www.f5.com/) 或 [cloudlb](https://rubygems.org/gems/cloudlb/) 等，使得客户端和负载均衡器通信代替和特定的 Kylin 实例通信。

#### 3.2.1、下载安装包和依赖包

nginx安装包：http://nginx.org/download/nginx-1.17.9.tar.gz

pcre依赖包：https://nchc.dl.sourceforge.net/project/pcre/pcre/8.43/pcre-8.43.tar.gz

openssl依赖包：https://www.openssl.org/source/old/1.1.1/openssl-1.1.1d.tar.gz

zlib依赖包：https://www.zlib.net/zlib-1.2.11.tar.gz

#### 3.2.2、解压安装包

```
[kylin@bdnode12 opt]$ tar -zxf pcre-8.43.tar.gz -C kylin/
[kylin@bdnode12 opt]$ tar -zxf zlib-1.2.11.tar.gz -C kylin/
[kylin@bdnode12 opt]$ tar -zxf openssl-1.1.1d.tar.gz -C kylin/
[kylin@bdnode12 opt]$ tar -zxf nginx-1.17.9.tar.gz -C kylin/
```

#### 3.2.3、编译环境

pcre编译

```
# ./configure 
# make && make install
# pcre-config --version
8.43
```

zlib编译

```
# ./configure 
# make && make install
```

openssl编译

```
# ./Configure darwin64-x86_64-cc --prefix=/opt/kylin/openssl-1.1.1d
# make && make install
```

nginx编译

```
# ./configure \
--prefix=/opt/kylin/nginx-1.17.9 \
--error-log-path=/opt/kylin/nginx-1.17.9/logs/error.log \
--http-log-path=/opt/kylin/nginx-1.17.9/logs/access.log \
--with-pcre=/opt/kylin/pcre-8.43 \
--with-zlib=/opt/kylin/zlib-1.2.11 \
--with-openssl=/opt/kylin/openssl-1.1.1d \
--with-http_ssl_module \
--with-stream \
--without-http_empty_gif_module \
--with-mail=dynamic 

# make & make install

```

启动nginx：

```
bin/nginx						出现下面的页面：安装nginx成功
```

![image-20200409180713216](E:\笔记\kylin\img\image-20200409180713216.png)

### 3.3、集群搭建

修改kylin.properties

```
kylin.server.mode=all								### 主节点为all 其他为query
kylin.server.cluster-servers=bdnode12:7070,bdnode13:7070,bdnode15:7070,bdnode16:7070,bdnode17:7070,bdnode18:7070
#### 使用中的服务器列表，这使一个实例可以在发生事件更改时通知其他服务器
kylin.rest.servers=bdnode12:7070,bdnode13:7070,bdnode15:7070,bdnode16:7070,bdnode17:7070,bdnode18:7070

### 使用多任务引擎，你可以在多个 Kylin 节点上配置它的角色为 job 或 all。为了避免它们之间产生竞争，需要启用分布式任务锁
kylin.job.scheduler.default=2
kylin.job.lock=org.apache.kylin.storage.hbase.util.ZookeeperJobLock
```

分发kylin.properties到每个节点

### 3.4、启动

1）在每个节点登陆kerberos用户

赋权给kylin用户运行权限

 chown kylin:kylin /etc/security/keytabs/kylin.keytab 

:a:为保证以后不报权限异常，设置定时执行任务；

```
crontab -e

* */1 * * *  kinit -kt /etc/security/keytabs/kylin.keytab kylin


crontab -l 				列出任务列表
```



2）启动：

```
[kylin@bdnode13 root]$ kylin.sh start
Retrieving hadoop conf dir...
KYLIN_HOME is set to /opt/kylin/apache-kylin-2.6.5-bin-hbase1x
(No info could be read for "-p": geteuid()=1019 but you should be root.)
Retrieving hive dependency...
find: failed to restore initial working directory: 权限不够
Retrieving hbase dependency...
Retrieving hadoop conf dir...
Retrieving kafka dependency...
Retrieving Spark dependency...
spark not found, set SPARK_HOME, or run bin/download-spark.sh
```

:a:提示，因为集群是使用ambari服务部署的，会自动加载各配置文件，如果异常，就添加该配置的环境变量

vim ~/.bashrc

```
export KYLIN_HOME=/opt/kylin/apache-kylin-2.6.5-bin-hbase1x
export SPARK_HOME=/usr/hdp/current/spark2-client

PATH=$PATH:$KYLIN_HOME/bin:$SPARK_HOME/bin
```

source ~/.bashrc

再次启动

```
[kylin@bdnode13 root]$ kylin.sh start
.........................
A new Kylin instance is started by kylin. To stop it, run 'kylin.sh stop'
Check the log at /opt/kylin/apache-kylin-2.6.5-bin-hbase1x/logs/kylin.log
Web UI is at http://bdnode13:7070/kylin
```

### 3.5、官网案例测试

1. 运行 `${KYLIN_HOME}/bin/sample.sh`；重启 Kylin 服务器刷新缓存;

2. 用默认的用户名和密码 ADMIN/KYLIN 登陆 Kylin 网站，选择 project 下拉框（左上角）中的 `learn_kylin` 工程;

3. 选择名为 `kylin_sales_cube` 的样例 Cube，点击 “Actions” -> “Build”，选择一个在 2014-01-01 之后的日期（覆盖所有的 10000 样例记录);

4. 点击 “Monitor” 标签，查看 build 进度直至 100%;

5. 点击 “Insight” 标签，执行 SQLs，例如:

   ```
   select part_dt, sum(price) as total_sold, count(distinct seller_id) as sellers from kylin_sales group by part_dt order by part_dt
   ```


![image-20200409200352192](F:\Typora\resources\app\Docs\img\image-20200409200352192-1586487647100.png)



![image-20200409200419960](F:\Typora\resources\app\Docs\img\image-20200409200419960.png)

## 4、hive库案例测试

### 4.1、kylin创建project

![image-20200410111214320](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410111214320.png)

2）设置项目名

![image-20200410111311107](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410111311107.png)

### 4.2、建立数据源（data source）

1）创建数据源

![image-20200410111810387](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410111810387.png)

数据加载方式介绍：

```
（1）第一个图标：Load Hive Table
手工填入需要加载的Hive列表，表之间用逗号分隔,单击Sync同步表信息

（2）第二个图标：Load Hive Table From Tree
单击数据库kylin_flat_db后列出库下面所有的表，然后单击需要同步的表即可，最后单击Sync同步

（3）最后一个图标：Add Streaming Table
添加实时数据流的表，格式必须是Json格式的。从1.5.2版本开始，官网给出了在Kylin中基于Kafka定义Streaming Table，从而完成准实时Cube的构建
```

2）选择需要的数据源

![Kylin中建立数据模型（Model）](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410112407709.png)

3）加载成功

![image-20200410113208674](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410113208674.png)

### 4.3、Kylin中建立数据模型（Model）

1）创建model

![image-20200410113448032](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410113448032.png)

2）输入名称和介绍

![image-20200410113952992](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410113952992.png)





3）选择要加载的表

![image-20200410114303064](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410114303064.png)3）关联列

![image-20200410114546847](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410114546847.png)

4）模型维度

![image-20200410114704883](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410114704883.png)

5）加载指标

![image-20200410115100364](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410115100364.png)

如果分区字段为空，那么每次全量刷新Cube。

如果你分区的字段的值，比如日期和时间是分开的，那么还需要指定额外独立的时间字段，如图9-15所示，填写好字段和时间格式。

![image-20200410115313083](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410115313083.png)

在创建Model的过程中，有几个步骤的点需要说明：

- Model Name：全局唯一。
- Data Model：星型模型，一个事实表和多个维度表。
- Dimensions：选择事实表和维度表的维度，这里设置的维度表在新建Cube时使用。
- Measures：设置度量，只能来自事实表。
- Settings：如果事实表是日增量数据，Partition Date Column可以选择事实表的日期分区字段。在Kylin 1.5.2版本中新增加小时分区设置filter条件，用于对表中的数据进行过滤。

### 4.4、模型检查

单击模型名称，显示下面视图

这里有三种方式查看，分别为Grid（表格）、Visualization（可视化）、JSON（JSON格式）。

（1）Grid（表格）：刚才创建Model的整个过程

（2）Visualization（可视化）：可视化事实表和维度表的关联

（3）JSON（JSON格式）：以JSON格式配置了Model模型中的表关联、维度字段、度量字段、Cube刷新方式、过滤条件等

![image-20200410115857454](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410115857454.png)

### 4.5、Kylin中建立Cube

1）选择new cube

![image-20200410141456498](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410141456498.png)

2）给cube命名

![image-20200410141817595](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410141817595.png)

3）添加维度

![image-20200410142105713](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410142105713.png)

4）优化策略

![image-20200410142645249](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410142645249.png)

- Auto Merge Thresholds：根据自己的需求，设计merge策略。
- Retention Threshold：默认为0，保留所有历史的Cube Segments。当然你也可以设置保留最新的多少天Cube Segments。
- Partition Start Date：Cube增量刷新的开始时间，根据你业务需求设计从哪一天开始计算计算Cube。

5）高级设置

![image-20200410143208277](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410143208277.png)

5）单击“+Property”，可以设置Cube级别的参数值，此处配置的参数值将覆盖kylin.prperties文件中的值。

![image-20200410143252944](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410143252944.png)

6）建成

![image-20200410143742663](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410143742663.png)

上面用方框标记出来的每一步都可以单击查看，如果发现Cube有问题的话，可以单击Action中的Edit进行修改，当然如果你觉得对这个Cube设计有问题的话，那也可以从Action中使用Drop删除。

### 4.6、执行

![image-20200410144040291](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410144040291.png)

![image-20200410144143611](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410144143611.png)

Action下拉框选项显示Cube操作有：

（1）Drop

删除此Cube。

（2）Edit

如果发现Cube设计有问题，可以选择Edit进行修改。

（3）Build

执行构建Cube操作，如果是增量Cube，则需要指定开始和结束时间，这两个时间区间标识本次构建的segment的数据源只选择这个时间范围内的数据。对于Build操作而言，startTime是不需要的，因为它总是会选择最后一个segment的结束时间作为当前segment的起始时间。

由于Kylin基于预计算的方式提供数据查询，构建操作是指将原始数据（存储在Hadoop中，通过Hive获取）转换成目标数据（存储在HBase中）的过程。

（4）Refresh

对某个已经构建过的Cube Segment，重新从数据源抽取数据并构建，从而获得更新。

（5）Merge

对于增量Cube，即设置分区字段，这样的Cube就可以进行多次Build，每一次的Build会生成一个segment，每一个segment对应着一个时间区间的Cube，这些segment的时间区间是连续并且不重合的，对于拥有多个segment的cube可以执行merge，相当于将一段时间区间内部的segment合并成一个，可以减少Segment的数量，同时减少Cube的存储空间。

（6）Enable

使Cube生效。如果Cube处于disabled状态时改变Cube Schema，那么Cube的所有segments将因为Data和Schema不匹配而被丢弃。

（7）Disabled

使Cube失效，此时无法再通过SQL查询Cube数据。如果再执行Enable的话，就可以继续查询了。

（8）Purge

将Cube的所有Cube Segment删除。

（9）Clone

如果我们想保留原先已经创建好的Cube，但是又想创建一个类似的Cube，那么此时就可以使用Clone功能重新克隆一个一模一样的Cube，然后对这个Cube进行修改等操作。

### 4.7、查看执行状态

1）在Monitor中查看状态

![image-20200410145122852](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410145122852.png)

Monitor中的Job状态显示为FINISHED，并且Progress显示为100%。

![image-20200410145055931](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410145055931.png)

这里补充一下Job的几种状态：

- NEW：新任务，刚刚创建。
- PENDING：等待被调度执行的任务。
- RUNNING：正在运行的任务。
- FINISHED：正常完成的任务（终态）。
- ERROR：执行出错的任务。
- DISCARDED：丢弃的任务（终态）。

2）Cube的状态显示为Ready了

![image-20200410145246082](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410145246082.png)

3）Build Cube完成后，我们可以从Cube的详细信息中看到HBase的信息

![image-20200410145552209](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410145552209.png)

hbase可以看到表

![image-20200410145709661](../../%E7%AC%94%E8%AE%B0kylinimg/image-20200410145709661.png)