

# XML Mapper

MyBatis 的真正强大在于它的语句映射，这是它的魔力所在。由于它的异常强大，映射器的 XML 文件就显得相对简单。如果拿它跟具有相同功能的 JDBC 代码进行对比，你会立即发现省掉了将近 95% 的代码。MyBatis 致力于减少使用成本，让用户能更专注于 SQL 代码。

SQL 映射文件只有很少的几个顶级元素（按照应被定义的顺序列出）：

- `cache` – 该命名空间的缓存配置。
- `cache-ref` – 引用其它命名空间的缓存配置。
- `resultMap` – 描述如何从数据库结果集中加载对象，是最复杂也是最强大的元素。
- `parameterMap` – 老式风格的参数映射。此元素已被废弃，并可能在将来被移除！请使用行内参数映射。文档中不会介绍此元素。
- `sql` – 可被其它语句引用的可重用语句块。
- `insert` – 映射插入语句。
- `update` – 映射更新语句。
- `delete` – 映射删除语句。
- `select` – 映射查询语句。

下一部分将从语句本身开始来描述每个元素的细节。

# select

一个简单查询的 select 元素是非常简单的。比如：

~~~xml
<select id="selectPerson" parameterType="int" resultTpye="hashmap">
    select * from person where ID=#{id}
</select>
~~~

这个语句名为 selectPerson，接受一个 int（或 Integer）类型的参数，并返回一个 HashMap 类型的对象，其中的键是列名，值便是结果行中的对应值。

参数符号`#{id}`表示这是一个预处理语句（PreparedStatement）参数，在 JDBC 中，这样的一个参数在 SQL 中会由一个“?”来标识，并被传递到一个新的预处理语句中，就像这样：

~~~java
// 近似的 JDBC 代码，非 MyBatis 代码...
String selectPerson = "SELECT * FROM PERSON WHERE ID=?";
PreparedStatement ps = conn.prepareStatement(selectPerson);
ps.setInt(1,id);
~~~

select 元素允许你配置很多属性来配置每条语句的行为细节。

~~~xml
<select
  id="selectPerson"
  parameterType="int"
  parameterMap="deprecated"
  resultType="hashmap"
  resultMap="personResultMap"
  flushCache="false"
  useCache="true"
  timeout="10"
  fetchSize="256"
  statementType="PREPARED"
  resultSetType="FORWARD_ONLY">
~~~



| 属性            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `id`            | 在命名空间中唯一的标识符，可以被用来引用这条语句。           |
| `parameterType` | 将会传入这条语句的参数的类全限定名或别名。这个属性是可选的，因为 MyBatis 可以根据语句中实际传入的参数计算出应该使用的类型处理器（TypeHandler），默认值为未设置（unset）。 |
| `resultType`    | 期望从这条语句中返回结果的类全限定名或别名。 注意，如果返回的是集合，那应该设置为集合包含的类型，而不是集合本身的类型。 resultType 和 resultMap 之间只能同时使用一个。 |
| `resultMap`     | 对外部 resultMap 的命名引用。结果映射是 MyBatis 最强大的特性，如果你对其理解透彻，许多复杂的映射问题都能迎刃而解。 resultType 和 resultMap 之间只能同时使用一个。 |
| `flushCache`    | 将其设置为 true 后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：false。 |
| `useCache`      | 将其设置为 true 后，将会导致本条语句的结果被二级缓存缓存起来，默认值：对 select 元素为 true。 |
| `timeout`       | 这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为未设置（unset）（依赖数据库驱动）。 |
| `fetchSize`     | 这是一个给驱动的建议值，尝试让驱动程序每次批量返回的结果行数等于这个设置值。 默认值为未设置（unset）（依赖驱动）。 |
| `statementType` | 可选 STATEMENT，PREPARED 或 CALLABLE。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。 |
| `resultSetType` | FORWARD_ONLY，SCROLL_SENSITIVE, SCROLL_INSENSITIVE 或 DEFAULT（等价于 unset） 中的一个，默认值为 unset （依赖数据库驱动）。 |
| `databaseId`    | 如果配置了数据库厂商标识（databaseIdProvider），MyBatis 会加载所有不带 databaseId 或匹配当前 databaseId 的语句；如果带和不带的语句都有，则不带的会被忽略。 |
| `resultOrdered` | 这个设置仅针对嵌套结果 select 语句：如果为 true，则假设结果集以正确顺序（排序后）执行映射，当返回新的主结果行时，将不再发生对以前结果行的引用。 这样可以减少内存消耗。默认值：`false`。 |
| `resultSets`    | 这个设置仅适用于多结果集的情况。它将列出语句执行后返回的结果集并赋予每个结果集一个名称，多个名称之间以逗号分隔。 |
| `affectData`    | Set this to true when writing a INSERT, UPDATE or DELETE statement that returns data so that the transaction is controlled properly. . Default: `false` (since 3.5.12) |

# insert、update、delete

数据变更语句 insert，update 和 delete 的实现非常接近：

```xml
<insert
  id="insertAuthor"
  parameterType="domain.blog.Author"
  flushCache="true"
  statementType="PREPARED"
  keyProperty=""
  keyColumn=""
  useGeneratedKeys=""
  timeout="20">

<update
  id="updateAuthor"
  parameterType="domain.blog.Author"
  flushCache="true"
  statementType="PREPARED"
  timeout="20">

<delete
  id="deleteAuthor"
  parameterType="domain.blog.Author"
  flushCache="true"
  statementType="PREPARED"
  timeout="20">
```

它们可用的属性如下：

| 属性               | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| `id`               | 在命名空间中唯一的标识符，可以被用来引用这条语句。           |
| `parameterType`    | 将会传入这条语句的参数的类全限定名或别名。这个属性是可选的，因为 MyBatis 可以根据语句中实际传入的参数计算出应该使用的类型处理器（TypeHandler），默认值为未设置（unset）。 |
| `parameterMap`     | 用于引用外部 parameterMap 的属性，目前已被废弃。请使用行内参数映射和 parameterType 属性。 |
| `flushCache`       | 将其设置为 true 后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：（对 insert、update 和 delete 语句）true。 |
| `timeout`          | 这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数。默认值为未设置（unset）（依赖数据库驱动）。 |
| `statementType`    | 可选 STATEMENT，PREPARED 或 CALLABLE。这会让 MyBatis 分别使用 Statement，PreparedStatement 或 CallableStatement，默认值：PREPARED。 |
| `useGeneratedKeys` | （仅适用于 insert 和 update）这会令 MyBatis 使用 JDBC 的 getGeneratedKeys 方法来取出由数据库内部生成的主键（比如：像 MySQL 和 SQL Server 这样的关系型数据库管理系统的自动递增字段），默认值：false。 |
| `keyProperty`      | （仅适用于 insert 和 update）指定能够唯一识别对象的属性，MyBatis 会使用 getGeneratedKeys 的返回值或 insert 语句的 selectKey 子元素设置它的值，默认值：未设置（`unset`）。如果生成列不止一个，可以用逗号分隔多个属性名称。 |
| `keyColumn`        | （仅适用于 insert 和 update）设置生成键值在表中的列名，在某些数据库（像 PostgreSQL）中，当主键列不是表中的第一列的时候，是必须设置的。如果生成列不止一个，可以用逗号分隔多个属性名称。 |
| `databaseId`       | 如果配置了数据库厂商标识（databaseIdProvider），MyBatis 会加载所有不带 databaseId 或匹配当前 databaseId 的语句；如果带和不带的语句都有，则不带的会被忽略。 |

下面是 insert，update 和 delete 语句的示例：

```xml
<insert id="insertAuthor">
  insert into Author (id,username,password,email,bio)
  values (#{id},#{username},#{password},#{email},#{bio})
</insert>

<update id="updateAuthor">
  update Author set
    username = #{username},
    password = #{password},
    email = #{email},
    bio = #{bio}
  where id = #{id}
</update>

<delete id="deleteAuthor">
  delete from Author where id = #{id}
</delete>
```

插入语句里面有一些额外的属性和子元素用来处理主键的生成,如果你的数据库支持自动生成主键的字段（比如 MySQL 和 SQL Server），那么你可以设置 useGeneratedKeys=”true”，然后再把 keyProperty 设置为目标属性,这样插入语句执行结束后，生成的主键将被设置到入参对象keyProperty 指定的属性中。

~~~xml
<insert id="insertAuthor" useGeneratedKeys="true"
    keyProperty="id">
  insert into Author (username,password,email,bio)
  values (#{username},#{password},#{email},#{bio})
</insert>
~~~

# sql标签

这个元素可以用来定义可重用的 SQL 代码片段，以便在其它语句中使用。 参数可以静态地（在加载的时候）确定下来，并且可以在不同的 include 元素中定义不同的参数值。比如：

~~~xml
<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
~~~

这个 SQL 片段可以在其它语句中使用，例如：

```xml
<select id="selectUsers" resultType="map">
  select
    <include refid="userColumns"><property name="alias" value="t1"/></include>,
    <include refid="userColumns"><property name="alias" value="t2"/></include>
  from some_table t1
    cross join some_table t2
</select>
```

也可以在 include 元素的 refid 属性或内部语句中使用属性值，例如：

```xml
<sql id="sometable">
  ${prefix}Table
</sql>

<sql id="someinclude">
  from
    <include refid="${include_target}"/>
</sql>

<select id="select" resultType="map">
  select
    field1, field2, field3
  <include refid="someinclude">
    <property name="prefix" value="Some"/>
    <property name="include_target" value="sometable"/>
  </include>
</select>
```

# 参数传递

可以使用`#{xxx}`占位符向sql语句传递参数值，例如：

~~~xml
<select id="selectUsers" resultType="User">
  select id, username, password
  from users
  where id = #{id}
</select>
~~~

在上例这个简单的命名参数映射中，参数类型（parameterType）会被自动设置为 `int`，这个参数可以随意命名。

如果传入一个复杂的对象，情况就会不一样，例如：

~~~xml
<insert id="insertUser" parameterType="User">
  insert into users (id, username, password)
  values (#{id}, #{username}, #{password})
</insert>
~~~

mabatis会查找传入的User 类型的参数对象的 id、username 和 password 属性，然后将它们的值传入预处理语句的参数中。

在参数占位符中可以指定以下属性：

* `javaType,jdbcType`:和 MyBatis 的其它部分一样，参数也可以指定一个特殊的数据类型：

  ~~~xml
  #{property,javaType=int,jdbcType=NUMERIC}
  ~~~

  和 MyBatis 的其它部分一样， 根据参数对象的类型就可以确定 javaType，所以一般不需要指定javaType。除非传入的参数是一个`HashMap`,此时需要显式指定 `javaType` 来确保正确的类型处理器（`TypeHandler`）被使用

  **提示** JDBC 要求，如果一个列允许使用 null 值，并且会使用值为 null 的参数，就必须要指定 JDBC 类型（jdbcType）。

* `typeHandler`要更进一步地自定义类型处理方式，可以指定一个特殊的类型处理器类（或别名），比如：

  ```xml
  #{age,javaType=int,jdbcType=NUMERIC,typeHandler=MyTypeHandler}
  ```

* `numericScale` 属性可以设置 数值类型小数点后保留的位数。

  ~~~xml
  #{height,javaType=double,jdbcType=NUMERIC,numericScale=2}
  ~~~

## 字符串替换

默认情况下，使用 `#{}` 参数语法时，MyBatis 会创建 `PreparedStatement` 参数占位符，并通过占位符安全地设置参数（就像使用 `? `一样）。 这样做更安全，更迅速，通常也是首选做法，不过有时你就是想直接在 SQL 语句中直接插入一个不转义的字符串。 比如 `ORDER BY `子句，这时候你可以：

```
ORDER BY ${columnName}
```

这样，MyBatis 就不会修改或转义该字符串了。

当 SQL 语句中的元数据（如表名或列名）是动态生成的时候，字符串替换将会非常有用。 举个例子，如果你想 `select` 一个表任意一列的数据时，不需要这样写：

~~~java
@Select("select * from user where id = #{id}")
User findById(@Param("id") long id);

@Select("select * from user where name = #{name}")
User findByName(@Param("name") String name);

@Select("select * from user where email = #{email}")
User findByEmail(@Param("email") String email);
~~~

而是可以只写这样一个方法：

```java
@Select("select * from user where ${column} = #{value}")
User findByColumn(@Param("column") String column, @Param("value") String value);
```

其中 `${column}` 会被直接替换，而 `#{value}` 会使用 `?` 预处理。

这种方式也同样适用于替换表名的情况。

**提示** 用这种方式接受用户的输入，并用作语句参数是不安全的，会导致潜在的 SQL 注入攻击。因此，要么不允许用户输入这些字段，要么自行转义并检验这些参数。

# 结果映射初步

`resultMap` 元素是 MyBatis 中最重要最强大的元素。ResultMap 的设计思想是，对简单的语句做到零配置，对于复杂一点的语句，只需要描述语句之间的关系就行了。

## 自动映射

可以直接使用`resultType="map"`将所有的列映射到 `HashMap` 的键上:

~~~xml
<select id="selectUsers" resultType="map">
  select id, username, hashedPassword
  from some_table
  where id = #{id}
</select>
~~~

但是 `HashMap `并不是一个很好的领域模型。一般会使用 JavaBean 或 POJO（Plain Old Java Objects，普通老式 Java 对象）作为领域模型。MyBatis 对两者都提供了支持。

对于下面这个 JavaBean：

~~~java
package com.someapp.model;

@Getter
@Setter
public class User {
  private int id;
  private String username;
  private String hashedPassword;
}
~~~

如果列名与这个JavaBean属性名一致，Mybatis会 隐式自动创建一个 `ResultMap`，再根据属性名来自动映射列到 JavaBean 的属性上：

~~~xml
<select id="selectUsers" resultType="com.someapp.model.User">
  select id, username, hashedPassword
  from some_table
  where id = #{id}
</select>
~~~

如果列名和属性名不能匹配上，可以在 SELECT 语句中设置列别名（这是一个基本的 SQL 特性）来完成匹配。比如：

```xml
<select id="selectUsers" resultType="com.someapp.model.User">
  select
    user_id             as "id",
    user_name           as "userName",
    hashed_password     as "hashedPassword"
  from some_table
  where id = #{id}
</select>
```

## 简单结果映射

通过上面例子，我们看到在列名或者列别名和Java对象属性一致时，完全可以不用显式地配置`resultMap`，Mabatis会帮我们自动配置。

当然我们可以显示得定义一个`resultMap`，来解决列名和对象属性不匹配的问题：

~~~xml
<resultMap id="userResultMap" type="User">
  <id property="id" column="user_id" />
  <result property="username" column="user_name"/>
  <result property="password" column="hashed_password"/>
</resultMap>
~~~

然后在引用它的语句中设置 `resultMap` 属性就行了（注意此时不需要 `resultType` 属性）。比如:

~~~xml
<select id="selectUsers" resultMap="userResultMap">
  select user_id, user_name, hashed_password
  from some_table
  where id = #{id}
</select>
~~~



# resultMap详解

以上是`resultMap`标签的简单应用，它还支持更多其他子标签和属性，以提供更复杂的映射关系。

其支持的子标签有：

* `constructor `- 用于在实例化类时，注入结果到构造方法中
  * `idArg `- ID 参数；标记出作为 ID 的结果可以帮助提高整体性能
  * `arg `- 将被注入到构造方法的一个普通结果
*  `id `– 一个 ID 结果；标记出作为 ID 的结果可以帮助提高整体性能
*  `result `– 注入到字段或 JavaBean 属性的普通结果
*  `association `– 一个复杂类型的关联；许多结果将包装成这种类型
  * 嵌套结果映射 – 关联可以是 resultMap 元素，或是对其它结果映射的引用
*  `collection `– 一个复杂类型的集合
  * 嵌套结果映射 – 集合可以是 resultMap 元素，或是对其它结果映射的引用
*  `discriminator `– 使用结果值来决定使用哪个 resultMap
  * `case `– 基于某些值的结果映射
    * 嵌套结果映射 – case 也是一个结果映射，因此具有相同的结构和元素；或者引用其它的结果映射

ResultMap 的支持的属性：

| 属性          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `id`          | 当前命名空间中的一个唯一标识，用于标识一个结果映射。         |
| `type`        | 类的完全限定名, 或者一个类型别名（关于内置的类型别名，可以参考上面的表格）。 |
| `autoMapping` | 如果设置这个属性，MyBatis 将会为本结果映射开启或者关闭自动映射。 这个属性会覆盖全局的属性 autoMappingBehavior。默认值：未设置（unset）。 |

## 示例模型

为了详细介绍`resultMap`的功能，我们以一个数据库模型实例为例，其schema如下：

~~~mysql
create table author( # 作者
    id       int primary key auto_increment,
    username varchar(100) not null, # 用户名
    password varchar(100) not null, # 密码
    email    varchar(100)           # 邮箱
);
create table blog( # 博客
    id           int primary key auto_increment,
    title        varchar(100) not null, # 标题
    author_id    int          not null, # 所属作者
    co_author_id int                    # 共同作者
);
create table post( #文章
    id int primary key  auto_increment,
    blog_id int, # 所属blog
    author_id int, # 所属作者
    created_on datetime, # 创作时间
    subject varchar(100), #主题
    body varchar(1000) # 文章内容
);
create table  tag( # 标签
    id int primary key  auto_increment,
    name varchar(100) not null  # 标签名
);
drop table if exists post_tag;
create table  post_tag( # 文章_标签关系表
    post_id int not null ,  # 文章id
    tag_id int not null     # 标签id
);
~~~

一个博客只有一个作者，但一个博客又很多文章，一个文章可以又多个标签。

为其添加一些数据：

~~~mysql
insert into author (id,username,password,email) values (1,'wangziming','123456','441678514@qq.com');
insert into author (id,username,password,email) values (2,'wangmengbing','456789','wangmengbing@gmail.com');

insert into blog (id,title,author_id,co_author_id) values (1,'王梓铭个人博客',1,2);

insert into post (id,blog_id,author_id,created_on,subject,body) values (1,1,1,now(),'Spring的AOP支持','Spring AOP是用纯Java实现的。不需要特殊的编译过程。Spring AOP不需要控制类加载器的层次结构，因此适合在Servlet容器或应用服务器中使用。......');
insert into post (id,blog_id,author_id,created_on,subject,body) values (2,1,1,now(),'基于注解的Controller','Spring MVC提供了一个基于注解的编程模型，通过`@Controller`、`@RequestMapping`注解来表达请求映射、请求输入、异常处理等内容。......');

insert into tag (id, name) values (1,'spring');
insert into tag (id, name) values (2,'java');
insert into tag (id, name) values (3,'aop');
insert into tag (id, name) values (4,'controller');

insert into post_tag (post_id, tag_id) VALUES (1,1);
insert into post_tag (post_id, tag_id) VALUES (1,2);
insert into post_tag (post_id, tag_id) VALUES (1,3);
insert into post_tag (post_id, tag_id) VALUES (2,1);
insert into post_tag (post_id, tag_id) VALUES (2,2);
insert into post_tag (post_id, tag_id) VALUES (2,4);
~~~



其相对应的领域模型(domain model)对象为：

~~~java
@Data
public class Author {
    private Integer id ;
    private String username;
    private String password;
    private String email;
}
@Data
public class Blog {
    private Integer id;
    private String title;c
    private Author author;
    private Author coAuthor;
    private List<Post> posts;
}
@Data
public class Post {
    private Integer id;
    private Integer blogId;
    private Integer authorId;
    private LocalDateTime createOn;
    private String subject;
    private String body;
    private List<Tag> tags;
}
@Data
public class Tag {
    private Integer id;
    private String name;
}
~~~



## id&result

~~~xml
<id property="id" column="post_id"/>
<result property="subject" column="post_subject"/>
~~~

这些元素是结果映射的基础。*id* 和 *result* 元素都将一个列的值映射到一个简单数据类型（String, int, double, Date 等）的属性或字段。

这两者之间的唯一不同是，***id* 元素对应的属性会被标记为对象的标识符，在比较对象实例时使用。 这样可以提高整体的性能，尤其是进行缓存和嵌套结果映射（也就是连接映射）的时候**。

两个元素都有下面属性：

| 属性          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `property`    | 映射到列结果的字段或属性。如果 JavaBean 有这个名字的属性（property），会先使用该属性。否则 MyBatis 将会寻找给定名称的字段（field）。 无论是哪一种情形，你都可以使用常见的点式分隔形式进行复杂属性导航。 比如，你可以这样映射一些简单的东西：“username”，或者映射到一些复杂的东西上：“address.street.number”。 |
| `column`      | 数据库中的列名，或者是列的别名。一般情况下，这和传递给 `resultSet.getString(columnName)` 方法的参数一样。 |
| `javaType`    | 一个 Java 类的全限定名，或一个类型别名（关于内置的类型别名，可以参考上面的表格）。 如果你映射到一个 JavaBean，MyBatis 通常可以推断类型。然而，如果你映射到的是 HashMap，那么你应该明确地指定 javaType 来保证行为与期望的相一致。 |
| `jdbcType`    | JDBC 类型，所支持的 JDBC 类型参见下表。 只需要在可能执行插入、更新和删除的且允许空值的列上指定 JDBC 类型。这是 JDBC 的要求而非 MyBatis 的要求。如果你直接面向 JDBC 编程，你需要对可以为空值的列指定这个类型。 |
| `typeHandler` | 我们在前面讨论过默认的类型处理器。使用这个属性，你可以覆盖默认的类型处理器。 这个属性值是一个类型处理器实现类的全限定名，或者是类型别名。 |

MyBatis 通过内置的 jdbcType 枚举类型支持下面的 JDBC 类型。

| `BIT`      | `FLOAT`   | `CHAR`        | `TIMESTAMP`     | `OTHER`   | `UNDEFINED` |
| ---------- | --------- | ------------- | --------------- | --------- | ----------- |
| `TINYINT`  | `REAL`    | `VARCHAR`     | `BINARY`        | `BLOB`    | `NVARCHAR`  |
| `SMALLINT` | `DOUBLE`  | `LONGVARCHAR` | `VARBINARY`     | `CLOB`    | `NCHAR`     |
| `INTEGER`  | `NUMERIC` | `DATE`        | `LONGVARBINARY` | `BOOLEAN` | `NCLOB`     |
| `BIGINT`   | `DECIMAL` | `TIME`        | `NULL`          | `CURSOR`  | `ARRAY`     |

## constructor

通过`setter`方法注入对象属性能够满足大部分领域模型的要求。但某些情况下可能需要映射到不可变对象。此时可以使用`constructor`标签进行构造器注入

例如对于下面构造方法:

```java
public class User {
   //...
   public User(Integer id, String username, int age) {
     //...
  }
//...
}
```

为了将结果注入构造方法，MyBatis 需要通过某种方式定位相应的构造方法。 在下面的例子中，MyBatis 搜索一个声明了三个形参的构造方法，参数类型以 `java.lang.Integer`, `java.lang.String` 和 `int` 的顺序给出：

~~~xml
<constructor>
   <idArg column="id" javaType="int"/>
   <arg column="username" javaType="String"/>
   <arg column="age" javaType="_int"/>
</constructor>
~~~

注意`constructor`标签中定义的`arg`顺序必须和Java构造方法入参的顺序对应。

这样当在处理一个带有多个形参的构造方法时，很容易搞乱 arg 元素的顺序。 所以可以用`name`属性指定参数名称的前提下，以任意顺序编写 arg 元素。 

 为了通过名称来引用构造方法参数，你可以添加 `@Param` 注解，或者使用 '-parameters' 编译选项并启用 `useActualParamName` 选项（默认开启）来编译项目。下面是一个等价的例子，尽管函数签名中第二和第三个形参的顺序与 constructor 元素中参数声明的顺序不匹配：

~~~xml
<constructor>
   <idArg column="id" javaType="int" name="id" />
   <arg column="age" javaType="_int" name="age" />
   <arg column="username" javaType="String" name="username" />
</constructor>
~~~

如果存在名称和类型相同的可写属性，那么可以省略 `javaType` 。此时对应的Bean对象构造器如下：

~~~java
public class User {
   //...
  public User(@Param("id") Interger id,
              @Param("username") String username,
              @Param("age") int password) {
     //...
  }
//...
}
~~~

`<idArg>`和`<arg>`的其他属性与`<id>`和`<result>`是一样的：

| 属性          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `column`      | 数据库中的列名，或者是列的别名。一般情况下，这和传递给 `resultSet.getString(columnName)` 方法的参数一样。 |
| `javaType`    | 一个 Java 类的完全限定名，或一个类型别名（关于内置的类型别名，可以参考上面的表格）。 如果你映射到一个 JavaBean，MyBatis 通常可以推断类型。然而，如果你映射到的是 HashMap，那么你应该明确地指定 javaType 来保证行为与期望的相一致。 |
| `jdbcType`    | JDBC 类型，所支持的 JDBC 类型参见这个表格之前的“支持的 JDBC 类型”。 只需要在可能执行插入、更新和删除的且允许空值的列上指定 JDBC 类型。这是 JDBC 的要求而非 MyBatis 的要求。如果你直接面向 JDBC 编程，你需要对可能存在空值的列指定这个类型。 |
| `typeHandler` | 我们在前面讨论过默认的类型处理器。使用这个属性，你可以覆盖默认的类型处理器。 这个属性值是一个类型处理器实现类的完全限定名，或者是类型别名。 |
| `select`      | 用于加载复杂类型属性的映射语句的 ID，它会从 column 属性中指定的列检索数据，作为参数传递给此 select 语句。具体请参考关联元素。 |
| `resultMap`   | 结果映射的 ID，可以将嵌套的结果集映射到一个合适的对象树中。 它可以作为使用额外 select 语句的替代方案。它可以将多表连接操作的结果映射成一个单一的 `ResultSet`。这样的 `ResultSet` 将会将包含重复或部分数据重复的结果集。为了将结果集正确地映射到嵌套的对象树中，MyBatis 允许你 “串联”结果映射，以便解决嵌套结果集的问题。想了解更多内容，请参考下面的关联元素。 |
| `name`        | 构造方法形参的名字。通过指定具体的参数名，你可以以任意顺序写入 arg 元素。参看上面的解释。 |

## association

`<association>`用于处理嵌套对象。即映射的JavaBean属性不是简单类型，而是又一个复杂对象的情况。

关联结果映射和其它类型的映射工作方式差不多。 你需要指定目标属性名以及属性的`javaType`（很多时候 MyBatis 可以自己推断出来），在必要的情况下你还可以设置 JDBC 类型，如果你想覆盖获取结果值的过程，还可以设置类型处理器。

关联的不同之处是，你需要告诉 MyBatis 如何加载关联。MyBatis 有两种不同的方式加载关联：

- 嵌套 Select 查询：通过执行另外一个 SQL 映射语句来加载期望的复杂类型。
- 嵌套结果映射：使用嵌套的结果映射来处理连接结果的重复子集。

`<association>`和其他`resultMap`子标签一样支持下面属性:

| 属性          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `property`    | 映射到列结果的字段或属性。如果用来匹配的 JavaBean 存在给定名字的属性，那么它将会被使用。否则 MyBatis 将会寻找给定名称的字段。 无论是哪一种情形，你都可以使用通常的点式分隔形式进行复杂属性导航。 比如，你可以这样映射一些简单的东西：“username”，或者映射到一些复杂的东西上：“address.street.number”。 |
| `javaType`    | 一个 Java 类的完全限定名，或一个类型别名（关于内置的类型别名，可以参考上面的表格）。 如果你映射到一个 JavaBean，MyBatis 通常可以推断类型。然而，如果你映射到的是 HashMap，那么你应该明确地指定 javaType 来保证行为与期望的相一致。 |
| `jdbcType`    | JDBC 类型，所支持的 JDBC 类型参见这个表格之前的“支持的 JDBC 类型”。 只需要在可能执行插入、更新和删除的且允许空值的列上指定 JDBC 类型。这是 JDBC 的要求而非 MyBatis 的要求。如果你直接面向 JDBC 编程，你需要对可能存在空值的列指定这个类型。 |
| `typeHandler` | 我们在前面讨论过默认的类型处理器。使用这个属性，你可以覆盖默认的类型处理器。 这个属性值是一个类型处理器实现类的完全限定名，或者是类型别名。 |

除此之外，它还有与嵌套select查询和嵌套结果映射相关的属性，接下来会一一介绍：

### 嵌套 Select 查询

嵌套Select查询由下面属性支持:

| 属性        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `column`    | 数据库中的列名，或者是列的别名。一般情况下，这和传递给 `resultSet.getString(columnName)` 方法的参数一样。 注意：在使用复合主键的时候，你可以使用 `column="{prop1=col1,prop2=col2}"` 这样的语法来指定多个传递给嵌套 Select 查询语句的列名。这会使得 `prop1` 和 `prop2` 作为参数对象，被设置为对应嵌套 Select 语句的参数。 |
| `select`    | 用于加载复杂类型属性的映射语句的 ID，它会从 column 属性指定的列中检索数据，作为参数传递给目标 select 语句。 具体请参考下面的例子。注意：在使用复合主键的时候，你可以使用 `column="{prop1=col1,prop2=col2}"` 这样的语法来指定多个传递给嵌套 Select 查询语句的列名。这会使得 `prop1` 和 `prop2` 作为参数对象，被设置为对应嵌套 Select 语句的参数。 |
| `fetchType` | 可选的。有效值为 `lazy` 和 `eager`。 指定属性后，将在映射中忽略全局配置参数 `lazyLoadingEnabled`，使用属性的值。 |

一个示例：

~~~xml
<resultMap id="blogResult" type="Blog">
  <association property="author" column="author_id" javaType="Author" select="selectAuthor"/>
</resultMap>

<select id="selectBlog" resultMap="blogResult">
  SELECT * FROM BLOG WHERE ID = #{id}
</select>

<select id="selectAuthor" resultType="Author">
  SELECT * FROM AUTHOR WHERE ID = #{id}
</select>
~~~

在上述示例的`blogResult`结果映射中，我们利用`association`标签，将`selectAuthor`对应的查询的结果映射到`Blog`的`author`属性指定的`Author`对象中。

这种方式虽然很简单，但在大型数据集或大型数据表上表现不佳。这个问题被称为“N+1 查询问题”。 概括地讲，N+1 查询问题是这样子的：

- 你执行了一个单独的 SQL 语句来获取结果的一个列表（就是“+1”）。
- 对列表返回的每条记录，你执行一个 select 查询语句来为每条记录加载详细信息（就是“N”）。

这会导致执行大量SQL语句， 所以还有另一种方法

### 嵌套结果映射

嵌套结果映射由下面属性支持：

| 属性            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `resultMap`     | 结果映射的 ID，可以将此关联的嵌套结果集映射到一个合适的对象树中。 它可以作为使用额外 select 语句的替代方案。它可以将多表连接操作的结果映射成一个单一的 `ResultSet`。这样的 `ResultSet` 有部分数据是重复的。 为了将结果集正确地映射到嵌套的对象树中, MyBatis 允许你“串联”结果映射，以便解决嵌套结果集的问题。使用嵌套结果映射的一个例子在表格以后。 |
| `columnPrefix`  | 当连接多个表时，你可能会不得不使用列别名来避免在 `ResultSet` 中产生重复的列名。指定 columnPrefix 列名前缀允许你将带有这些前缀的列映射到一个外部的结果映射中。 详细说明请参考后面的例子。 |
| `notNullColumn` | 默认情况下，在至少一个被映射到属性的列不为空时，子对象才会被创建。 你可以在这个属性上指定非空的列来改变默认行为，指定后，Mybatis 将只在这些列中任意一列非空时才创建一个子对象。可以使用逗号分隔来指定多个列。默认值：未设置（unset）。 |
| `autoMapping`   | 如果设置这个属性，MyBatis 将会为本结果映射开启或者关闭自动映射。 这个属性会覆盖全局的属性 autoMappingBehavior。注意，本属性对外部的结果映射无效，所以不能搭配 `select` 或 `resultMap` 元素使用。默认值：未设置（unset）。 |

下面例子演示嵌套结果映射：

~~~xml
<select id="selectBlog" resultMap="blogResult">
    select
        B.id            as blog_id,
        B.title         as blog_title,
        B.author_id     as blog_author_id,
        A.id            as author_id,
        A.username      as author_username,
        A.password      as author_password,
        A.email         as author_email
    from Blog B , Author A 
    where B.author_id = A.id
    and B.id = #{id}
</select>
~~~

这里为了确保结果能够拥有唯一且清晰的名字，我们设置的别名，这个sql对应的结果映射为：

~~~xml
<resultMap id="blogResult" type="com.wzm.domain.Blog">
    <id property="id" column="blog_id" />
    <result property="title" column="blog_title"/>
    <association property="author" column="blog_author_id" javaType="com.wzm.domain.Author" resultMap="authorResult"/>
</resultMap>

<resultMap id="authorResult" type="com.wzm.domain.Author">
    <id property="id" column="author_id"/>
    <result property="username" column="author_username"/>
    <result property="password" column="author_password"/>
    <result property="email" column="author_email"/>
</resultMap>
~~~

可以看到，在这个示例中，我们通过关联查询获取嵌套对象`Author`的属性值，然后通过`authorResult`将这些属性值映射到子属性`Author`中。

之前已经提过，这里需要在此强调：id 标签在嵌套结果映射中扮演着非常重要的角色。一定要指定一个或多个可以唯一标识结果的属性。 如果不指定，会产生严重的性能问题。 id需要只需要指定可以唯一标识结果的最少属性。可以选择主键（复合主键）。

上面的示例使用了外部的结果映射元素来映射关联。这使得 Author 的结果映射可以被重用。如果不需要重用，或者希望所有映射关系都呈现在一个`resultMap`中，可以直接将结果映射作为子元素嵌套在`<association>`中：

~~~xml
<resultMap id="blogResult" type="com.wzm.domain.Blog">
    <id property="id" column="blog_id" />
    <result property="title" column="blog_title"/>
    <association property="author" column="blog_author_id" javaType="com.wzm.domain.Author">
        <id property="id" column="author_id"/>
        <result property="username" column="author_username"/>
        <result property="password" column="author_password"/>
        <result property="email" column="author_email"/>
    </association>
</resultMap>
~~~

如果考虑博客（blog）会有一个共同作者（co-author）,那么对应的SQL如下：

~~~xml
<select id="selectBlog" resultMap="blogResult">
    select
        B.id            as blog_id,
        B.title         as blog_title,
        A.id            as author_id,
        A.username      as author_username,
        A.password      as author_password,
        A.email         as author_email,
        CA.id           as co_author_id,
        CA.username     as co_author_username,
        CA.password     as co_author_password,
        CA.email        as co_author_email
    from Blog B,Author A, Author CA
    where B.author_id = A.id
    and B.co_author_id = CA.id
    and B.id = #{id}
</select>
~~~

 我们已经定义了`authorResult`的结果映射：

~~~xml
<resultMap id="authorResult" type="com.wzm.domain.Author">
    <id property="id" column="author_id"/>
    <result property="username" column="author_username"/>
    <result property="password" column="author_password"/>
    <result property="email" column="author_email"/>
</resultMap>
~~~

此时可以指定 `columnPrefix` 以便重复使用这个结果映射：

~~~xml
<resultMap id="blogResult" type="com.wzm.domain.Blog">
    <id property="id" column="blog_id" />
    <result property="title" column="blog_title"/>
    <association property="author" resultMap="authorResult" />
    <association property="coAuthor" resultMap="authorResult" columnPrefix="co_" />
</resultMap>
~~~

## collection

集合元素和关联元素几乎是一样的，例如，对于下面SQL：

~~~xml
<select id="selectBlog" resultMap="blogResult">
    select
        B.id            as blog_id,
        B.title         as blog_title,
        P.id            as post_id,
        P.subject       as post_subject,
        P.body          as post_body
    from Blog B,Post P
    where P.blog_id = B.id
    and B.id = #{id}
</select>
~~~

可以用下面resultMap来映射：

~~~xml
<resultMap id="blogResult" type="com.wzm.domain.Blog">
    <id property="id" column="blog_id" />
    <result property="title" column="blog_title"/>
    <collection property="posts" ofType="com.wzm.domain.Post">
        <id property="id" column="post_id"/>
        <result property="subject" column="post_subject"/>
        <result property="body" column="post_body"/>
    </collection>
</resultMap>
~~~

接下来我们主要介绍`collection`与`association`的不同之处

### 嵌套 Select 查询

使用嵌套Select查询来关联集合的示例：

~~~xml
<resultMap id="blogResult" type="com.wzm.domain.Blog">
    <collection property="posts" javaType="ArrayList" column="id" ofType="com.wzm.domain.Post" select="selectPostsForBlog"/>
</resultMap>

<select id="selectBlog" resultMap="blogResult">
    SELECT * FROM BLOG WHERE ID = #{id}
</select>

<select id="selectPostsForBlog" resultType="com.wzm.domain.Post">
    SELECT * FROM POST WHERE BLOG_ID = #{id}
</select>
~~~

我们可以注意到`javaType`这里指定的是`ArrayList`，并且用`ofType`指定了这个`ArrayList`集合的泛型。即“posts 是一个存储 Post 的 ArrayList 集合”	

一般情况下，MyBatis 可以推断 javaType 属性，因此并不需要填写。所以很多时候你可以简略成：

~~~xml
<collection property="posts" column="id" ofType="Post" select="selectPostsForBlog"/>
~~~

### 嵌套结果映射

集合的嵌套结果映射也是一样，除了新增了`ofType`属性与`association`完全相同

对于下面SQL:

~~~xml
<select id="selectBlog" resultMap="blogResult">
    select
        B.id            as blog_id,
        B.title         as blog_title,
        P.id            as post_id,
        P.subject       as post_subject,
        P.body          as post_body
    from Blog B,Post P
    where P.blog_id = B.id
    and B.id = #{id}
</select>
~~~

对应的`resultMap`为：

~~~xml
<resultMap id="blogResult" type="com.wzm.domain.Blog">
    <id property="id" column="blog_id" />
    <result property="title" column="blog_title"/>
    <collection property="posts" ofType="com.wzm.domain.Post">
        <id property="id" column="post_id"/>
        <result property="subject" column="post_subject"/>
        <result property="body" column="post_body"/>
    </collection>
</resultMap>
~~~

## 自动映射

我们已经知道，简单的场景下，MyBatis 可以自动映射查询结果。我们来深入了解一下自动映射是怎样工作的。

通常数据库列使用大写字母组成的单词命名，单词间用下划线分隔；而 Java 属性一般遵循驼峰命名法约定。为了在这两种命名方式之间启用自动映射，需要将 `mapUnderscoreToCamelCase` 设置为 true。

在我们显示提供了结果映射`resultMap`后，自动映射也能工作。对于每个结果列，如果该列没有显式设置映射，那么将被自动映射。

在下面的例子中，*id* 和 *userName* 列将被自动映射，*hashed_password* 列将根据配置进行映射：

~~~xml
<resultMap id="userResultMap" type="User">
  <result property="password" column="hashed_password"/>
</resultMap>

<select id="selectUsers" resultMap="userResultMap">
  select
    user_id             as "id",
    user_name           as "userName",
    hashed_password
  from some_table
  where id = #{id}
</select>
~~~

可以用`autoMappingBehavior`控制自动映射行为：

- `NONE` - 禁用自动映射。仅对手动映射的属性进行映射。
- `PARTIAL` - 对除在内部定义了嵌套结果映射（也就是连接的属性）以外的属性进行映射
- `FULL` - 自动映射所有属性。

无论设置的自动映射等级是哪种，你都可以通过在结果映射上设置 `autoMapping` 属性来为指定的结果映射设置启用/禁用自动映射：

~~~xml
<resultMap id="userResultMap" type="User" autoMapping="false">
  <result property="password" column="hashed_password"/>
</resultMap>
~~~

# cache

MyBatis 内置了一个强大的事务性查询缓存机制，它可以非常方便地配置和定制。 

默认情况下，只启用了本地的会话缓存，它仅仅对一个会话中的数据进行缓存。 要启用全局的二级缓存，只需要在你的 SQL 映射文件中添加一行：

~~~xml
<cache/>
~~~

这个语句效果如下：

- XML Mapper文件中的所有 select 语句的结果将会被缓存。
- XML Mapper文件中的所有 insert、update 和 delete 语句会刷新缓存。
- 缓存会使用最近最少使用算法（LRU, Least Recently Used）算法来清除不需要的缓存。
- 缓存不会定时进行刷新（也就是说，没有刷新间隔）。
- 缓存会保存列表或对象（无论查询方法返回哪种）的 1024 个引用。
- 缓存会被视为读/写缓存，这意味着获取到的对象并不是共享的，可以安全地被调用者修改，而不干扰其他调用者或线程所做的潜在修改。

**提示** 缓存只作用于 cache 标签所在的映射文件中的语句。如果你混合使用 Java API 和 XML 映射文件，在共用接口中的语句将不会被默认缓存。你需要使用` @CacheNamespaceRef` 注解指定缓存作用域。

多个命名空间中共享相同的缓存配置和实例。要实现这种需求，你可以使用 cache-ref 元素来引用另一个缓存：

~~~xml
<cache-ref namespace="com.someone.application.data.SomeMapper"/>
~~~

可以通过属性来修改 cache行为，比如：

~~~xml
<cache
  eviction="FIFO"
  flushInterval="60000"
  size="512"
  readOnly="true"/>	
~~~

这个更高级的配置创建了一个 FIFO 缓存，每隔 60 秒刷新，最多可以存储结果对象或列表的 512 个引用，而且返回的对象被认为是只读的，因此对它们进行修改可能会在不同线程中的调用者产生冲突。这些属性值具体解释如下：

* `eviction`缓存清除策略：

  - `LRU` – 最近最少使用：移除最长时间不被使用的对象。

  - `FIFO` – 先进先出：按对象进入缓存的顺序来移除它们。

  - `SOFT` – 软引用：基于垃圾回收器状态和软引用规则移除对象。

  - `WEAK` – 弱引用：更积极地基于垃圾收集器状态和弱引用规则移除对象。

* `flushInterval`（刷新间隔）属性可以被设置为任意的正整数，设置的值应该是一个以毫秒为单位的合理时间量。 默认情况是不设置，也就是没有刷新间隔，缓存仅仅会在调用语句时刷新。

* `size `（引用数目）属性可以被设置为任意正整数，要注意欲缓存对象的大小和运行环境中可用的内存资源。默认值是 1024。

* `readOnly`（只读）属性可以被设置为 true 或 false。只读的缓存会给所有调用者返回缓存对象的相同实例。 因此这些对象不能被修改。这就提供了可观的性能提升。而可读写的缓存会（通过序列化）返回缓存对象的拷贝。 速度上会慢一些，但是更安全，因此默认值是 false。

## 自定义缓存

可以使用自定义缓存，来完全覆盖缓存行为：

~~~xml
<cache type="com.domain.something.MyCustomCache"/>
~~~

type 属性指定的类必须实现 org.apache.ibatis.cache.Cache 接口，且提供一个接受 String 参数作为 id 的构造器。 这个接口是 MyBatis 框架中许多复杂的接口之一，但是行为却非常简单：

~~~java
public interface Cache {
  String getId();
  int getSize();
  void putObject(Object key, Object value);
  Object getObject(Object key);
  boolean hasKey(Object key);
  Object removeObject(Object key);
  void clear();
}
~~~

可以通过`property `向自定义缓存传递属性：

~~~xml
<cache type="com.domain.something.MyCustomCache">
  <property name="cacheFile" value="/tmp/my-custom-cache.tmp"/>
</cache>
~~~

