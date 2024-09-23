# 简介和快速开始

使用`Mybatis`进行数据访问时，需要进行一系列重复的操作，对数据库中表：

* 根据数据库表设计领域模型，pojo类的属性通常和表字段一一对应。
* 根据表编写Mapper xml文件和对应的Mapper接口，至少需要对应的简单增删改查。其他的复杂查询根据业务需要来。

这一过程显示是机械的。 MyBatis Generator可以帮助我们完成上面提到的大部分工作，为我们根据表结构生成基本的Pojo类和对应的增删改查xml配置和Mapper代码。

使用 MyBatis Generator生成代码的的步骤如下：

* 在项目中引入相关依赖
* 新建和编写相关配置文件
* 编写启动generator的java代码
* 执行代码以生成mabaits代码，注意代码执行的工作目录必须是当前项目的根目录

mybatis generator通过targetRuntime配置不同的代码风格，这个属性有如下可选项：

* `MyBatis3DynamicSql`:默认值，生成Java代码;不生成XML文件，使用Mapper注解，生成的代码量较小；生成的代码依赖于 MyBatis 动态 SQL 库
* `MyBatis3Kotlin`: 生成 Kotlin 代码
* `MyBatis3` 生成 Java 代码； 生成的代码没有外部依赖项； 生成的代码量非常大；生成的代码查询构造能力有限，扩展难度大
* `MyBatis3Simple` 生成 Java 代码；是上一个配置的简化版本； 生成的代码量相对较小

我们以`MyBatis3Simple`为示例：

首先引入下面依赖：

~~~xml
<dependency>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-core</artifactId>
    <version>1.4.0</version>
</dependency>
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.3.0</version>
</dependency>
~~~

然后在项目的`resources`下新建`generatorConfig.xml`配置文件，内容为：

~~~xml
<!DOCTYPE generatorConfiguration PUBLIC
        "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <context id="simple" targetRuntime="MyBatis3Simple">
        <jdbcConnection driverClass="com.mysql.cj.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/test"
                        userId="root"
                        password="123456"/>

        <javaModelGenerator targetPackage="com.wzm.model" targetProject="src/main/java"/>

        <javaClientGenerator type="ANNOTATEDMAPPER" targetPackage="com.wzm.mapper" targetProject="src/main/java"/>

        <table tableName="BLOG" />
    </context>
</generatorConfiguration>
~~~

然后编写下面代码：

~~~java
public class App {

    public static void main(String[] args) throws Exception {
        generator();
    }

    public static void generator() throws Exception {
        List<String> warnings = new ArrayList<String>();
        boolean overwrite = true;
        String path = App.class.getClassLoader().getResource("generatorConfig.xml").getPath();
        File configFile = new File(path);

        ConfigurationParser cp = new ConfigurationParser(warnings);
        Configuration config = cp.parseConfiguration(configFile);
        DefaultShellCallback callback = new DefaultShellCallback(overwrite);
        MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config, callback, warnings);
        myBatisGenerator.generate(null);
    }

}
~~~

执行以上代码以生成mabatis代码。

以上就是整个过程

**注意**:`generator`生成mapper配置文件和接口后，并不会将其自动加入mybatis的配置文件中。所以需要将其手动添加到`mybatis-config.xml`配置文件中。

# 通过maven运行MybatisGenerator

可以以多种方式运行`MybatisGenerator`:

* 通过命令行
* 使用Ant
* 使用Maven
* 使用Java代码

在快速开始中，我们给了Java代码的演示，这里再给出使用Maven执行`MybatisGenerator`:

Mybatis生成器提供一个maven插件：mybatis-generator-maven-plugin，用于集成到maven build中。

最简单的使用该插件的pom.xml配置如下：

~~~xml
<project ...>
    ...
    <build>
        ...
        <plugins>
            ...
            <plugin>
                <groupId>org.mybatis.generator</groupId>
                <artifactId>mybatis-generator-maven-plugin</artifactId>
                <version>1.4.2</version>
            </plugin>
            ...
        </plugins>
        ...
    </build>
    ...
</project>
~~~

可以通过下面命令行来执行插件：

~~~shell
mvn mybatis-generator:generate
~~~

也可以附加一些参数：

~~~shell
mvn -Dmybatis.generator.overwrite=true mybatis-generator:generate
~~~

但是像这样的配置无法直接运行，因为`mybatis-generator-maven-plugin`插件依赖于数据库驱动，可以在`plugin`标签中添加依赖：

~~~xml
<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.4.2</version>
    <dependencies>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.3.0</version>
        </dependency>     
    </dependencies>
</plugin>
~~~

但是generator提供了更好的方法：可以使用配置参数`includeCompileDependencies`将当前项目的依赖项都作为插件的依赖项，如：

~~~xml
<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.4.2</version>
    <configuration>
        <includeCompileDependencies>true</includeCompileDependencies>
    </configuration>
</plugin>
~~~

插件提供的配置参数如下：

| Parameter                    | Type               | Comments                                                     |
| ---------------------------- | ------------------ | ------------------------------------------------------------ |
| `configurationFile`          | `java.io.File`     | 生成器配置文件地址，默认值为:`${basedir}/src/main/resources/generatorConfig.xml` |
| `contexts`                   | `java.lang.String` | 要运行的上下文，可以用逗号`,`分隔来指定多个上下文。指定后，将运行配置文件中`id`与之匹配的`context`标签。默认为空，即运行所有上下文。 |
| `jdbcDriver`                 | `java.lang.String` | 如果指定 sqlScript，指定完全限定的JDBC驱动程序类名           |
| `jdbcPassword`               | `java.lang.String` | 如果指定 sqlScript，指定JDBC连接数据库的密码                 |
| `jdbcURL`                    | `java.lang.String` | 如果指定 sqlScript，指定JDBC连接数据库的URL                  |
| `jdbcUserId`                 | `java.lang.String` | 如果指定 sqlScript，指定JDBC连接数据库的用户名               |
| `outputDirectory`            | `java.io.File`     | MBG 生成的文件的配置目录。当配置文件中的 `targetProject` 设置为`MAVEN`时，会用到该目录。 |
| `overwrite`                  | `boolean`          | 如果为 true，则如果发现现有 Java 文件与生成的文件同名，则现有 Java 文件将被覆盖。如果未指定，并且已经存在与生成的文件同名的 Java 文件，则 MBG 会将新生成的 Java 文件写入具有唯一名称的适当目录（例如 MyClass.java.1、MyClass.java.2 等）。**重要说明：MBG 将始终合并和覆盖 XML 文件。**默认为`false` |
| `sqlScript`                  | `java.lang.String` | 在生成代码之前要运行的 SQL 脚本文件的位置。默认为`null`，如果该值不为`null`，则必须指定``jdbcDriver`, `jdbcURL`，如果数据库需要验证，还需指定`jdbcUserId` 和`jdbcPassword` |
| `tableNames`                 | `java.lang.String` | 用逗号分隔的字符串，指定多个表名。用于进一步限定配置中指定生效的表。该属性中指定的表名必须在配置文件中的表中；默认为空，即`context`中所有的表都会生效。如果不为空，则只有该字段指定的表会生效。 |
| `verbose`                    | `boolean`          | 如果为 true，则 MBG 会将进度消息写入构建日志。               |
| `includeCompileDependencies` | `boolean`          | 如果为 true，则范围为 “compile”、“provided” 和 “system” 的依赖项将被添加到生成器的 Classpath 中。 |
| `includeAllDependencies`     | `boolean`          | 如果为 true，则具有任何范围的依赖项都将添加到生成器的 Classpath 中。 |



# MybatisGenerator配置文件

MyBatis Generator 是由 XML 配置文件驱动的。配置文件告诉 MBG：

*  如何连接到数据库
* 要生成哪些对象以及如何生成它们
*  应该使用哪些表来生成对象

一个典型的配置文件结构如下：

~~~xml

        <!-- Mapper映射生成器，即配置生成生成XxxMapper.xml的位置 -->
        <sqlMapGenerator targetPackage="cn.jq.jqmybatis.dao" targetProject=".\src\main\java">
            <!-- 在targetPackage的基础上，根据数据库的schema再生成一层package，最终生成的类放在这个package下，默认为false -->
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <!--Mapper接口生成器, 即配置生成生成的 Mapper接口的位置，注意，如果没有配置该元素，那么默认不会生成Mapper接口
                type：选择怎么生成mapper接口（在MyBatis3/MyBatis3Simple下）：
                    1，ANNOTATEDMAPPER：会生成使用Mapper接口+Annotation的方式创建（SQL生成在annotation中），不会生成对应的XML；
                    2，MIXEDMAPPER：使用混合配置，会生成Mapper接口，并适当添加合适的Annotation，但是XML会生成在XML中；
                    3，XMLMAPPER：会生成Mapper接口，接口完全依赖XML；
                注意，如果context是MyBatis3Simple：只支持ANNOTATEDMAPPER和XMLMAPPER
        -->
        <javaClientGenerator type="XMLMAPPER" targetPackage="cn.jq.jqmybatis.dao" targetProject=".\src\main\java">
            <!-- 在targetPackage的基础上，根据数据库的schema再生成一层package，最终生成的类放在这个package下，默认为false -->
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <!-- 指定数据库中的数据表（可同时指定多张表）进行生成 -->
        <!--字段命名策略过程: table标签对应数据库中的table表-->
        <table tableName="t_user" domainObjectName="User"></table>
        <table tableName="t_role" domainObjectName="Role"></table>

    </context>
</generatorConfiguration>
~~~

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>

    <!--指定mysql数据库驱动-->
    <!--<classPathEntry location="E://repository-p2p//mysql//mysql-connector-java//5.1.43//mysql-connector-java-5.1.43.jar"/>-->

    <!--导入属性配置   配置信息中有链接数据库的参数-->
    <!--<properties resource="generator.properties"></properties>-->

    <!--指定特定数据库的jdbc驱动jar包的位置
        注意：驱动类需要找到他在本机的绝对位置   不能有空格和中文路径
    -->
    <!--TODO 需要修改-->
    <classPathEntry location="D:/Data/Learning/Java/repository/mysql/mysql-connector-java/8.0.15/mysql-connector-java-8.0.15.jar"/>

    <context id="default" targetRuntime="MyBatis3">

        <!-- optional，旨在创建class时，对注释进行控制，false生成注释,true无注释 -->
        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>

        <!--jdbc的数据库连接
            逆向工程会扫描你数据库中的指定表  你多个数据库可能会出现同名的表
            他会找第一个出现的表名
            nullCatalogMeansCurrent：无视其他同名表  只生成指定数据库中的表
            在url上配置参数
            jdbc:mysql://localhost:3306/sh2203?serverTimezone=UTC&nullCatalogMeansCurrent=true
         -->
        <jdbcConnection
                driverClass="com.mysql.cj.jdbc.Driver"
                connectionURL="jdbc:mysql://localhost:3306/empproject?serverTimezone=UTC"
                userId="root"
                password="root">
            <property name="nullCatalogMeansCurrent" value="true"/>
        </jdbcConnection>


        <!-- 非必需，类型处理器，在数据库类型和java类型之间的转换控制-->
        <javaTypeResolver>
            <!--
                true：使用BigDecimal对应DECIMAL和 NUMERIC数据类型
                false：默认,
                    scale>0;length>18：使用BigDecimal;
                    scale=0;length[10,18]：使用Long；
                    scale=0;length[5,9]：使用Integer；
                    scale=0;length<5：使用Short；      
			-->
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>


        <!-- java模型创建器，即配置生成java POJO实体类的位置
               负责：1，key类（见context的defaultModelType）；2，java类；3，查询类
               targetPackage：生成的类要放的包，真实的包受enableSubPackages属性控制；
               targetProject：目标项目，指定一个存在的目录下，生成的内容会放到指定目录中，如果目录不存在，MBG不会自动建目录
        -->
        <javaModelGenerator targetPackage="com.bjpn.bean"
                            targetProject="D:/Data/Learning/Java/workspace/ssm/src/main/java">

            <!-- 是否允许根据数据库的schema生成子包，即targetPackage.schemaName.tableName -->
            <property name="enableSubPackages" value="false"/>
            <!-- 是否对model添加 构造函数 true添加，false不添加
                不生成有参构造  否则mapper.xml文件的关系映射会使用有参构造  不利于我们操作
            -->
            <property name="constructorBased" value="false"/>
            <!-- 是否对类CHAR类型的列的数据进行trim操作 -->
            <property name="trimStrings" value="true"/>
            <!-- 建立的Model对象是否 不可改变  即生成的Model对象不会有 setter方法，只有构造方法 -->
            <property name="immutable" value="false"/>
        </javaModelGenerator>

        <!-- Mapper映射生成器，即配置生成生成XxxMapper.xml的位置 -->
        <sqlMapGenerator targetPackage="com.bjpn.mapper"
                         targetProject="D:/Data/Learning/Java/workspace/ssm/src/main/java">
            <property name="enableSubPackages" value="false"/>
        </sqlMapGenerator>

        <!--Mapper接口生成器, 即配置生成生成的 Mapper接口的位置，注意，如果没有配置该元素，那么默认不会生成Mapper接口
                type：选择怎么生成mapper接口（在MyBatis3/MyBatis3Simple下）：
                    1，ANNOTATEDMAPPER：会生成使用Mapper接口+Annotation的方式创建（SQL生成在annotation中），不会生成对应的XML；
                    2，MIXEDMAPPER：使用混合配置，会生成Mapper接口，并适当添加合适的Annotation，但是XML会生成在XML中；
                    3，XMLMAPPER：会生成Mapper接口，接口完全依赖XML；
                注意，如果context是MyBatis3Simple：只支持ANNOTATEDMAPPER和XMLMAPPER
        -->
        <javaClientGenerator targetPackage="com.bjpn.mapper"
                             targetProject="D:/Data/Learning/Java/workspace/ssm/src/main/java" type="XMLMAPPER">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>
        <!--4、根据指定的表  生成对应的信息
            tableName：表名
            domainObjectName：根据这个表生成的实体类的名字
            enableCountByExample="false" enableUpdateByExample="false"
               enableDeleteByExample="false" enableSelectByExample="false"
               selectByExampleQueryId="false"
               不会产生一个额外的Example类
               只会产生基本增删改查方法
        -->
        <table tableName="emp" domainObjectName="Employee"
               enableCountByExample="false" enableUpdateByExample="false"
               enableDeleteByExample="false" enableSelectByExample="false"
               selectByExampleQueryId="false">
        </table>
        <table tableName="dept" domainObjectName="Department"
               enableCountByExample="false" enableUpdateByExample="false"
               enableDeleteByExample="false" enableSelectByExample="false"
               selectByExampleQueryId="false">
        </table>
        <table tableName="admin" domainObjectName="Administrator"
               enableCountByExample="false" enableUpdateByExample="false"
               enableDeleteByExample="false" enableSelectByExample="false"
               selectByExampleQueryId="false">
        </table>
    </context>
</generatorConfiguration>
~~~

完整属性配置项请参照<a href="https://mybatis.org/generator/configreference/xmlconfig.html ">官方文档</a>

