---
title: Mysql插入性能调优
date: 2019-05-16 17:58:03
categories:
- mysql
tags:
- mysql
---
&emsp;&emsp;前面一篇篇幅有点长， [Mysql单机压力测试](../../mysql/mysql-stress-testing/ "Mysql单机压力测试") 实际并没有一个准确的指标数据，比如TPS，QPS，并发连接数等。现在开始通过mysql自带的压测工具mysqlslap来看看实际的性能如何。mysqlslap是MySQL5.1.4之后自带的benchmark基准测试工具，该工具可以模拟多个客户端同时并发的向服务器发出查询请求，给出了性能测试数据而且提供了多种引擎的性能比较。

``` sh
[root@env1 test]# mysqlslap --create-schema=`test` --concurrency=50,100 --number-of-queries=100000 --query="SELECT * from test.emp where create_time BETWEEN '2019-05-14 17:01:37.824' and '2019-05-14 17:50:37.824' limit 100" -uroot -p
Enter password: 
Benchmark
	Average number of seconds to run all queries: 8.294 seconds
	Minimum number of seconds to run all queries: 8.294 seconds
	Maximum number of seconds to run all queries: 8.294 seconds
	Number of clients running queries: 50
	Average number of queries per client: 2000

Benchmark
	Average number of seconds to run all queries: 8.431 seconds
	Minimum number of seconds to run all queries: 8.431 seconds
	Maximum number of seconds to run all queries: 8.431 seconds
	Number of clients running queries: 100
	Average number of queries per client: 1000

```

&emsp;&emsp;由于之前一篇[Mysql单机压力测试](../../mysql/mysql-stress-testing/ "Mysql单机压力测试") 篇幅过长，所以只能把这部分单独抽出来写，逻辑也显得清晰。该篇文章都是基于Innodb存储引擎做的优化。

暂时不知道怎么恢复这5000多万数据，只能清空表了。从上面可看到数据的插入效率并不高，有没有什么办法可以做下调整呢？在网上看到了部分参数设置： innodb_flush_log_at_trx_commit ， innodb_log_buffer_size，innodb_log_file_size，以及 innodb_buffer_pool_size 、innodb_buffer_pool_chunk_size。

**1. innodb_flush_log_at_trx_commit :** 该参数有三个可选项：
- **0** : log buffer将每秒一次地写入log file中，并且同时执行log file的flush(刷到磁盘)操作。该模式下在事务提交的时候，不会主动触发写入磁盘的操作。该模式速度最快，但不太安全，mysqld进程的崩溃会导致上一秒钟所有事务数据的丢失。

- **1** ：每次事务提交时系统都会把log buffer的数据写入log file， 并且执行flush(刷到磁盘)操作，该模式为系统默认。
 
- **2** ：每次事务提交时系统都会把log buffer的数据写入log file，但是flush(刷到磁盘)操作并不会同时进行。该模式下，MySQL会每秒执行一次 flush(刷到磁盘)操作。该模式速度较快，也比0安全，只有在操作系统崩溃或者系统断电的情况下，上一秒钟所有事务数据才可能丢失。

**2. innodb_log_buffer_size :** InnoDB的写操作为了保证数据完整性，会首先将数据写入到日志中，防止MySQL崩溃或宕机后待MySQL服务重启时可以在日志中快速恢复该数据。ElasticSearch在写数据时也有类似的机制，叫translog。而写日志的过程中并不会直接写入，而是首先将数据写入到日志的缓冲区中，由于InnoDB在事务提交前，并不将改变的日志写入到磁盘中，因此在大事务中，可以减轻磁盘I/O的压力，当前版本默认是16M。看[官网](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_log_buffer_size)对该参数的描述：
``` sh
  A large log buffer enables large transactions to run without the need to write the log to disk 
before the transactions commit. Thus, if you have transactions that update, insert, or delete 
many rows, making the log buffer larger saves disk I/O. For related information
```

**3. innodb_log_file_size :** 此配置项作用设定innodb 数据库引擎日志的大小；从而减少数据库checkpoint操作。设置的太小：当一个日志文件写满后，innodb会自动切换到另外一个日志文件，而且会触发数据库的检查点（Checkpoint），这会导致innodb缓存脏页的小批量刷新，会明显降低innodb的性能。设置很大以后虽然减少了checkpoint，并且由于redo log是顺序I/O，大大提高了I/O性能。但是如果数据库意外出现了问题，比如意外宕机，那么需要重放日志并且恢复已经提交的事务（也就是实例恢复中的前滚, 利用redo从演变化来恢复buffer cache中的数据），如果日志很大，那么将会导致恢复时间很长。当前版本默认是48M。

**4. innodb_buffer_pool_size :** 该参数用于指定缓存innodb表的索引，数据，插入数据时的缓冲区大小，该参数与innodb_buffer_pool_chunk_size密切相关。当前MySQL版本默认的是128M，即 134217728 比特。两者的关系为 ：
**innodb_buffer_pool_size &emsp; = &emsp; innodb_buffer_pool_chunk_size &emsp; * &emsp; innodb_buffer_pool_instances.**
在调整 innodb_buffer_pool_size 时，若 innodb_buffer_pool_size 的值不是innodb_buffer_pool_chunk_size 与 innodb_buffer_pool_instances 乘积的整数倍，那么系统会将 innodb_buffer_pool_size 系统调整为前面两个参数乘积的整数倍（一般是将innodb_buffer_pool_size在物理内存限制内向上取最小整数倍）。一般建议将该参数设置系统总内存的70%-80%。[innodb_buffer_pool_size 官网文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool-resize.html)
 

**5. innodb_buffer_pool_chunk_size :** 该参数用于指定缓存innodb表的索引，数据，插入数据时的缓冲区中块的大小，内存会分块（block），每块还分多个页，具体分多少页，就得看块的大小及页的大小。页的大小参数是innodb_page_size。 该参数调整的最小单位时 1 M，而该参数的单位时比特Bit，所以不是很好阅读，当前MySQL版本默认的也是128M，即 134217728 比特。在调整该参数后：[innodb_buffer_pool_chunk_size 官网文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool-resize.html)
- 若将该参数调大，使得 innodb_buffer_pool_chunk_size * innodb_buffer_pool_instances 的结果大于 innodb_buffer_pool_size ，那么innodb_buffer_pool_chunk_size将会被系统缩减为 innodb_buffer_pool_size / innodb_buffer_pool_instances 的大小
- 若将该参数调小，那么 innodb_buffer_pool_size 将会被系统调整为  innodb_buffer_pool_chunk_size 与 innodb_buffer_pool_instances 两者乘积的最小整数倍。为了避免出现性能问题，innodb_buffer_pool_size 与 innodb_buffer_pool_chunk_size的比值最好不要超过 1000.


``` sh
To avoid potential performance issues, the number of chunks
 (innodb_buffer_pool_size / innodb_buffer_pool_chunk_size) should not exceed 1000. [innodb_buffer_pool_size 官网文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-buffer-pool-resize.html)
```

&emsp;&emsp;通过下面命令可以查看当前系统中的相关参数：
``` sql
show GLOBAL VARIABLES like '%innodb_flush_log_at_trx_commit%';

show GLOBAL VARIABLES like '%innodb_log_buffer_size%'; 

show GLOBAL VARIABLES like '%innodb_log_file_size%';

show GLOBAL VARIABLES like '%innodb_buffer_pool_size%';

show GLOBAL VARIABLES like '%innodb_buffer_pool_chunk_size%';

```
## 调整参数 innodb_flush_log_at_trx_commit
下面做临时修改，先调整前面两个参数的值，再做百万数据插入看看结果：
``` sql
set GLOBAL innodb_flush_log_at_trx_commit = 0;
```

执行百万数据插入：
``` sql
call insert_emp(1,1000000)
> OK
> 时间: 1152.921s
```
结果发现花了接近19分钟，这个还是在有索引的情况下执行的操作。前面在无索引无参数调整的情况下插入百万数据耗时是30分钟，磁盘占用 134 M；而这次是在有索引做了部分参数调整的情况下的执行情况，接近19分钟，磁盘占用 244 M。所以调整这个两个参数还是起了不少作用。但是innodb_flush_log_at_trx_commit设置为 0 就太不安全。

``` shell
[root@env1 test]# ll -h

-rw-r----- 1 mysql mysql   61 5月  10 18:09 db.opt
-rw-r----- 1 mysql mysql 8.5K 5月  10 18:12 dept.frm
-rw-r----- 1 mysql mysql 176K 5月  13 11:11 dept.ibd
-rw-r----- 1 mysql mysql 8.9K 5月  13 18:00 emp.frm
-rw-r----- 1 mysql mysql 244M 5月  14 15:47 emp.ibd
-rw-r----- 1 root  root  1.9G 5月  13 16:06 emp.ibd.bak
-rw-r----- 1 root  root   17G 5月  14 13:59 emp.ibd.bak2
-rw-r----- 1 mysql mysql 8.9K 5月  10 18:12 order.frm
-rw-r----- 1 mysql mysql  96K 5月  10 18:12 order.ibd

```
如果我把索引删除后再插入效果会如何呢？首先来删除索引：
``` sql
-- 删除主键以外的索引
alter table emp drop index index_emp_weight;
alter table emp drop index index_createtime;
alter table emp drop index index_emp_mobile_ename;

-- 清空表数据
TRUNCATE table emp;
```
执行Sql并查看执行结果：
``` sql
call insert_emp(1,1000000)
> OK
> 时间: 1037.407s
```
在看看磁盘占用，136 M：
``` shell
[root@env1 test]# ll -h
总用量 19G
-rw-r----- 1 mysql mysql   61 5月  10 18:09 db.opt
-rw-r----- 1 mysql mysql 8.5K 5月  10 18:12 dept.frm
-rw-r----- 1 mysql mysql 176K 5月  13 11:11 dept.ibd
-rw-r----- 1 mysql mysql 8.9K 5月  14 15:57 emp.frm
-rw-r----- 1 mysql mysql 136M 5月  14 16:16 emp.ibd
-rw-r----- 1 root  root  1.9G 5月  13 16:06 emp.ibd.bak
-rw-r----- 1 root  root   17G 5月  14 13:59 emp.ibd.bak2
-rw-r----- 1 mysql mysql 8.9K 5月  10 18:12 order.frm
-rw-r----- 1 mysql mysql  96K 5月  10 18:12 order.ibd
```
可以看到在无索引的情况下会快个一两分钟。
## 调整 innodb_log_file_size 及 innodb_log_buffer_size
接下来再调整下后面两个参数
``` sql
set GLOBAL innodb_log_buffer_size = 33554432; -- 默认是 16777216，即 16 M，现在改为 32 M
set GLOBAL innodb_log_file_size = 134217728;  -- 默认是 50331648， 即 48 M，现在改为 128 M
```
执行后发现 innodb_log_file_size 与 innodb_log_buffer_size 是只读属性，无法修改 ：
``` sql
set GLOBAL innodb_log_buffer_size = 33554432
> 1238 - Variable 'innodb_log_buffer_size' is a read only variable
> 时间: 0.002s

set GLOBAL innodb_log_file_size = 50331648
> 1238 - Variable 'innodb_log_file_size' is a read only variable
> 时间: 0.006s
```
那就试试直接在配置文件中修改该配置，将两者都设置为512M，修改完成后重启，通过SQL看看是否修改成功：
``` sh
innodb_log_buffer_size = 512M

innodb_log_file_size = 512M
```
修改成功后执行百万数据插入：
``` sql
call insert_emp(1, 1000000)
> OK
> 时间: 1013.731s

call insert_orders(1000000)
> OK
> 时间: 1223.921s
```
如果这时将 innodb_flush_log_at_trx_commit 参数设置为 2 ，一个稍微保险的值。然后再看看性能会如何：

``` sql
call insert_emp(1, 1000000)
> OK
> 时间: 1061.904s


call insert_orders(1000000)
> OK
> 时间: 1314.815s

```

## 调整 innodb_buffer_pool_size 及 innodb_buffer_pool_chunk_size
再来看看后面两参数，为了保险起见还是将 innodb_flush_log_at_trx_commit 设置为2。这次直接在mysql配置文件中修改，修改完成后重启MySQL服务。
``` sh
# 默认是 134217728字节，即128M。这里先试试16G
innodb_buffer_pool_size = 16G
innodb_buffer_pool_chunk_size = 4G
innodb_buffer_pool_instances = 4

innodb_flush_log_at_trx_commit = 2
```
重启完成后可以通过sql来查看是否已修改成功：
``` sql
show GLOBAL VARIABLES like '%innodb_buffer_pool_size';
show GLOBAL VARIABLES like '%innodb_buffer_pool_chunk_size';
SHOW VARIABLES LIKE '%innodb_buffer_pool_instances';
show GLOBAL VARIABLES like '%innodb_flush_log_at_trx_commit%';
```
接下来使用truncate命令将Order表数据清空，同时删除该表所有索引，然后首先测试插入百万数据的情况。

``` sql
call insert_emp(1, 1000000)
> OK
> 时间: 1129.966s

call insert_orders(1000000)
> OK
> 时间: 1345.076s
```