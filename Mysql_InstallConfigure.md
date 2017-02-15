## CentOS 6.4下编译安装MySQL 5.6.14
**概述：**
CentOS 6.4下通过yum安装的MySQL是5.1版的，比较老，所以就想通过源代码安装高版本的5.6.14。

**正文：**
### 一：卸载旧版本

**使用下面的命令检查是否安装有MySQL Server**

>rpm -qa | grep mysql

**有的话通过下面的命令来卸载掉**
>rpm -e mysql   //普通删除模式
>
>rpm -e --nodeps mysql    // 强力删除模式，如果使用上面命令删除时，提示有依赖的其它文件，则用该命令可以对其进行强力删除

### 二：添加用户设置权限
**使用下面的命令查看是否有mysql用户及用户组**
>cat /etc/passwd 查看用户列表
>
>cat /etc/group  查看用户组列表

**如果没有就创建**
>groupadd mysql
>
>useradd -g mysql mysql


**修改/usr/local/mysql权限**
>chown -R mysql:mysql /usr/local/mysql


### 三：安装MySQL
**安装编译代码需要包(在安装前，需先安装相关的依赖库)**

>yum -y install make gcc-c++ cmake bison-devel  ncurses-devel
>
> sudo yum install -y cmake,make,gcc,gcc-c++,bison, ncurses,ncurses-devel// MySQL5.7.13安装（下同）

- **cmake:**
MySQL使用cmake跨平台工具预编译源码，用于设置mysql的编译参数。如：安装目录、数据存放目录、字符编码、排序规则等。安装最新版本即可。
- **make3.75:**
mysql源代码是由C和C++语言编写，在Linux下使用make对源码进行编译和构建，要求必须安装make 3.75或以上版本
- **gcc4.4.6:**
GCC是Linux下的C语言编译工具，mysql源码编译完全由C和C++编写，要求必须安装GCC4.4.6或以上版本
- **Boost1.59.0:**
mysql源码中用到了C++的Boost库，要求必须安装boost1.59.0或以上版本
- **bison2.1**
Linux下C/C++语法分析器
- **ncurses**
字符终端处理库


**下载Mysql5.6.14**
>wget http://cdn.mysql.com/Downloads/MySQL-5.6/mysql-5.6.14.tar.gz
>
>tar xvf mysql-5.6.14.tar.gz
>
>cd mysql-5.6.14

**配置mysql预编译参数**

```
cmake \
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
-DMYSQL_DATADIR=/usr/local/mysql/data \
-DSYSCONFDIR=/etc \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_MEMORY_STORAGE_ENGINE=1 \
-DWITH_READLINE=1 \
-DMYSQL_UNIX_ADDR=/var/lib/mysql/mysql.sock \
-DMYSQL_TCP_PORT=3306 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DEXTRA_CHARSETS=all \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci


```
- **-DCMAKE_INSTALL_PREFIX：**安装路径
- **-DMYSQL_DATADIR：**数据存放目录
- **DWITH_BOOST：**boost源码路径
- **-DSYSCONFDIR：**my.cnf配置文件目录
- **-DEFAULT_CHARSET：**数据库默认字符编码
- **-DDEFAULT_COLLATION：**默认排序规则
- **-DENABLED_LOCAL_INFILE：**允许从本文件导入数据
- **-DEXTRA_CHARSETS：**安装所有字符集

**编译并安装**
>make -j `grep processor /proc/cpuinfo | wc -l`
>
> make install

`-j参数表示根据CPU核数指定编译时的线程数，可以加快编译速度。默认为1个线程编译，经测试单核CPU，1G的内存，编译完需要将近1个小时。`


注明:[更多编译的参数可以参考][1](整个过程需要30分钟左右……漫长的等待[MySQL5.7.13的参考][2])

**初始化系统数据库**
>cd /usr/local/mysql
>
>chown -R mysql:mysql
>
>./bin/mysql_install_db --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data#MySQL 5.7.6之前的版本执行这个脚本初始化系统数据库(本文使用此方式初始化)
>
>./bin/mysqld --initialize-insecure --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data#5.7.6之后版本初始系统数据库脚本


注：在启动MySQL服务时，会按照一定次序搜索my.cnf，先在/etc目录下找，找不到则会搜索"$basedir/my.cnf"，在本例中就是 /usr/local/mysql/my.cnf，这是新版MySQL的配置文件的默认位置！

`注意：在CentOS 6.4版操作系统的最小安装完成后，在/etc目录下会存在一个my.cnf，需要将此文件更名为其他的名字，如：/etc/my.cnf.bak，否则，该文件会干扰源码安装的MySQL的正确配置，造成无法启动。`

`在使用"yum update"更新系统后，需要检查下/etc目录下是否会多出一个my.cnf，如果多出，将它重命名成别的。否则，MySQL将使用这个配置文件启动，可能造成无法正常启动等问题。`

**配置文件及参数优化**

```
shell> cp support-files/my-default.cnf /etc/my.cnf
shell> vim /etc/my.cnf

[client]
port=3306
socket=/usr/local/mysql/mysql.sock
[mysqld]
character-set-server=utf8
collation-server=utf8_general_ci

skip-external-locking
skip-name-resolve

user=mysql
port=3306
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
tmpdir=/usr/local/mysql/temp
# server_id = .....
socket=/usr/local/mysql/mysql.sock
log-error=/usr/local/mysql/logs/mysql_error.log
pid-file=/usr/local/mysql/mysql.pid
open_files_limit=10240
back_log=600
max_connections=500
max_connect_errors=6000
wait_timeout=605800
#open_tables=600
#table_cache = 650
#opened_tables = 630

max_allowed_packet=32M
sort_buffer_size=4M
join_buffer_size=4M
thread_cache_size=300
query_cache_type=1
query_cache_size=256M
query_cache_limit=2M
query_cache_min_res_unit=16k

tmp_table_size=256M
max_heap_table_size=256M

key_buffer_size=256M
read_buffer_size=1M
read_rnd_buffer_size=16M
bulk_insert_buffer_size=64M

lower_case_table_names=1

default-storage-engine=INNODB

innodb_buffer_pool_size=2G
innodb_log_buffer_size=32M
innodb_log_file_size=128M
innodb_flush_method=O_DIRECT
#####################
thread_concurrency=32
long_query_time=2
slow-query-log=on
slow-query-log-file=/usr/local/mysql/logs/mysql-slow.log

[mysqldump]
quick
max_allowed_packet=32M

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```
**配置mysql服务**

```
shell> service mysqld start       # 启动mysql服务
shell> service mysqld stop        # 停止mysql服务
shell> service mysqld restart     # 重新启动mysql服务
```
**设置数据库密码**

```
shell> /usr/local/mysql/bin/mysql -e "grant all privileges on *.* to root@'127.0.0.1' identified by "root" with grant option;"
shell> /usr/local/mysql/bin/mysql -e "grant all privileges on *.* to root@'localhost' identified by "root" with grant option;"
# 开启远程登录(将host设为%即可)
/usr/local/mysql/bin/mysql -e "grant all privileges on *.* to root@'%' identified by "root" with grant option;"
```
**配置mysql环境变量**

```
shell> vim /etc/profile
shell> export PATH=/usr/local/mysql/bin:$PATH
shell> source /etc/profile
```
**其他注意事项**

如果中途编译失败了，需要删除cmake生成的预编译配置参数的缓存文件和make编译后生成的文件，再重新编译。

```
shell> cd /opt/mysql-server
shell> rm -f CMakeCache.txt
shell> make clean
```
**搜狐镜像下载地址：**
 
[http://mirrors.sohu.com/mysql/MySQL-5.5/] [3]

[http://mirrors.sohu.com/mysql/MySQL-5.6/][4]
 
[http://mirrors.sohu.com/mysql/MySQL-5.7/][5]

**Mysql其他编译安装参考链接**

[MySQL5.7.13源码编译安装与配置][6]

[CentOS 6.4下编译安装MySQL 5.6.14][7]

[MySQL 5.6.23 Install][8]


[1]: [http://dev.mysql.com/doc/refman/5.5/en/source-configuration-options.html。]
[2]:http://dev.mysql.com/doc/refman/5.7/en/source-configuration-options.html#cmake-general-options
[3]:http://mirrors.sohu.com/mysql/MySQL-5.5/ 
[4]:http://mirrors.sohu.com/mysql/MySQL-5.6/ 
[5]:http://mirrors.sohu.com/mysql/MySQL-5.7/
[6]:http://blog.csdn.net/xyang81/article/details/51792144
[7]:http://www.cnblogs.com/xiongpq/p/3384681.html
[8]:https://github.com/Hackeruncle/MySQL/blob/master/MySQL%205.6.23%20Install.txt





