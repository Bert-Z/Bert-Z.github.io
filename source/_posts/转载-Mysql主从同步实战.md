---
title: '[转载]Mysql主从同步实战'
date: 2020-08-13 07:15:05
tags: Mysql
categories:
cover: https://user-gold-cdn.xitu.io/2019/3/29/169c78436838fbe6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1
---
<meta name="referrer" content="no-referrer" />

## 前言

Mysql主从同步实战，知其然，知其所以然也是很重要的，不然你永远无法进阶，但是，俗话说学会走前要学会爬，在大多数的中小型公司中，业务需要用到某一项技术，是没有太多的时间给你慢慢研究其原理的，大多是囫囵吞枣的看一下，找点攻略，快速搭建起来，先干活再说，至于后期的扩展和优化那就是慢慢来的事情，事有轻重缓急，我们要结合自身所处的环境来调整才能不断的前行，那么今天我们就来爬一爬

## 实战

### 准备

#### 软件

mysql5.7.2 [cdn.mysql.com//Downloads/…](https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.25-linux-glibc2.12-x86_64.tar)

##### 硬件

| 主机配置            | IP地址       | 操作系统  | 用途          | 部署路径         |
| ------------------- | ------------ | --------- | ------------- | ---------------- |
| 2.6GHz(1core)内存2G | 10.211.55.7  | Centos7.0 | mysql-masterr | /usr/local/mysql |
| 2.6GHz(1core)内存2G | 10.211.55.13 | Centos7.0 | mysql-slave   | /usr/local/mysql |

### 部署

#### mysql安装

```
# 上传mysql的tar包
re -be 
# 解压上传的命令
tar -xvf  https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.25-linux-glibc2.12-x86_64.tar
cd  https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.25-linux-glibc2.12-x86_64
# 创建mysql安装以及数据保存的目录
mkdir /usr/local/mysql
mkdir /usr/local/mysql/data  
mv * /usr/local/mysql
# 删除可能存在的mysql rpm包
rpm -qa|grep -i mysql
rpm -qa | grep maridb
yum remove  mysql mysql-server mysql-libs mysql-server
rpm -e --nodeps `rpm -qa | grep mysql`
rpm -e --nodeps `rpm -qa | grep maridb`
# 知道可能遗留的mysql相关文件,rm掉
find / -name mysql 
rpm -qa|grep mysql
# 因为我们是使用源码的方式安装，所有需要创建mysql用户组
userdel mysql 
groupdel mysql
groupadd mysql    
mkdir /usr/local/mysql
useradd -g mysql -d /usr/local/mysql mysql 
# 编译源码
sh /usr/local/mysql/bin/mysqld --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --initialize
# 编译成功后会在控制台看到打印的mysql的临时密码，记住，后面需要用到
# 启动mysql
sh /usr/local/mysql/support-files/mysql.server start
# 控制台打印success成功启动
# 创建mysql客户端软链接
ln -s /usr/local/mysql/bin/mysql /usr/bin/mysql
# 修改密码，输入上面步骤控制台打印的临时密码
mysql -u root -p
set password = password("mysql");
# 刷新缓存
flush privileges;
# 授权远程访问
grant all privileges on *.* to 'root'@'%' identified by 'mysql';
# 查看防火墙状态
firewall-cmd --state
# 停止防火请
systemctl stop firewalld.service
# 禁止防火墙开机启动
systemctl disable firewalld.service 
# 配置mysql服务开机自动启动
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld 
# 增加执行权限,检查开机启动列表，没有则加入
chmod 755 /etc/init.d/mysqld
chkconfig --list mysqld 
chkconfig --add mysqld
chkconfig mysqld on
# 以后mysql的启动就可以使用下面命令启动停止了
service mysqld start/stop/restart
复制代码
```

### 配置一

#### mysql-master配置

遇到的问题：
1：很多文章都会告诉你在 /etc/ 或者 安装目录的support-file下面有my.cnf配置文件，但是我们安装版本为5.7的时候怎么都找不到，后来经过查阅资料，发现5.7版本的mysql已经不自带这个配置文件了，所以需要自己手动配置

```
touch /etc/my.cnf
chmod 644 /etc/my.cnf
# 如果遇到权限问题请使用root账户
vim /etc/my.cnf
# 写入内容
# mysql主节点编号
[mysqld] 
server-id = 1 #Mysql服务的唯一编号 每个mysql服务Id需唯一
log-bin=mysql-bin # logbin的名字
binlog-do-db=test03 #需要同步的数据库的名字
binlog-do-db=test05 #需要同步的数据库的名字
binlog-ignore-db=test01 #不需要同步的数据库的名字
log-slave-updates=1 # log更新间隔
slave-skip-errors=1 # 是跳过错误，继续执行复制操作(可选)
复制代码
```

#### mysql-slave配置

```
[mysqld] 
#Mysql服务的唯一编号 每个mysql服务Id需唯一
server-id = 2
# read_only=1只读模式，可以限定普通用户进行数据修改的操作，但不会限定具有super权限的用户（如超级管理员root用户）的数据修改操作。如果想保证super用户也不能写操作，就可以就需要执行给所有的表加读锁的命令 “flush tables with read lock;”
read_only = 1
复制代码
```

重启 service mysqld restart

### 配置二

#### mysql-master配置

```
[mysqld] 
#Mysql服务的唯一编号 每个mysql服务Id需唯一
server-id = 1
log-bin=mysql-bin # logbin的名字
复制代码
```

#### mysql-slave配置

```
[mysqld] 
#Mysql服务的唯一编号 每个mysql服务Id需唯一
server-id = 2
# read_only=1只读模式，可以限定普通用户进行数据修改的操作，但不会限定具有super权限的用户（如超级管理员root用户）的数据修改操作。如果想保证super用户也不能写操作，就可以就需要执行给所有的表加读锁的命令 “flush tables with read lock;”
read_only = 1
replicate-do-db=test02 #需要复制的数据库名，如果复制多个数据库，重复设置这个选项即可
replicate-ignore-db=test05 #需要复制的数据库名，如果复制多个数据库，重复设置这个选项即可
复制代码
```

重启 service mysqld restart

### 设置主从同步

#### 操作主数据库

1：使用root用户进入mysql创建同步的账户并且赋权

```
CREATE USER 'slave'@'%' IDENTIFIED BY 'mysql';
GRANT REPLICATION SLAVE ON *.* TO 'slave'@'%';
FLUSH PRIVILEGES;
# 查看赋权状态
use mysql;
select  User,authentication_string,Host from user;
复制代码
```



![img](https://user-gold-cdn.xitu.io/2019/3/29/169c776ee8e2f7b4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



```
# 查看 日志开启状态
show variables like 'log_bin';
复制代码
```



![img](https://user-gold-cdn.xitu.io/2019/3/29/169c777c8a19c6fa?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

成功开启



```
# 查看主节点状态
show master status;
复制代码
```



![img](https://user-gold-cdn.xitu.io/2019/3/29/169c77881b3b244c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

记住file那么的名字和position的值，因为主从同步，从数据库是通过读取主数据库的日志文件来完成同步的，所以需要文件名字和日志的当前位置



#### 操作从数据库

```
# 停止正在进行的slave(如果有，此方法也用于修改slave的值(如果参数不对))
stop slave;
# 需要主机名，上面步骤的账户密码以及日志文件名字和位置(请根据实际情况自行修改)
change master to master_host='10.211.55.7', master_user='slave', master_password='mysql', master_log_file='mysql-bin.000003', master_log_pos=1639;
# 启动
start slave;
# 查看状态
show slave status\G;
复制代码
```

如果一切配置正常那么则会看到:
Slave_IO_Running: Yes
Slave_SQL_Running: Yes

![img](https://user-gold-cdn.xitu.io/2019/3/29/169c77e0c6e8a68b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

如果遇到 no的情况一般会原因有：
1：机器网络不通
2：配置项写错，例如ip，密码等
3：防火墙问题



### 优劣势对比

在master上设置binlog_do_弊端：
1、过滤操作带来的负载都在master上
2、无法做基于时间点的复制（利用binlog）
建议使用配置二，但是我使用的是配置一

### 测试

分别查询主从数据库

```
use test03;
select * from test01;
复制代码
```



![img](https://user-gold-cdn.xitu.io/2019/3/29/169c7831c0dd1ae3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

在主数据库插入一条数据





![img](https://user-gold-cdn.xitu.io/2019/3/29/169c783a4a5f168f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

查询从数据库





![img](https://user-gold-cdn.xitu.io/2019/3/29/169c78436838fbe6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

主从同步成功



## 总结

今天的实战就这么结束了，怎么样有没有收获，那么现在我们学会了爬，下一篇文档我们就要学一学怎么走，怎么跑了，下篇预告MySQL主从同步知其然，知其所以然。