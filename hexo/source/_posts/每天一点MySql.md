---
title: 每天一点MySql
date: 2019-01-22 10:09:19
tags:
	- 学习笔记
	- mySql
---
## 写在前面
时至今日,已经是我毕业工作的第0.5个年头了.遥想大二当年学的数据库原理,更多的是基于sql server 2008的,其中大部分知识已经回归了书本的怀抱.即使身为一名前端工程师,但平时工作中更多的是使用Node和sequlize框架和mysql数据库进行交互,感觉到有必要强化学习一下mysql知识.

买了一本MySQL必知必会,每天至少保证一小时的阅读时间,并根据自己的水平,记录对应的阅读笔记.

## 基础

####连接到mysql

1. 本地服务器 localhost
2. 默认的端口 3306
3. 用户
4. 密码

#### 了解mysql的简单指令

~~~
SHOW TABLES 				// 返回当前选择的数据库内可用表的列表
SHOW COLUMNS FROM 表名	   // 返回所有列(Field, Type, Null, Key, Default, Extra)
DESCRIBE 表名 			   // 同上,为SHOW COLUMNS的一种快捷方式
SHOW STATUS					// 显示广泛的服务器状态信息
SHOW CREATE DATABASE		// 显示创建数据库SQL语句
SHOW CREATE TABLE			// 显示创建表SQL语句
SHOW GRANTS					// 显示授予用户的安全权限
SHOW ERRORS					// 显示服务器错误消息
SHOW WARNINGS				// 显示服务器警告消息
HELP SHOW					// 显示允许的SHOW语句
~~~

#### SQL语句常用命令

> 多条SQL语句必须以分号 ( ; ) 分隔
>
> SQL语句不区分大小写,一般关键字使用大写,列名表名使用小写,更易于阅读和调试
>
> 处理SQL语句是,其中所有空格都被忽略

```
SELECT *						// 通配符 ( * ), 返回所有
SELECT DISTINCT XX FROM 表名	   // 检索出不同的值,DISTINCT应用于所有列而非前置列
SELECT XX FROM 表名 LIMIT 5      // 检索返回不多于5行(LIMIT 3,4 返回从行3开始的4行)
									MYSQL5 支持LIMIT 4 OFFSET 3 从行3开始取4行
```

* 排序数据 ORDER BY

数据一般将以它在底层表中的出现的顺序显示,无排序检索应该认为顺序无异议; 如果数据后来进行过更新或删除, 则此顺序将会受到MySQL重用回收存储空间的影响

```
ORDER BY 						// 检索数据以默认ASC升序A~Z排序(ORDER BY xx DESC 降序)
* 应位于FROM子句之后,若使用LIMIT,需位于ORDER BY之后;
	是否区分大小写取决于数据库的设置
```

* WHERE子句操作符

```
<> 等价于 != 即不等于,不匹配检查
字符串值用单引号 '' 括起
BETWEEN 范围检查
IS NULL 空值检查
* NULL与不匹配,匹配过滤或不匹配过滤时不返回

* 组合WHERE子句
AND 							// 与
OR  							// 或

* 计算次序
AND在计算次序中优先级更高
... WHERE vid = 10002 OR vid = 10003 AND price >= 10;
理解为 检索条件由id为10002,或者id为10003且价格大于等于10的
=> ... WHERE vid = 10002 OR (vid = 10003 AND price >= 10);

如果要检索由id为10002或10003,且价格都大于等于10的,应使用括号提高优先级
=> ... WHERE (vid = 10002 OR vid = 10003) AND price >= 10;

检索id为10002或10003,除了使用OR操作符外,还可以
... WHERE vid IN (10002, 10003);
* IN操作符一般比OR操作符执行更快,语法清楚且直观

NOT 操作符用来否定后跟条件的关键字
... WHERE vid NOT IN (10002, 10003);
```

* 通配符过滤

```
LIKE '%p'					// p结尾
LIKE 'p%'					// p开头
LIKE '%p%'					// 包含p

* '_' 用途与'%'一样,但只匹配单个字符而不是多个
* 尾空格可能会干扰通配符匹配,最好的办法是使用函数去掉首尾空格,亦或者首尾使用%
* %通配符无法匹配 NULL
```

* MySQL正则表达式

```
REGEXP (BINARY区分大小写) '.000'		// 匹配 000尾部
* LIKE匹配整个列,REGEXP在列值内匹配
* \\ 转义字符
\\f 换页; \\n 换行; \\r 回车; \\t 制表; \\v 纵向制表; \\\ 匹配反斜杠\

字符类
[:alnum:]				任意字母和数字([a-zA-Z0-9])
[:alpha:]				任意字符([a-zA-Z])
[:blank:]				空格和制表([\\t])
[:cntrl:]				ASCII控制字符(ASCII 0到31和127)
[:digit:]				任意数字([0-9])
[:graph:]				与[:print:]相同,但不包括空格
[:lower:]				任意小写字母([a-z])
[:print:]				任意可打印字符
[:punct:]				既不在[:alnum:]又不在[:cntrl:]中的任意字符
[:space:]				包括空格在内的任意空白字符([\\f\\n\\r\\t\\v])
[:upper:]				任意大写字母([A-Z])
[:xdigit:]				任意十六进制数字([a-fA-F0-9])

重复元字符
*						0个或多个匹配
+						1个或多个匹配({1,})
?						0个或1个匹配({0,1})
{n}						指定数目的匹配
{n,}					不少于指定数目的匹配
{n,m}					匹配数目的范围(m不超过255)

定位元字符
^						文本的开始
$						文本的结束
[[:<:]]					词的开始
[[:>:]]					词的结束

简单的正则表达式测试
SELECT 'hello' REGEXP '[0-9]';		=> 返回0
REGEXP检查总返回0或1
```

* 函数

```
Concat()				// 拼接函数
Trim()					// 去除首尾空格(RTrim(), LTrim())
Now()					// 返回当前日期和时间
Left()					// 返回串左边的字符
Length()				// 返回串的长度
Locate()				// 找出串的一个子串
Lower()					// 将串转换为小写
Right()					// 返回串右边的字符
Soundex()				// 返回串的SOUNDEX值
SubString()				// 返回子串的字符
Upper()					// 将串转换为大写

日期和时间处理函数
NOW()					// 返回当前的日期和时间
CURDATE()				// 返回当前的日期
CURTIME()				// 返回当前的时间
DATE()					// 提取日期或日期/时间表达式的日期部分
EXTRACT()				// 返回日期/时间按的单独部分
DATE_ADD()				// 给日期添加指定的时间间隔
DATE_SUB()				// 从日期减去指定的时间间隔
DATEDIFF()				// 返回两个日期之间的天数
DATE_FORMAT()			// 用不同的格式显示日期/时间

常用数值处理函数
Abs()					// 绝对值
Cos()					// 角度余弦
Exp()					// 指数值
Mod()					// 除操作的余数
Pi()					// 圆周率
Rand()					// 随机数
Sin()					// 角度正弦
Sqrt()					// 平方根
Tan()					// 角度正切

SQL聚集函数
AVG()					// 平均值,忽略NULL行
COUNT()					// 行数,COUNT(*)包括空值NULL和非空值,COUNT(column)忽略NULL行
MAX()					// 最大值
MIN()					// 最小值
SUM()					// 列值和
* MySQL允许MIN()和MAX()返回文本数据列
* 指定DISTINCT参数只包含不同值,必须指定列名
```

* 别名 AS

1. 通过`AS`别名执行算术计算`SELECT a*b AS c FROM 表名`
2. 表别名不返回到客户机,列别名返回客户机

* 分组数据 GROUP BY

```
GROUP BY 				// 必须出现在WHERE之后,ORDER BY之前
WITH ROLLUP				// 可以得到分组汇总
HAVING					// 过滤分组,WHERE过滤行
* WHERE分组前过滤,HAVING分组后过滤
* 使用GROUP BY的时候应给也给出ORDER BY子句
* 顺序: SELECT => FROM => WHERE => GROUP BY => HAVING => ORDER BY => LIMIT
```

* 子查询

子查询嵌套不如联结,不建议疯狂套子查询;

* 联结 JION

```
* 完全限定列名,用.分隔表名和列名
* 笛卡尔积 没有WHERE子句设定联结条件的表关系返回笛卡尔积,即n*m,故所有联结都应有WHERE子句

1.内部联结/等值联结
SELECT v_name, p_name, p_price
FROM v INNER JOIN p
ON v.v_id = p.v_id;
* 性能考虑,联结的表越多,性能下降越厉害

2.自联结
SELECT p1.p_id, p1.p_name
FROM p AS p1, p AS p2
WHERE p1.v_id = p2.v_id
AND p2.p_id = 'DTNTR';

3.自然联结
* 排除多次出现,使每个列只返回一次
* 自己选择SELECT 列来完成自然联结

4.外部联结
LEFT OUTER JOIN				// 左联结
RIGHT OUTER JOIN			// 右联结
```

* 组合查询 UNION

```
UNION 每个查询必须包含相同的列,表达式或聚集函数(次序无需相同)
* UNION从查询结果集中自动去除了重复行;如不需自动去重,使用UNION ALL
* 只能使用一条ORDER BY子句,必须出现在最后一条SELECT语句后
* 可以使用UNION组合查询应用不同的表
```

* 全文本搜索 FULLTEXT

```
* 并不是所有引擎都支持全文本搜索,MyISAM支持全文本搜索,InnoDB不支持
* 启用全文本搜索支持在创建表时接受FULLTEXT(列名),自动维护索引
* 导入数据应该先导入,再修改表,定义FULLTEXT
Match()指定被搜索的列, Against指定要使用的搜索表达式
WITH QUERY EXPANSION 查询扩展
IN BOOLEAN MODE 布尔文本搜索
```

* 插入数据 INSERT

```
INSERT操作可能很耗时,特别是有很多索引需要更新时,遇到重要的SELECT可以给INSERT操作降权
INSERT LOW_PRIORITY INTO
* 单条INSERT插入多条语句比多条INSERT语句快

插入检索出的数据, 无需关注SELECT中的列名,可包含WHERE子句
INSERT SELECT
```

* 更新和删除数据 UPDATE DELETE

```
UPDATE 表名 SET 列名 = ''(, 列名 = '') WHERE ...;
* IGNORE关键字 UPDATE IGNORE 表名... 错误发生前更新的所有行恢复原值
删除指定列使用UPDATE ... SET 列名 = NULL

DELETE FROM 表名 WHERE ...; 
* 生产上一般使用软删除,不使用物理删除
* 不要忽略WHERE子句
* 更快的删除 TRUNATE TABLE,实际是删除原来的表并重建一个表,而不是逐行删除
```

* 创建和操纵表 CREATE/ALTER

```
ddl 数据定义语言(data define language)
dml 数据操纵语言(data manipulation language)

* 表需要主键,使用AUTO_INCREMENT,自增长,每个表只允许存在一个,且必须被索引
CREATE TABLE 表名
(
	列名		类型		NOT NULL	AUTO_INCREMENT,
	...
	PRIMARY KEY(列名),
	FULLTEXT(列名)
) ENGINE=MyISAM;
* 表不存在时创建,应在表名后加 IF NOT EXISTS
* NULL值是没有值,不是空串(''),空串是一个有效值
* 默认值,MySQL不允许使用函数作为默认值,只支持常量

引擎类型
1.InnoDB 可靠的事务处理引擎,不支持全文本搜索
2.MEMORY 功能等同于MyISAM,数据存储在内存(非磁盘),速度很快,适用于临时表
3.MyISAM 性能极高的引擎,支持全文本搜索,不支持事务处理
* 引擎类型可以混用
* 外键不能跨引擎,使用一个引擎的表不能引用具有使用不同引擎的表的外键

ALTER TABLE 更新表
* 使用前应做一个完整的备份(模式和数据的备份),数据库表的操作不能更改

DROP TABLE 删除表
RENAME TABLE 表名 TO 新表名 
```

* 视图 VIEW

```
CREATE VIEW 创建视图
SHOW CREATE VIEW viewname 查看创建视图的语句
DROP VIEW viewname
CREATE OR REPLACE VIEW 更新视图

* 视图是虚拟的表,只包含使用时动态检索数据的查询
* 使用多个联结和过滤创建复杂视图会发现性能下降得很厉害
* 一般,视图用于检索SELECT,不用于INSERT,UPDATE和DELETE
```

* 存储过程 PROCEDURE

```
使用存储过程
CALL procedurename(arg, arg...)

创建存储过程
CREATE PROCEDURE procedurename()
BEGIN
	...
	
END;

* mysql命令行客户机的分隔符
为防止语法错误,可以临时更改命令行使用程序的语句分隔符:
DELIMITER //
CREATE PROCEDURE procedurename()
BEGIN
	SELECT ...
	FROM 表名;
END //
DELIMITER ;

删除存储过程
DROP PROCEDURE (IF EXISTS) procedurename; (IF EXISTS 仅当存在时删除,可以避免错误)

* 一般存储过程并不显示结果,而是把结果返回给你指定的变量  DECIMAL(8,2)=>8位数字,小数点2位
CREATE PROCEDURE procedurename(
	OUT pl DECIMAL(8,2),
	OUT ph DECIMAL(8,2),
	OUT pa DECIMAL(8,2),
)
BEGIN
	...
END;

* MySQL支持IN(传递给存储过程),OUT(从存储过程中传出)和INOUT(对存储过程传入和传出)

使用procedurename存储过程
call procedurename(@pl, @ph, @pa);
* 所有MySQL变量都必须以@开始

检查存储过程
SHOW PROCEDURE STATUS
显示过程状态结果(指定过滤模式)
SHOW PROCEDURE STATUS LIKE 'ordertotal';
```

* 游标 CURSOR

```
游标主要用于交互式应用,其中给用户需要滚动屏幕上的数据,并对数据进行浏览或作出更改.

创建游标
CREATE PROCEDURE procedurename()
BEGIN
	DECLARE cursorname CURSOR
	FOR
	SELECT order_num FROM orders;
END;
* 存储过程处理完成后,游标就消失

打开和关闭游标
OPEN cursorname;
CLOSE cursorname;
* 使用声明过的游标不需要再次声明,用OPEN语句打开即可
* 隐含关闭,MySQL会在END时自动关闭它

* DECLARE定义的局部变量必须在定义任意游标或句柄之前定义,而句柄必须在游标之后定义
* REPEAT 语句对游标进行循环
```

* 触发器 TRIGGER

```
事件发生时自动执行

创建触发器 *保持每个数据库的触发器名唯一
CREATE TRIGGER triggername AFTER INSERT ON 表名
FOR EACH ROW SELECT 'Product added';

* 仅有表支持触发器,视图不支持
* 每个表最多支持6个触发器
			INSERT
BEFORE		UPDATE		AFTER
			DELETE
通常,BEFORE用于数据验证和净化,确保传入表中的数据确实是需要的数据
			
删除触发器
DROP TRIGGER triggername;
* 触发器不能更新或覆盖,必须先删除,然后重新创建

INSERT触发器
TODO
DELETE触发器
TODO
UPDATE触发器
TODO

* MySQL触发器不支持CALL语句,这表示不能从触发器内调用存储过程,存储过程代码需复制到触发器内
```

* 事务 TRANSACTION

```
用来维护数据库的完整性,保证成批的MySQL操作要么完全执行,要么完全不执行

事务 transaction 指一组SQL语句
回退 rollback 指撤销指定SQL语句的过程
提交 commit 指将未存储的SQL语句结果写入数据库表
保留点 savepoint 指事务中是指的临时占位符,可以对它发布回退

控制事务处理
1.将SQL语句分解为逻辑块
2.明确规定数据何时回退,何时不回退

使用ROLLBACK回退
SELECT * FORM 表名;
START TRANSATION;
DELETE FROM 表名;
ROLLBACK;
* 无法回退SELECT语句,这样做也没有意义
* 不能回退CREATE或DROP操作,不会被撤销

使用COMMIT
START TRANSACTION;
DELETE FROM 表1 WHERE o_num = 20010;
DELETE FROM 表2 WHERE o_num = 20010;
COMMIT;
如果第一条DELETE起作用,第二条失败,则DELETE不会被提交

* COMMIT或ROLLBACK执行后,事务会自动关闭

支持部分事务回退
使用保留点,标识唯一名字
SACEPOINT pointname
* 保留点越多越好,在执行一条ROLLBACK或COMMIT后自动释放
* MySQL5之后可以用RELEASE SAVEPOINT释放保留点
```

* 全球化和本地化

```
字符集/编码/校对
字符集我们一般使用utf8mb4

SHOW CHARACTER SET;		// 显示所有可用字符集及其默认描述和校对
SHOW COLLATION;			// 显示所有可用的校对
```

* 安全管理

```
本地学习实践可以使用root用户,生产上不使用root用户,防止无意的错误

创建用户账号
CREATE USER name IDENTIFIED BY 'password';
重命名用户账号
RENAME USER name TO newname;
删除用户账户
DROP USER name;

 查看用户权限
 SHOW GRANTS FOR name;
 'USAGE' => 无权限
 
 GRANT SELECT ON 库名.* TO 用户名;
 允许用户在库所有表上使用SELECT
 * GRANT反操作为REVOKE,用来撤销特定的权限
 简化多次授权
 GRANT SELECT, INSERT ON 库名.* TO 用户名;
 
 更改口令
 SET PASSWORD FOR 用户名 = Password('密码');
 * 不指定用户时,更新当前用户登录口令
```

* 数据库维护

```
FLUSH TABLES			// 刷新未写数据

* 可以使用BACKUP TABLE或SELECT INTO OUTFILE转储所有数据到某个外部文件
* 数据可以用RESTORE TABLE来复原

ANALYZE TABLE 表名;		// 检查表键是否正确
CHECK TABLE 表名;
* 尽量手动启动服务器

查看日志文件
1.错误日志 hostname.err 位于data目录,可用--log-error命令行选项更改
2.查询日志 hostname.log 位于data目录,可用--log命令行选项更改
3.二进制日志 hostname-bin 位于data目录,可用--log-bin命令行选项更改
4.缓慢查询日志 hostname-slow.log 位于data目录,可用--log-slow-queries命令行选项更改

FLUSH LOGS来刷新和重新开始所有日志文件
```

* 改善性能

```
MySQL是一个多用户多线程的DBMS,如果任务重某一个执行缓慢,则所有请求都会执行缓慢
SHOW PROCESSLIST 显示所有活动进程及其线程ID和执行时间
KILL 终结某个特定的进程,需管理员权限

* 索引改善数据检索的性能,但损害数据插入,删除和更新的性能.根据情况添加索引.
* LIKE很慢,一般最好使用FULLTEXT

* 最重要的规则就是,每条规则在某些条件下都会被打破
我的理解就是 任何无视环境背景情况的规则都是瞎逼逼
```



##  进阶

~~这本书真的很基础,进阶的内容需要我从另外的地方去探索!~~
