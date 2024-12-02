
**大纲**


**1\.Explain概述**


**2\.Explain详解**


**3\.索引优化数据准备**


**4\.索引优化原则详解**


**5\.慢查询设置与测试**


**6\.慢查询SQL优化思路**


 


**1\.Explain概述**


使用Explain关键字可以模拟查询优化器来执行SQL查询语句，从而知道MySQL是如何处理SQL语句的，从而分析出查询语句和表结构的性能瓶颈。


 


**MySQL查询过程：**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/3f01a01845364ed085a36ccedbfb7960~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=0sHqcIEXh0mPZ27XM8yYpYtI6xg%3D)
![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/32b377e4a82e4276b1820573dc5e5223~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=6R%2FAEp3FsJJIUvwPB0upjeBqVSk%3D)
通过Explain可以获得以下信息：


一.表的读取顺序


二.数据读取操作的操作类型


三.哪些索引可以被使用


四.哪些索引真正被使用


五.表的直接引用


六.每张表的有多少行被优化器查询了


 


Explain使用方式：Explain \+ SQL语句，通过执行Explain可以获得SQL语句执行的相关信息。



```
EXPLAIN SELECT * FROM L1;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d8932d24e126419b83012764e71145fd~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=QF6lbJS0K7ruPa%2FpLS%2B4OeNptUo%3D)
 


**2\.Explain详解**


**(1\)数据准备**


**(2\)ID字段说明**


**(3\)select\_type和table字段说明**


**(4\)type字段说明**


**(5\)possible\_keys与key说明**


**(6\)key\_len字段说明**


**(7\)ref字段说明**


**(8\)rows字段说明**


**(9\)filtered字段说明**


**(10\)extra字段说明**


 


**(1\)数据准备**



```
-- 创建数据库
CREATE DATABASE test CHARACTER SET 'utf8';


-- 创建表
CREATE TABLE L1(id INT PRIMARY KEY AUTO_INCREMENT,title VARCHAR(100) );
CREATE TABLE L2(id INT PRIMARY KEY AUTO_INCREMENT,title VARCHAR(100) );
CREATE TABLE L3(id INT PRIMARY KEY AUTO_INCREMENT,title VARCHAR(100) );
CREATE TABLE L4(id INT PRIMARY KEY AUTO_INCREMENT,title VARCHAR(100) );


-- 每张表插入3条数据
INSERT INTO L1(title) VALUES('test001'),('test002'),('test003');
INSERT INTO L2(title) VALUES('test004'),('test005'),('test006');
INSERT INTO L3(title) VALUES('test007'),('test008'),('test009');
INSERT INTO L4(title) VALUES('test010'),('test011'),('test012');
```

**(2\)ID字段说明**


ID字段代表SELECT查询的序列号，它是一组数字，表示的是查询或操作表的顺序。


 


**一.ID相同，执行顺序就是由上至下**



```
EXPLAIN SELECT * FROM  L1,L2,L3 WHERE L1.id=L2.id AND L2.id = L3.id;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/c7c28c79cc5b44978aed8f78bdd62ff2~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=XqVQ1XppB8d2%2BuUa9uNQR4AtsaE%3D)
**二.ID不同，如果有子查询，ID号会递增**


ID值越大优先级越高，越先被执行。



```
EXPLAIN SELECT * FROM L2 WHERE id = (
    SELECT id FROM L1 WHERE id = (SELECT L3.id FROM L3 WHERE L3.title = 'test007')
);
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/85aff9c5d2ca45fd899d9ddafcfdcb0d~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=oAXODXOSAOG%2FLJ96KTAAxEYocNo%3D)
**(3\)select\_type和table字段说明**


select\_type表示查询类型，主要用于区别普通查询还是子查询等，table表示被操作的表。


 


**一.SIMPLE：简单的SELECT查询，查询中不包含子查询或者UNION**



```
EXPLAIN SELECT * FROM L1 where id = 1;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d143ba34a4d74002b391dcb4ee1f8245~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=PuFseINpZGQlltsWD4gQai2OHtY%3D)
**二.PRIMARY：在有子查询的情况下，最外层被标记为PRIMARY**


 


**三.SUBQUERY：在SELECT或WHERE列表中包含了子查询**



```
EXPLAIN SELECT * FROM L2 WHERE id = (
    SELECT id FROM L1 WHERE id = (SELECT L3.id FROM L3 WHERE L3.title = 'test08')
);
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/b5f9cc1d3d67405e9f64ceb58e0a7667~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=B1PP0HirI8dZoLO3W4kmWHoE3Mk%3D)
**四.UNION：UNION连接的两个SELECT查询**


在使用UNION时，左边的表的select\_type是DERIVED，右边的表的select\_type是UNION。


 


**五.DERIVED：在FROM列表中包含的子查询被标记为DERIVED派生表**


MySQL会递归执行这些被标记为DERIVED的子查询，然后把结果放到临时表中。


 


**六.UNION RESULT：UNION的结果**



```
EXPLAIN SELECT * FROM (SELECT * FROM L3 UNION SELECT * FROM L4) a;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/c2bb94e4129d435b9d93d05473acd89a~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=uP1ScKaJ1xMH0UveQb87O%2B0Nhnc%3D)
**(4\)type字段说明**


type字段表示的是连接类型，描述了找到所需数据而使用的扫描方式。


 


下面给出各种连接类型，按照从最好类型到最差类型进行排序：system \-\> const \-\> eq\_ref \-\> ref \-\> fulltext \-\> ref\_or\_null \-\> index\_merge \-\> unique\_subquery \-\> index\_subquery \-\> range \-\> index \-\> ALL


 


简化后，可以只关注以下几种 ：system \-\> const \-\> eq\_ref \-\> ref \-\> range \-\> index \-\> ALL


 


一般来说，需要保证查询至少达到range级别，最好能达到ref级别，否则就要就行SQL的优化调整。


 


下面介绍type字段不同值表示的含义：


 


**一.system**


表示表中仅有一行数据，这是const连接类型的一个特例，很少出现。


 


**二.const**


表示命中主键索引(primary key)或唯一索引(unique)，通过主键索引或唯一索引一次就找到了数据。因为只匹配一条记录，所以被连接的部分是一个常量。如果将主键放在where条件中，MySQL就能将该查询转换为一个常量。这种类型非常快，例如以下查询：



```
EXPLAIN SELECT * FROM L1 WHERE id = 3;
  
-- 为L1表的title字段添加唯一索引
ALTER TABLE L1 ADD UNIQUE(title);
EXPLAIN SELECT * FROM L1 WHERE title = 'test001';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/482c25e314d6455b87b5505d4698df2c~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=dihzeHcUlhnQ31eV3E%2BtPEjm7Ro%3D)
**三.eq\_ref**


表示的是使用了唯一索引。比如连表查询中，对于前一个表中的每一行，后表只有一行被扫描。除了system和const类型之外，这是最好的连接类型。只有在连表时使用的索引都是主键或唯一索引时，才会出现这种类型，例如以下查询：



```
EXPLAIN SELECT L1.id,L1.title FROM L1 LEFT JOIN L2 ON L1.id = L2.id;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4499ab8b15364f038175da76399a9e8f~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=ZuRPtYfd%2BV0wb43hm%2FztCT2dk3U%3D)
**四.ref**


表示使用了普通索引，即非唯一性索引。比如连表时对于前表的每一行，后表可能有多于一行的数据被扫描，例如以下查询：



```
-- 为L1表的title字段添加普通索引
ALTER TABLE L1 ADD INDEX idx_title (title);
EXPLAIN SELECT * FROM L1 INNER JOIN L2 ON L1.title = L2.title;
-- 如果L1表的title字段没有唯一索引，只有普通索引，如下查询也是ref
EXPLAIN SELECT * FROM L1 WHERE title = 'test001';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/22bae808168e41f7a829a0301e00694b~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=hcWYAiL2xesqKmJO1pgM1p4nLqs%3D)
**五.range**


表示的是进行了索引上的范围查询，检索了给定范围的行，比如between、in函数、\>都是典型的范围查询，例如以下查询：



```
EXPLAIN SELECT * FROM L1 WHERE L1.id BETWEEN 1 AND 10;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/ae87f84a07834228b7064dc72381f181~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=M0K4gYmkfGC1Fv%2FpF3GM51hk5rE%3D)
注意：当in函数中的数据很大时，可能会导致效率下降，最终不走索引。


 


**六.index**


当可以使用索引覆盖，但需要扫描全部索引记录时，则type为index。


 


当需要执行全表扫描，且需要对主键进行排序时，则type也为index。


 


所以如果type的值等于index，那么就需要进行优化了。因为出现index表示没有通过索引进行过滤，需要扫描索引的全部数据。index会遍历扫描索引树，比ALL快一些。如果索引文件过大，index的速度还是会很慢的。


 


**总结：**


当遍历二级索引不需要回表或者主键排序全表扫描时，type就为index。


 


**注意：**


使用索引进行排序分组时，可能会出现这种type值为index的情况。比如进行统计操作时，会出现type值为index的情况。



```
EXPLAIN SELECT * FROM L2 GROUP BY id ORDER BY id;
-- 该count查询需要通过扫描索引上的全部数据来计数
EXPLAIN SELECT count(*) FROM L2;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/ff0f99fa5cda461ebbb150ee58b18462~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=eUg7T0Tu9pzhylDEufhU8V8OdE0%3D)
**七.ALL**


表示没有使用到任何索引，连表查询时对于前表的每一行，后表都要被全表扫描。



```
EXPLAIN SELECT * FROM L3 inner join L4 on L3.title = L4.title;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/fbae1542ea9a48f2a50925a9fb505fa1~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=2G5Pgec9NATvchZY37WHHoQl1pU%3D)
**总结各类type类型的特点：**


system：不进行磁盘IO，查询系统表，仅仅返回一条数据。


 


const：查找主键索引，最多返回1条或0条数据，属于精确查找。


 


eq\_ref：查找唯一性索引，返回数据最多一条，属于精确查找。


 


ref：查找非唯一性索引，返回匹配的多条数据，属于精确查找。


 


range：查找索引中给定范围的行，属于范围查找(\>、\<、in、between)。


 


index：使用了索引但扫描全部了，比all快，因索引文件比数据文件小。


 


index：比如遍历二级索引不需要回表或者主键排序全表扫描。


 


all：不使用任何索引，直接进行全表扫描。


 


**(5\)possible\_keys与key说明**


**一.possible\_keys**


表示可能用于查询的表上的索引。查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询实际使用。


 


**二.key**


表示实际使用的索引。若为null，则表示没有使用到索引或索引失效。查询中若使用了覆盖索引，则该索引仅出现在key列表中。


 


情形一：理论和实际都用到了索引



```
EXPLAIN SELECT * FROM L1 WHERE id = 1;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/a4087013e6fb4201be316a0cb9067f85~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=GMOj1WXaiIzllU3Pk4vxE5CNDjk%3D)
情形二：理论上没有使用索引，但实际上使用了



```
EXPLAIN SELECT L3.id FROM L3;
```

![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d23b05ae2d574f25954bce7094ea3b4a~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=nzPtXVa2i7%2BDcrSliBMo64K9H3Q%3D)
情形三：理论和实际上都没有使用索引



```
EXPLAIN SELECT * FROM L3 WHERE title = 'test007';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/481efb8df10042308b900cee0548a3a9~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=lyND%2F8b9fbsXPBOX%2F8ErUSCJobY%3D)
**(6\)key\_len字段说明**


表示索引中使用的字节数，通过该列可以计算查询中使用索引的长度。key\_len字段能够帮我们检查是否充分利用了索引，ken\_len越长越好，说明索引使用的越充分。


 


**一.创建表**



```
CREATE TABLE L5(
    a INT PRIMARY KEY,
    b INT NOT NULL,
    c INT DEFAULT NULL,
    d CHAR(10) NOT NULL
);
```

**二.使用EXPLAIN进行测试**



```
-- 下面的查询只用到了主键a的索引
EXPLAIN SELECT * FROM L5 WHERE a > 1 AND b = 1;
```

观察key\_len的值，用到了主键索引，是int类型的，所以key\_len是4字节。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/cd15579e02fe4fee89a41b2f4139cbe5~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=wRc8uSi06Bm61Xc936xTGnqm0DM%3D)
**三.为b字段添加索引，进行测试**



```
ALTER TABLE L5 ADD INDEX idx_b(b);
-- 执行SQL，这次将b字段也作为条件
EXPLAIN SELECT * FROM L5 WHERE a > 1 AND b = 1;
```

ken\_len还是4。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/3e0e0d54ff194f47bdbebe3d0fb6f6d5~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=Vpe0eP%2FukIKjy12x3ZmhEzRmfGE%3D)
**四.为c、d字段添加联合索引，然后进行测试**



```
ALTER TABLE L5 ADD INDEX idx_c_b(c,d);
EXPLAIN SELECT * FROM L5 WHERE c = 1 AND d = 'A';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/3b13966bb43e46a2acadb822d0144641~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=a9QaKjGtc%2BwyHK7i2TgNw5qpKMc%3D)
c字段是int类型4个字节，d字段char(10\)代表的是10个字符30个字节。因为数据库的字符集是utf8，一个字符3个字节。d字段是char(10\)代表的是10个字符相当30个字节。多出的一个字节用来表示是联合索引。


 


下面这个例子虽然使用了联合索引，但没充分利用索引，还有优化空间。因为可以根据ken\_len的长度推测出该联合索引只使用一部分。



```
EXPLAIN SELECT * FROM L5 WHERE c = 1;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/9fbece85b71440bfa09d7edd02a7e504~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=uiwnQJqgy%2F8Z42ujfXw6Dvb8I%2BE%3D)
**(7\)ref字段说明**


表示的是显示索引的哪一列被使用了，如果可能的话，最好是一个常数。表示的是哪些列或常量被用于查找索引列上的值。


 


如下的"L1\.id\=1"中，由于1是常量，所以ref \= const，此时的ref \= const表示着查询过程中使用到了常量。



```
EXPLAIN SELECT * FROM L1 WHERE L1.id = 1;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/c2b5ba55e2534c509d59b04ecc41e5e2~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=9yfb3GPaaB2P5MTO0Lko2VmKrbg%3D)
**(8\)rows字段说明**


表示MySQL为了找到所需的记录，一共访问了多少行(预估的)。L3中的title没有添加索引，所以L3中有3条记录，就需要访问3条记录。



```
EXPLAIN SELECT * FROM L3,L4 WHERE L3.id = L4.id AND L3.title LIKE 'test007';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/402341c749f242bda4249afb76a381e3~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=Mo24Xbc5iD9UMzDNGIEjnJDbBjE%3D)
需要注意的是rows只是一个估算值，并不准确。所以rows行数过大的问题并不值得过多考虑，主要分析的还是索引是否使用正确了。


 


**(9\)filtered字段说明**


它指返回结果的行占需要读到的行(rows列的值)的百分比。


 


**(10\)extra字段说明**


Extra是EXPLAIN输出中另外一个很重要的列，该列显示MySQL在查询过程中的一些详细信息。


 


**一.准备数据**



```
CREATE TABLE users (
    uid INT PRIMARY KEY AUTO_INCREMENT,
    uname VARCHAR(20),
    age INT(11)
);
INSERT INTO users VALUES(NULL, 'lisa', 10);
INSERT INTO users VALUES(NULL, 'lisa', 10);
INSERT INTO users VALUES(NULL, 'rose', 11);
INSERT INTO users VALUES(NULL, 'jack', 12);
INSERT INTO users VALUES(NULL, 'sam', 13);
```

**二.Using filesort(需要进行文件排序)**


执行结果Extra为Using filesort，说明得到所需结果集，需要对所有记录进行文件排序。表示执行的SQL语句性能极差，需要进行优化。


 


下面就是在一个没有建立索引的列上进行order by，此时会触发filesort。优化方案是，在order by的列上添加索引，避免每次查询都全量排序。



```
EXPLAIN SELECT * FROM users ORDER BY age;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/5a2f2011a94849a085eea673c895d336~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=3G9rg9WozeJOmasxaKTkT3ZDOVw%3D)
**三.Using temporary**


表示使用了临时表来存储结果集，常见于排序和分组查询。



```
EXPLAIN SELECT COUNT(*),uname FROM users GROUP BY uname;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/40c623dabd2b4f27bcb7cc95f3bbf7dc~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=Ny4I77mmoMbxxLD90CvCnPWD1qo%3D)
**四.Using where**


表示使用了全表扫描或者在查找时使用索引的情况下，还有查询条件不在索引字段中需要回表。


 


注意一：返回所有记录的SQL，不使用where条件过滤数据，大概率不符合预期，这类SQL往往需要进行优化。


 


注意二：使用了where条件的SQL，并不代表不需要优化，往往需要配合explain结果中的type(连接类型)来综合判断。例如下面查询的age未设置索引，所以返回的type为ALL，仍有优化空间，可建立索引优化查询。



```
EXPLAIN SELECT * FROM users WHERE age = 10;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/e9981a3776d64a99b77e89bf34b956ca~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=YI6BAo0Kn64jI5VsrA%2BxY5xbkrU%3D)
**五.Using index**


表示直接访问索引就能获取所需数据(覆盖索引)，不需要回表。



```
-- 为uname创建索引
ALTER TABLE users ADD INDEX idx_uname(uname);
EXPLAIN SELECT uid,uname FROM users WHERE uname='lisa';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/bf8ff5032e3d426994a6dc43582f8902~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=CjKZ%2FhKv1Hy5%2B9%2FUpZG6BqpSHFs%3D)
**六.Using join buffer**


表示使用了连接缓存，还会显示join连接查询时使用的算法。



```
EXPLAIN SELECT * FROM users u1 LEFT JOIN 
    (SELECT * FROM users WHERE age = 1) u2 ON u1.age = u2.age;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/c848f99dc0944c0b9ce3c6100b9f3330~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=MkE3a9dVZ3GtHvWTouokLMK%2BMZA%3D)
Using join buffer(Block Nested Loop)说明，需要进行嵌套循环计算。这里每个表都有五条记录，内外表查询的type都为ALL。两个表通过字段age进行关联，且age字段未建立索引。


 


**七.Using index condition**


表示的是使用了索引，但是只使用了索引的一部分。一般发生在使用联合索引时，需要回表查询。



```
EXPLAIN SELECT * FROM L5 WHERE c > 10 AND d = '';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/8802bc1fb0534722a851a8755832f85a~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=PF6pRFdE%2Ftn5AQlLLH1BNLU7yXk%3D)
**八.Extra主要指标的含义总结**


using index：查找时使用了覆盖索引的时候就会出现，不需要回表。


 


using where：查找时使用索引的情况下需要回表或全表扫描。


 


using index condition：查找时使用了索引但只用一部分索引需要回表。


 


Using filesort：在一个没有建立索引的列上order by，发生文件排序。


 


Using temporary：使用了临时表存储结果集，常见于排序和分组查询。


 


当遍历二级索引不需要回表或者主键排序全表扫描时，type就为index。


 


查找非唯一性索引，返回匹配的多条数据，type就为ref。


 


查找唯一性索引，返回匹配的数据最多一条，type就为eq\_ref。


 


查找索引中给定范围的行，type就为range。


 


**3\.索引优化数据准备**


**(1\)创建数据库、表，插入数据**



```
create database idx_optimize character set 'utf8';
CREATE TABLE users(
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_name VARCHAR(20) NOT NULL COMMENT '姓名',
    user_age INT NOT NULL DEFAULT 0 COMMENT '年龄',
    user_level VARCHAR(20) NOT NULL COMMENT '用户等级',
    reg_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '注册时间'
);
INSERT INTO users(user_name,user_age,user_level,reg_time)
VALUES('tom',17,'A',NOW()),('jack',18,'B',NOW()),('lucy',18,'C',NOW());
```

**(2\)创建联合索引**



```
ALTER TABLE users ADD INDEX idx_nal (user_name,user_age,user_level) USING BTREE;
```

 


**4\.索引优化原则详解**


**(1\)最左侧列匹配和最左前缀匹配法则**


**(2\)不要在索引列上做任何计算**


**(3\)范围之后全失效**


**(4\)避免使用is null、is not null、!\= 、or**


**(5\)like以%开头会使索引失效**


**(6\)索引优化原则总结**


 


**(1\)最左侧列匹配和最左前缀匹配法则**


如果创建的是联合索引，就要遵循该法则。where后面的条件需从索引的最左侧列开始，且不能跳过索引中的列。如果where只匹配一个列，那么该列在索引最左侧，且只匹配前缀字段。


 


**一.最左侧列匹配和最左前缀匹配的场景**


场景1：按照索引字段顺序使用，三个字段都使用了索引，没有问题。



```
EXPLAIN SELECT * FROM users 
WHERE user_name = 'tom' AND user_age = 17 AND user_level = 'A';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/7c43aeb32379499ebe43ace341e50c86~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=P%2BppCb6PLf9gaQ%2BLiz4ZVcuX0rM%3D)
场景2：直接跳过user\_name使用索引字段，索引无效，未使用到索引。



```
EXPLAIN SELECT * FROM users WHERE user_age = 17 AND user_level = 'A';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4f3925a6f1534ee48a15eb38a2acd46a~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=e9EbAIdJ5UVznhwUitJuJyD6gLE%3D)
场景3: 不按照创建联合索引的顺序，使用索引。



```
EXPLAIN SELECT * FROM users WHERE user_age = 17 AND user_name = 'tom' AND user_level = 'A';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/72852562e9a44412999cafdfc5b892fe~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=YSIWunvUxMxFi71i5dCGbRruN9E%3D)
where后面查询条件顺序是user\_age、user\_level、user\_name，这与创建的索引顺序user\_name、user\_age、user\_level不一致。为什么还是使用了索引，原因是MySQL底层优化器对其进行了优化。


 


场景4：只要包含最左侧字段，索引就可以生效


但从key\_len可知只是用到索引的一部分。



```
EXPLAIN SELECT * FROM users WHERE user_name = 'tom';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/fd4c87221be149e6ab29acfae41eb717~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=pE7TAjH4Wr5J48RJG7H%2B%2FIa0p5M%3D)
**二.最左侧列匹配和最左前缀匹配的原理**


InnoDB创建联合索引的规则是：


首先会对联合索引最左边的字段进行排序，例子中是user\_name。在第一个字段的基础之上再对第二个字段进行排序，例子中是user\_age。所以最佳左前缀原则其实是和B\+树的结构有关系，最左字段肯定是有序的，第二个字段则是无序的。


 


联合索引的排序方式是：


先按第一个字段进行排序，如果第一个字段相等再根据第二个字段排序。所以如果直接使用第二个字段user\_age通常是使用不到索引的。


 


**(2\)不要在索引列上做任何计算**


不要在索引列上做任何操作，否则会导致索引失效，从而转向全表扫描。比如计算、使用函数、自动或手动进行类型转换(字符串不加双引号)。


 


**一.插入数据**



```
INSERT INTO users(user_name,user_age,user_level,reg_time) VALUES('11223344',22,'D',NOW());
```

场景1：使用系统函数left()函数，对user\_name进行操作



```
EXPLAIN SELECT * FROM users WHERE LEFT(user_name, 6) = '112233';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/916ad6f440114c4d8945588fc46cfdb5~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=d4jXm9CtElryiFfw8gfHZyzGEj0%3D)
场景2：字符串不加单引号(隐式类型转换)


对于varchar类型的字段，如果查询时不加单引号就会进行隐式转换，导致索引失效转向全表扫描。



```
EXPLAIN SELECT * FROM users WHERE user_name = 11223344;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/bba0bd2394fd440eb2b0f3ec16897d33~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=xUQcGIxiq7xDyXvgucnPqEtOa8Q%3D)
**(3\)范围之后全失效**


where条件中如果有范围条件，并且范围条件之后还有其他过滤条件，那么范围条件之后的列就都将会索引失效。


 


场景1：条件单独使用user\_name时，type\=ref、key\_len\=62。



```
-- 条件只有一个 user_name
EXPLAIN SELECT * FROM users WHERE user_name = 'tom';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/cb78484f50b04f01a43036061a518536~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=XOIPdWU017vEk1BMBfqX2dfmjX0%3D)
场景2：条件增加一个user\_age(使用常量等值)，type\= ref、key\_len \= 66。



```
EXPLAIN SELECT * FROM users WHERE user_name = 'tom' AND user_age = 17;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/5b6e263e270b4f8ba91c3a9ab9243a94~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=c2UR%2BKQQanfog1DYB1t%2FiBdg%2FRI%3D)
场景3：使用全值匹配，type \= ref、key\_len \= 128，索引都利用上了。



```
EXPLAIN SELECT * FROM users WHERE user_name = 'tom' AND user_age = 17 AND user_level = 'A';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/ea4e2a9c4ef24eb9822add18464d74ac~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=wTH4O7rruxwcRmEEiX5HXjrNoos%3D)
场景4：使用范围条件时，avg \> 17、type \= range、key\_len \= 66。与场景3比较，可发现user\_level索引没用上。



```
EXPLAIN SELECT * FROM users WHERE user_name = 'tom' AND user_age > 17 AND user_level = 'A';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/ea2f3def35d34d6ba8ce133f75063a96~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=E4TEorh8iPOegfBJ0Bk5ET4li3c%3D)
**(4\)避免使用is null、is not null、!\= 、or**


**一.使用is null会使索引失效**



```
EXPLAIN SELECT * FROM users WHERE user_name IS NULL;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/10458d9126dc4584b30f7976430d3a1d~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=YSN52q8o%2BZRYpMZjO5bzJ9W0lbU%3D)
Impossible Where：表示where条件不成立，不能返回任何行。


 


**二.使用is not null会使索引失效**



```
EXPLAIN SELECT * FROM users WHERE user_name IS NOT NULL;
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/204be82af9d343fe843e14714536824d~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=qTO6iNU8uE3P200rJGAASL%2B2614%3D)
**三.使用!\=和or会使索引失效**



```
EXPLAIN SELECT * FROM users WHERE user_name != 'tom';
EXPLAIN SELECT * FROM users WHERE user_name = 'tom' or user_name = 'jack';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/aa82d8acf3044ba1b254d4e3de65a798~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=TjiLAjLSEaQpIr1BVegDoz9b1A0%3D)
**(5\)like以%开头会使索引失效**


**一.like查询中%出现在左边则索引失效，%出现在右边索引未失效**


场景1：两边都有%或者%在左边，索引都会失效



```
EXPLAIN SELECT * FROM users WHERE user_name LIKE '%tom%';
EXPLAIN SELECT * FROM users WHERE user_name LIKE '%tom';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/bac2a19b38514c87ae3448fedd94cd8e~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=oRug%2FyC8ANon2dhgH2HW%2BClE760%3D)
场景2：%在右边，索引生效



```
EXPLAIN SELECT * FROM users WHERE user_name LIKE 'tom%';
```

![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/9ed040c69147449e9deccbe32fbd4456~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=7SyPrScgS%2FgWzglXO5gj2kTxZd0%3D)
**二.解决%出现在左边索引失效的方法——使用覆盖索引**



```
EXPLAIN SELECT user_name FROM users WHERE user_name LIKE '%jack%';
EXPLAIN SELECT user_name,user_age,user_level FROM users WHERE user_name LIKE '%jack%';
```

![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/6fbeaea076cf4ffaa0775f5554f8d675~tplv-obj.image?lk3s=ef143cfe&traceid=20241201165602CA3B637C11E74634A84C&x-expires=2147483647&x-signature=lAsmAPo9HPvmNt4PSRs8w7qyB7Q%3D)
对比场景1可以知道：通过使用覆盖索引type \= index，并且extra \= Using index，从原来的全表扫描变成了全索引扫描，也就是索引的全表扫描。


 


**三.like失效的原理**


**原理一：%号在右**


由于B\+树的索引顺序，是按照首字母的大小进行排序，而%号在右时的匹配又会匹配首字母，所以能在B\+树上进行有序的查找。也就是查找出首字母符合要求的数据，所以%号在右可以用到索引。


 


**原理二：%号在左是匹配字符串尾部的数据**


由于尾部的字母是没有顺序的，所以不能按索引顺序查询，用不到索引。


 


**原理三：两个%%号**


这个是查询任意位置的字母满足条件即可。只有首字母是进行索引排序的，其他位置的字母都是相对无序的，所以查找任意位置的字母是用不上索引的。


 


**(6\)索引优化原则总结**


一.最左侧列匹配和最左前缀匹配


二.索引列上不计算不转换


三.范围之后全失效


四.最好使用覆盖索引


五.!\=、is null、is not null、or会索引失效


六.like百分号加右边，加左边导致索引失效的解决方法是使用覆盖索引


 


**5\.慢查询设置与测试**


**(1\)慢查询介绍**


**(2\)慢查询参数**


**(3\)慢查询配置方式**


**(4\)慢查询测试**


**(5\)慢日志内容**


 


**(1\)慢查询介绍**


MySQL的慢查询全名是慢查询日志，是MySQL提供的一种日志记录。慢查询日志会记录在MySQL中响应时间超过阈值的语句。MySQL数据库默认不启动慢查询日志，需要手动来设置这个参数。


 


如果不是调优需要的话，一般不建议启动该参数。因为开启慢查询日志会或多或少带来一定的性能影响，慢查询日志支持将日志记录写入文件和数据库表。


 


**(2\)慢查询参数**


执行下面的语句



```
mysql> show variables like '%slow_query_log%';
+---------------------+------------------------------+
| Variable_name       | Value                        |
+---------------------+------------------------------+
| slow_query_log      | ON                           |
| slow_query_log_file | /var/lib/mysql/test-slow.log |
+---------------------+------------------------------+


mysql> show variables like '%long_query%';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
```

MySQL慢查询的相关参数解释：


一.slow\_query\_log：是否开启慢查询日志。


二.slow\-query\-log\-file：慢查询日志存储路径。


三.long\_query\_time：慢查询阈值，查询时间多于设定阈值则记录日志。


 


**(3\)慢查询配置方式**


**一.默认情况下slow\_query\_log的值为OFF，表示慢查询日志是禁用的**



```
mysql> show variables like '%slow_query_log%';
+---------------------+------------------------------+
| Variable_name       | Value                        |
+---------------------+------------------------------+
| slow_query_log      | ON                           |
| slow_query_log_file | /var/lib/mysql/test-slow.log |
+---------------------+------------------------------+
```

**二.可以通过设置slow\_query\_log的值来开启**



```
mysql> set global slow_query_log=1;
```

**三.set global slow\_query\_log\=1开启慢查询日志当前生效重启失效**


如果要永久生效，就必须修改配置文件my.cnf，其它系统变量也是如此。



```
-- 编辑配置
vim /etc/my.cnf


-- 添加如下内容
slow_query_log =1
slow_query_log_file=/var/lib/mysql/ruyuan-slow.log


-- 重启MySQL
service mysqld restart


mysql> show variables like '%slow_query%';
+---------------------+--------------------------------+
| Variable_name       | Value                          |
+---------------------+--------------------------------+
| slow_query_log      | ON                             |
| slow_query_log_file | /var/lib/mysql/ruyuan-slow.log |
+---------------------+--------------------------------+
```

**四. 开启了慢查询日志后，什么样的SQL才会记录到慢查询日志里**


这个由参数long\_query\_time控制，默认long\_query\_time的值为10秒。



```
mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+


mysql> set global long_query_time=1;
Query OK, 0 rows affected (0.00 sec)


mysql>  show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
```

**五.修改变量long\_query\_time，但查询值还是10**


执行命令set global long\_query\_time\=1后，需要重新连接或者打开新开会话才能看到修改值。



```
mysql> show variables like 'long_query_time';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 1.000000 |
+-----------------+----------+
```

**六.log\_output参数是指定日志的存储方式**


log\_output\=FILE表示将日志存入文件，默认值是FILE。log\_output\=TABLE表示将日志存入数据库，这样日志信息就会被写入到mysql.slow\_log表中。


 


MySQL数据库可以同时支持两种日志存储方式，配置的时候以逗号隔开即可，如：log\_output\='FILE,TABLE'。


 


日志记录到系统的专用日志表中，要比记录到文件耗费更多的系统资源。因此如果启用慢查询日志\+获得更高系统性能，则建议优先记录到文件。



```
mysql> SHOW VARIABLES LIKE '%log_output%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+
```

**七.开启系统变量让未使用索引的查询也被记录到慢查询日志中**


这个系统变量就是log\-queries\-not\-using\-indexes，所以在进行调优时，可以开启这个选项。



```
mysql> show variables like 'log_queries_not_using_indexes';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| log_queries_not_using_indexes | OFF   |
+-------------------------------+-------+


mysql> set global log_queries_not_using_indexes=1;
Query OK, 0 rows affected (0.00 sec)


mysql> show variables like 'log_queries_not_using_indexes';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| log_queries_not_using_indexes | ON    |
+-------------------------------+-------+
```

**(4\)慢查询测试**


一.执行test\_index.sql脚本，监控慢查询日志内容



```
[root@localhost mysql]# tail -f /var/lib/mysql/test-slow.log 
/usr/sbin/mysqld, Version: 5.7.30-log (MySQL Community Server (GPL)). started with:
Tcp port: 0  Unix socket: /var/lib/mysql/mysql.sock
Time                 Id Command    Argument
```

二. 执行下面的SQL，执行超时(超过1秒)我们去查看慢查询日志



```
SELECT * FROM test_index WHERE 
hobby = '20009951' OR hobby = '10009931' OR hobby = '30009931'
OR dname = 'name4000' OR dname = 'name6600' ;
```

**(5\)慢日志内容**


我们得到慢查询日志后，最重要的一步就是去分析这个日志。先来看慢日志里到底记录了哪些内容，如下是慢日志里其中一条记录，可以看到有时间戳、用户、查询时长及具体的SQL等信息。



```
Time                 Id Command    Argument
# Time: 2022-02-23 T03:55:15. 336037Z
# User@Host: root[root] @ localhost []  Id:     6
# Query_time: 2.375219  Lock_time: 0.000137 Rows_sent: 3  Rows_examined: 5000000
use db4;
SET timestamp=1645588515;
SELECT * FROM test_index WHERE  hobby = '20009961' OR hobby = '10009941' OR hobby = '30009961' OR dname = 'name4001' OR dname = 'name6601';
```

Time：执行时间；


Users：用户信息；


Query\_time：查询时长；


Lock\_time：等待锁时长；


Rows\_sent：结果行统计数量；


Rows\_examined：扫描的行数；


 


**6\.慢查询SQL优化思路**


**(1\)SQL性能下降的原因**


**(2\)慢查询优化思路**


 


**(1\)SQL性能下降的原因**


导致SQL执行性能下降的原因可体现在以下两方面：


 


**一.等待时间长**


锁表导致查询一直处于等待状态。


 


**二.执行时间长**


查询语句没优化、索引失效、关联查询太多join、机器及参数没调优。


 


**(2\)慢查询优化思路**


**一.优先选择优化高并发执行的SQL**


因为高并发的SQL出现问题带来后果更严重，比如下面两种情况：SQL1每小时执行10000次，每次20个IO，优化后每次18个IO，每小时节省2万次IO；SQL2每小时10次，每次20000个IO，每次优化减少2000个IO，每小时节省2万次IO。此时SQL2更难优化，SQL1更好优化。但是第一种属于高并发SQL，更急需优化，因为成本更低。


 


**二.定位优化对象的性能瓶颈**


在去优化SQL时，选择优化分方向有三个：


方向1：IO，数据访问消耗了太多时间，查看是否正确使用索引。


方向2：CPU，数据运算花费了太多时间，数据的运算分组、排序是不是有问题。


方向3：网络带宽，加大网络带宽。


 


**三.明确优化目标**


根据数据库当前状态、当前SQL的具体功能，来确定最好情况下消耗的资源和最差情况下消耗的资源。因为优化的结果只有一个，即给用户一个好的体验。


 


**四.从explain执行计划入手**


只有explain能告诉我们当前SQL的执行状态。


 


**五.永远用小的结果集驱动大的结果集**


小的数据集驱动大的数据集，减少内层表读取次数。



```
//类似于嵌套循环
for (int i = 0; i < 5; i++) {
    for (int i = 0; i < 1000; i++) {


    }
}
```

**六.尽可能在索引中完成排序**


排序操作用得比较多，所以order by后面的字段尽量使用上索引。因为索引本来就是排好序的，所以速度很快。没有索引的话，就需要从表中拿数据，在内存中进行排序。如果内存空间不够还会发生临时文件落盘操作。


 


**七.只获取自己需要的列**


不要使用select \*，因为select \* 很可能不使用索引，而且数据量过大。


 


**八.只使用最有效的过滤条件**


where后面的条件并非越多越好，应该用最短的路径访问到数据。


 


**九.尽可能避免复杂的join和子查询**


每条SQL的JOIN操作建议不要超过三张表。将复杂的SQL，拆分成多个小的SQL，单个表执行，然后对获取的结果在程序中进行封装。因为如果join占用的资源比较多，会导致其他进程等待时间变长。


 


**十.合理设计并利用索引**


也就是合理判断是否需要创建索引，以及合理选择合适索引。


 


**(3\)如何判定是否需要创建索引**


一.频繁作为查询条件的字段应该创建索引


二.唯一性太差的字段不适合单独创建索引，即使它频繁作为查询条件


唯一性太差的字段主要是指哪些呢？如状态字段、类型字段等。这些字段中的数据可能总共就是那么几个几十个数值重复使用。当一条Query所返回的数据超过了全表的15%时，就不应该再使用索引扫描来完成这个Query了。


三.更新非常频繁的字段不适合创建索引


因为索引中的字段被更新时，不仅要更新表的数据，还要更新索引数据。


四.不会出现在WHERE子句中的字段不该创建索引


 


**(4\)如何选择合适索引**


一.单键索引，尽量选择针对当前Query过滤性更好的索引。


二.联合索引，当前查询中过滤性最好的字段在索引字段顺序中排列靠前。


 


 本博客参考[slower加速器](https://chundaotian.com)。转载请注明出处！
