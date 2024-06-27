# mysql索引失效的几种场景


# 1.准备工作

## 1.1创建表
```sql
DROP TABLE IF EXISTS `movie`;
CREATE TABLE `movie` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `code` varchar(50) DEFAULT NULL COMMENT '电影编号',
  `type` int(11) DEFAULT NULL COMMENT '电影类型',
  `name` varchar(200) DEFAULT NULL COMMENT '电影名称',
  `ranks` int(11) DEFAULT NULL COMMENT '排名',
  `remark` varchar(200) DEFAULT NULL COMMENT '描述',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 1.2创建索引
```sql
ALTER TABLE movie ADD index idx_code_type_name (code, type, name);
ALTER TABLE movie ADD index idx_ranks (ranks);
```

## 1.3插入数据
```sql
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('101', 3, '战狼', 1, '战狼:备注');
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('202', 2, '流浪地球', 3, '流浪地球:备注');
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('303', 1, '独行月球', 5, '独行月球:备注');
```

# 2.索引失效场景

## 2.1 不满足最左匹配原则
```sql
EXPLAIN
SELECT * from movie where code = '101' and type = 1 and name = '战狼';  #走索引
EXPLAIN
SELECT * from movie where code = '101' and type = 1; #走索引
EXPLAIN
SELECT * from movie where code = '101' ; #走索引
EXPLAIN
SELECT * from movie where code = '101' and name = '战狼'; #走索引，只是用上了code字段索引
EXPLAIN
SELECT * from movie where name = '战狼' and code = '101'; #走索引
EXPLAIN
SELECT * from movie where type = 1 and name = '战狼';  #不走索引
```
 因为联合索引的情况下，数据是按照索引第一列排序，在第一列数据相同才会按照第二列排序，第三列又是根据前面两列排序后再来排序
中间断层这种，其实他只是走了最左边的字段索引，剩下的没用上
查询条件中，只要有最左边的索引字段，就会走索引；和where条件字段顺序无关，即使中间断了层也能走上索引

## 2.2 使用了 select *
```sql
EXPLAIN
select * from movie ;#不走索引
```
sql中使用了select *，从执行结果来看，走了全表扫描，没有用到任何索引，查询效率非常低
```sql
EXPLAIN
SELECt code, type, name from movie;#走索引
```
如果select语句中的查询列都是索引列，那么这些列被称为覆盖索引。这种情况下，查询的相关字段都能走索引，索引查询的效率相对高一些

## 2.3 order by导致索引失效
```sql
EXPLAIN
SELECT code,type,name,ranks from movie ORDER BY ranks;#不走索引
EXPLAIN
SELECT ranks from movie ORDER BY ranks;#走索引

```
在我们的开发过程中，经常会使用到order by，为了保证order by的查询效率，建议将order by字段和select字段添加联合索引;但是如果需求经常变更需要增加字段，不能总是去修改select字段的联合索引；可以通过给order by增加索引，先通过子查询查询出id,然后再手动回表查询所需要的字段

## 2.4 索引列上有计算/使用了函数
```sql
EXPLAIN
SELECT * from movie where IFNULL(ranks,0)=0;#不走索引

EXPLAIN
SELECT * from movie where ranks+1=2;#不走索引
```
 会对索引列进行一次重新计算

## 2.5 字段类型不同
```sql
EXPLAIN
SELECT * from movie where code='101';#走索引

EXPLAIN
SELECT * from movie where code=101;#不走索引

EXPLAIN
SELECT * from movie where rank='101';#走索引
```
mysql如果是int类型字段作为查询条件时，他会自动将该字段的参数进行隐式转换，把字符串换成int类型

## 2.6 like左边包含了%
```sql
EXPLAIN
SELECT * from movie where code like '%1';#不走索引
EXPLAIN
SELECT * from movie where code like '1%';#走索引
EXPLAIN
SELECT * from movie where code like '%1%';#不走索引
EXPLAIN
SELECT code, name, type from movie where  code LIKE '%1%';#走索引
```
当like语句中的%出现在查询条件的左边时，索引会失效；但是如果我们使用覆盖索引就不会失效

## 2.7 列对比
```sql
EXPLAIN
SELECT * from movie where id=ranks;#不走索引
EXPLAIN
SELECT ranks from movie where id=ranks;#走索引
EXPLAIN
SELECT code, type, name from movie where id=type;#走索引
```
如果把两个单独建了索引的列用来做对比时索引会失效
走覆盖索引的话是可以走索引的，比如只查询ranks字段或者将where条件字段换成我们的聚合索引上面的字段

## 2.8 使用or关键字
```sql
EXPLAIN
SELECT * from movie where id=1 or ranks=1;# or字段前后两个字段都增加了索引，mysql8会走索引，mysql5.7不会走索引

EXPLAIN
SELECT * from movie where id=1 or remark='战狼:备注';#remark字段没有索引，增加or后会不走索引

```
如果使用了or关键字，那么他前面和后面的字段都要加索引，不然所有的索引都会失效。这里是一个大坑

## 2.9 <>不等于导致索引失效
```sql
EXPLAIN
SELECT * from movie where ranks=1;#走索引

EXPLAIN
SELECT * from movie where ranks<>1;#不走索引
```
mysql5.7版本<>不等于会导致索引失效；但是在mysql8.0还是会走索引
不走索引场景解析：其实它是可以走索引的，如果返回结果集大于20%，那么它就不会走索引。

## 2.10. 范围查询数量过多导致索引失效

```sql
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('104', 4, '流浪地球1', 2, '流浪地球1:备注');
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('105', 5, '独行月球2', 4, '独行月球2:备注');
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('106', 6, '战狼3', 6, '战狼3:备注');
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('107', 7, '流浪地球4', 7, '流浪地球4:备注');
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('108', 8, '独行月球5', 8, '独行月球5:备注');
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('109', 9, '战狼6', 9, '战狼6:备注');
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('110', 10, '流浪地球7', 10, '流浪地球7:备注');
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('111', 11, '独行月球8', 11, '独行月球8:备注');
INSERT INTO `movie` (`code`, `type`, `name`, `ranks`, `remark`) VALUES ('112', 12, '战狼9', 12, '战狼9:备注');
```
```sql
EXPLAIN
SELECT *  from movie WHERE ranks>1;#不走索引
EXPLAIN
SELECT *  from movie WHERE ranks>10;走索引
```
where条件大于1的情况下，因为扫描的行数比较多，还不如全表扫描；但是当ranks大于10的时候，数据比较少，回表成本也比较小，所以还是会选择走索引；
在我们开发过程中如果遇到范围条件，尽可能去精确一点


# 3. 总结
我们在实际开发过程中 ，一定要注意以上这些场景，我们要灵活运用我们的索引；尽量使用覆盖索引，减少select * 的这种写法，更要关心我们的查询条件的数据量




> 原文: <https://www.yuque.com/tulingzhouyu/sfx8p0/gxbfeagi2g49gipa>