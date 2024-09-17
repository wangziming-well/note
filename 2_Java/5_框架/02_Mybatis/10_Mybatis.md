# 动态SQL

动态SQL提供了动态拼接SQL语句的功能

## if

属性：

* test：条件，条件为真，则拼接if中的SQL语句，否则不拼接

示例：

~~~xml
<select id="findUser" resultType="user">
	select * from user
    where sex="male"
    <if test="likeName !=null and likeName !=''">
    	and name like #{likeName}
    </if>
</select>
~~~

## where

当有多个where条件时，使用if语句可能会导致最终的SQL语句多出and关键字

所以mybatis设置了where标签，它：

* 自动拼装where关键字
* 自动去掉第一个条件的and

示例：

~~~xml
<select id="findUser" resultType="user">
	select * from user
    <where>
        <if test="sex !=null and sex !=''">
    		and sex="male"
        </if>
        <if test="likeName !=null and likeName !=''">
            and name like #{likeName}
        </if>
        <if test=test="age !=null and age !=''">
            and age =10
        </if>
    </where>
</select>
~~~

## set

执行修改语句时，当设有多个set字段时，使用if语句可能会导致最终SQL语句多出`,`

所以mybatis设置了set标签，它：

* 自动拼装set关键字
* 自动去掉最后一个参数的`,`

示例：

~~~xml
<update id="updateUser">
  update user
    <set>
      <if test="username != null">username=#{username},</if>
      <if test="password != null">password=#{password},</if>
      <if test="email != null">email=#{email},</if>
    </set>
  where id=#{id}
</update>
~~~



## trim

能指定sql语句的前后缀并去掉指定的sql关键字或字符

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



## forEach

能遍历指定的集合，在in关键字中有应用

属性：

* open ：指定开始的字符
* separator：指定分隔符
* close：指定结束字符
* collection：指定要遍历的集合类型
* item：指定遍历的元素名
* index：指定遍历下标的变量名

示例：

~~~xml
<delete id="deleteBatch">
    delete from account where aid in
    <foreach collection="array" item="aid" open="(" close=")" separator=",">
        #{aid}
    </foreach>
</delete>
~~~

# SqlSession

SqlSession是使用Mybatis的主要Java接口，通过这个接口，我们可以:

* 执行命令
* 获取映射器示例和事务管理



## 获取SQLSession实例

* 通过(org.apache.ibatis.io)Resources工具类中的以下方法加载配置文件：

    InputStream getResourceAsStream(String resource)

* 通过SqlSessionFactoryBuilder类中的以下方法创建SqlSessionFactory实例：

    SqlSessionFactory build(InputStream inputStream)

* 通过SqlSessionFactory类中的以下方法创建SqlSession实例：

    SqlSession openSession()

注意：SqlSessionFactory是懒汉式单例模式

实例：

~~~java
//一个获得SqlSession的方法
public SqlSession getSqlSession(){
    if(sqlSessionFactory == null){
        try {
            InputStream in = Resources.getResourceAsStream("mybatis.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(in);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    SqlSession sqlSession = sqlSessionFactory.openSession();
    return sqlSession;
}
~~~



## 语句执行方法

* \<T> T selectOne(String statement, Object parameter)

    执行单条查询

* \<E> List\<E> selectList(String statement, Object parameter)

    执行多条查询

* int insert(String statement, Object parameter)

    执行插入语句

* int update(String statement, Object parameter)

    执行更新语句

* int delete(String statement, Object parameter)

    执行删除语句

statement参数定位到具体的语句，格式为`"mapperUsername.语句id"`

parameter为传给语句的参数

## 事务控制方法

* void commit()
    事务提交
* void rollback()
    事务回滚

## 关闭SqlSession

* void close()

    关闭

## 使用映射器

* \<T> T getMapper(Class\<T> type)

采用接口，使用代理的方式创建接口的实现类

type为mapper接口的Class文件

映射器接口需要满足：

* mapper映射文件名与mapper接口名

* namespace为mapper接口的全路径名
* sql标签中的id与mapper接口中对应方法名一致
* sql标签中的parameterType和mapper接口中的对应方法参数类型一致
* sql标签中的resultType和mapper接口中对应方法的返回值类型一致

**注意：**

* 如果映射方法需要接受多个参数，可以在参数前使用`@Param`注解来定义每个参数的名字

* 增删改的方法返回值除了int也可以是boolean类型，mybatis会自动转换
