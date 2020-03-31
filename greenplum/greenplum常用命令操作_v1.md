# greenplum常用命令操作

## 1、管理工具

中文官网介绍：https://docs.greenplum.cn/6-0/utility_guide/admin_utilities/util_ref.html

英文官网介绍：https://gpdb.docs.pivotal.io/6-4/utility_guide/utility-programs.html

### 1.1、gpcopy

#### 1.1.1、下载

https://network.pivotal.io/products/gpdb-data-copy/

#### 1.1..2、安装

1）解压

```
tar xzvf gpcopy-1-5.tar.gz
# 解压后有两个执行文件：gpcopy、gpcopy_helper
```

2）拷贝

在greenplum master host 节点上将这两个文件拷贝到$GPHOME/bin目录下

```
cp gpcopy $GPHOME/bin
cp gpcopy_helper $GPHOME/bin
```

3）赋予执行权限

```
chmod 755 $GPHOME/bin/gpcopy
chmod 755 $GPHOME/bin/gpcopy_helper
```

4）将文件gpcopy_helper文件拷贝到greenplum segment 的 $GPHOME/bin目录下

```
gpscp -h sdw1 -h sdw2 gpcopy_helper =:/usr/local/greenplum-db-6.0.0/bin
```

5）greenplum segment节点设置读写执行权限在gpadmin用户模式下

```
$ gpssh -h sdw1 -h sdw2 -e 'chmod 755 /usr/local/greenplum-db-6.0.0/bin/gpcopy_helper'
```

#### 1.1.3、介绍

​	1、支持不同greenplum版本之间的迁移，支持将数据从Greenplum 5系统迁移到Greenplum Database 6系统。

​	2、gpcopy界面包括用于转移一个或多个完整数据库或一个或多个数据库表的选项。完整的数据库传输包括数据库架构，表数据，索引，视图，角色，用户定义的函数，资源队列和资源组。如果目标系统中不存在复制的表或数据库， gpcopy 自动创建它，并根据需要创建索引。

​	3、先决条件

```
a、源Greenplum数据库系统和目标Greenplum数据库系统必须已经存在，并且所有主机之间都具有网络访问权限
b、如果要在具有不同版本的Greenplum数据库系统之间复制数据，则每个系统必须具有 gpcopy在本地安装。
```

​	4、源系统和目标系统的局限性

```
a、gpcopy仅从用户数据库传输数据；Postgres， template0和 template1 数据库无法传输。管理员必须手动传输配置文件，并使用以下命令将扩展安装到目标数据库中：gppkg。
b、gpcopy 无法复制大于1GB的行。
c、gpcopy 目前不支持SSL加密的连接。
d、gpcopy当复制由作为外部表的叶表定义的分区表，或者如果叶表是由与根分区表不同的分发策略定义的，则不支持表数据分发检查。您可以将这些表复制到gpcopy 操作并指定 --no-distribution-check 选项禁用数据分发检查。
```

#### 1.1.4、命令讲解

```
gpcopy
   { {-F | --full} | 此选项将所有用户定义数据库的所有数据库对象（包括表，表数据，索引，视图，用户，角色，函数和资源队列）复制到其他目标系统。
     { {-d | --dbname} database1[, database2 ... ] | 要复制的源数据库。要将多个数据库复制到目标系统，请指定以逗号分隔的数据库列表，名称之间不得有空格。所有用户定义的表和表数据都将复制到目标系统。如果源数据库不存在， gpcopy返回错误并退出。如果目标数据库不存在，则会创建一个数据库。
        {-D | --dest-dbname} dest-db1[, dest-db2 ... ] ] } | 目标数据库的名称。对于多个数据库，请指定以逗号分隔的数据库列表，名称之间不得有空格。数据库名称的数量必须与在数据库中指定的名称的数量匹配 --dbname选项。该实用程序按列出的顺序将源数据库复制到目标数据库。
     {-t | --include-table} db.schema.table[, db.schema1.table1 ... ] | 从源数据库系统复制一个或多个表。您必须提供完全限定表名（database.schema.table），您不能指定实例化视图或系统目录表。对于依赖于其他表的表，还必须指定从属表。要复制多个表，请包含以逗号分隔的表名列表，或使用正则表达式来描述一组表。模式和表部分中使用Go语言正则表达式来定义一组输入表。正则表达式模式必须用斜杠（/ RE_pattern /）
        [ --dest-table db.schema.table[, db.schema1.table1 ... ] | （可选）更改数据库，架构或表，
     {-T | --include-table-file} table-file1 | 定义要复制的表和数据的文本文件的位置和名称。要使用多个文件，请为每个文件指定此选项。
        [{-T | --include-table-file} table-file2] ... ] | 
     --include-table-json json-table-file1 | 定义要复制的表和数据的JSON格式文件的位置和名称。
        [ --include-table-json json-table-file2] ... ] }
   [ {-m | --metadata-only} ] | 仅创建命令指定的架构。数据不传输。
   [ {-e | --exclude-table} db.schema.table[, db.schema1.table1 ... ] ] | 来自源数据库系统的要从传输中排除的表。必须指定标准表名（database.schema.table）。要排除多个表，请指定以逗号分隔的表名列表。
   [ {-E | --exclude-table-file} table-file1 ] 
      [ {-E | --exclude-table-file} table-file2 ] ... ] ]

   { --dest-host dest_host [ --dest-port dest_port ] | 目标Greenplum数据库主段主机名或IP地址。目标Greenplum数据库主网段端口号。如果 -目标端口 未指定，则默认值为5432。
      [ --dest-user dest_user ] [ --dest-mapping-file host_ip_map_file ]} | 用于连接到目标Greenplum主站的用户ID。如果未指定，则默认值为gpadmin。
   [ --source-host source_host [ --source-port source_port ] | 源Greenplum数据库主段主机名或IP地址。如果未指定，则默认主机为正在运行的系统gpcopy （127.0.0.1）。源Greenplum数据库主端口号。如果未指定，则默认值为5432。
   [ --source-user source_user ] ] | 用于连接到源Greenplum数据库系统的用户ID。如果未指定，则默认值为gpadmin。
   [ --enable-receive-daemon = {true | false }]
   [ --jobs int ] | 可处理的最大数量 gpcopy并行运行。默认值为4。范围是1到64512。产生 2 * n +1数据库连接。默认值为4，创建9个连接。
   [ {-o | --on-segment-threshold} int ] | 指定确定何时执行的行数 gpcopy 使用Greenplum数据库源和目标主数据库而不是源和目标段实例复制表。默认值为10000行。如果一个表包含10000行或更少的行，则使用Greenplum数据库主表复制该表。
   [ {-p | --parallelize-leaf-partitions} = { true | false} ] 基于根分区表并行或作为单个表复制分区表的叶分区表。默认是 true，并行复制叶子分区表。要将分区表复制为单个表，请将此选项设置为 false。
   [ --data-port-range lower_port-upper_port ]

   { --skip-existing | --truncate | --drop | --append } 如果此表已存在于目标数据库中，请指定此选项以跳过从源数据库复制表的操作。
   [ {-a | --analyze} ] 跑过 分析非系统表上的命令。默认是不运行分析命令。复制表数据后，将对每个表执行该操作
   [ --no-compression ] | 如果指定，数据将不压缩地传输。默认情况下， gpcopy 将数据复制到其他主机时，在从源数据库到目标数据库的传输过程中压缩数据。
   [ --no-distribution-check ] | 指定此选项可禁用表数据分发检查。默认情况下， gpcopy执行数据分发检查，以确保将数据正确分发到细分实例。如果分发检查失败，则表副本将失败。
   [ --truncate-source-after [--yes ] ]
   [ {-v | --validate} type ] | 复制表数据后，对目标数据库中的表数据执行数据验证。这些是受支持的验证类型。

   [ --dry-run ] | 指定此选项时， gpcopy生成将使用指定选项执行的迁移操作的列表。数据不迁移。该信息显示在命令行中，并写入日志文件。
   [ --dumper "utility" ]
   [ --quiet | --debug ]

gpcopy --version
gpcopy {-h | --help}
```

#### 1.1.5、例子

此命令使用以下命令将源系统中所有用户创建的数据库复制到目标系统： - full 选项。如果目标中已经存在该表，则将其删除并再次创建它。

```
gpcopy --source-host mytest --source-port 1234 --source-user gpuser \ 
    --dest-host demohost --dest-port 1234 --dest-user gpuser \ 
    --full --drop
```

从指定数据库迁移目标数据库，添加校验，并指定并行数，

```
gpcopy --source-host 192.168.158.111 --source-port 5432 --source-user gpadmin \
   --dest-host 192.168.158.111 --dest-port 5432 --dest-user testuser \
   --dbname testdb --dest-dbname test \
   --jobs 9 --truncate --analyze --validate count
```

指定目标表文件迁移到目标表

```
gpcopy --source-host 192.168.158.111 --source-port 5432 --source-user gpadmin \
   --dest-host 192.168.158.111 --dest-port 5432 --dest-user testuser \
   --include-table-file /home/gpadmin/xxx --jobs 7 \
   --jobs 9 --truncate --analyze --validate count
```

### 1.2、gpconfig 配置属性

#### 1.2.1、命令讲解

```
gpconfig  -c  PARAM_NAME  通过将新设置添加到服务器底部来更改配置参数设置 postgresql.conf 文件。
		-v  值 您通过以下命令指定的配置参数要使用的值，值将应用于所有段，它们的镜像，主服务器和备用主服务器。
		[ -m  master_value | 此值仅适用于主服务器和备用主服务器
		--masteronly ]  gpconfig 只会编辑母版 postgresql.conf 文件
       | -r  param_name [ --masteronly ]
       | -l 列出受支持的所有配置参数 gpconfig 效用
       [ --skipvalidation ] [ --verbose ] [ --debug ]

gpconfig  -s  param_name [ --file | --file-compare ] [ --verbose ] [ --debug ]

gpconfig  --help
```

#### 1.2.2、例子

设置 max_connections 在所有网段上设置为100，在主网段上设置为10：

```
gpconfig -c max_connections -v 100 -m 10
```

注释掉所有实例 default_statistics_target 配置参数，并恢复系统默认值：

```
gpconfig -r default_statistics_target
```

列出受支持的所有配置参数 gpconfig：

```
gpconfig -l
```

显示整个系统中特定配置参数的值：

```
gpconfig -s max_connections
```

### 1.3、gpcheckperf

#### 1.3.1、命令讲解

```
gpcheckperf -d test_directory [-d test_directory ...] 对于磁盘I / O测试，指定要测试的文件系统目录位置。您必须对性能测试所涉及的所有主机上的测试目录具有写访问权
    {-f hostfile_gpcheckperf 对于磁盘I / O和流测试，请指定文件名称，该文件的每个主机包含一个主机名，该主机名将参与性能测试。
    | - h hostname [-h hostname ...]} 
    [-r ds] 
    [-B block_size] 指定用于磁盘I / O测试的块大小（以KB或MB为单位）。默认值为32KB，与Greenplum数据库页面大小相同。最大块大小为1 MB。
    [-S file_size] [-D] [-v|-V]

gpcheckperf -d temp_directory 
    {-f hostfile_gpchecknet |为了进行网络性能测试，主机文件中的所有条目都必须针对同一子网内的主机地址。
    [ -r n|N|M 磁盘I / O测试（d）、流测试（s）、依次进行网络性能测试（ñ）
    [--duration time] [--netperf] ] [-D]  报告每个主机的磁盘I / O测试性能结果。
    [-v | -V] 详细信息

gpcheckperf -?

gpcheckperf --version
```

#### 1.3.2、例子

使用/ data1和/ data2的测试目录 在文件host_file中的所有主机上运行磁盘I / O和内存带宽测试 ：

```
gpcheckperf -f hostfile_gpcheckperf -d /data1 -d /data2 -r ds
```

使用/ data1的测试目录仅在名为sdw1和 sdw2的主机上运行磁盘I / O测试。显示单个主机结果并以详细模式运行：

```
gpcheckperf -h sdw1 -h sdw2 -d / data1 -rd -D -v
```

### 1.4、gpaddmirrors 

```
gpaddmirrors [-p port_offset] 可选的。此数字用于计算用于镜像段的数据库端口。默认偏移量是1000
	[-m datadir_config_file [-a]] [-s] 一个配置文件，其中包含将在其中创建镜像数据目录的文件系统位置的列表。
   [-d master_data_directory] 主数据目录。如果未指定，则为 $ MASTER_DATA_DIRECTORY 将会被使用。
   [-B parallel_processes] 
   [-l logfile_directory]  写入日志文件的目录。默认为 〜/ gpAdminLogs。
   [-v]

gpaddmirrors -i mirror_config_file 一个配置文件，其中包含要创建的每个镜像段的一行。您必须为系统中的		每个主段列出一个镜像段。
	[-a] [-d master_data_directory]
   [-B parallel_processes] [-l logfile_directory] [-v]

gpaddmirrors -o output_sample_mirror_config 该实用程序将提示您输入镜像段数据目录的位置
	[-s] [-m datadir_config_file]

gpaddmirrors -? 

gpaddmirrors --version
```

#### 1.4.1、生成mirror配置文件

```
[gpadmin@gw_mdw1 ~]$ gpaddmirrors -o ./addmirror
20190326:00:56:21:030831 gpaddmirrors:gw_mdw1:gpadmin-[INFO]:-Obtaining Segment details from master...
Enter mirror segment data directory location 1 of 4 >
/data/mirror                                                # 填入mirror所在目录
Enter mirror segment data directory location 2 of 4 >
/data/mirror
Enter mirror segment data directory location 3 of 4 >
/data/mirror
Enter mirror segment data directory location 4 of 4 >
/data/mirror
```

查看文件内容如下

```
[gpadmin@gw_mdw1 ~]$ cat addmirror 
filespaceOrder=
mirror0=0:gw_sdw2:41000:42000:43000:/data/mirror/gpseg0
mirror1=1:gw_sdw2:41001:42001:43001:/data/mirror/gpseg1
mirror2=2:gw_sdw2:41002:42002:43002:/data/mirror/gpseg2
mirror3=3:gw_sdw2:41003:42003:43003:/data/mirror/gpseg3
```

#### 1.4.2、执行添加命令

```
[gpadmin@gw_mdw1 ~]$ gpaddmirrors -i addmirror 
20190326:01:08:45:031106 gpaddmirrors:gw_mdw1:gpadmin-[INFO]:-Starting gpaddmirrors with args: -i addmirror
20190326:01:08:45:031106 gpaddmirrors:gw_mdw1:gpadmin-[INFO]:-local Greenplum Version: 'postgres (Greenplum Database) 4.3.1.0 build 6'
20190326:01:08:45:031106 gpaddmirrors:gw_mdw1:gpadmin-[INFO]:-master Greenplum Version: 'PostgreSQL 8.2.15 (Greenplum Database 4.3.1.0 build 6) on x86_64-unknown-linux-gnu, compiled by GCC gcc (GCC) 4.4.2 compiled on Jun 11 2014 17:23:40'
```

#### 1.4.3、命令没有报错，查看mirror节点的情况

使用gpstate -m查看，发现所有的mirror正在同步数据，同步完成了，此时再执行gpstate -m就可以看到Data Status的状态是Synchronized（已同步的）

```
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:--------------------------------------------------------------
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:--Current GPDB mirror list and status
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:--Type = Group
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:--------------------------------------------------------------
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:-   Mirror    Datadir               Port    Status    Data Status       
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:-   gw_sdw2   /data/mirror/gpseg0   41000   Passive   Resynchronizing
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:-   gw_sdw2   /data/mirror/gpseg1   41001   Passive   Resynchronizing
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:-   gw_sdw2   /data/mirror/gpseg2   41002   Passive   Resynchronizing
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:-   gw_sdw2   /data/mirror/gpseg3   41003   Passive   Resynchronizing
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:-   gw_sdw1   /data/mirror/gpseg4   41000   Passive   Resynchronizing
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:-   gw_sdw1   /data/mirror/gpseg5   41001   Passive   Resynchronizing
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:-   gw_sdw1   /data/mirror/gpseg6   41002   Passive   Resynchronizing
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:-   gw_sdw1   /data/mirror/gpseg7   41003   Passive   Resynchronizing
20190326:01:09:51:031359 gpstate:gw_mdw1:gpadmin-[INFO]:--------------------------------------------------------------
```

### 1.5、gpinitstandby 初始化一个后备Master

#### 1.5.1、命令讲解

```
gpinitstandby { -s standby_hostname 	# 备用主控主机的主机名。
	[-P port] 							# 此选项指定Greenplum数据库备用主服务器使用的端口
	| -r |								# 从Greenplum数据库系统中删除当前配置的备用主实例。
    -n } 								# 指定此选项可启动已配置但由于某种原因已停止的Greenplum数据库备用主服务器
	[-a] 								# 不要提示用户进行确认。
	[-q] 								# 在安静模式下运行。命令输出未显示在屏幕上，但仍被写入日志文件。
    [-D]				 				# 将日志记录级别设置为调试。
    [-S standby_data_directory] 		# 用于新的备用主服务器的数据目录。默认值为活动主服务器使用的目录
    [-l logfile_directory] 				# 写入日志文件的目录。默认为 〜/ gpAdminLogs
```

#### 1.5.2、例子

**添加**

```
gpinitstandby -s centos02	
```

**验证**

```
gpstate										[检查节点运行环境]

[INFO]:-----------------------------------------------------
[INFO]:-   Master instance                  = Active
[INFO]:-   Master standby                   = centos02   [含有节点名代表添加成功]
[INFO]:-   Standby master state             = Standby host passive
[INFO]:-   Total segment instance count from metadata                = 8
[INFO]:-----------------------------------------------------
```

#### 1.5.3、完整案例

1）创建master镜像

​	1.1）在镜像master配置环境变量：vim ~/.bashrc

~~~
PATH=$PATH:$HOME/bin
export PGPORT=5432
export MASTER_DATA_DIRECTORY=/data/greenplum/gpmaster/gpseg-1
source /usr/local/greenplum/greenplum-db/greenplum_path.sh

export PGDATABASE=testdb

export PATH
~~~

​	1.2）在镜像master配置免密：在/home/gpadmin下创建all_nodes文件，添加集群所有节点

​	设置GP环境生效：source /usr/local/greenplum/greenplum-db-5.19.0/greenplum_path.sh

​	设置GP环境免密：gpssh-exkeys -f all_nodes

~~~
[STEP 1 of 5] create local ID and authorize on local host
  ... /home/gpadmin/.ssh/id_rsa file exists ... key generation skipped

[STEP 2 of 5] keyscan all hosts and update known_hosts file

[STEP 3 of 5] authorize current user on remote hosts
  ... send to centos01
  ... send to centos03

[STEP 4 of 5] determine common authentication file content

[STEP 5 of 5] copy authentication files to all remote hosts
  ... finished key exchange with centos01
  ... finished key exchange with centos03

[INFO] completed successfully
~~~

​	1.3）验证GP集群免密：gpssh -f all_nodes 

~~~
=> pwd
[centos01] /home/gpadmin
[centos02] /home/gpadmin
[centos03] /home/gpadmin
=> 
~~~

​	1.4）在镜像master节点创建数据目录：mkdir -p /data/greenplum/gpmaster

​	1.5）在主master运行命令：gpinitstandby -s centos02

~~~
gpadmin-[INFO]:-Validating environment and parameters for standby initialization...
gpadmin-[INFO]:-Checking for filespace directory /data/greenplum/gpmaster/gpseg-1 on centos02
gpadmin-[INFO]:------------------------------------------------------
gpadmin-[INFO]:-Greenplum standby master initialization parameters
gpadmin-[INFO]:------------------------------------------------------
gpadmin-[INFO]:-Greenplum master hostname               = centos01
gpadmin-[INFO]:-Greenplum master data directory         = /data/greenplum/gpmaster/gpseg-1
gpadmin-[INFO]:-Greenplum master port                   = 5432
gpadmin-[INFO]:-Greenplum standby master hostname       = centos02
gpadmin-[INFO]:-Greenplum standby master port           = 5432
gpadmin-[INFO]:-Greenplum standby master data directory = /data/greenplum/gpmaster/gpseg-1
gpadmin-[INFO]:-Greenplum update system catalog         = On
gpadmin-[INFO]:------------------------------------------------------
gpadmin-[INFO]:- Filespace locations
gpadmin-[INFO]:------------------------------------------------------
gpadmin-[INFO]:-pg_system -> /data/greenplum/gpmaster/gpseg-1
ialization? Yy|Nn (default=N):
> y
gpadmin-[INFO]:-Syncing Greenplum Database extensions to standby
gpadmin-[INFO]:-The packages on centos02 are consistent.
gpadmin-[INFO]:-Adding standby master to catalog...
gpadmin-[INFO]:-Database catalog updated successfully.
gpadmin-[INFO]:-Updating pg_hba.conf file...
gpadmin-[INFO]:-pg_hba.conf files updated successfully.
gpadmin-[INFO]:-Updating filespace flat files...
gpadmin-[INFO]:-Filespace flat file updated successfully.
gpadmin-[INFO]:-Starting standby master
gpadmin-[INFO]:-Checking if standby master is running on host: centos02  in directory: /data/greenplum/gpmaster/gpseg-1
gpadmin-[INFO]:-Cleaning up pg_hba.conf backup files...
gpadmin-[INFO]:-Backup files of pg_hba.conf cleaned up successfully.
gpadmin-[INFO]:-Successfully created standby master on centos02
~~~

​	1.6）查看集群状态：gpstate

~~~
[INFO]:-Greenplum instance status summary
[INFO]:-----------------------------------------------------
[INFO]:-   Master instance                                           = Active
[INFO]:-   Master standby                                            = centos02
[INFO]:-   Standby master state                                      = Standby host passive
[INFO]:-   Total segment instance count from metadata                = 8
[INFO]:-----------------------------------------------------
[INFO]:-   Primary Segment Status
[INFO]:-----------------------------------------------------
[INFO]:-   Total primary segments                                    = 4
[INFO]:-   Total primary segment valid (at master)                   = 4
[INFO]:-   Total primary segment failures (at master)                = 0
[INFO]:-   Total number of postmaster.pid files missing              = 0
[INFO]:-   Total number of postmaster.pid files found                = 4
[INFO]:-   Total number of postmaster.pid PIDs missing               = 0
[INFO]:-   Total number of postmaster.pid PIDs found                 = 4
[INFO]:-   Total number of /tmp lock files missing                   = 0
[INFO]:-   Total number of /tmp lock files found                     = 4
[INFO]:-   Total number postmaster processes missing                 = 0
[INFO]:-   Total number postmaster processes found                   = 4
[INFO]:-----------------------------------------------------
~~~

​	1.7）当主master故障激活standby master：在standby master上运行命令：

gpactivatestandby -d /data/greenplum/gpmaster/gpseg-1

~~~
[INFO]:------------------------------------------------------
[INFO]:-Standby data directory    = /data/greenplum/gpmaster/gpseg-1
[INFO]:-Standby port              = 5432
[INFO]:-Standby running           = no
[INFO]:-Force standby activation  = yes
[INFO]:------------------------------------------------------
n (default=N):
> y
[INFO]:-Starting standby master database in utility mode...
[INFO]:-Updating transaction files filespace flat files...
[INFO]:-Updating temporary files filespace flat files...
[INFO]:-Reading current configuration...
[DEBUG]:-Connecting to dbname='testdb'
[INFO]:-Writing the gp_dbid file - /data/greenplum/gpmaster/gpseg-1/gp_dbid...
[INFO]:-But found an already existing file.
[INFO]:-Hence removed that existing file.
[INFO]:-Creating a new file...
[INFO]:-Wrote dbid: 1 to the file.
[INFO]:-Now marking it as read only...
[INFO]:-Verifying the file...
~~~

2）将故障master重新加入集群，作为standby节点

​	2.1）先备份之前的主节点数据

~~~
mv /data/greenplum/gpmaster/gpseg-1 /data/greenplum/gpmaster/gpseg-1_bak20200110
~~~

​	2.2）将节点作为standby加入集群

~~~
gpinitstandby -s  故障节点主机名
~~~

​	2.3）如果想在切换的同时创建一个新的Standby，可以执行如下命令

~~~
gpactivatestandby -d /data/greenplum/gpmaster/gpseg-1 -c new_standby_hostname
~~~



### 1.6、集群恢复

|命令参数| 作用 
|gprecoverseg -a | 快速恢复

|gprecoverseg -B 要并行恢复的段数

|gprecoverseg -i | 指定恢复文件
|gprecoverseg -d | 指定数据目录
|gprecoverseg -l | 指定日志文件
|gprecoverseg -r | 平衡数据
|gprecoverseg -s | 指定配置空间文件
|gprecoverseg -o | 指定恢复配置文件
|gprecoverseg -p | 指定额外的备用机
|gprecoverseg -S | 指定输出配置空间文件

### 1.7、gpfdist并行装载数据

#### 1.7.1、介绍

​		gpfdist并行文件服务程序通过外部表的形式读取需要装载到Greenplum数据库的数据文件，凭借这种方式装载保证所有segment节点直接并行加装数据。通过gpfdist加载数据的步骤如下

#### 1.7.2、启动gpfdist

在启动过程中，首先指定gpfdist所要提供服务的目录（即需要装载数据文件所属的目录，或者所属目录的上级目录）， 及端口（默认为HTTP端口8080）

```
$nohup gpfdist -d /var/load_files -p 18086 -l /home/gpadmin/log &

【其中/var/load_files即gpfdist所要提供服务的目录，18086即端口，/home/gpadmin/log即记录输出信息及错误信息】
```

如果启动了多个gpfdist连接某台主机（例如ETL主机），那么需要给每一个gpfdist提供一个不同的端口，及服务目录，例如

```
$nohup gpfdist -d /var/load_files1 -p 8081 -l /home/gpadmin/log1 &
$nohup gpfdist -d /var/load_files2 -p 8082 -l /home/gpadmin/log2 &
```

#### 1.7.3、检查gpdfdist是否启动成功

```
[root@centos01 ~]# ps -ef | grep 8081
gpadmin    3528   3082  0 15:16 pts/2    00:00:00 gpfdist -d /home/gpadmin/wang/ -p 8081 -l /home/gpadmin/log
root       3645   3609  0 15:30 pts/3    00:00:00 grep --color=auto 8081

```

#### 1.7.4、在Greenplum数据库中创建表

1）目标表

```
CREATE TABLE wang.LINEITEM (
L_ORDERKEY INTEGER NOT NULL,
L_PARTKEY INTEGER NOT NULL,
L_SUPPKEY INTEGER NOT NULL,
L_LINENUMBER INTEGER,
L_QUANTITY DECIMAL,
L_EXTENDEDPRICE DECIMAL,
L_DISCOUNT DECIMAL,
L_TAX DECIMAL,
L_RETURNFLAG CHAR(1),
L_LINESTATUS CHAR(1),
L_SHIPDATE DATE,
L_COMMITDATE DATE,
L_RECEIPTDATE DATE,
L_SHIPINSTRUCT CHAR(25),
L_SHIPMODE CHAR(10),
L_COMMENT VARCHAR(44)
);
```

2）外部表

```
CREATE EXTERNAL TABLE wang.ext_LINEITEM (
L_ORDERKEY INTEGER,
L_PARTKEY INTEGER,
L_SUPPKEY INTEGER,
L_LINENUMBER INTEGER,
L_QUANTITY DECIMAL,
L_EXTENDEDPRICE DECIMAL,
L_DISCOUNT DECIMAL,
L_TAX DECIMAL,
L_RETURNFLAG CHAR(1),
L_LINESTATUS CHAR(1),
L_SHIPDATE DATE,
L_COMMITDATE DATE,
L_RECEIPTDATE DATE,
L_SHIPINSTRUCT CHAR(25),
L_SHIPMODE CHAR(10),
L_COMMENT VARCHAR(44)
)
LOCATION ('gpfdist://192.168.158.111:8081/*') 
FORMAT 'TEXT' (DELIMITER '|');
```

#### 1.7.5、通过外部表向目标表装载数据

```
insert into wang.lineitem select * from wang.ext_lineitem;
```

### 1.8、pg_dump和pg_restore

#### 1.8.1、介绍

​	是用于备份数据库的标准PostgreSQL实用程序，在Greenplum Database中也受支持。它创建一个（非并行）转储文件。对于Greenplum数据库的常规备份，最好使用Greenplum数据库备份实用程序，[gpbackup](https://gpdb.docs.pivotal.io/6-4/utility_guide/ref/gpbackup.html#topic1)，以获得最佳性能。

```
pg_dump [connection-option ...] [dump_option ...] [dbname]
控制输出内容的选项
  -a, --data-only          只备份数据
  -c, --clean              在创建之前，先删除schema
  -d, --inserts            使用insert命令方式备份数据，而不是copy命令
  -D, --column-inserts     使用insert命令方式备份数据，并显示列信息
  -E, --encoding=ENCODING  指定备份数据使用的字符集
  -f, --file=FILENAME      备份文件的名称
  -F p | c | d | t | --format = plain | custom | directory | tar	选择输出格式。格式可以是以下之一
  -n, --schema=SCHEMA      只备份指定的shcema
  -N, --exclude-schema=SCHEMA 不备份指定的schema
  -o, --oids               备份中包含oid
  -O, --no-owner           采用文本格式时，不输出授权语句。
  -s, --schema-only        只备份schema的定义
  -S, --superuser=NAME     采用文本格式备份时，指定超级管理员
  -t, --table=TABLE        只备份特定的表，视图，索引和序列
  -T, --exclude-table=TABLE   不备份特定的表，视图，索引和序列
es)
  -x, --no-privileges     不备份授权语句
  --use-set-session-authorization 使用SESSION AUTHORIZATION替代
                              ALTER OWNER 命令去设置所有权
连接选项
  -h, --host=HOSTNAME      数据库服务主机（ip）或者socket目录
  -p, --port=PORT          数据库服务端口号
  -U, --username=NAME      连接用户
  -W, --password           强制口令提示，默认选项

Greenplum专有选项
  --gp-c                  用gzip进行内置备份压缩
  --gp-d=BACKUPFILEDIR    指定备份文件存放目录，这些目录应该存在于所有master主机和segment主机，或者为这些主机所共享。默认目录就是每个段和master的数据目录。
  --gp-r=REPORTFILEDIR    指定备份报告文件的存放目录，这些目录应该存在于所有master主机和segment主机，或者为这些主机所共享。默认目录就是每个段和master的数据目录。

--gp-s=BACKUPSET        备份指示 - p表示备份所有primary segment。i指定特定的primary segment进行备份，要跟上特定segment的dbid。
比如我们要对sales_history进行全库备份。

```

```
pg_restore [OPTIONS]

常用选项
  -d, --dbname=NAME        指定数据库名称
  -i, --ignore-version     即使服务版本不匹配，忽略版本信息。
  -v, --verbose            用verbose模式. 并向每个段的状态文件中添加这些信息
  --help                   显示帮助然后退出
  --version               输出版本信息，然后退出

控制输出内容的选项
  -a, --data-only          只恢复数据
  -c, --clean              创建之前，删除schema
  -s, --schema-only        只恢复schema定义

连接选项
  -h, --host=HOSTNAME      指定数据库服务主机名或者socket目录
  -p, --port=PORT          指定连接端口号
  -U, --username=NAME      指定连接的用户
  -W, --password           强制口令提示

Greenplum 特有选项
  --gp-c                  如果备份时是压缩格式，必须恢复使用该选项解压
  --gp-d=BACKUPFILEDIR    指定备份文件目录，默认就是数据目录。
  --gp-i                  忽略错误 --gp-k=KEY              指定备份14位时间戳信息，必须指定。
  --gp-r=REPORTFILEDIR    指定报告存放目录
  --gp-l=FILELOCATIONS    指定备份文件是在所有主段（p),还是特定的主段上。

```

#### 1.8.2、例子

```
备份：
pg_dump -f /home/gpadmin/sales_bak -v -F c -p 5432 -C -n public testdb
提供master的连接信息，把数据库sales_history的schema sales_history备份到/home/gpadmin/backup/sals_bak中，并含有建库句法。 

恢复：
pg_restore -F c -d tpchdb -v  -t test1 -p 5432  /home/gpadmin/sales_bak
如果pg_dump备份的文件格式是txt平面文件，则不能用pg_restore进行恢复，应该考虑使用psql进行恢复。
```

下面进行恢复，再恢复之前，必须先重建数据库

### 1.9、analyzedb 收集统计信息

~~~
 -d <database name>    Database name. Required.
  -s <schema name>      Specify a schema to analyze. All tables in the schema
                        will be analyzed.
  -t <schema name>.<table name>
                        Analyze a single table. Table name needs to be
                        qualified with schema name.
  -i <column1>,<column2>,...
                        Columns to include to be analyzed, separated by comma.
                        All columns will be analyzed if not specified.
  -x <column1>,<column2>,...
                        Columns to exclude to be analyzed, separated by comma.
                        All columns will be analyzed if not specified.
  -f <config_file>, --file=<config_file>
                        Config file that includes a list of tables to be
                        analyzed. Table names must be qualified with schema
                        name. Optionally a list of columns (separated by
                        comma) can be specified using -i or -x.
  -l, --list            List the tables to be analyzed without actually
                        running analyze (dry run).
  -p <parallel level>   Parallel level, i.e. the number of tables to be
                        analyzed in parallel. Valid numbers are between 1 and
                        10. Default value is 5.
  --skip_root_stats     Skip refreshing root partition stats if any of the
                        leaf partitions is analyzed.
  --gen_profile_only    Create cached state files to indicate specified table
                        or all tables have been analyzed.
  --full                Analyze without using incremental. All tables
                        requested by the user will be analyzed.
  --clean_last          Clean the state files generated by last analyzedb run.
                        All other options except -d will be ignored.
  --clean_all           Clean all the state files generated by analyzedb. All
                        other options except -d will be ignored.
  -h, -?, --help        Show this help message and exit.
  -v, --verbose         Print debug messages.
  -a                    Quiet mode. Do not prompt for user confirmation.
<u>下划线</u>
~~~



## 2、客户端常用操作

### 2.1、greenplum启动、停止、查看段主机命令

#### 2.1.1 gpstart

| 命令 参数  | 作用     |
| :--------- | :------- |
| gpstart -a | 快速启动 |

|gpstart -d | 指定数据目录（默认值：$MASTER_DATA_DIRECTORY）
|gpstart -q | 在安静模式下运行。命令输出不显示在屏幕，但仍然写入日志文件。
|gpstart -m | 以维护模式连接到Master进行目录维护。例如：$ PGOPTIONS='-c gp_session_role=utility' psql postgres
|gpstart -R | 管理员连接
|gpstart -v | 显示详细启动信息

#### 2.1.2 gpstop

|命令参数 	| 	作用 |
|gpstop -a | 快速停止
|gpstop -d | 指定数据目录（默认值：$MASTER_DATA_DIRECTORY）
|gpstop -m | 维护模式
|gpstop -q | 在安静模式下运行。命令输出不显示在屏幕，但仍然写入日志文件。
|gpstop -r | 停止所有实例，然后重启系统
|gpstop -u | 重新加载配置文件 postgresql.conf 和 pg_hba.conf
|gpstop -v | 显示详细启动信息
|gpstop -M fast      | 快速关闭。正在进行的任何事务都被中断。然后滚回去。
|gpstop -M immediate | 立即关闭。正在进行的任何事务都被中止。不推荐这种关闭模式，并且在某些情况下可能导致数据库损坏需要手动恢复。
|gpstop -M smart     | 智能关闭。如果存在活动连接，则此命令在警告时失败。这是默认的关机模式。
|gpstop --host hostname | 停用segments数据节点，不能与-m、-r、-u、-y同时使用 

#### 2.1.3 gpstate

|命令 参数  | 作用 |
|gpstate -b | 显示简要状态
|gpstate -c | 显示主镜像映射
|gpstart -d | 指定数据目录（默认值：$MASTER_DATA_DIRECTORY）
|gpstate -e | 显示具有镜像状态问题的片段
|gpstate -f | 显示备用主机详细信息
|gpstate -i | 显示GRIPLUM数据库版本
|gpstate -m | 显示镜像实例同步状态
|gpstate -p | 显示使用端口
|gpstate -Q | 快速检查主机状态
|gpstate -s | 显示集群详细信息
|gpstate -v | 显示详细信息

### 2.2、赋权

#### 2.2.1、表赋权

指定pg_tables库下某个schemaname下所有表赋予read_role用户**查询**权限

```
select ‘grant SELECT on table ’ || schemaname || ‘.’ || tablename || ’ to read_role;’ from pg_tables where schemaname = ‘vt_profile’
```

指定pg_tables库下某个schemaname下所有表赋予read_role用户**所有**权限

```
select ‘grant all on table ’ || schemaname || ‘.’ || tablename || ’ to read_role;’ from pg_tables where schemaname = ‘vt_profile’
```

指定pg_tables库下某个schemaname下所有表赋予read_role用户**插入**权限

```
select ‘grant INSERT on table ’ || schemaname || ‘.’ || tablename || ’ to read_role;’ from pg_tables where schemaname = ‘vt_profile’
```

设置默认新建表都赋予用户拥有**所有**权限

```
ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT all ON TABLES TO PUBLIC;
ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT all ON TABLES TO webuser;
```

#### 2.2.2、schema赋权

查询数据库下所有schema并赋予tuser用户**所有**schema的权限

```
select 'grant all on SCHEMA ' || schemaname || ' to tuser;' as grant_script from pg_tables group by schemaname;
```

给数据库下某个单一的schema所有权限赋予testuser用户

```
grant all on schema zjb to testuser;
```

#### 2.2.3、库赋权

赋予用户建库的权限

```
alter user lwdg [with] createdb
```

#### 2.2.4、协议赋权

```
grant insert on protocol pxf to user;
```

### 2.3、创建用户、模式

|createdb  |创建一个新数据库
|createlang|定义一种新的过程语言
|createuser|定义一个新的数据库角色
|dropdb|移除一个数据库
|droplang|移除一种过程语言
|dropuser|移除一个角色
|psql|PostgreSQL交互式终端
|reindexdb|对一个数据库重建索引
|vacuumdb|对一个数据库进行垃圾收集和分析

#### 2.3.1、用户

```
CREATE USER testuser WITH PASSWORD '123456' SUPERUSER NOINHERIT; 	# 超级用户
CREATE USER testuser WITH PASSWORD '123456';                    	# 普通用户
alter user gpadmin encrypted password 'gpadmin';					# 修改密码

CREATE DATABASE test OWNER testuser;							 	# 创建库并指定所有者
CREATE DATABASE test;                                            	# 创建库在默认用户下
GRANT ALL PRIVILEGES ON DATABASE test TO testuser;               	# 设置用户拥有库所有权限
```

#### 2.3.2、模式

1）创建

```
CREATE SCHEMA myschema;
```

2）删除

```
DROP SCHEMA myschema;
```

```
CREATE USER testuser WITH PASSWORD '123456' SUPERUSER NOINHERIT; # 超级用户
CREATE USER testuser WITH PASSWORD '123456';                     # 普通用户
```

### 2.4、获取table、schema及其database大小及分布

#### 2.4.1、获取表的大小

1、获取某一个特定表的大小

```
select pg_size_pretty(pg_relation_size('schema_name.table_name'));

# 如果这里是一个分区表，那么查询到的结果为0bytes,原因在于:GP的分区表的主表只是一个表定义,其实际数据内容存储在继承父表的分区子表里面了
```

2、查询一个库下所有schema下所有表的空间

```
select schemaname  || '.' || tablename, pg_size_pretty(pg_relation_size( schemaname || '.' || tablename)) from pg_tables;
```

3、查询某一个schema下各表的空间

```
select schemaname  || '.' || tablename, pg_size_pretty(pg_relation_size( schemaname  || '.' || tablename)) from pg_tables t inner join pg_namespace d on t.schemaname=d.nspname  where schemaname='public';
```

4、表和索引

```
select pg_size_pretty(pg_total_relation_size('gp_test'));
```

#### 2.4.2、获取库的大小

查看某一个数据库大小

```
select pg_size_pretty(pg_database_size('MyDatabase')); 
```

查看所有数据库大小

```
 select datname,pg_size_pretty(pg_database_size(datname)) from pg_database;
```

#### 2.4.3、获取schema的大小

1、获取一个库下所有schema的大小

```
select pg_size_pretty(cast( sum(pg_relation_size( schemaname  || '.' || tablename)) as bigint)), schemaname from pg_tables t inner join pg_namespace d on t.schemaname=d.nspname  group by schemaname;
```

2、获取单一的schema的大小

```
select pg_size_pretty(cast( sum(pg_relation_size( schemaname  || '.' || tablename)) as bigint)) from pg_tables t inner join pg_namespace d on t.schemaname=d.nspname  where schemaname='public';
```

#### 2.4.4、查看数据分布情况和磁盘空间

查看数据分布

```
gp_segment_id,count(*) from gp_test group by gp_segment_id order by 1;
```

磁盘空间

```
select dfhostname, dfspace,dfdevice from gp_toolkit.gp_disk_free order by dfhostname;
```

查看磁盘剩余空间：

```
SELECT * FROM gp_toolkit.gp_disk_free ORDER BY dfsegment;
```

#### 2.4.5、查看实例

```
select * from gp_segment_configuration order by 1;
```

#### 2.4.6、收集统计信息，回收空间

```
# 定期使用回收垃圾和收集统计信息，尤其在大数据量删除，导入以后，非常重要
Vacuum analyze tablename
```

## 3、sql操作命令

中文官网地址介绍：https://docs.greenplum.cn/6-0/ref_guide/sql_commands/COPY.html

英文官网地址介绍：https://gpdb.docs.pivotal.io/6-4/ref_guide/sql_commands/sql_ref.html

### 3.1、copy

```
COPY table_name [ ( column_name [, ...] ) ]
    FROM { 'filename' | PROGRAM 'command' | STDIN }
    [ [ WITH ] ( option [, ...] ) ]

COPY { table_name [ ( column_name [, ...] ) ] | ( query ) }
    TO { 'filename' | PROGRAM 'command' | STDOUT }
    [ [ WITH ] ( option [, ...] ) ]

where option can be one of:

    FORMAT format_name
    OIDS [ boolean ]
    FREEZE [ boolean ]
    DELIMITER 'delimiter_character'
    NULL 'null_string'
    HEADER [ boolean ]
    QUOTE 'quote_character'
    ESCAPE 'escape_character'
    FORCE_QUOTE { ( column_name [, ...] ) | * }
    FORCE_NOT_NULL ( column_name [, ...] )
    ENCODING 'encoding_name'
```

#### 3.1.1、介绍

COPY`在PostgreSQL表和文件之间交换数据。 `COPY TO`把一个表的所有内容都拷贝到一个文件，而`COPY FROM`从一个文件里拷贝数据到一个表里(把数据附加到表中已经存在的内容里)。 `COPY TO`还能拷贝`SELECT`查询的结果。

#### 3.1.2、案例：copy表数据到hive里面

1）在hive中创建一张外部表

```
create external table test_copy(
id int,
name string
)
row format delimited 
fields terminated by '|'
location '/hive';
```

2）导出greenplum数据出来成csv文件

```
postgres=# copy (select id,name from public.test_copy_gp) to '/data/test_copy.csv' with csv delimiter '|';						# 表名大写时表名需要用双引号标注 public."TEST_COPY"
或
psql -d postgres -h ip -p 5432 -c"copy (select id,name from public.test_copy_gp) to '/data/test_copy.csv' with csv delimiter '|'"
```

3）将数据load到hive表中

```
load data local inpath '/data/test_copy.csv' into table database.tbname;
```















































