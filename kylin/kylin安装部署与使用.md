# kylin安装部署与使用

## 1、kylin集群安装

### 1.1、部署环境

| 环境      | 版本            |
| --------- | --------------- |
| Linux系统 | centos-7.6.1810 |
| ambari    | ambari-2.6.2.2  |
| kylin     | kylin-2.6.5     |
|           | nginx-1.17.9    |
|           | pcre-8.43       |

### 1.2、环境准备

#### 1.2.1、下载安装包和依赖包

kylin安装包：https://mirror.bit.edu.cn/apache/kylin/apache-kylin-2.6.5/apache-kylin-2.6.5-bin-hbase1x.tar.gz

nginx安装包：http://nginx.org/download/nginx-1.17.9.tar.gz

pcre依赖包：https://nchc.dl.sourceforge.net/project/pcre/pcre/8.43/pcre-8.43.tar.gz

openssl依赖包：https://www.openssl.org/source/old/1.1.1/openssl-1.1.1d.tar.gz

zlib依赖包：https://www.zlib.net/zlib-1.2.11.tar.gz

#### 1.2.2、解压安装包：采用ansible解压

安装包存放在/opt/kylin目录下

vim tar.yml

```
---
- hosts: 192.168.158.111
  remote_user: root
  tasks:
    - name:  01 unzip  pcre package
      unarchive: "src=/opt/kylin/pcre-8.43.tar.gz dest=/opt/kylin"
    - name:  02 unzip  openssl package
      unarchive: "src=/opt/kylin/openssl-1.1.1d.tar.gz dest=/opt/kylin"
    - name:  03 unzip  nginx package
      unarchive: "src=/opt/kylin/nginx-1.17.9.tar.gz dest=/opt/kylin"
    - name:  04 unzip  zlib package
      unarchive: "src=/opt/kylin/zlib-1.2.11.tar.gz dest=/opt/kylin"
    - name:  05 unzip  kylin package
      unarchive: "src=/opt/kylin/apache-kylin-2.6.5-bin-hbase1x.tar.gz dest=/opt/kylin"
```

运行：ansible-playbook -vv tar.yml

#### 1.2.3、编译环境

pcre编译

```
[root@bdnode10 opt]# cd kylin/pcre-8.43/
[root@bdnode10 pcre-8.43]# ./configure
[root@bdnode10 pcre-8.43]# make && make install
[root@bdnode10 pcre-8.43]# pcre-config --version
8.43
```

### 1.3、安装

nginx安装

```
[root@bdnode10 kylin]# cd nginx-1.17.9/
[root@bdnode10 pcre-8.43]# ./configure
[root@bdnode10 pcre-8.43]# make
[root@bdnode10 pcre-8.43]# make install



```



