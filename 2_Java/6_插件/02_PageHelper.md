# 分页查询

在要查询的数据量大的情况下，一次性查询所有数据会导致：

* java内存容器不足以存储这些数据
* 前台一次性显示所有数据不利于阅读

所以出现了分页技术，对查询结果进行分页，一次只提供查询结果的指定内容

mysql的分页查询方言为

`limit start,step`

放到查询语句的最后，以实行分页

## 手动实现分页

需要实现：

* bean:加入分页需要的属性：当前页，分页起始，分页步长

* dao: 在sql语句中加入分页方言语句
* service：传入的实体类中需要包含有分页起始，分页步长

* controller:需要从前端接收当前页与每页条数，并计算数据量，得出总页数并传递给前端

# pageHelper

pageHelper为mybatis提供了分页功能，在mybatis的SqlSessionFactory生成SqlSession时，为指定的查询提供了分页，并返回分页需要的前端信息

## maven依赖

5.0以上的版本才提供自动为不同的数据库提供分页方言

~~~xml
<!-- pagehelper -->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>5.1.1</version>
</dependency>
<!-- pagehelper整合springboot -->
 <dependency>
     <groupId>com.github.pagehelper</groupId>
     <artifactId>pagehelper-spring-boot-starter</artifactId>
     <version>1.4.1</version>
     <!-- 这个依赖会整合mybatis，如果当前项目不需要，必须exclusion，否则会因为没有配置数据源而保存-->
     <exclusions>
         <exclusion>
             <groupId>org.mybatis.spring.boot</groupId>
             <artifactId>mybatis-spring-boot-starter</artifactId>
         </exclusion>
     </exclusions>
</dependency>
~~~

## 配置

* spring配置

在spring配置SqlSessionFactory时，为其配置pageHelper插件：

~~~xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="myDataSource"/>
    <property name="configLocation" value="classpath:mybatis.xml"/>
    <!--配置pageHelper插件-->
    <property name="plugins">
        <array>
            <bean class="com.github.pagehelper.PageInterceptor">
                <property name="properties">
                    <value>
                        offsetAsPageNum=true
                        rowBoundsWithCount=true
                        pageSizeZero=true
                        reasonable=true
                    </value>
                </property>
            </bean>
        </array>
    </property>
</bean>
~~~

* springboot配置

~~~yaml
pagehelper:
    helperDialect: mysql
    reasonable: true
    supportMethodsArguments: true
    params: count=countSql
~~~

## 使用pageHeler

* 必须紧靠着查询语句执行之前，声明开启分页
* 查询完毕后，将结果交给pageInfo，能得到所有的分页信息，以便交给前端

~~~java
//service层
public PageInfo<Object> queryLoansByType(int pageNum, int pageSize) {
    //开启分页，该语句后面必须是请求
    PageHelper.startPage(pageNum,pageSize);
    List<Object> objects = loanInfoMapper.selectObjects();
    PageInfo<Object> pageInfo = new PageInfo<>(objects);
    return pageInfo;
}
~~~

## pageInfo

pageInfo提供了分页常用的属性

~~~java
private int pageNum; //当前页码
private int pageSize;//设置每页多少条数据
private int size;//当前页有多少条数据
private int startRow;//当前页码的开始条
private int endRow;//当前页码结束条
private int pages;//总页数
private int prePage;//上一页（页面链接使用）
private int nextPage;//下一页（页面链接使用）
private boolean isFirstPage;//是否为第一页
private boolean isLastPage;//是否为最后一页
private boolean hasPreviousPage;//是否有前一页
private boolean hasNextPage;//是否有下一页
private int navigatePages;//导航页码数(就是总共有多少页)
private int[] navigatepageNums;//导航页码数(就是总共有多少页),可以用来遍历
private int navigateFirstPage;//首页号
private int navigateLastPage;//尾页号
//其父类方法
protected long total;//查询总数据量
protected List<T> list;//查询的具体数据对象表
~~~
