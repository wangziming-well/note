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
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.16</version>
</dependency>
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













