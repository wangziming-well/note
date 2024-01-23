# SQL简介

* SQL语句：Stuctured Query Language 结构化查询语句。
* SQL语句不依赖于任何平台，对所有的数据库是通用的。
* SQL具有查询、操纵、定义和控制关系型数据库的功能
* SQL是一个非过程性的语句，每个SQL执行完都有有一个具体的结果出现。

## SQL分类

* DDL：数据定义语言
* DML：数据处理语言
* DCL：数据控制语言
* DQL：数据查询语言



# 数据库的操作

~~~cmd
连接数据库
mysql （-h ip -P端口）-u username -p password
连接本机时可省略ip和端口
~~~

~~~sql
#查询数据库#
show databases;#查询所有的数据仓库
show create database 数据库名;#查询指定数据库创建规则

#创建数据库#
create database 数据库名; 
#创建时没有指定编码表，因此会使用安装数据库时默认的编码表。
create database 数据库名 character set 编码表名; 
#创建数据库会使用指定的编码表。
create database 数据库名 character set 编码表名 collate 排序规则; 
#使用指定的编码表同时还可以根据编码表指定排序规则。 规则查看MySQL_5.1 版本的API。

drop database 数据库名; #删除数据库

alter database 数据库名 character set 字符集 collate 比较规则;
# 修改数据库编码集或者比较规则

select database(); #查询正在使用的数据库

use 数据库名;#切换/使用数据库
~~~

# 数据类型与运算

数据类型：

~~~sql
# 字符型
varchar(列长)
char(列长)

# 数值型
tinyint
smallint
int
bigint
float
double

#逻辑形
BIT

#日期型
DATE 年月日
TIME 时分秒
DATETIME 年月日时分秒
TIMESTAMP 年月日时分秒（自动更新保存数据时的当前时间。）

# 大数据类型
BLOB 保存字节数据
TEXT 保存字符数据
~~~



运算：

~~~sql
比较运算：相等 =   不等 <> 
		大于 > , &gt;
		小于 < , &lt;
		大于等于 >= , &gt;=
		小于等于 <= , &lt;=

逻辑运算： and or not 

区间判断： between ...and... (取闭区间)

列表取值：in(值，值，值)

模糊匹配 like 'pattern'   %表示任意字符串，_表示单个字符

正则匹配 regexp 'pattern' 

null空判断: is null  和 is not null
~~~



# 表结构的操作

~~~sql
# 查看当前数据库所有表
show tables; 
# 查看表信息 
show create table tableName;
#查看表的列信息
show columns from tableName;
desc tableName;
#创建表
create table tableName(
    column1 type [constraint1]，
 	column1 type [constraint2]，
 	...
);

# 删除表
drop table 表名;

# 表单创建时约束

列名 列的类型 unique; # 唯一约束
列名 列的类型 not null; # 非空约束
列名 列的类型 primary key; # 主键约束（唯一且非空）
列名 列的类型 auto_increment; #自增长
列名 列的类型 default 默认值# 设置默认值

#修改删除表结构
rename table 旧表名 to 新表名;# 修改表名
alter table 表名 add 列名 类型 约束;  # 增加列
alter table 表名 modify 列名 类型 约束;  # 修改现有列的类型和约束
alter table 表名 change 旧列名 新列名 类型 约束;  # 修改现有列名称 类型和约束
alter table 表名 drop 列名; #删除列
alter table 表名 character set 字符集;# 修改表字符集

~~~

# 表数据的操作

~~~sql
# distinct 只返回不同的值
# columnName :列名
# alias :别名
# tableName : 表名
# filterCondition:过滤条件 
select [distinct] columnName1 [ [as] alias1] [,columnName2 [ [as] alias2]...] | *
from tableName
[where filterCondition]
[group by column1 [, column2...]]
[having filterCondition]
[order by columnNameA [asc]|desc [,columnNameB [asc]|desc...]]
[limit  [start,]step]


#low_priority 降低该语句的执行优先级，让其他语句先执行
insert [low_priority] into tableName(column1，column2...) values (value11，value12...)[,(value21，value22...)...];

insert into tableName1(column1，column2...) select columnA，columnB... from tableName2

#ignore 在执行多条更新语句时，如果其中的一行发送错误，那么整个update操作会被取消，想要让即使发送错误，也能继续更新，可以使用ignore关键字
update [ignore] tableName set column1=value1,column2=value2.... where filterCondition; 
delete from tableName [where filterCondition]；

#截断表 删除表中所有数据 TRUNCATE实际是删除原来的表并重新创建一个表，而不是逐行删除表中的数据不可回滚 重置表的自增值
truncate table tableName;
~~~

# Select查询

## 分组查询

### group by

分组是在SELECT语句的GROUP BY子句中建立的。  

* GROUP BY子句可以包含任意数目的列。这使得能对分组进行嵌套，为数据分组提供更细致的控制。  

* ==?如果在GROUP BY子句中嵌套了分组，数据将在最后规定的分组上进行汇总。换句话说，在建立分组时，指定的所有列都一起计算（所以不能从个别的列取回数据)==  

* GROUP BY子句中列出的每个列都必须是检索列或有效的表达式（但不能是聚集函数）。如果在SELECT中使用表达式，则必须在GROUP BY子句中指定相同的表达式。不能使用别名。
* 除聚集计算语句外， SELECT语句中的每个列都必须在GROUP BY子句中给出。
* 如果分组列中具有NULL值，则NULL将作为一个分组返回。如果列中有多行NULL值，它们将分为一组。
* GROUP BY子句必须出现在WHERE子句之后， ORDER BY子句之前。  

使用WITH ROLLUP关键字，可以得到每个分组以及每个分组汇总级别（针对每个分组）的值  

### having

HAVING非常类似于WHERE。事实上，目前为止所学过的所有类型的WHERE子句都可以用HAVING来替代。唯一的差别是WHERE过滤行，而HAVING过滤分组。  

这里有另一种理解方法，WHERE在数据分组前进行过滤， HAVING在数据分组后进行过滤。这是一个重要的区别， WHERE排除的行不包括在分组中。这可能会改变计算值，从而影响HAVING子句中基于这些值过滤掉的分组。  

## 内联查询

~~~sql
select * from 表名1,表名2 where 表名1.列名 = 表名2.列名;
select * from 表名1 inner join 表名2 on 条件;
~~~

## 外联查询

~~~sql
# 右外连接
select * from 表1 left outer join 表2 on 条件;
# 左外连接
select * from 表1 right outer join 表2 on 条件;
# 全连接
select * from 表1 full outer join 表2 on 条件; # 但是mysql数据库不支持此语法。 

#在sql语句全连接，其实就是左外链接和右外连接之和，并且使用union去掉重复的数据。
select * from 表1 left outer join 表2 on 条件 
union all 
select * from 表1 right outer join 表2 on 条件;


~~~

## 组合查询

~~~sql
查询语句1
union
查询语句2
...
~~~

* `union`查询的结果集会自动去重，如果不想去重，可以使用` union all`
* 对组合查询结果排序，可以在最后一条select语句用`order by`

## SQL关联子查询

* 子查询：把一个SQL语句的查询结果当做另一个SQL语句查询的参数

## 全文本搜索

要使用全文本搜索：

* 需要使用支持全文本搜索的引擎：创建表时，可以指定engine=MyISAM
* 需要对要进行全文本搜索的列指定索引：`fulltext(colunm)`

在索引之后，使用两个函数`Match()`和`Against()`执行全文本搜索，其中`Match()`指定被搜索的列， `Against()`指定要使用的搜索表达式。  



# SQL函数

与其他大多数计算机语言一样， SQL支持利用函数来处理数据。函数一般是在数据上执行的，它给数据的转换和处理提供了方便。

**注意**：函数没有SQL的可移植性强,几乎每种主要的DBMS的实现都支持其他实现不支持的函数，而且有时差异还很大。  

## 文本处理函数

* `Concat()`拼接文本

* `Trim()`去掉文本右侧的所有空格
* `LTrim() `去掉串左边的空格
* `RTrim()` 去掉串右边的空格
* `Upper()`将文本转换为大写
* `Lower() `将串转换为小写

* `Left()` 返回串左边的字符
* `Right()` 返回串右边的字符
* `Length() `返回串的长度
* `Locate() `找出串的一个子串

* `Soundex()` 返回串的SOUNDEX值
* `SubString()` 返回子串的字符



## 日期和时间处理函数

* `AddDate()` 增加一个日期（天、周等）
* `AddTime() `增加一个时间（时、分等）
* `CurDate() `返回当前日期
* `CurTime() `返回当前时间
* `Date() `返回日期时间的日期部分
* `DateDiff() `计算两个日期之差
* `Date_Add() `高度灵活的日期运算函数
* `Date_Format() `返回一个格式化的日期或时间串
* `Day()` 返回一个日期的天数部分
* `DayOfWeek() `对于一个日期，返回对应的星期几
* `Hour() `返回一个时间的小时部分
* `Minute() `返回一个时间的分钟部分
* `Month() `返回一个日期的月份部分
* `Now() `返回当前日期和时间
* `Second() `返回一个时间的秒部分
* `Time()` 返回一个日期时间的时间部分
* `Year()` 返回一个日期的年份部分



## 数值处理函数

* `Abs() `返回一个数的绝对值
* `Cos() `返回一个角度的余弦
* `Exp() `返回一个数的指数值
* `Mod() `返回除操作的余数
* `Pi()` 返回圆周率
* `Rand() `返回一个随机数
* `Sin() `返回一个角度的正弦
* `Sqrt() `返回一个数的平方根
* `Tan() `返回一个角度的正切



## 聚集函数

我们经常需要汇总数据而不用把它们实际检索出来，为此MySQL提供了专门的函数。

* `AVG() `返回某列的平均值
    * AVG()只能用来确定特定数值列的平均值，而且列名必须作为函数参数给出。为了获得多个列的平均值，必须使用多个AVG()函数。
    * AVG()函数忽略列值为NULL的行。  
* `COUNT() `返回某列的行数
    * 使用COUNT(*)对表中行的数目进行计数， 不管表列中包含的是空值（ NULL）还是非空值  
    * 使用COUNT(column)对特定列中具有值的行进行计数，忽略NULL值。  
* `MAX() `返回某列的最大值
* `MIN() `返回某列的最小值
* `SUM()` 返回某列值之和

以上五个函数参数是列名时，可以使用 distinct关键字，只计算不同列值的部分



# 多表设计

## 外键约束

* 外键： 在一个表中去引用另外一张表的主键作为该表的字段，这个字段被称为外键，一旦有了外键，表与表之间就产生了外键约束

* 作用：减少数据冗余，维护多表之间的数据完整性，减少垃圾数据

* 主表：主键被引用的表

* 从表：存在外键的表

* **必须保证从表的外键值在主表的主键值中存在**

* 语法：

    ~~~sql
    #表存在时：
    alter table 从表 add constraint 外键约束名 foreign key(从表的列) references 主表(主表的主键列)
    #表创建时；
    constraint 外键约束名 foreign key(从表的列) references 主表(主表的主键列)
    #直接在属性后：
    references 主表(主表的主键列)
    s
    #解除外键约束
    alter table 表名 drop foreign key 外键约束名;
    ~~~



## 级联操作

* 通过操作主表的数据，从而影响到从表的数据。

* **操作风险很大，一般默认不进行级联操作**

* 语法格式:

    ~~~sql
    #级联更新 ：
    外键约束语句 on update cascade;
    #级联删除 ：
    外键约束语句 on delete cascade;
    ~~~

    

## 数据库设计三大范式

* 第一范式：要求表的每个字段必须是不可分割的独立单元。
* 第二范式：要求每张表只表达一个意思，表的每个字段和表的主键有依赖
* 第三范式：要求每张表的主键之外的其他字段都只能和主键有直接决定的依赖



# SQL高级



## 存储过程

存储过程简单来说，就是为以后的使用而保存的一条或多条MySQL语句的集合。  

它可以：

* 通过把处理封装在容易使用的单元中，简化复杂的操作（正如前面例子所述）。
* 由于不要求反复建立一系列处理步骤，这保证了数据的完整性。如果所有开发人员和应用程序都使用同一（试验和测试）存储过程，则所使用的代码都是相同的。这一点的延伸就是防止错误。需要执行的步骤越多，出错的可能性就越大。防止错误保证了数据的一致性。
* 简化对变动的管理。如果表名、列名或业务逻辑（或别的内容）有变化，只需要更改存储过程的代码。使用它的人员甚至不需要知道这些变化。

* 提高性能。因为使用存储过程比使用单独的SQL语句要快。
* 存在一些只能用在单个请求中的MySQL元素和特性，存储过程可以使用它们来编写功能更强更灵活的代码（在下一章的例子中可以看到。）  

### 创建存储过程

~~~sql
create procedure demo(
    in param2 dateType1,
    out param1 dateType2,
    
	...
)[comment '该存储过程的描述']
begin
	select count(1) into param2
	from u_user where u_money > param1;
end ;
~~~

以上存储过程

* 类似于函数的参数： 需要调用者传入参数param1 ，来参入存储过程

* 类似于函数的返回值：将 select语句的结果count(1)存入声明的参数变量param2中这样调用者就能通过sql变量获取到该值

**注意**：如果命令行实用程序要解释存储过程自身内的;字符，则它们最终不会成为存储过程的成分，这会使存储过程中的SQL出现句法错误  

解决办法是临时更改命令行实用程序的语句分隔符  :

~~~sql
delimiter //
create procedure demo()
begin
	select count(1) as amount
	from u_user;
end //
delimiter ;
~~~

### 删除存储过程

~~~sql
drop procedure procedureName;
~~~

### 检查存储过程

~~~sql
#显示用来创建一个存储过程的CREATE语句
show create procedure procedureName;
#获得包括何时、由谁创建等详细信息的存储过程列表
#SHOW PROCEDURE STATUS列出所有存储过程。为限制其输出，可使用LIKE指定一个过滤模式
show procedure status [like 'procudureName']
~~~



### 使用存储过程

~~~sql
call demo(param @amount);
select @amount;
~~~

调用者需要传入存储过程需要的数据值，和接受结果的变量存储过程的结果会存入到变量中

所有MySQL变量都必须以@开始

### 高级存储过程

在定义存储过程中：

* 可以定义局部变量 

    ~~~sql
    declare paramName paramType [default value1];
    ~~~

    同样用`into`关键字将值赋值给局部变量

* 可以使用流程控制

    * if else:

        ~~~sql
        if condition0 then SQL;
        elseif condition1 then SQL;
        else DQL;
        end if;
        ~~~

    * case:

        ~~~sql
        case parameter
        when condition1 then SQL;
        when condition2 then SQL;
        else SQL;
        end case;
        ~~~

    * while:

        ~~~sql
        while condtion  do
        	set function1;
        	set function2;
        end while;
        ~~~

        set 关键字可以改变变量的值，在这里可以控制循环出口。

    * loop：

        ~~~sql
        cyclic_name:loop
        	set function1; # 每次循环都会执行该语句
            set function2; 
        if condition0 then
        	iterate cyclic_name;#继续循环
        endif;
        if condition1 then
        	leave cyclic_name; #循环出口
        endif;
        end loop;
        ~~~

    * repeat

        ~~~sql
        repeat
        	set function2;
        	set i=i+1;
        until condition # 循环出口 注意这里没有分号
        end repeat;
        ~~~

**注意:**如果循环的中需要查询某个表，循环结束的条件是查询完整个表，或者游标遍历完整个表，可以使用下面的语句：

~~~sql
declare continue handler for sqlstate '02000'|NOT FOUND set done=1;
~~~

该语句使下面的执行过程中，一旦有空记录（查询没有返回结果），或者一旦有异常，就会把done设置为1

这样循环就可以根据done的值作为循环出口

## 游标

游标（ cursor） 是一个存储在MySQL服务器上的数据库查询，它不是一条SELECT语句，而是被该语句检索出来的结果集。在存储了游标之后，应用程序可以根据需要滚动或浏览其中的数据。  

通过游标可以对结果集进行逐行处理

MySQL游标只能用于存储过程（和函数）。  

### 声明游标

~~~sql
declare cursorName cursor       
[ local | global ]                                   --游标的作用域
[ forword_only | scroll ]                            --游标的移动方向
[ static | keyset | dynamic | fast_forward ]         --游标的类型
[ read_only | scroll_locks | optimistic ]            --游标的访问类型
[ type_warning]                                      --类型转换警告语句
FOR SELECT DQL                                      --SELECT查询语句
[ for { read only | update [of columName]}][,...n]      --可修改的列
~~~

* 作用域：
    * `Local`： (默认)作用域为局部，只在定义它的批处理，存储过程或触发器中有效。
    * `Global `：作用域为全局，由连接执行的任何存储过程或批处理中，都可以引用该游标。

* 移动方向：
    * `Forward_Only`:指定游标智能从第一行滚到最后一行。Fetch Next是唯一支持的提取选项。
    * `Scroll`

* 类型：

    - `static `: 静态游标的结果集，在游标打开的时候建立在TempDB中，不论你在操作游标的时候，如何操作数据库，游标中的数据集都不会变。例如你在游标打开的时候，对游标查询的数据表数据进行增删改，操作之后，静态游标中select的数据依旧显示的为没有操作之前的数据。如果想与操作之后的数据一致，则重新关闭打开游标即可。

    - `dynamic `: 这个则与静态游标相对，滚动游标时，动态游标反应结果集中的所有更改。结果集中的行数据值、顺序和成员在每次提取时都会变化。所有用户做的增删改语句通过游标均可见。如果使用API函数或T-SQL Where Current of子句通过游标进行更新，他们将立即可见。在游标外部所做的更新直到提交时才可见。

    - `fast_forward`：只进游标不支持滚动，只支持从头到尾顺序提取数据，数据库执行增删改，在提取时是可见的，但由于该游标只能进不能向后滚动，所以在行提取后对行做增删改是不可见的。

    - `keyset`：打开键集驱动游标时，该有表中的各个成员身份和顺序是固定的。打开游标时，结果集这些行数据被一组唯一标识符标识，被标识的列做删改时，用户滚动游标是可见的，如果没被标识的列增该，则不可见，比如insert一条数据，是不可见的，若可见，须关闭重新打开游标。 静态游标在滚动时检测不到表数据变化，但消耗的资源相对很少。动态游标在滚动时能检测到所有表数据变化，但消耗的资源却较多。键集驱动游标则处于他们中间，所以根据需求建立适合自己的游标，避免资源浪费。

* 访问类型

    - `Read_Only`:不能通过游标对数据进行删改。

    - `Scroll_Locks`：将行读入游标是，锁定这些行，确保删除或更新一定会成功。如果指定啦Fast_Forward或Static，就不能指定他啦。

    - `Optimistic`：指定如果行自读入游标以来已得到更新，则通过游标进行的定位更新或定位删除不成功。当将行读入游标时，sqlserver不锁定行，它改用timestamp列值的比较结果来确定行读入游标后是否发生了修改，如果表不行timestamp列，它改用校验和值进行确定。如果已修改改行，则尝试进行的定位更新或删除将失败。如果指定啦Fast_Forward,则不能指定他。

**注意：**

* 如果在指定游标的移动方向为 Forward_Only，游标类型默认为Dynamic。

* 如果游标类型指定为：
    * Static、KeySet、Dynamic 游标移动方向默认为 Scroll，
    * Fast_Forward 游标移动方向默认为 Forward_Only

### 操作游标

~~~sql
#打开游标
open cursorName;
#关闭游标
close cursorName;
#使用游标
fetch
[[ next | prior | first | last | absolute{n|@nvar }| relative { n|@nvar }] FROM ] 
{{[ GLOBAL] 游标名称} | @游标变量名称 } 
[ INTO @游标变量名称 ][,...n] -- 将读取的游标数据存放到指定变量中


~~~

* `Next`表示返回结果集中当前行的下一行记录，如果第一次读取则返回第一行。默认的读取选项为Next
* `Prior `表示返回结果集中当前行的前一行记录，如果第一次读取则没有行返回，并且把游标置于第一行之前。
* `First`表示返回结果集中的第一行，并且将其作为当前行。
* `Last`表示返回结果集中的最后一行，并且将其作为当前行。
* `Absolute`　n　如果n为正数，则返回从游标头开始的第n行，并且返回行变成新的当前行。如果n为负，则返回从游标末尾开始的第n行，并且返回行为新的当前行，如果n为0，则返回当前行。
* `Relative　`n　如果n为正数，则返回从当前行开始的第n行，如果n为负,则返回从当前行之前的第n行，如果为0，则返回当前行。
* ` ` 缺省 ：返回当前行，并把游标置于下一行

关于游标的状态信息可以通过全局变量获取：

* `@@CORSOR_ROWS` 用来记录游标内的数据行数
    * -m	表示仍在从基础表向游标读入数据，m表示当前在游标中的数据行数
    * -1	该游标是一个动态游标，其返回值无法确定
    * 0	无符合调剂的记录或游标已经关闭
    * n	从基础表向游标读入数据已结束，n 为游标中已有的数据记录行数

* `@@FETCH_STATUS `返回上次执行 FETCH 命令的状态
    * 0	FETCH命令被成功执行
    * 1	FETCH命令失败或者行数据超过游标数据结果集的范围
    * 2	所读取的数据已经不存在

## 触发器

触发器提供某些语句在事件发生时自动执行的功能

触发器只支持响应下面语句：`delete`、`insert`、`update`

### 声明触发器

~~~sql
CREATE TRIGGER trigger_name
trigger_time
trigger_event ON tbl_name
FOR EACH ROW
trigger_stmt
~~~

* `trigger_name`：标识触发器名称，用户自行指定；
* `trigger_time`：标识触发时机，取值为 BEFORE 或 AFTER；
* `trigger_event`：标识触发事件，取值为 INSERT、UPDATE 或 DELETE；
* `tbl_name`：标识建立触发器的表名，即在哪张表上建立触发器；
* `trigger_stmt`：触发器程序体，可以是一句SQL语句，或者用 BEGIN 和 END 包含的多条语句。

trigger_time和trigger_event组合共有6中触发器

**注意:**MYSQL5以后，不允许触发器返回任何结果,先要然触发器传递信息，可以将信息赋值给变量

### 使用触发器

* 删除触发器

    ~~~sql
    drop trigger triggerName;
    ~~~

根据触发时机和触发事件的不同，mysql的触发器有6中

* 触发时机：
    * `BEFORE `在事件发生前执行
    * `AFTER `在事件发生后执行
* 触发事件：
    * `INSERT`
        * 在INSERT触发器代码内，可引用一个名为NEW的虚拟表，访问被插入的行；
        * 在BEFORE INSERT触发器中， NEW中的值也可以被更新（允许更改被插入的值）；
        * 对于AUTO_INCREMENT列， NEW在INSERT执行之前包含0，在INSERT执行之后包含新的自动生成值。  
    * `DELETE`
        * 在DELETE触发器代码内，你可以引用一个名为OLD的虚拟表，访问被删除的行；
        * OLD中的值全都是只读的，不能更新。  
    * `UPDATE`
        * 在UPDATE触发器代码中，你可以引用一个名为OLD的虚拟表访问以前（ UPDATE语句前）的值，引用一个名为NEW的虚拟表访问新更新的值；
        * 在BEFORE UPDATE触发器中， NEW中的值可能也被更新（允许更改将要用于UPDATE语句中的值）；
        * OLD中的值全都是只读的，不能更新。  



## 全球化和本地化

### 字符集和校对顺序

字符集影响编码，校对影响排序

在讨论多种语言和字符集时，将会遇到以下重要术语：

* 字符集为字母和符号的集合
* 编码为某个字符集成员的内部表示
* 校对为规定字符如何比较的指令  

可以 通过下面指令查看字符集和校对：

~~~sql
#显示所有可用的字符集以及每个字符集的描述和默认校对。
show character set;
#显示所有可用的校对，以及它们适用的字符集
show collation;
#查看当前数据库的字符集和校对
show variables like 'character%';
show variables like 'collation%';
~~~

可以在创建表时为表，或者具体的列指定字符集：

~~~sql
create table tableName(
    column1 type [constraint1]，
 	column1 type [constraint2]
    character set latin1 collate latin1_general_ci，
 	...
)default character set utf8mb4
collate utf8mb4_0900_ai_ci;
~~~

一般， MySQL如下确定使用什么样的字符集和校对。

* 如果指定CHARACTER SET和COLLATE两者，则使用这些值。
* 如果只指定CHARACTER SET，则使用此字符集及其默认的校对（如SHOW CHARACTER SET的结果中所示）。
* 如果既不指定CHARACTER SET，也不指定COLLATE，则使用数据库默认。  

### 时区设置

==todo==



## 安全管理

### 管理用户

~~~sql
#查看用户
user mysql;
select user from user;

#创建用户
create user userName identified by 'password';
# 重命名账户名
rename user oldName to newName;
#更改密码
set [for userName] password = PassWord('password');
# 删除用户账户
drop user userName;
~~~

**注意：**

* dentified by 指定的口令为纯文本，MySQL将在保存到user表之前对其进行加密
* insert 语句也可以创建用户，但不建议这么做

### 设置访问权限

~~~sql
#查看用户权限
show grants for userName;
#设置访问权限
grant {select|update|insert|.... }  		--权限类型
on {database.table |database.* | *.* }		--权限范围(*表示通配符)
to userName;								--权限角色
#取消访问权限
revoke {select|update|insert|... }  		--权限类型
on {database.table |database.* | *.* }		--权限范围(*表示通配符)
to userName;							    --权限角色
~~~



# 数据库原理

## 索引

在数据库中可以给表的字段添加索引，已提高查询效率

### 原理

将数据组织为B+树，通过B+树来执行搜索

### 优缺点

优点：

* 提高查询的效率
* 再使用分组和排序子句进行数据检索时，可以减少查询中分组和排序的时间

缺点：

* 创建和维护索引要耗费时间
* 索引需要占用物理空间
* 在对有索引的表进行增删改时需要耗费较多的时间，因为需要维护数据结构的性质

### 声明语法

以普通索引为例

* 建表时创建索引：

    ~~~sql
    index 索引名 (字段名称) # 普通索引 字段的值可以重复
    
    index(字段1,字段2...)    
    # 组合索引的使用：组合索引存在最左原则，左侧的字段1如果如果在查询的时候使用则是有效索引，如果左侧的字段1没有使用，只是使用了字段2，那么索引也不会效果
    ~~~

* 表创建后创建索引

    ~~~sql
    create index 字段名 on 表名(字段名);
    ALTER TABLE table_name ADD INDEX index_name(col_name);
    ~~~

### 索引类型

#### 按逻辑分

* 主键索引 `primary key`

    一张表只能有一个主键索引，不允许重复、不允许为NULL；

* 唯一索引`unique`

    数据列不允许重复，允许为 NULL 值，一张表可有多个唯一索引，索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。

* 普通索引`index`

    一张表可以创建多个普通索引，一个普通索引可以包含多个字段，允许数据重复，允许 NULL 值插入；

* 全文索引`fulltext index`

    加速通过关键字的匹配来进行的查询

    需要配合match 和 against 关键字使用

    - `match()`函数中指定的列必须是设置为全文索引的列
    - `against()`函数标识需要模糊查找的关键字

#### 按列数分

* 单例索引：一个索引只包含一个列，一个表可以有多个单例索引。
* 组合索引：一个组合索引包含两个或两个以上的列。查询的时候遵循 mysql 组合索引的 “最左前缀”原则，即使用 where 时条件要按照建立索引的时候字段的排列方式放置索引才会生效。

#### 物理分类

* 聚簇索引

    数据存储与索引放到了一块，找到索引也就找到了数据。

    数据访问更快

    对于主键的排序查找和范围查找速度非常快

* 非聚簇索引

    数据和索引是分开的，B+树叶子节点存放的不是数据表的行记录。而是数据的引用

### 索引失效

* like查询以`%`开头

* 字符串类型的字段 查询的时候如果不加引号`'' `

* or语句前后没有同时使用索引

* 组合索引中不是使用第一列索引(违背最左前缀原则)

* 在索引列上使用`IS NULL`或`IS NOT NULL`操作

* 在索引字段上使用`not`，`<>`，`!=`等等

* 在where子句中对字段进行算数运算操作和函数操作

* 联合索引中不遵守最左前缀法则

## 锁机制

锁是计算机协调多个进程或线程并发访问某一资源的机制

是为了保证多线程环境下，对相同资源进行操作时的资源安全

数据库中锁有以下分类：

* 按锁的粒度划分，可分为行级锁、表级锁、页级锁。（mysql支持）

* 按锁级别划分，可分为共享锁、排他锁

* 按使用方式划分，可分为乐观锁、悲观锁

### 乐观锁、悲观锁

乐观锁和悲观锁是一种加锁的思想，并不是具体的加锁方式，它们有各自不同的实现方式

* 悲观锁：悲观地认为修改数据时一定会遇到并发修改的情况，所以悲观锁在修改数据之前就对数据进行锁定，让其他事务无法修改数据，当前事务结束后才会释放锁
    * 优点：能为数据安全提供可靠保证
    * 缺点：效率相对低，因为处理加锁机制会让数据库产生额外的开销，还会产生死锁的可能
* 乐观锁：乐观地认为修改数据时一般不会有并发冲突，所以只有在数据进行提交更新的时候，才会对数据的冲突进行检测，如果发生了冲突，就放弃修改
    * 实现：一般通过为表添加一个版本字段，通过版本来实现乐观锁
    * 优点：不会产生死锁，效率高(因为并没有真正的加锁)
    * 缺点：如果没有控制好加锁的颗粒度，那么容易发生业务失败

### 共享锁、排他锁

* 共享锁又称为读锁，简称S锁，就是多个事务对于同一数据可以共享一把锁，都能访问到数据，但是只能读不能修改。

* 排它锁又称为写锁，简称X锁，一个排它锁只能由一个事务持有，获取排他锁的事务是可以对数据就行读取和修改

### 行级锁、表级锁、页级锁

* 行级锁：表示只针对当前操作的行进行加锁；其加锁粒度最小，但加锁的开销也最大。行级锁分为共享锁和排他锁。

* 表级锁：表示对当前操作的整张表加锁；它实现简单，资源消耗较少，被大部分MySQL引擎支持；表级锁定分为表共享读锁（共享锁）与表独占写锁（排他锁）。

* 页级锁：是MySQL中锁定粒度介于行级锁和表级锁中间的一种锁。表级锁速度快，但冲突多，行级冲突少，但速度慢。所以取了折衷的页级，一次锁定相邻的一组记录。

    特点：开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般。



## Log

MySQL 中有以下几种日志，包括：

* redo log（重做日志）
* undo log（回滚日志）
* binary log（二进制日志、归档日志）
* error log（错误日志）
* slow query log（慢查询日志）
* general log（一般查询日志）
* relay log（中继日志）

其中前三个日志比较重要 redo log和undo log 也叫作事务日志

### Redo Log

mysql 通过redo log 记录事务对数据页做的修改，保证了事务的持久性

redo log 包括两部分：一个是内存中的日志缓冲( redo log buffer )，另一个是磁盘上的日志文件( redo logfile)。

mysql 每执行一条 DML 语句，先将记录写入 redo log buffer，后续某个时间点再一次性将多个操作记录写到 redo log file。

InnoDB 存储引擎为 redo log 的刷盘策略提供了 innodb_flush_log_at_trx_commit 参数，它支持三种策略：

* 0 ：设置为 0 的时候，表示每次事务提交时不进行刷盘操作
* 1 ：设置为 1 的时候，表示每次事务提交时都将进行刷盘操作（默认值）
* 2 ：设置为 2 的时候，表示每次事务提交时都只把 redo log buffer 内容写入 page cache

### Undo Log

用以保证事务的原子性；保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读，也即非锁定读

undo log主要记录了数据的逻辑变化，比如一条 INSERT 语句，对应一条DELETE 的 undo log ，对于每个 UPDATE 语句，对应一条相反的 UPDATE 的 undo log ，这样在发生错误时，就能回滚到事务之前的数据状态。

### Binary Log

逻辑日志，记录的是语句原始的逻辑，不论什么存储引擎，只要发生了表数据的更新，都会产生 binlog日志。

可以用于：

* 在主从复制中，从库利用主库上的binlog进行重播，实现主从同步。

* 数据库的基于时间点的还原。

## MVCC

MVCC，Multi-Version Concurrency Control ，多版本并发控制

MVCC 是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问，以提高并发性能。

MVCC只在 读已提交（RC） 和 可重复读（RR） 两个隔离级别下工作。

MVCC使得大部分支持行锁的事务引擎，如，InnoDB，Falcon以及PBXT，不再单纯的使用行锁来进行数据库的并发控制，取而代之的是把数据库的行锁与行的多个版本结合起来，只需要很小的开销,就可以实现非锁定读，从而大大提高数据库系统的并发性能。

## 事务

要求在逻辑上的一组（insert、update、delete）SQL语句，在执行时，要么全部成功，要么全部失败。

对任何一个操作，数据库都有一个隐含提交(implicit commit)的动作，正常情况下，语句是自动提交的

当开启事务时，自动提交失效，需要手动提交(commit,rollback)

也可以通过`set autucommit = 0`关闭自动提交

~~~sql
start transaction;#此语句后面的所有SQL语句都处在在事务中，更改的内容不会自动提交，需要手动提交
savepoint pointName;
rollback [to pointName];#事务的回滚，回滚到事务开始时的状态，如果指定了回滚点，则会滚到回滚点

commit;#事务的提交，表示全部成功，事务结束，数据更改
~~~

`rollback`关键字能退回大部分操作，但不能退回`SELECT`(也没有意义)语句和`CREAT`和`DROP`操作

### 事务特性

事务的特性不是事务的性质，是事务应该遵守的原则，数据库在设计事务时，应当保证事务遵循下面四个特性：

* 原子性

    事务是一个原子操作单元,其对数据的修改,要么全都执行,要么全都不执行。

* 一致性

    在事务开始和完成时,数据都必须保持一致状态。这意味着所有相关的数据规则都必须应用于事务的修改,以保持数据的完整性

* 隔离性

    数据库系统提供一定的隔离机制,保证事务在不受外部并发操作影响的“独立”环境执行。这意味着事务处理过程中的中间状态对外部是不可见的,反之亦然。

* 持久性

    事务完成之后,它对于数据的修改是永久性的,即使出现系统故障也能够保持。

### 事务并发问题

在数据库并发执行多个事务时，如果不加操作，将破会事务的性质(一致性、隔离性):

* 脏读

    一个事务读取了另一个事务未提交的数据。(因为该数据未提交所以是不确定的，所以称为脏数据；如果该事务回滚了，该数据就是错误的数据了)

* 不可重复读

    在同一个事务中，使用相同的查询，读取到的结果的值是不一致的

    这是因为在两次查询间，该数据被其他事务修改(update)并提交了

* 幻读

    一个事务两次执行相同的条件查询，两次查询的记录数不同

    因为有其他事务在两次读取之间增加(insert)或删除(delete)表中数据并提交了

* 第一类更新丢失（回滚丢失）

    一个事务对某数据的回滚操作覆盖了其他事务对该行数据的更新操作。

* 第二类更新丢失(覆盖丢失)

    一个事务对某数据的更新操作覆盖了其他事务对该行数据的更新操作。

### 事务的隔离级别

定义了事务之间的隔离程度

* `Read Uncommitted `读未提交 

    最低级别，只能保证不读取物理上损坏的数据

    可以读到别的事务未提交的数据

    可能发生：脏读 不可重复读 幻读

    * 事务对当前被读取的数据不加锁；
    * 事务在更新某数据的瞬间（就是发生更新的瞬间），必须先对其加 行级共享锁，直到事务结束才释放。

* `Read Committed`读已提交 

    只可以读到别的事务已经提交 的(更新，插入)数据

    可能发生：不可重复读 幻读

    * 事务对当前被读取的数据加 行级共享锁（当读到时才加锁），一旦读完该行，立即释放该行级共享锁；
    * 事务在更新某数据的瞬间（就是发生更新的瞬间），必须先对其加 行级排他锁，直到事务结束才释放。

* `Repeatable Read`可重复读 ( MySQL 的默认隔离级别)

    事务开启，开始读取数据后，不允许其他事务对读取的数据进行更新操作

    可能发生： 幻读

    * 事务在读取某数据的瞬间（就是开始读取的瞬间），必须先对其加 行级共享锁，直到事务结束才释放；
    * 事务在更新某数据的瞬间（就是发生更新的瞬间），必须先对其加 行级排他锁，直到事务结束才释放。

* `Serializable`可串行化 

    事务序列化执行，事务只能一个接着一个地执行，不能并发执行 

    会解决所有的并发事务带来的问题，但效率极低，开发中很少用到

    * 事务在读取数据时，必须先对其加 表级共享锁 ，直到事务结束才释放；
    * 事务在更新数据时，必须先对其加 表级排他锁 ，直到事务结束才释放。

## 数据库优化

### 架构优化

分布式缓存、读写分离(主从复制)、分库分表

### SQL优化

* 查询语句中不要使用select *
* 尽量减少子查询，使用关联查询（left join,right join,inner join）替代(因为子查询会让数据库在内存中建立临时表，并且如果有索引的话，关联查询会走索引)
* 减少使用IN或者NOT IN ,使用exists，not exists或者关联查询语句替代
* or 的查询尽量用 union或者union all 代替(在确认没有重复数据或者不用剔除重复数据时，union all会更好)

