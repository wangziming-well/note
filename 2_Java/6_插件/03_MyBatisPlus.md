# MyBatisPlus简介

MyBatisPlus是一个MyBatis增强工具，在 MyBatis 的基础上只做增强不做改变，以简化开发、提高效率

特性

- **无侵入**：只做增强不做改变，引入它不会对现有工程产生影响，如丝般顺滑
- **损耗小**：启动即会自动注入基本 CURD，性能基本无损耗，直接面向对象操作
- **强大的 CRUD 操作**：内置通用 Mapper、通用 Service，仅仅通过少量配置即可实现单表大部分 CRUD 操作，更有强大的条件构造器，满足各类使用需求
- **支持 Lambda 形式调用**：通过 Lambda 表达式，方便的编写各类查询条件，无需再担心字段写错
- **支持主键自动生成**：支持多达 4 种主键策略（内含分布式唯一 ID 生成器 - Sequence），可自由配置，完美解决主键问题
- **支持 ActiveRecord 模式**：支持 ActiveRecord 形式调用，实体类只需继承 Model 类即可进行强大的 CRUD 操作
- **支持自定义全局通用操作**：支持全局通用方法注入（ Write once, use anywhere ）
- **内置代码生成器**：采用代码或者 Maven 插件可快速生成 Mapper 、 Model 、 Service 、 Controller 层代码，支持模板引擎，更有超多自定义配置等您来使用
- **内置分页插件**：基于 MyBatis 物理分页，开发者无需关心具体操作，配置好插件之后，写分页等同于普通 List 查询
- **分页插件支持多种数据库**：支持 MySQL、MariaDB、Oracle、DB2、H2、HSQL、SQLite、Postgre、SQLServer 等多种数据库
- **内置性能分析插件**：可输出 SQL 语句以及其执行时间，建议开发测试时启用该功能，能快速揪出慢查询
- **内置全局拦截插件**：提供全表 delete 、 update 操作智能分析阻断，也可自定义拦截规则，预防误操作

# 依赖与配置

在springboot项目中的配置：

* 依赖

~~~xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>最新版本</version>
</dependency>
~~~

* Java配置

~~~java
//配置 MapperScan 注解
@SpringBootApplication
@MapperScan("com.baomidou.mybatisplus.samples.mapper")
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
~~~

* springboot核心配置文件

~~~yaml
#配置数据源
spring:
    datasource:
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: root
        url: jdbc:mysql://localhost:3306/security_demo?serverTimezone=UTC
#配置日志输出级别
mybatis-plus:
    configuration:
        log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
~~~



# 注解

MB通过注解建立实体类与数据库表字段的映射

## `@TableName`

- 描述：表名注解，标识实体类对应的表
- 使用位置：实体类

| 属性             | 类型     | 必须指定 | 默认值 | 描述                                                         |
| :--------------- | :------- | :------- | :----- | :----------------------------------------------------------- |
| value            | String   | 否       | ""     | 表名                                                         |
| schema           | String   | 否       | ""     | schema                                                       |
| keepGlobalPrefix | boolean  | 否       | false  | 是否保持使用全局的 tablePrefix 的值（当全局 tablePrefix 生效时） |
| resultMap        | String   | 否       | ""     | xml 中 resultMap 的 id（用于满足特定类型的实体类对象绑定）   |
| autoResultMap    | boolean  | 否       | false  | 是否自动构建 resultMap 并使用（如果设置 resultMap 则不会进行 resultMap 的自动构建与注入） |
| excludeProperty  | String[] | 否       | {}     | 需要排除的属性名                                             |

# `@TableId`

- 描述：主键注解
- 使用位置：实体类主键字段

| 属性  | 类型   | 必须指定 | 默认值      | 描述         |
| :---- | :----- | :------- | :---------- | :----------- |
| value | String | 否       | ""          | 主键字段名   |
| type  | Enum   | 否       | IdType.NONE | 指定主键类型 |

| 属性  | 类型   | 必须指定 | 默认值      | 描述         |
| :---- | :----- | :------- | :---------- | :----------- |
| value | String | 否       | ""          | 主键字段名   |
| type  | Enum   | 否       | IdType.NONE | 指定主键类型 |

IdType:

| 值          | 描述                                                         |
| :---------- | :----------------------------------------------------------- |
| AUTO        | 数据库 ID 自增                                               |
| NONE        | 无状态，该类型为未设置主键类型（注解里等于跟随全局，全局里约等于 INPUT） |
| INPUT       | insert 前自行 set 主键值                                     |
| ASSIGN_ID   | 分配 ID(主键类型为 Number(Long 和 Integer)或 String)(since 3.3.0),使用接口`IdentifierGenerator`的方法`nextId`(默认实现类为`DefaultIdentifierGenerator`雪花算法) |
| ASSIGN_UUID | 分配 UUID,主键类型为 String(since 3.3.0),使用接口`IdentifierGenerator`的方法`nextUUID`(默认 default 方法) |

## `@TableField`

- 描述：字段注解（非主键）

| 属性             | 类型     | 必须指定 | 默认值             | 描述                                                         |
| :--------------- | :------- | :------- | :----------------- | :----------------------------------------------------------- |
| value            | String   | 否       | ""                 | 数据库字段名                                                 |
| exist            | boolean  | 否       | true               | 是否为数据库表字段                                           |
| condition        | String   | 否       | ""                 | 字段 `where` 实体查询比较条件，有值设置则按设置的值为准，没有则为默认全局的 `%s=#{%s}` |
| update           | String   | 否       | ""                 | 字段 `update set` 部分注入，例如：当在version字段上注解`update="%s+1"` 表示更新时会 `set version=version+1` （该属性优先级高于 `el` 属性） |
| fill             | Enum     | 否       | FieldFill.DEFAULT  | 字段自动填充策略                                             |
| select           | boolean  | 否       | true               | 是否进行 select 查询                                         |
| keepGlobalFormat | boolean  | 否       | false              | 是否保持使用全局的 format 进行处理                           |
| jdbcType         | JdbcType | 否       | JdbcType.UNDEFINED | JDBC 类型 (该默认值不代表会按照该值生效)                     |
| numericScale     | String   | 否       | ""                 | 指定小数点后保留的位数                                       |

# IService

MB封装了IService接口，提供了通用的ServiceCRUD方法

普通的Service类通过继承IService类来获取这些方法

IService采用采用 `get 查询单行` `remove 删除` `list 查询集合` `page 分页` 前缀命名方式

泛型 `T` 为任意实体对象

## save



~~~java
// 插入一条记录（选择字段，策略插入）
boolean save(T entity);
// 插入（批量）
boolean saveBatch(Collection<T> entityList);
// 插入（批量） batchSize	插入批次数量
boolean saveBatch(Collection<T> entityList, int batchSize);
~~~

## SaveOrUpdate

~~~java
// 若主键存在则更新，否则插入
boolean saveOrUpdate(T entity);
// 根据updateWrapper尝试更新，否继续执行saveOrUpdate(T)方法
boolean saveOrUpdate(T entity, Wrapper<T> updateWrapper);
// 批量修改插入
boolean saveOrUpdateBatch(Collection<T> entityList);
// 批量修改插入
boolean saveOrUpdateBatch(Collection<T> entityList, int batchSize);
~~~

**注意：**saveOrUpdate会先执行主键查询，根据查询结果决定是更新还是插入

所有如无必要，不要使用saveOrUpdate

## Remove

~~~java
// 根据 entity 条件，删除记录
boolean remove(Wrapper<T> queryWrapper);
// 根据 ID 删除
boolean removeById(Serializable id);
// 根据 columnMap 条件，删除记录
boolean removeByMap(Map<String, Object> columnMap);
// 删除（根据ID 批量删除）
boolean removeByIds(Collection<? extends Serializable> idList);
~~~

## Update

~~~java
// 根据 UpdateWrapper 条件，更新记录 需要设置sqlset
boolean update(Wrapper<T> updateWrapper);
// 根据 whereWrapper 条件，更新记录
boolean update(T updateEntity, Wrapper<T> whereWrapper);
// 根据 ID 选择修改
boolean updateById(T entity);
// 根据ID 批量更新
boolean updateBatchById(Collection<T> entityList);
// 根据ID 批量更新
boolean updateBatchById(Collection<T> entityList, int batchSize);
~~~

## Get

~~~java
// 根据 ID 查询
T getById(Serializable id);
// 根据 Wrapper，查询一条记录。结果集，如果是多个会抛出异常，随机取一条加上限制条件 wrapper.last("LIMIT 1")
T getOne(Wrapper<T> queryWrapper);
// 根据 Wrapper，查询一条记录
T getOne(Wrapper<T> queryWrapper, boolean throwEx);
// 根据 Wrapper，查询一条记录
Map<String, Object> getMap(Wrapper<T> queryWrapper);
// 根据 Wrapper，查询一条记录
<V> V getObj(Wrapper<T> queryWrapper, Function<? super Object, V> mapper);
~~~

## List

~~~java
// 查询所有
List<T> list();
// 查询列表
List<T> list(Wrapper<T> queryWrapper);
// 查询（根据ID 批量查询）
Collection<T> listByIds(Collection<? extends Serializable> idList);
// 查询（根据 columnMap 条件）
Collection<T> listByMap(Map<String, Object> columnMap);
// 查询所有列表
List<Map<String, Object>> listMaps();
// 查询列表
List<Map<String, Object>> listMaps(Wrapper<T> queryWrapper);
// 查询全部记录
List<Object> listObjs();
// 查询全部记录
<V> List<V> listObjs(Function<? super Object, V> mapper);
// 根据 Wrapper 条件，查询全部记录
List<Object> listObjs(Wrapper<T> queryWrapper);
// 根据 Wrapper 条件，查询全部记录
<V> List<V> listObjs(Wrapper<T> queryWrapper, Function<? super Object, V> mapper);

~~~

## Page

~~~java
// 无条件分页查询
IPage<T> page(IPage<T> page);
// 条件分页查询
IPage<T> page(IPage<T> page, Wrapper<T> queryWrapper);
// 无条件分页查询
IPage<Map<String, Object>> pageMaps(IPage<T> page);
// 条件分页查询
IPage<Map<String, Object>> pageMaps(IPage<T> page, Wrapper<T> queryWrapper);

~~~

## Count

~~~java
// 查询总记录数
int count();
// 根据 Wrapper 条件，查询总记录数
int count(Wrapper<T> queryWrapper);
~~~

## Chain

### query

~~~java
// 链式查询 普通
QueryChainWrapper<T> query();
// 链式查询 lambda 式。注意：不支持 Kotlin
LambdaQueryChainWrapper<T> lambdaQuery();

// 示例：
query().eq("column", value).one();
lambdaQuery().eq(Entity::getId, value).list();
~~~

### update

~~~java
// 链式更改 普通
UpdateChainWrapper<T> update();
// 链式更改 lambda 式。注意：不支持 Kotlin
LambdaUpdateChainWrapper<T> lambdaUpdate();

// 示例：
update().eq("column", value).remove();
lambdaUpdate().eq(Entity::getId, value).update(entity);
~~~

# BaseMapper

MB封账了BaseMapper接口，提供了mapper层通用的crud方法

普通mapper接口通过继承它来获取使用这些方法

## Insert

~~~java
// 插入一条记录
int insert(T entity);
~~~

## Delete

~~~java
// 根据 entity 条件，删除记录
int delete(@Param(Constants.WRAPPER) Wrapper<T> wrapper);
// 删除（根据ID 批量删除）
int deleteBatchIds(@Param(Constants.COLLECTION) Collection<? extends Serializable> idList);
// 根据 ID 删除
int deleteById(Serializable id);
// 根据 columnMap 条件，删除记录
int deleteByMap(@Param(Constants.COLUMN_MAP) Map<String, Object> columnMap);
~~~

## Update

~~~java
// 根据 whereWrapper 条件，更新记录
int update(@Param(Constants.ENTITY) T updateEntity, @Param(Constants.WRAPPER) Wrapper<T> whereWrapper);
// 根据 ID 修改
int updateById(@Param(Constants.ENTITY) T entity);

~~~

## Select

~~~java
// 根据 ID 查询
T selectById(Serializable id);
// 根据 entity 条件，查询一条记录
T selectOne(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

// 查询（根据ID 批量查询）
List<T> selectBatchIds(@Param(Constants.COLLECTION) Collection<? extends Serializable> idList);
// 根据 entity 条件，查询全部记录
List<T> selectList(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
// 查询（根据 columnMap 条件）
List<T> selectByMap(@Param(Constants.COLUMN_MAP) Map<String, Object> columnMap);
// 根据 Wrapper 条件，查询全部记录
List<Map<String, Object>> selectMaps(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
// 根据 Wrapper 条件，查询全部记录。注意： 只返回第一个字段的值
List<Object> selectObjs(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

// 根据 entity 条件，查询全部记录（并翻页）
IPage<T> selectPage(IPage<T> page, @Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
// 根据 Wrapper 条件，查询全部记录（并翻页）
IPage<Map<String, Object>> selectMapsPage(IPage<T> page, @Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
// 根据 Wrapper 条件，查询总记录数
Integer selectCount(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
~~~



# Wrapper

条件构造器提供crud时的condition条件

继承关系

* AbstractWrapper
    * QueryWrapper
    * UpdateWrapper
    * AbstractLambdaWrapper
        * LambdaQueryWrapper
        * LambdaUpdateWrapper

## AbstractWrapper

当condition 值为true时，该条件才会加入最后是生成的sql中

### allEq

~~~java
allEq(Map<R, V> params)
allEq(Map<R, V> params, boolean null2IsNull)
allEq(boolean condition, Map<R, V> params, boolean null2IsNull)
~~~

* `params` : `key`为数据库字段名,`value`为字段值
* `null2IsNull` : 为`true`则在`map`的`value`为`null`时调用isNull方法,为`false`时则忽略`value`为`null`的

~~~java
allEq(BiPredicate<R, V> filter, Map<R, V> params)
allEq(BiPredicate<R, V> filter, Map<R, V> params, boolean null2IsNull)
allEq(boolean condition, BiPredicate<R, V> filter, Map<R, V> params, boolean null2IsNull) 
~~~

### 常用条件

~~~java
//eq 等于
eq([boolean condition,] R column, Object val);
//ne 不等于
ne([boolean condition,] R column, Object val);
//gt 大于
gt([boolean condition,] R column, Object val);
//ge 大于等于
ge([boolean condition,] R column, Object val);
//lt 小于
lt([boolean condition,] R column, Object val);
//le 小于等于
le([boolean condition,] R column, Object val);
//between 在两值之间
between([boolean condition,] R column, Object val1, Object val2);
//notBetween 不在两值之间
notBetween([boolean condition,] R column, Object val1, Object val2);
//like 模糊查询 %值%
like([boolean condition,] R column, Object val);
//notLike 
notLike([boolean condition,] R column, Object val);
//isNull 字段是null
isNull([boolean condition,] R column);
//isNotNull 字段不是null
isNotNull([boolean condition,] R column);
//in 字段在执行的值中
in([boolean condition,] R column, Collection<?> value);
in([boolean condition,] R column, Object... values);
//groupBy 分组
groupBy([boolean condition,] R... columns);
//orderBy 排序
orderByAsc([boolean condition,] R... columns);
orderByDesc([boolean condition,] R... columns);
orderBy([boolean condition,] boolean isAsc, R... columns)
~~~

### 逻辑运算

~~~java
//或者
or(boolean condition);
or(boolean condition, Consumer<Param> consumer);
//并且
and(boolean condition, Consumer<Param> consumer);
~~~

## QueryWrapper

**Select**

~~~java
select(String... sqlSelect)
select(Predicate<TableFieldInfo> predicate)
select(Class<T> entityClass, Predicate<TableFieldInfo> predicate)
~~~

## UpdateWrapper

**Set**

~~~java
set(String column, Object val)
set(boolean condition, String column, Object val)
~~~

