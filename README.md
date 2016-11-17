# MySQL优化

## 索引
**建立索引的四个原则**
1. 较频繁的作为查询条件的字段应该创建索引
2. 唯一性太差的字段不适合单独创建索引，即使频繁作为查询条件
3. 更新非常频繁的字段不适合创建索引
4. 不会出现在WHERE子句中的字段不创建索引

**MySQL 中索引的限制**
1. MyISAM 存储引擎索引键长度总和不能超过1000 字节
2. BLOB 和TEXT 类型的列只能创建前缀索引
3. MySQL 目前不支持函数索引
4. 使用不等于（!= 或者<>）的时候MySQL 无法使用索引
5. 过滤字段使用了函数运算后（如abs(column)），MySQL 无法使用索引
6. Join 语句中Join 条件字段类型不一致的时候MySQL 无法使用索引
7. 使用LIKE 操作的时候如果条件以通配符开始（ '%abc...'）MySQL 无法使用索引
8. 使用非等值查询的时候MySQL 无法使用Hash 索引

自建索引，对全文本搜索有奇效，可用于解决like查询的扫表问题


## SQL语句
## 主从复制
## 分库分表
### 垂直分表
将表按照功能模块、关系密切程度划分出来，部署到不同的数据表上。
* 比如user表中和user_details表。
* 比如博客表中的title和content表
* 大字段垂直切分

垂直切分的优点
数据库的拆分简单明了，拆分规则明
* 应用程序模块清晰明确，整合容易
* 数据维护方便易行，容易定位

垂直切分的缺点
◆部分表关联无法在数据库级别完成，需要在程序中完成
◆对于访问极其频繁且数据量超大的表仍然存在性能瓶颈，不一定能满足要求
◆事务处理相对更为复杂
◆切分达到一定程度之后，扩展性会遇到限制
◆过度切分可能会带来系统过渡复杂而难以维护


### 水平分表
依据的条件可以是时间、地域、功能等比较清晰的条件
* 比如财务报表、薪资发放就可以用时间进行水平分割；
* 比如商品库存就可以用地域进行分割
* 比如用户表的普通用户、商户就可以用功能来进行划分


**水平通用分表策略**
1. 以uuid作为全局唯一标识，为每一个新生成的用户生成uuid
2. 将uuid进行md5加密，生成16进制随机字符串，取随机字符串前两位进行10进制转换，对分表数量的取余，获取插入的表后缀名。
3. 比如建立8张表，对8取余，则会生成user_0...user_7，每个用户会随机插入这8张表中

**分表后，如何统计数据？**
所有统计数据都是根据业务需求而来的，原始数据存在的情况，我们可以进行自建索引，实现具体的业务需求。比如根据添加时间自建索引

** 分表后查询效率的问题？** 
* 根据自建索引表，获取uuid，再根据uuid获取数据每一行的数据。只不过多了一个10次的for循环而已，而php的10for循环可以说是微秒级的。
* 结果集存储的是指针
* mysql_fetch_row()读取磁盘文件
**水平切分的优点**
* 表关联基本能够在数据库端全部完成
* 不会存在某些超大型数据量和高负载的表遇到瓶颈的问题
* 应用程序端整体架构改动相对较少
* 事务处理相对简单
* 只要切分规则能够定义好，基本上较难遇到扩展性限制

**水平切分的缺点**
* 切分规则相对更为复杂，很难抽象出一个能够满足整个数据库的切分规则
* 后期数据的维护难度有所增加，人为手工定位数据更困难
* 应用系统各模块耦合度较高，可能会对后面数据的迁移拆分造成一定的困难

### 垂直与水平联合切分的使用
**联合切分的优点**
* 可以充分利用垂直切分和水平切分各自的优势而避免各自的缺陷
* 让系统扩展性得到最大化提升
**联合切分的缺点**
* 数据库系统架构比较复杂，维护难度更大；
* 应用程序架构也相对更复杂；


## 锁表
MySQL 各存储引擎使用了三种类型（级别）的锁定机制：行级锁定，页级锁定和表级锁定
### 行级锁定（row-level）
* 能够给予应用程序尽可能大的并发处理能力而提高一些需要高并发应用系统的整体性能
* 由于锁定资源的颗粒度很小，所以每次获取锁和释放锁需要做的事情也更多，带来的消耗自然也就更大了。此外，行级锁定也最容易发生死锁。
 

### 表级锁定（table-level）
* 实现逻辑非常简单，带来的系统负面影响最小。所以获取锁和释放锁的速度很快。由于表级锁一次会将整个表锁定，所以可以很好的避免困扰我们的死锁问题。
* 最大的负面影响就是出现锁定资源争用的概率也会最高，致使并发度大打折扣
### 页级锁定（page-level）
MySQL 中比较独特的一种锁定级别，在其他数据库管理软件中也并不是太常见

* 锁定颗粒度介于行级锁定与表级锁之间，所以获取锁定所需要的资源开销，以及所能提供的并发处理能力也同样是介于上面二者之间
* 使用表级锁定的主要是MyISAM，Memory，CSV 等一些非事务性存储引擎，而使用行级锁定的主要是Innodb 存储引擎和NDB Cluster 存储引擎，页级锁定主要是BerkeleyDB 存储引擎的锁定方式。

### MyISAM 表锁优化
1. 缩短锁定时间,让我们的Query 执行时间尽可能的短。
	* 尽量减少大的复杂Query，将复杂Query 分拆成几个小的Query 分步进行；
	* 尽可能的建立足够高效的索引，让数据检索更迅速；
	* 尽量让MyISAM 存储引擎的表只存放必要的信息，控制字段类型；
	
2. MyISAM 的存储引擎有Concurrent Insert（并发插入）的特性。制是否打开Concurrent Insert 功能的参数选项：concurrent_insert，可以设置为0，1 或者2。三个值的具体说明如下：
	* concurrent_insert=2，无论MyISAM 存储引擎的表数据文件的中间部分是否存在因为删除数据而留下的空闲空间，都允许在数据文件尾部进行Concurrent Insert;
	* concurrent_insert=1，当MyISAM 存储引擎表数据文件中间不存在空闲空间的时候，可以从文件尾部进行Concurrent Insert;
	* concurrent_insert=0，无论MyISAM 存储引擎的表数据文件的中间部分是否存在因为删除数据而留下的空闲空间，都不允许Concurrent Insert。

如果我们的系统是一个以读为主，而且要优先保证查询性能的话，我们可以通过设置系统参数选项low_priority_updates=1，将写的优先级设置为比读的优先级低，即可让告诉MySQL 尽量先处理读请求。 可以利用这个特性，将concurrent_insert 参数设置为1，甚至如果数据被删除的可能性很小的时候，如果对暂时性的浪费少量空间并不是特别的在乎的话，将concurrent_insert 参数设置为2 都可以尝试。当然，数据文件中间留有空余空间，在浪费空间的时候，还会造成在查询的时候需要读取更多的数据，所以如果删除量不是很小的话，还是建议将concurrent_insert 设置为1 更为合适。

### Innodb 行锁优化
虽然在锁定机制的实现方面所带来的性能损耗可能比表级锁定会要更高一些，但是在整体并发处理能力方面要远远优于MyISAM 的表级锁定的。当系统并发量较高的时候，Innodb 的整体性能和MyISAM 相比就会有比较明显的优势了

* 合理利用Innodb 的行级锁定
	* 尽可能让所有的数据检索都通过索引来完成，从而避免Innodb 因为无法通过索引键加锁而升级为表级锁定；
	* 合理设计索引，让Innodb 在索引键上面加锁的时候尽可能准确，尽可能的缩小锁定范围，避免造成不必要的锁定而影响其他Query 的执行；
	* 尽可能减少基于范围的数据检索过滤条件，避免因为间隙锁带来的负面影响而锁定了不该锁定的记录；
	* 尽量控制事务的大小，减少锁定的资源量和锁定时间长度；
	* 在业务环境允许的情况下，尽量使用较低级别的事务隔离，以减MySQL 因为实现事务隔离级别所带来的附加成本；

* 减少死锁产生概率建议
	* 类似业务模块中，尽可能按照相同的访问顺序来访问，防止产生死锁；
	* 在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁产生概率；
	* 对于非常容易产生死锁的业务部分，可以尝试使用升级锁定颗粒度，通过表级锁定来减少死锁产生的概率
* 系统锁定争用情况查询
	* 表级锁定的争用状态变量：mysql> show status like 'table%';
	+-----------------------+-------+
	| Variable_name | Value |
	+-----------------------+-------+
	| Table_locks_immediate | 100 |
	| Table_locks_waited | 0 |
	+-----------------------+-------+
	Table_locks_immediate：产生表级锁定的次数；
	Table_locks_waited：出现表级锁定争用而发生等待的次数；
	两个状态值都是从系统启动后开始记录，每出现一次对应的事件则数量加1。如果这里的Table_locks_waited 状态值比较高，那么说明系统中表级锁定争用现象比较严重，就需要进一步分析为什么会有较多的锁定资源争用了
		1. 以固定的顺序访问表和行。比如两个job批量更新的情形，简单方法是对id列表先排序，后执行，这样就避免了交叉等待锁的情形；又比如，将两个事务的sql顺序调整为一致，也能避免死锁。
		2. 大事务拆小。大事务更倾向于死锁，如果业务允许，将大事务拆小。
		3. 在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁概率。
		4. 降低隔离级别。如果业务允许，将隔离级别调低也是较好的选择，比如将隔离级别从RR调整为RC，可以避免掉很多因为gap锁造成的死锁。
		5. 为表添加合理的索引。可以看到如果不走索引将会为表的每一行记录添加上锁，死锁的概率大大增大。

	* 行级锁定，系统中是通过另外一组更为详细的状态变量来记录的，如下：
	mysql> show status like 'innodb_row_lock%';
	+-------------------------------+--------+
	| Variable_name | Value |
	+-------------------------------+--------+
	| Innodb_row_lock_current_waits | 0 |
	| Innodb_row_lock_time | 490578 |
	| Innodb_row_lock_time_avg | 37736 |
	| Innodb_row_lock_time_max | 121411 |
	| Innodb_row_lock_waits | 13 |
	+-------------------------------+--------+
	Innodb 的行级锁定状态变量不仅记录了锁定等待次数，还记录了锁定总时长，每次平均时长，以及最大时长，此外还有一个非累积状态量显示了当前正在等待锁定的等待数量。对各个状态量的说明如下：
		Innodb_row_lock_current_waits：当前正在等待锁定的数量；
		Innodb_row_lock_time：从系统启动到现在锁定总时间长度；
		Innodb_row_lock_time_avg：每次等待所花平均时间；
		Innodb_row_lock_time_max：从系统启动到现在等待最长的一次所花的时间；
		Innodb_row_lock_waits：系统启动后到现在总共等待的次数；
	对于这5 个状态变量，比较重要的主要是Innodb_row_lock_time_avg（ 等待平均时长） ，Innodb_row_lock_waits（等待总次数）以及Innodb_row_lock_time（等待总时长）这三项。尤其是当等待次数很高，而且每次等待时长也不小的时候，我们就需要分析系统中为什么会有如此多的等待，然后根据分析结果着手指定优化计划。

### 死锁解决
```
#查看正在锁的事务
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS; 
 
#查看等待锁的事务
SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS; 
```


## 数据类型
## 缓存





## 数据备份和恢复
**热备份**
mysqldump
利用第三方工具：mysqlhotcopy等
**冷备份**
复制数据文件

### mysqldump的使用
```
#基本的备份命令
mysqldump -h hostname -u username -p password databasename > backupfile.sql
#基本还原命令
mysql -h hostname -u username -p password databasename < backupfile.sql
#帮助
mysqldump --help
```
**mysqldump参数**
```
--single-transaction（针对innodb）
示例：
mysqldump -uroot --single-transaction --databases db1 db2 > test.sql
--single-transaction 的设置使整个导出的过程为一个事务，避免了锁。

--lock-tables
示例：
mysqldump -uroot --lock-tables --databases db1 db2 > test.sql
它在导出db1的时候,会对db1所有的表上锁,导出结束之后释放锁.然后再同样导出db2.
也就是说在db1导出的时候,db2的数据可能还在变化.

--master-data
它对所有数据库的所有表上了锁,并且查询binlog的位置，并填充到导出的文件中。请注意它会使用FLUSH TABLES。

--flush-logs
用于关闭并重新打开所有的日志文件。如果您已经指定了一个更新日志文件或一个二进制日志文件

```

>一张每秒都有数据在写入的 InnoDB 表，如果用 mysqldump --single-transaction 来备份的话会发生什么情况？
>答：
>只有在开始备份那一刻时已经存在的数据会被备份，备份过程中新加入的数据不会被备份。InnoDB内部有每个数据的时间戳，如果时间戳大于备份开始时的时间，该数据就会被忽略。如果有数据被删除且被删除的时间戳大于备份开始时的时间，该数据就会在备份中被还原。这样保证了所有数据的可靠性。

### 利用二进制bin-log日志增量备份
```
#查看bin-log日志
mysqlbinlog /var/log/mysql/mysql-bin.000001
mysqlbinlog /var/log/mysql/mysql-bin.000001 > /tmp/log.sql

#指定时间恢复
mysqlbinlog --start-date="2005-04-20 10:01:00" /var/log/mysql/bin.123456  | mysql -u root -pmypwd 
mysqlbinlog --stop-date="2005-04-20 10:01:00" /var/log/mysql/bin.123456  | mysql -u root -pmypwd 
#指定位置恢复
mysqlbinlog --stop-position="368312" /var/log/mysql/bin.123456  | mysql -u root -pmypwd 
mysqlbinlog --start-position="368312" /var/log/mysql/bin.123456  | mysql -u root -pmypwd 

#根据数据库名来进行还原 -d
#在这里是小写的d，请不要把它和mysqldump中的－D搞混了
[root@BlackGhost mysql]# /usr/local/mysql/bin/mysqlbinlog -d test  /var/lib/mysql/mysql-bin.000002

#根据数据库所在IP来分－h
[root@BlackGhost mysql]# /usr/local/mysql/bin/mysqlbinlog -h 192.1681.102  /var/lib/mysql/mysql-bin.000002

#根据数据库所占用的端口来分－P
#有的时候，我们的mysql用的不一定是3306端口，注意是大写的P
[root@BlackGhost mysql]# /usr/local/mysql/bin/mysqlbinlog -P 13306  /var/lib/mysql/mysql-bin.000002

#根据数据库serverid来还原–server-id
#在数据库的配置文件中，都有一个serverid并且同一集群中serverid是不能相同的。
[root@BlackGhost mysql]# /usr/local/mysql/bin/mysqlbinlog –server-id=1  /var/lib/mysql/mysql-bin.000002

#备份所有数据，并清空所有bin-log日志
./mysqldump –flush-logs -u root  –all-databases > /tmp/alldatabase.sql
```


冷备份文件位置
```
SHOW VARIABLES LIKE '%datadir%';
```

