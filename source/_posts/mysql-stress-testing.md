---
title: Mysql单机插入性能测试
date: 2019-05-10 13:50:15
categories:
- mysql
tags:
- mysql
---

&emsp;&emsp;为了验证Mysql单机下单表数据量很大时的查询及插入性能，所以打算使用公司的App测试物理服务器做一个测试。之前也做过一些测试，只不过测试机用的是自己的电脑（2013年联想出的游戏本，i5，12G内存，1T硬盘，可惜我不玩什么游戏...）。机器配置信息如下：

## 测试机硬件及软件信息
当前测试的服务器是2路8核超线程也就是32核，32G内存，1T硬盘。
![服务器型号](./product.png)
![CPU信息](./cpu.png)
![CPU个数](./cpus.png)
![磁盘容量](./disk.png)
![内存信息](./memory.png)

``` sh
[root@env1 test]# dmidecode|grep -A5 "Memory Device"|grep Size|grep -v Range
	Size: 8192 MB
	Size: No Module Installed
	Size: No Module Installed
	Size: 8192 MB
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: 8192 MB
	Size: No Module Installed
	Size: No Module Installed
	Size: 8192 MB
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed
	Size: No Module Installed

```
测试系统： ** CentOS Linux release 7.1.1503 (Core) **

## Mysql版本
测试用到的Mysql数据库是社区版的 5.7.6：
![Mysql版本](./mysql版本.png)

## 创建数据表及相关函数、存储过程
首先创建测试使用的数据表，有部门、员工及订单三个表。表的创建语句如下：

``` sql
-- 部门表
CREATE TABLE `test`.`dept`  (
  `id` int(32) NOT NULL AUTO_INCREMENT,
	deptno MEDIUMINT UNSIGNED not null default 0,
	dname varchar(50) not null default '',
	loc VARCHAR(13) not null DEFAULT '',
	create_time datetime(3) ,
		PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 
COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- 员工表
CREATE TABLE `test`.`emp`  (
  `id` int(32) NOT NULL AUTO_INCREMENT,
	empno MEDIUMINT UNSIGNED not null default 0,
	deptno MEDIUMINT UNSIGNED not null default 0,
	mgr MEDIUMINT UNSIGNED not null default 0,
	ename varchar(20) not null default '',
	job varchar(20) not null default '',
	hiredate date ,
	sal DECIMAL(7,2) default 0,
	comm DECIMAL(7,2) default 0,
	create_time datetime(3) ,
	
	email VARCHAR(30) default '',
	mobile VARCHAR(20) DEFAULT '',
	weight int DEFAULT 0,
	married int DEFAULT 0,
	pwd VARCHAR(20) DEFAULT '123456',
	
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 
COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- 订单表
CREATE TABLE `test`.`order`  (
  `id` int(32) NOT NULL AUTO_INCREMENT,
	pay_status int not null default 0,
	trade_status int not null default 0,
  order_no VARCHAR(30) not null default '',
  mobile VARCHAR(20) not null default '',
  user_id int not null default 0,
	total DECIMAL(20,2) not null default 0,
  real_total DECIMAL(20,2) not null default 0,
  channel_id int not null default 0,
  open_id VARCHAR(50) not null default '',
  union_id VARCHAR(50) not null default '',
  transaction_no VARCHAR(50) not null default '',
  pay_time datetime(6),
	create_time datetime(3) ,
  deleted bit(1) not null default 0,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 
COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

```

接着创建执行插入语句需要用到的一些随机函数：

``` sql

-- 员工随机ID
delimiter $$
create function rand_userid() returns int(5)
    begin
    declare i int default 0;
    set i = floor(1000000+rand()*10);
    return i;
end $$

-- 随机字符串
delimiter $$
create function rand_num_str(length int) returns varchar(255)
begin

   declare chars_str varchar(100) default '0123456789';
   declare return_str varchar(255) default '';
   declare i int default 0;
   while i < length do 
       set return_str = concat(return_str, substring(chars_str,floor(1+rand()*10),1));
       set i = i +1;
   end while;
   return return_str;
end $$

-- 随机数字
delimiter $$
create function rand_num(max int) returns int
begin
   return floor(rand()* max);
end $$

-- 随机部门ID
delimiter $$
create function rand_deptid() returns int(5)
begin
   declare i int default 0;
   set i = floor(1000+rand()*10);
   return i;
end $$

-- 随机指定长度字符串
delimiter $$
create function rand_string(n int) returns varchar(255)
begin

   declare chars_str varchar(100) default 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
   declare return_str varchar(255) default '';
   declare i int default 0;
   while i < n do 
       set return_str = concat(return_str, substring(chars_str,floor(1+rand()*52),1));
       set i = i +1;
   end while;
   return return_str;
end $$

-- 随机手机号
delimiter $$
CREATE FUNCTION `rand_mobile`() RETURNS varchar(11) CHARSET utf8
BEGIN
	#Routine body goes here...
	declare chars_str varchar(100) default '0123456789';
	declare prefix_str varchar(100) default '356789';
  declare return_str varchar(255) default '';
	declare i int default 0;
	
	set return_str = concat('1',return_str, substring(prefix_str,floor(1+rand()*6),1));
	
  while i < 9 do 
      set return_str = concat(return_str, substring(chars_str,floor(1+rand()*10),1));
      set i = i + 1;
  end while;
	
  return return_str;
END
end $$

-- 随机邮箱
delimiter $$
CREATE FUNCTION `rand_email`() RETURNS varchar(30) CHARSET utf8
BEGIN
	#Routine body goes here...
	declare chars_str varchar(100) default 'abcdefghijklmnopqrstuvwxyz';
  declare return_str varchar(50) default '';
  declare i int default 0;
	declare type int default 0;
	
  while i < 5 do 
      set return_str = concat(return_str, substring(chars_str,floor(1+rand()*26),1));
      set i = i +1;
  end while;
	set return_str = concat(return_str, '.');
	while i < 10 do 
      set return_str = concat(return_str, substring(chars_str,floor(1+rand()*26),1));
      set i = i +1;
  end while;
	
 	set return_str = concat(return_str, '@');
 	set type= floor(1+rand()*7);

	if type=0 then
	   set return_str = concat(return_str, 'qq');
	elseif type=1 then
	   set return_str = concat(return_str, '163');
	elseif type=2 then
	   set return_str = concat(return_str, 'gmail');
	elseif type=3 then
	   set return_str = concat(return_str, 'wt');
	elseif type=4 then
	   set return_str = concat(return_str, '126');
	elseif type=5 then
	   set return_str = concat(return_str, 'yahoo');
	elseif type=6 then
	   set return_str = concat(return_str, 'hotmail');   -- msn  sogou inbox  sina
 	elseif type=7 then
	   set return_str = concat(return_str, 'msn');
	end if;
		
	set return_str = concat(return_str, '.com');
	
  return return_str;
END
end $$
```
函数创建完成后接着创建存储过程：
``` sql
-- 插入订单
delimiter $$
create procedure insert_orders(in max_num int(10))
begin
   declare i int default 0;
   declare total decimal(20,2) default 0;
   declare real_total decimal(20,2) default 0;
   set autocommit = 0;
   repeat
      set i = i+1;
	  set total = rand()*1000;
	  set real_total = total - rand()*100;
	  set mins = concat('00:', floor(rand()* 20),':', floor(rand()* 30));

      insert into `order`(pay_status,trade_status,order_no, mobile, 
	  user_id, total, real_total ,channel_id ,open_id, union_id, 
	  transaction_no , pay_time, create_time)
      values( rand_num(4), rand_num(8),
	  concat( '4081' , date_format(sysdate(),'%Y%m%d%h%i%s'), rand_num_str(5)),
      concat( '185' ,rand_num_str(8)), rand_userid(), total, real_total,
	  rand_deptid(),  rand_string(15), replace(UUID(),'-',''),
	  concat( '4200000' , rand_num_str(16)),
	  date_add(now(), interval mins Hour_second) , now(3));

      until i = max_num

   end repeat;
   commit;
end $$

-- 插入员工
delimiter $$
create procedure insert_emp(in start int(10),in max_num int(10))
begin
   declare i int default 0;
   set autocommit = 0;
   repeat
      set i = i+1;
      insert into emp(empno,ename,job,mgr,hiredate,sal,comm,deptno,create_time)
      values((start+1), concat('employee',rand_string(7)), 'saleman', 0001, curdate(),2000,400,rand_deptid(),now(3));
      until i=max_num

   end repeat;
   commit;
end $$

-- 插入部门
delimiter $$
create procedure insert_dept(in start int(10),in max_num int(10))
begin
   declare i int default 0;
   set autocommit = 0;
   repeat
      set i = i+1;
      insert into dept(deptno,dname,loc,create_time)values((start+1),
      concat( 'department_' , rand_string(10)), rand_string(12),now(3));
      until i=max_num

   end repeat;
   commit;
end $$

```
## 百万级数据测试
&emsp;&emsp; 插入百万耗时30分钟，磁盘占用 134 M，之前在自己电脑上测试同样的案例数据所花的时间是 7分20秒。
![插入百万员工数据](./插入百万员工数据.png)
![百万员工数据总量](./百万员工数据总量.png)

&emsp;&emsp;做一个简单查询看看耗时：
![跨90万员工查询](./skill_90_thousand_search.png)

## 千万级数据数据测试
&emsp;&emsp;现在来测试表容量达到500万以上的查询效率，这次再来插入700万，让容量达到800万。
![插入700万员工耗时](./插入700万员工耗时.png)
&emsp;&emsp;为了方便阅读，我将 7000000 写成了 700*10000，等数据插入完成后发现实际插入了1400万数据，所以前面耗时应该是插入1400万数据的耗时，不知道是不是我改成了方便阅读写法的问题。既来之则安之，那就使用这个量级来做测试吧。

&emsp;&emsp;在执行过程中让我们来服务器的CPU使用情况，CPU已被mysql占满：
![CPU使用率](./CPU使用率.png)
&emsp;&emsp;插入完成后查看磁盘占用情况：
![1500万员工磁盘占用](./1500/1500万员工磁盘占用.png)

### 无索引查询及性能分析
下面开始执行在无索引情况下的查询，首先根据任一手机号或员工姓名做一个查询耗时对比：
![根据手机号或员工姓名查询耗时对比](./1500/根据手机号或员工姓名查询耗时对比.png)
结果发现他们都花了将近10秒的时间才查询到响应结果。再使用explain来做下性能分析：
![手机号查询性能分析](./explain/explain_mobile.png)
![姓名查询性能分析](./explain/explain_ename.png)
可以看到他们都是用的是全表扫码，因为没有建相关索引。

再来看看limit与between （id）的查询对比
![between与limit查询对比](./1500/between_limit.png)
使用explain来做下性能分析：
![limit性能分析](./explain/limit.png)
![between性能分析](./explain/id_between_and.png)
between使用到了范围查找，查询的行数是100，而limit使用的却是全表扫描。如果给between及limit 查询都加上order by会是什么情况，直接看explain的结果：
![ orderby limit性能分析](./explain/orderby_limit.png)
![between orderby性能分析](./explain/between_orderby.png)
可以看到在不加orderby的情况下，limit使用的是全表扫描；在加orderby的情况下，limit使用的是部分扫描，并且查询结果越靠后查询耗时也随之增大。因为between使用的是id主键，所以自然使用到了主键索引，查询效率比较高。如果between使用的weight（无索引）是做范围查询，性能如何呢？
![根据weight做between范围查询性能分析](./explain/weight_between_and.png)
可以看到同样是全表扫描，性能也是比较底的。

### 有索引查询及性能分析
下面来建立索引，涉及的索引有员工表emp中的mobile，ename，weight三个属性，因为前两个都是字符串类型，所以打算将他们建一个符合索引，最后一个做一个单独索引。
``` sql
  CREATE INDEX index_emp_weight ON emp (`weight`);
  CREATE INDEX index_emp_mobile_ename ON emp (mobile,ename);
```
![创建索引](./create_index.png)
![查看表索引](./indexs.png)
可以看到大数据量比较大的时候创建索引也是要花费时间的，在数据更新及删除的时候也会重建索引，所以索引并不是越多越好。现在来看看weight范围查找性能分析：
![增加weight索引后的性能分析](./explain/explain_index_wight.png)
执行Sql发现耗时就几毫秒，效率就高了很多。

看看mobile的查询性能
![增加复合索引后mobile的性能分析](./explain/explain_index_mobile.png)
看看ename的查询性能，没有用到索引，因为没有遵循最左前缀法则，已经跳过了mobile这一列。只有查询条件中的列名顺序与索引中的列名顺序一致，查询时才会使用到相应的复合索引。
![增加复合索引后ename的性能分析](./explain/explain_index_ename.png)

![遵循最左前缀法则的复合索引性能分析](./explain/explain_index_mobile_ename.png)

![1500万查询耗时比较](./1500万查询耗时比较.png)

## 亿级数据测试
接下来试试5000万以上的有索引情况下的查询性能，首先插入5000万数据，预计耗时7小时左右。我是在下午6点多下班的时候开始执行的Sql，等到第二天8点多到公司通过 information_schema 库来查询已插入的一个大概数量发现已经达到6700万，但是还在执行插入。无奈只能通过systemctl暴力停止mysql服务，但是命令就一直卡住，然后又通过 kill -9 来关闭进程都没有成功，kill多次mysql服务又自行启动，还好这个是测试机只能重启，重启后竟然无法启动，泪奔...后来进机房连接了下显示器强按电源键关机后再重启，系统提示重启将会丢失内存数据，只能让它丢了，最后重启成功。后来才知道原来是因为我停止MySQL服务后未提交的数据做了回滚操作，导致服务迟迟不能停止（通过innodb_flush_log_at_trx_commit表可以看到有一行ROLLINGBACK的数据，就算重启服务后还是在做回滚操作，以确保数据完整性）。

``` sql
use information_schema;
select * from innodb_sys_tablestats where name='test/emp';
```
&emsp;&emsp;再次登录数据库，试着查询1500万之后的数据，发现并没有查询到相关数据。再看看磁盘空间，已经占用了17G。因为中途停止服务导致了事务回滚，对于已经存在的数据MySQL不会做回收，只是不会将他们放到表中。
![插入5000万后的磁盘占用](./50million_diskusage.png)
![查询1500万后的3000多条记录](./seach_after_15million.png)
![通过innodb_sys_tablestats查询实际有6300多万记录](./row_nums.png)


暂时不知道怎么恢复这5000多万数据，只能清空表了。主要还是想试试上千万及上亿数据的查询性能，所以先试试插入五千万数据：
``` sql
call insert_emp(1000000,50000000); # 前面已插入了100万数据，再插入5000万
```
第二天看下结果，实际插入的数据已翻倍：
``` sql
SELECT count(id) from emp
> OK
> 时间: 27.037s

count(id): 101000000

SELECT create_time from emp where id = 1000001;   # 第一条数据插入的时间   2019-05-14 17:04:11.766
SELECT create_time from emp where id = 101000000; # 最后一条数据插入的时间 2019-05-15 08:08:09.415
```
这个有个奇怪的地方，我插入百万数据都没有翻倍，插入千万数据就会翻倍，试过两次都是如此。看了下存储过程也没有看到哪里有问题。由于插入过程中丢失了连接，所以只能通过查询最前与最后一条记录的插入时间来统计花费的时间，实际花费15个小时。
``` shell
[root@env1 test]# ll -h
-rw-r----- 1 mysql mysql   61 5月  10 18:09 db.opt
-rw-r----- 1 mysql mysql 8.5K 5月  10 18:12 dept.frm
-rw-r----- 1 mysql mysql 176K 5月  13 11:11 dept.ibd
-rw-r----- 1 mysql mysql 8.9K 5月  14 15:57 emp.frm
-rw-r----- 1 mysql mysql  13G 5月  15 08:08 emp.ibd
-rw-r----- 1 mysql mysql 8.9K 5月  10 18:12 order.frm
-rw-r----- 1 mysql mysql  96K 5月  10 18:12 order.ibd
```
来看看between与limit的查询性能（该表尚未创建索引）：

``` sql
SELECT * from emp where id BETWEEN 29000000 and 29000100
> OK
> 时间: 0.01s

SELECT * from emp where id BETWEEN 101000000 and 101002000
> OK
> 时间: 0.003s

SELECT * from emp limit 90000000,100
> OK
> 时间: 52.95s
> 
#                     排序

SELECT * from emp ORDER BY id limit 90000000,100
> OK
> 时间: 52.445s

SELECT * from emp ORDER BY `weight` limit 50000000,100
> OK
> 时间: 417.639s

#                     分组
SELECT * from emp group BY id limit 90000000,100
> OK
> 时间: 52.737s

# 根据deptno单列来分组会报错： 
# > 1055 - Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'test.emp.id' which is not  
#          functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
#
# 因为mysql 5.7.5之后的 sql_mode 默认是 only_full_group_by，其常用值有：
#
# ONLY_FULL_GROUP_BY： 对于GROUP BY聚合操作，如果在SELECT中的列，没有在GROUP 
#                      BY中出现，那么这个SQL是不合法的，因为列不在GROUP BY从句中
#
# NO_AUTO_VALUE_ON_ZERO： 该值影响自增长列的插入。默认设置下，插入0或NULL代表生成下一个自#增长值。
#                        如果用户希望插入的值为0，而该列又是自增长的，那么这个选项就有用了。
#
# STRICT_TRANS_TABLES： 在该模式下，如果一个值不能插入到一个事务表中，则中断当前的操作，对非事务表不做限制 
#
# NO_ZERO_IN_DATE：     在严格模式下，不允许日期和月份为零
#
# NO_ZERO_DATE：        设置该值，mysql数据库不允许插入零日期，插入零日期会抛出错误而不是警告。
#
# ERROR_FOR_DIVISION_BY_ZERO： 在INSERT或UPDATE过程中，如果数据被零除，则产生错误而非警告。
#                              如果未给出该模式，那么数据被零除时MySQL返回NULL
#
# NO_AUTO_CREATE_USER： 禁止GRANT创建密码为空的用户
#
# NO_ENGINE_SUBSTITUTION： 如果需要的存储引擎被禁用或未编译，那么抛出错误。不设置此值时，用默认的存储引擎替代，并抛出一个异常
#
# PIPES_AS_CONCAT： 将”||”视为字符串的连接操作符而非或运算符，这和Oracle数据库是一样的，也和字符串的拼接函数Concat相类似 
#
# ANSI_QUOTES：     启用ANSI_QUOTES后，不能用双引号来引用字符串，因为它被解释为识别符
SELECT * from emp group BY deptno, id limit 50000000,100  # 根据部门分组
> OK
> 时间: 463.735s   

SELECT * from emp group BY married, id limit 90000000,100 # 根据婚否分组
> OK
> 时间: 377.36s


SELECT * from emp where mobile='17709585262'  # 查询最后一行记录中的手机号
> OK
> 时间: 62.272s


SELECT * from emp where create_time < '2019-05-14 17:01:37.824'
> OK
> 时间: 207.743s

```
可以看到在非索引键中做分组和排序的性能都很低，下面来创建索引试试：
``` sql
create INDEX index_emp_weight on emp(`weight`);
create INDEX index_emp_married on emp(married);
create INDEX index_emp_deptno on emp(deptno);
create INDEX index_emp_create_time on emp(create_time);
create INDEX index_emp_mobile_name on emp(mobile,ename);
```
创建一个索引需要花几分钟
``` sql
create INDEX index_emp_weight on emp(`weight`)
> OK
> 时间: 336.096s

create INDEX index_emp_married on emp(married)
> OK
> 时间: 263.363s

create INDEX index_emp_deptno on emp(deptno)
> OK
> 时间: 257.514s

create INDEX index_emp_create_time on emp(create_time)
> OK
> 时间: 265.776s


create INDEX index_emp_mobile_name on emp(mobile,ename)
> OK
> 时间: 568.012s

[root@env1 test]# ll -h     # 磁盘多了11个G的索引数据
-rw-r----- 1 mysql mysql   61 5月  10 18:09 db.opt
-rw-r----- 1 mysql mysql 8.5K 5月  10 18:12 dept.frm
-rw-r----- 1 mysql mysql 176K 5月  13 11:11 dept.ibd
-rw-r----- 1 mysql mysql 8.9K 5月  15 10:17 emp.frm
-rw-r----- 1 mysql mysql  24G 5月  15 10:26 emp.ibd
-rw-r----- 1 mysql mysql 8.9K 5月  10 18:12 order.frm
-rw-r----- 1 mysql mysql  96K 5月  10 18:12 order.ibd
```

``` sql

SELECT * from emp ORDER BY weight limit 1000000,100  # 不加任何条件的排序非常耗时
> OK
> 时间: ...很长

SELECT * from emp where deptno=1001 ORDER BY weight limit 1000000,100
> OK
> 时间: 142.618s

# 在有索引情况下无条件排序非常耗时， 因为没有使用到索引，使用的是全表扫描，还用到了文件内排序
SELECT * from emp group BY deptno,id limit 50000000,100  
> OK
> 时间: 472.974s

# 增加索引后查询很快
SELECT * from emp where mobile='17709585262'
> OK
> 时间: 0.171s

SELECT * from emp where mobile='17433959067' and ename='employeeYWPjAYp'
> OK
> 时间: 0.099s

SELECT * from emp where weight BETWEEN 1 and 40 limit 100
> OK
> 时间: 0.155s

SELECT * from emp where create_time < '2019-05-14 17:01:37.824'
> OK
> 时间: 144.509s  # 两分钟以上

SELECT * from emp where create_time < '2019-05-14 17:01:37.824' limit 100
> OK
> 时间: 0.012s

SELECT * from emp where create_time <= '2019-05-14 17:50:37.824' and create_time >= '2019-05-14 17:01:37.824' limit 100
> OK
> 时间: 0.013s

SELECT * from emp where create_time BETWEEN '2019-05-14 17:01:37.824' and '2019-05-14 17:50:37.824' limit 100;
> OK
> 时间: 0.011s
```
在无条件的情况下做的查询还是非常的慢，因为没有使用到相关索引。而在有条件语句中使用到了索引列（定值查询）也不一定很快，因为后面的排序有可能用到文件内排序。使用explain 分析下排序order by 发现这里用到了文件内排序，因为查询条件字段与排序字段不同。
mysql无法利用已有索引完成的排序操作称为文件排序，mysql会对数据使用一个外部的索引排序。出现该情况时表示查询性能已经很差。
![文件内排序](./filesort.png)
![非文件内排序](./not_filesort.png)

来看看create_time的查询性能：
![create_time范围查询分析](./explain/explain_create_time.png)
![create_time范围查询分析](./explain/explain_create_time_between_limit.png)
![create_time between范围查询分析](./explain/explain_create_time_range_limit.png)

如果删除create_time索引后性能又如何：
![create_time 无索引下范围查询分析](./explain/explain_create_time_range_all.png)
![create_time 无索引下between范围查询分析](./explain/explain_create_time_between_all.png)

## 再试插入亿级数据
&emsp;&emsp;好奇心作怪又试了下插入order表9000万数据，结果还是翻倍了，变成了1.8亿。最终发现这次数据插入明显慢了很多，耗时36个小时。第一条记录的创建时间是 **2019-05-15 17:48:23.615**，最后一条记录的创建时间是 **2019-05-17 05:51:50.106**。若果按照插入一亿数据耗时15个小时的比例计算，那么插入两亿的数据应该耗时30个小时，实际还没有到两亿已经花费了36个小时。

&emsp;&emsp;在数据插入过程中通过每隔一两秒刷新 information_schema.INNODB_SYS_TABLESTATS 中order对应的 NUM_ROWS 属性会发现，mysql的每秒钟的平均插入量大约在 2000 左右，当时只是简单做了一个心算：
``` sql
use information_schema;
SELECT * from information_schema.INNODB_SYS_TABLESTATS where `name` ='test/order';
```
更加精确的做法是通过查询每条记录的创建时间及主键ID来做统计，因为主键ID是自增主键，所以只要看看每秒钟生成了多少个主键即可。每次抽取大概一万条记录，然后对其该样本的前面完整的一秒及该样本的最后完整一秒做统计然后求平均数，简单做了一个抽样统计得出的一个大概的TPS：
``` sql
select id,create_time from `order` where id BETWEEN 1010000 and 1020000;  # 710 tps 

select id,create_time from `order` where id BETWEEN 4010000 and 4020000;  # 710 tps 

select id,create_time from `order` where id BETWEEN 5010000 and 5019000;  # 710 tps 

select id,create_time from `order` where id BETWEEN 5810000 and 5819000;  # 4000 tps 

select id,create_time from `order` where id BETWEEN 6010000 and 6019000;  # 4000 tps 

select id,create_time from `order` where id BETWEEN 9010000 and 9019000;  # 4000 tps 

select id,create_time from `order` where id BETWEEN 29010000 and 29023000;  # 3800 tps 

select id,create_time from `order` where id BETWEEN 89010000 and 89020000;  # 4000 tps 

select id,create_time from `order` where id BETWEEN 100010000 and 100022000;  # 3900 tps 

select id,create_time from `order` where id BETWEEN 110010000 and 110022000;  # 4000 tps 

select id,create_time from `order` where id BETWEEN 112010000 and 112022000;  # 3300 tps 

select id,create_time from `order` where id BETWEEN 114010000 and 114022000;  # 750 tps 

select id,create_time from `order` where id BETWEEN 116010000 and 116022000;  # 700 tps 

select id,create_time from `order` where id BETWEEN 129010000 and 129020000;  # 710 tps 
```
可以看到插入到580万左右的时候性能开始有很大的提升，580万以下的TPS大约是710，580万到1.12亿的TPS大约是3875，1.14亿之后的TPS大约是720。按1.8亿耗时36小时算，那平均的TPS是 180000000/(36*3600) = 1388.88。这里可以不考虑磁盘空间问题，因为插入数据完成后磁盘的占用是 71%，order表数据占用是 34 G：
``` sh
[root@env1 test]# ll -h
总用量 58G
-rw-r----- 1 mysql mysql   61 5月  10 18:09 db.opt
-rw-r----- 1 mysql mysql 8.5K 5月  10 18:12 dept.frm
-rw-r----- 1 mysql mysql 176K 5月  13 11:11 dept.ibd
-rw-r----- 1 mysql mysql 8.9K 5月  15 11:29 emp.frm
-rw-r----- 1 mysql mysql  24G 5月  15 11:34 emp.ibd
-rw-r----- 1 mysql mysql 8.9K 5月  10 18:12 order.frm
-rw-r----- 1 mysql mysql  34G 5月  17 05:51 order.ibd

[root@env1 test]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root  100G   71G   30G   71% /
devtmpfs                  16G     0   16G    0% /dev
tmpfs                     16G     0   16G    0% /dev/shm
tmpfs                     16G   74M   16G    1% /run
tmpfs                     16G     0   16G    0% /sys/fs/cgroup
/dev/sda2                497M  113M  384M   23% /boot
/dev/sda1                200M  9.8M  191M    5% /boot/efi
/dev/mapper/centos-home  112G  728M  111G    1% /home
/dev/mapper/centos-data  667G  100G  568G   15% /data
tmpfs                    3.1G     0  3.1G    0% /run/user/0

```
再来看看前面插入员工表的效率如何，同样做抽样统计：
``` sql
select id,create_time from `emp` where id BETWEEN 1010000 and 1020000;  # 930 tps 

select id,create_time from `emp` where id BETWEEN 4010000 and 4020000;  # 950 tps 

select id,create_time from `emp` where id BETWEEN 5010000 and 5019000;  # 950 tps 

select id,create_time from `emp` where id BETWEEN 6010000 and 6019000;  # 950 tps 

select id,create_time from `emp` where id BETWEEN 9010000 and 9023000;  # 4600 tps 

select id,create_time from `emp` where id BETWEEN 29010000 and 29023000;  # 3400 tps 

select id,create_time from `emp` where id BETWEEN 59010000 and 59023000;  # 4100 tps 

select id,create_time from `emp` where id BETWEEN 65010000 and 65023000;  # 4000 tps 

select id,create_time from `emp` where id BETWEEN 67010000 and 67024000;  # 3800 tps 

select id,create_time from `emp` where id BETWEEN 69010000 and 69023000;  # 960 tps 

select id,create_time from `emp` where id BETWEEN 79010000 and 79023000;  # 970 tps 

select id,create_time from `emp` where id BETWEEN 89010000 and 89020000;  # 970 tps 

select id,create_time from `emp` where id BETWEEN 100010000 and 100022000;  # 950 tps 
```
按1亿耗时15小时算，那平均的TPS是 100000000/(15*3600) = 1851.85 。
对比两个表的插入过程可以得出一个结果，就是他们在前后过程中的效率都比较低，而中间的效率就比较高。最终单机Myql在未做任何优化下的TPS预计在1500左右，能达到2000确实算很高了，这个结论也刚好符合我听说的预期。
&emsp;&emsp;接着为order创建索引，然后再试试查询性能。无索引的情况就不用考虑了，随便查一个无索引的键应该都要花上半个钟。
``` sql
create INDEX index_order_userid on `order`(user_id);
create INDEX index_order_mobile on `order`(mobile);
create INDEX index_order_orderno on `order`(order_no);
create INDEX index_order_channelid on `order`(channel_id);
create INDEX index_order_create_time on `order`(create_time);

-- 结果
create INDEX index_order_userid on `order`(user_id)
> OK
> 时间: 763.333s


create INDEX index_order_mobile on `order`(mobile)
> OK
> 时间: 1381.539s


create INDEX index_order_orderno on `order`(order_no)
> OK
> 时间: 1961.026s


create INDEX index_order_channelid on `order`(channel_id)
> OK
> 时间: 764.187s

create INDEX index_order_create_time on `order`(create_time)
> OK
> 时间: 836.578s
```
索引创建完成，磁盘增加了20G。来看看查询效率：
``` sql
SELECT * from `order`  where order_no = '40812019051508101611013'
> OK
> 时间: 0.076s

SELECT * from `order`  where order_no = '40812019051602502959435'
> OK
> 时间: 0.02s

SELECT * from `order`  where user_id = 1000003 limit 100
> OK
> 时间: 0.073s

# 因为mobile是字符串类型，所以查询时不加单引号会导致全表扫码
SELECT * from `order`  where mobile ='18574145487' limit 100 
> OK
> 时间: 0.015s

SELECT * from `order`  where user_id = 1000003 GROUP BY channel_id,id limit 100
> OK
> 时间: 0.021s

SELECT * from `order`  where create_time>'2019-05-16 17:48:23.948'  and create_time<'2019-05-16 22:48:23.948' limit 100
> OK
> 时间: 0.022s

# 下面这两条语句中的查询条件没有加索引都会导致全表扫描
SELECT * from `order` where total > 751.21 limit 100
SELECT * from `order` where pay_status = 1 limit 100

select * from `order` where channel_id=1002 ORDER BY create_time desc limit 100
> OK
> 时间: 0.019s

SELECT * from `order` a 
INNER join emp b on a.user_id = b.id
GROUP BY a.user_id,a.id
limit 100
> OK
> 时间: 0.036s

# 该SQL通过explain分析会发现它会使用到临时表及文件内排序，
# 这两个很影响的性能的东西它都沾上了，限于篇幅这里暂时不考虑优化
SELECT a.mobile,a.order_no,a.user_id,a.channel_id from `order` a  -- force index(index_order_mobile)
INNER join emp b on a.user_id = b.id
where  a.create_time >'2019-05-16 17:48:23.948'  
and a.create_time < '2019-05-16 22:48:23.948'
GROUP BY a.channel_id , a.mobile,a.order_no,a.user_id
limit 100
```

## 最后
&emsp;&emsp;通过这次的测试发现，在没有对MySQL优化的前提下，其平均TPS在1500左右。再一个是单表数据达到上亿时，通过添加相应索引后其查询效率也不低（还有什么问题没有考虑到吗？有点虚...）。但是单表上亿数据占用的磁盘比较大，实际项目中的表不是几个表而是几十个上百个表，所以按照目前测试的Order表1.8亿占用54G空间算的话，缩小10倍就是1800万的数据占用5.4G空间。相当于一个表维持1800万数据占用5.4G，如果项目中有50个表就占用270G，再考虑数据归档的话可以想想也是不少的磁盘占用。这篇幅有点长，下一篇再来试试对MySQL做部分优化后在同样的测试环境下的插入性能。
