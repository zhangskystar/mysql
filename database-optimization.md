### 数据库优化
#### 为什么要进行数据库优化？
> 一个成熟的数据库架构并不是一开始设计就具备高可用、高伸缩等特性的，它是随着用户量的增加，基础架构才逐渐完善。SQL命令因为语法简单、操作高效受到了很多用户的欢迎。但是，SQL命令的效率受到不同的数据库功能的限制，特别是在计算时间方面，再加上语言的高效率也不意味着优化会更容易，所以每个数据库都需要依据实际情况做特殊调整。

> 系统的吞吐量瓶颈往往出现在数据库的访问速度上
随着应用程序的运行，数据库的中的数据会越来越多，处理时间会相应变慢
数据是存放在磁盘上的，读写速度无法和内存相比

>优化的原则：减少系统瓶颈，减少资源占用，增加系统的反应速度。
#### 数据库优化问题从以下几个层面入手：
（1）根据服务层面：配置mysql性能优化参数；

（2）从系统层面增强mysql的性能：优化数据表结构、字段类型、字段索引、分表，分库、读写分离等等。

（3）从数据库层面增强性能：优化SQL语句，合理使用字段索引。

（4）从代码层面增强性能：使用缓存和NoSQL数据库方式存储，如MongoDB/Memcached/Redis来缓解高并发下数据库查询的压力。

（5）减少数据库操作次数，尽量使用数据库访问驱动的批处理方法。

（6）不常使用的数据迁移备份，避免每次都在海量数据中去检索。

（7）提升数据库服务器硬件配置，或者搭建数据库集群。

（8）编程手段防止SQL注入：使用JDBC PreparedStatement按位插入或查询；正则表达式过滤（非法字符串过滤）；

#### 具体优化案例：
1. 查询SQL尽量不要使用select *，而是具体字段
> 反例：SELECT * FROM student
正例：SELECT id,NAME FROM student
理由：
字段多时，大表能达到100多个字段甚至达200多个字段
只取需要的字段，节省资源、减少网络开销
select * 进行查询时，很可能不会用到索引，就会造成全表扫描

2. 避免在where子句中使用or来连接条件
> 反例：SELECT * FROM student WHERE id=1 OR salary=30000
正例：# 分开两条sql写
SELECT * FROM student WHERE id=1
SELECT * FROM student WHERE salary=30000
理由：
使用or可能会使索引失效，从而全表扫描
对于or没有索引的salary这种情况，假设它走了id的索引，但是走到salary查询条件时，它还得全表扫描。
也就是说整个过程需要三步：全表扫描+索引扫描+合并。如果它一开始就走全表扫描，直接一遍扫描就搞定。
虽然mysql是有优化器的，处于效率与成本考虑，遇到or条件，索引还是可能失效的

3. 使用varchar代替char
> 反例：`deptname` char(100) DEFAULT NULL COMMENT '部门名称'
正例：`deptname` varchar(100) DEFAULT NULL COMMENT '部门名称'
理由：
varchar变长字段按数据内容实际长度存储，存储空间小，可以节省存储空间
char按声明大小存储，不足补空格
其次对于查询来说，在一个相对较小的字段内搜索，效率更高

4. 尽量使用数值替代字符串类型
> 主键（id）：primary key优先使用数值类型int，tinyint
性别（sex）：0-代表女，1-代表男；数据库没有布尔类型，mysql推荐使用tinyint
支付方式（payment）：1-现金、2-微信、3-支付宝、4-信用卡、5-银行卡
服务状态（state）：1-开启、2-暂停、3-停止
商品状态（state）：1-上架、2-下架、3-删除

5. 查询尽量避免返回大量数据
> 如果查询返回数据量很大，就会造成查询时间过长，网络传输时间过长。
同时，大量数据返回也可能没有实际意义。如返回上千条甚至更多，用户也看不过来.
通常采用分页，一页习惯10/20/50/100条。

6. 使用explain分析你SQL执行计划
> EXPLAIN
SELECT * FROM student WHERE id=1
SQL很灵活，一个需求可以很多实现，那哪个最优呢？
SQL提供了explain关键字，它可以分析你的SQL执行计划，看它是否最佳。
Explain主要看SQL是否使用了索引。

7. 是否使用了索引及其扫描类型
> type：
	ALL 全表扫描，没有优化，最慢的方式
	index 索引全扫描
	range 索引范围扫描，常用语<，<=，>=，between等操作
	ref 使用非唯一索引扫描或唯一索引前缀扫描，返回单条记录，常出现在关联查询中
	eq_ref 类似ref，区别在于使用的是唯一索引，使用主键的关联查询
	const/system 单条记录，系统会把匹配行中的其他列作为常数处理，如主键或唯一索引查询
	null MySQL不访问任何表或索引，直接返回结果
key：
	真正使用的索引方式

8. 创建name字段的索引
> ALTER TABLE student ADD INDEX index_name (NAME)

9. 优化like语句
> 模糊查询，程序员最喜欢的就是使用like，但是like很可能让你的索引失效
反例：
EXPLAIN
SELECT id,NAME FROM student WHERE NAME LIKE '%1'
EXPLAIN
SELECT id,NAME FROM student WHERE NAME LIKE '%1%'
正例：
EXPLAIN
SELECT id,NAME FROM student WHERE NAME LIKE '1%'

10. 字符串怪现象
> 反例：
#未使用索引
EXPLAIN
SELECT * FROM student WHERE NAME=123
正例：
#使用索引
EXPLAIN
SELECT * FROM student WHERE NAME='123'
理由：
为什么第一条语句未加单引号就不走索引了呢？
这是因为不加单引号时，是字符串跟数字的比较，它们类型不匹配，
MySQL会做隐式的类型转换，把它们转换为数值类型再做比较

11. 索引不宜太多，一般5个以内
> 索引并不是越多越好，虽其提高了查询的效率，但却会降低插入和更新的效率
索引可以理解为一个就是一张表，其可以存储数据，其数据就要占空间
再者，索引表的一个特点，其数据是排序的，那排序要不要花时间呢？肯定要
insert或update时有可能会重建索引，如果数据量巨大，重建将进行记录的重新排序，
所以建索引需要慎重考虑，视具体情况来定
一个表的索引数最好不要超过5个，若太多需要考虑一些索引是否有存在的必要

12. 索引不适合建在有大量重复数据的字段上
> 如性别字段。因为SQL优化器是根据表中数据量来进行查询优化的，如果索引列有大量重复数据，
Mysql查询优化器推算发现不走索引的成本更低，很可能就放弃索引了。

13. where限定查询的数据
> 数据中假定就一个男的记录
反例：
SELECT id,NAME FROM student WHERE sex='男'
正例：
SELECT id,NAME FROM student WHERE id=1 AND sex='男'
理由：
需要什么数据，就去查什么数据，避免返回不必要的数据，节省开销

14. 避免在where中对字段进行表达式操作
> 反例：
EXPLAIN
SELECT * FROM student WHERE id+1-1=+1
正例：
EXPLAIN
SELECT * FROM student WHERE id=+1-1+1
EXPLAIN
SELECT * FROM student WHERE id=1
理由：
SQL解析时，如果字段相关的是表达式就进行全表扫描

15. 避免在where子句中使用!=或<>操作符
> 应尽量避免在where子句中使用!=或<>操作符，否则引擎将放弃使用索引而进行全表扫描。
记住实现业务优先，实在没办法，就只能使用，并不是不能使用。如果不能使用，SQL也就无需支持了。
反例：
EXPLAIN
SELECT * FROM student WHERE salary!=3000
EXPLAIN
SELECT * FROM student WHERE salary<>3000
理由：
使用!=和<>很可能会让索引失效

16. 去重distinct过滤字段要少
> #索引失效
EXPLAIN
SELECT DISTINCT * FROM student
#索引生效
EXPLAIN
SELECT DISTINCT id,NAME FROM student
EXPLAIN
SELECT DISTINCT NAME FROM student
理由：
带distinct的语句占用cpu时间高于不带distinct的语句。因为当查询很多字段时，如果使用distinct，
数据库引擎就会对数据进行比较，过滤掉重复数据，然而这个比较、过滤的过程会占用系统资源，如cpu时间

17. where中使用默认值代替null
> #修改表，增加age字段，类型int，非空，默认值0
ALTER TABLE student ADD age INT NOT NULL DEFAULT 0;

18. 批量插入性能提升
> 大量数据提交，上千，上万，批量性能非常快，mysql独有
多条提交：
INSERT INTO student (id,NAME) VALUES(4,'齐雷');
INSERT INTO student (id,NAME) VALUES(5,'刘昱江');
批量提交：
INSERT INTO student (id,NAME) VALUES(4,'齐雷'),(5,'刘昱江');
理由：
默认新增SQL有事务控制，导致每条都需要事务开启和事务提交；而批量处理是一次事务开启和提交。自然速度飞升
数据量小体现不出来

19. 批量删除优化
> 避免同时修改或删除过多数据，因为会造成cpu利用率过高，会造成锁表操作，从而影响别人对数据库的访问。
反例：
#一次删除10万或者100万+？
delete from student where id <100000;
#采用单一循环操作，效率低，时间漫长
for（User user:list）{
delete from student;
}
正例：
//分批进行删除，如每次500
for(){
delete student where id<500;
}
delete student where id>=500 and id<1000;
理由：
一次性删除太多数据，可能造成锁表，会有lock wait timeout exceed的错误，所以建议分批操作

20. 伪删除设计
> 商品状态（state）：1-上架、2-下架、3-删除
理由：
这里的删除只是一个标识，并没有从数据库表中真正删除，可以作为历史记录备查
同时，一个大型系统中，表关系是非常复杂的，如电商系统中，商品作废了，但如果直接删除商品，其它商品详情，物流信息中可能都有其引用。
通过where state=1或者where state=2过滤掉数据，这样伪删除的数据用户就看不到了，从而不影响用户的使用
操作速度快，特别数据量很大情况下

21. 提高group by语句的效率
> 可以在执行到该语句前，把不需要的记录过滤掉
反例：先分组，再过滤
select job，avg（salary） from employee
group by job
having job ='president' or job = 'managent';
正例：先过滤，后分组
select job，avg（salary） from employee
where job ='president' or job = 'managent'
group by job;

22. 复合索引最左特性
> 创建复合索引，也就是多个字段
ALTER TABLE student ADD INDEX idx_name_salary (NAME,salary)
满足复合索引的左侧顺序，哪怕只是部分，复合索引生效
EXPLAIN
SELECT * FROM student WHERE NAME='陈子枢'
没有出现左边的字段，则不满足最左特性，索引失效
EXPLAIN
SELECT * FROM student WHERE salary=3000
复合索引全使用，按左侧顺序出现 name,salary，索引生效
EXPLAIN
SELECT * FROM student WHERE NAME='陈子枢' AND salary=3000
虽然违背了最左特性，但MYSQL执行SQL时会进行优化，底层进行颠倒优化
EXPLAIN
SELECT * FROM student WHERE salary=3000 AND NAME='陈子枢'
理由：
复合索引也称为联合索引
当我们创建一个联合索引的时候，如(k1,k2,k3)，相当于创建了（k1）、(k1,k2)和(k1,k2,k3)三个索引，这就是最左匹配原则
联合索引不满足最左原则，索引一般会失效，但是这个还跟Mysql优化器有关的

23. 排序字段创建索引
> 什么样的字段才需要创建索引呢？原则就是where和order by中常出现的字段就创建索引。
#使用*，包含了未索引的字段，导致索引失效
EXPLAIN
SELECT * FROM student ORDER BY NAME;
EXPLAIN
SELECT * FROM student ORDER BY NAME,salary
#name字段有索引
EXPLAIN
SELECT id,NAME FROM student ORDER BY NAME
#name和salary复合索引
EXPLAIN
SELECT id,NAME FROM student ORDER BY NAME,salary
EXPLAIN
SELECT id,NAME FROM student ORDER BY salary,NAME
#排序字段未创建索引，性能就慢
EXPLAIN
SELECT id,NAME FROM student ORDER BY sex

24. 删除冗余和重复的索引
> SHOW INDEX FROM student
#创建索引index_name
ALTER TABLE student ADD INDEX index_name (NAME)
#删除student表的index_name索引
DROP INDEX index_name ON student ;
#修改表结果，删除student表的index_name索引
ALTER TABLE student DROP INDEX index_name ;
#主键会自动创建索引，删除主键索引
ALTER TABLE student DROP PRIMARY KEY ;

25. 不要有超过5个以上的表连接
> 关联的表个数越多，编译的时间和开销也就越大
每次关联内存中都生成一个临时表
应该把连接表拆开成较小的几个执行，可读性更高
如果一定需要连接很多表才能得到数据，那么意味着这是个糟糕的设计了
阿里规范中，建议多表联查三张表以下

26. inner join 、left join、right join，优先使用inner join
> 三种连接如果结果相同，优先使用inner join，如果使用left join左边表尽量小
inner join 内连接，只保留两张表中完全匹配的结果集
left join会返回左表所有的行，即使在右表中没有匹配的记录
right join会返回右表所有的行，即使在左表中没有匹配的记录
理由：
如果inner join是等值连接，返回的行数比较少，所以性能相对会好一点
同理，使用了左连接，左边表数据结果尽量小，条件尽量放到左边处理，意味着返回的行数可能比较少。
这是mysql优化原则，就是小表驱动大表，小的数据集驱动大的数据集，从而让性能更优

27. in子查询的优化
> 日常开发实现业务需求可以有两种方式实现：
一种使用数据库SQL脚本实现
一种使用程序实现
如需求：查询所有部门的所有员工：
#in子查询
SELECT * FROM tb_user WHERE dept_id IN (SELECT id FROM tb_dept);
#这样写等价于：
#先查询部门表
SELECT id FROM tb_dept
#再由部门dept_id，查询tb_user的员工
SELECT * FROM tb_user u,tb_dept d WHERE u.dept_id = d.id 
假设表A表示某企业的员工表，表B表示部门表，查询所有部门的所有员工，很容易有以下程序实现，
可以抽象成这样的一个嵌套循环：
List<> resultSet;
#部门表中嵌套员工表
for(int i=0;i<B.length;i++) {
	for(int j=0;j<A.length;j++) {
		if(A[i].id==B[j].id) 
		{
			resultSet.add(A[i]);
			break;
		}
	}
}
上面的需求使用SQL就远不如程序实现，特别当数据量巨大时。
理由：
数据库最费劲的就是程序链接的释放。假设链接了两次，每次做上百万次的数据集查询，查完就结束，
这样就只做了两次；相反建立了上百万次链接，申请链接释放反复重复，
就会额外花费很多实际，这样系统就受不了了，慢，卡顿

28. 尽量使用union all替代union
> #UNION 操作符用于合并两个或多个 SELECT 语句的结果集。
反例：
SELECT * FROM student
UNION
SELECT * FROM student
正例：
SELECT * FROM student
UNION ALL
SELECT * FROM student
理由：
union和union all的区别是，union会自动去掉多个结果集合中的重复结果，而union all则将所有的结果全部显示出来，不管是不是重复
union：对两个结果集进行并集操作，不包括重复行，同时进行默认规则的排序
union在进行表链接后会筛选掉重复的记录，所以在表链接后会对所产生的结果集进行排序运算，删除重复的记录再返回结果。
实际大部分应用中是不会产生重复的记录，最常见的是过程表与历史表UNION
