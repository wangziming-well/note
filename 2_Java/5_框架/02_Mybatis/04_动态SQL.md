# 动态SQL

动态 SQL 是 MyBatis 的强大特性之一。提供了动态拼接SQL语句的功能，相关标签：

- if
- choose (when, otherwise)
- trim (where, set)
- foreach

# if

属性：

* test：条件，条件为真，则拼接if中的SQL语句，否则不拼接

示例：

~~~xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
  <if test="author != null and author.name != null">
    AND author_name like #{author.name}
  </if>
</select>
~~~

# choose

有时候，我们不想使用所有的条件，而只是想从多个条件中选择一个使用。针对这种情况，MyBatis 提供了 choose 元素，它有点像 Java 中的 switch 语句。示例：

~~~xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <choose>
    <when test="title != null">
      AND title like #{title}
    </when>
    <when test="author != null and author.name != null">
      AND author_name like #{author.name}
    </when>
    <otherwise>
      AND featured = 1
    </otherwise>
  </choose>
</select>
~~~

# trim

在使用`if`判断条件子句时，可能会遇到下面这种情况：

~~~xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG
  WHERE
  <if test="state != null">
    state = #{state}
  </if>
  <if test="title != null">
    AND title like #{title}
  </if>
  <if test="author != null and author.name != null">
    AND author_name like #{author.name}
  </if>
</select>
~~~

对于这种语句，如果没有任何条件满足，最终SQL会是：

~~~sql
SELECT * FROM BLOG
WHERE
~~~

如果匹配的只是第二个条件，最终SQL会是：

~~~sql
SELECT * FROM BLOG
WHERE
AND title like ‘someTitle’
~~~

这样的SQL都是不合法的，会导致查询失败。

## where

为了解决这个问题，mybatis提供`<where>`标签，来代替sql中的`where`子句：

~~~xml
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG
  <where>
    <if test="state != null">
         state = #{state}
    </if>
    <if test="title != null">
        AND title like #{title}
    </if>
    <if test="author != null and author.name != null">
        AND author_name like #{author.name}
    </if>
  </where>
</select>
~~~

*where* 元素只会在子元素返回任何内容的情况下才插入 “WHERE” 子句。而且，若子句的开头为 “AND” 或 “OR”，*where* 元素也会将它们去除。

## set

同样的问题在`update`语句中也有，为此mybatis提供了`set`标签：

~~~xml
<update id="updateAuthorIfNecessary">
  update Author
    <set>
      <if test="username != null">username=#{username},</if>
      <if test="password != null">password=#{password},</if>
      <if test="email != null">email=#{email},</if>
      <if test="bio != null">bio=#{bio}</if>
    </set>
  where id=#{id}
</update>
~~~

这个例子中，*set* 元素会动态地在行首插入 SET 关键字，并会删掉额外的逗号（这些逗号是在使用条件语句给列赋值时引入的）。

## 原理

where和set标签实际上都是`trim`标签的一个特例：

`trim`标签能指定sql语句的前后缀并去掉指定的sql关键字或字符

属性：

* prefix：给sql语句拼装的前缀
* suffix：给sql语句拼装的后缀
* prefixOverrides：如果sql语句前面是prefixOverrides指定的字符，则去掉该字符
* suffixOverrids：如果sql语句后面是prefixOverrides指定的字符，则去掉该字符

示例:

~~~xml
<insert id="insert" parameterType="account">
        insert into account
        <trim prefix="(" suffix=")" suffixOverrides=",">
            <if test="money != null">
                money,
            </if>
            <if test="name != null and name != ''">
                name
            </if>
        </trim>
        <trim prefix="values(" suffix=")" suffixOverrides=",">
            <if test="money != null">
               #{money},
            </if>
            <if test="name != null and name != ''">
                #{name}
            </if>
        </trim>
</insert>
~~~

# forEach

动态 SQL 的另一个常见使用场景是对集合进行遍历（尤其是在构建 IN 条件语句的时候）。比如：

~~~xml
<select id="selectPostIn" resultType="domain.blog.Post">
  SELECT *
  FROM POST P
  <where>
    <foreach item="item" index="index" collection="list"
        open="ID in (" separator="," close=")" nullable="true">
          #{item}
    </foreach>
  </where>
</select>
~~~

属性：

* open ：指定开始的字符
* separator：指定分隔符
* close：指定结束字符
* collection：指定要遍历的集合类型
* item：指定遍历的元素名
* index：指定遍历下标的变量名

**提示** 你可以将任何可迭代对象（如 List、Set 等）、Map 对象或者数组对象作为集合参数传递给 *foreach*。当使用可迭代对象或者数组时，index 是当前迭代的序号，item 的值是本次迭代获取到的元素。当使用 Map 对象（或者 Map.Entry 对象的集合）时，index 是键，item 是值。

# script

script标签允许在注解Mapper中使用动态SQL，例如：

~~~java
@Update({"<script>",
  "update Author",
  "  <set>",
  "    <if test='username != null'>username=#{username},</if>",
  "    <if test='password != null'>password=#{password},</if>",
  "    <if test='email != null'>email=#{email},</if>",
  "    <if test='bio != null'>bio=#{bio}</if>",
  "  </set>",
  "where id=#{id}",
  "</script>"})
void updateAuthorValues(Author author);
~~~

# bind

`bind` 元素允许你在 OGNL 表达式以外创建一个变量，并将其绑定到当前的上下文。比如：

```xml
<select id="selectBlogsLike" resultType="Blog">
  <bind name="pattern" value="'%' + _parameter.getTitle() + '%'" />
  SELECT * FROM BLOG
  WHERE title LIKE #{pattern}
</select>
```

# 多数据库支持

如果配置了 `databaseIdProvider`，你就可以在动态代码中使用名为 “_databaseId” 的变量来为不同的数据库构建特定的语句。比如下面的例子：

```xml
<insert id="insert">
  <selectKey keyProperty="id" resultType="int" order="BEFORE">
    <if test="_databaseId == 'oracle'">
      select seq_users.nextval from dual
    </if>
    <if test="_databaseId == 'db2'">
      select nextval for seq_users from sysibm.sysdummy1"
    </if>
  </selectKey>
  insert into users values (#{id}, #{name})
</insert>
```
