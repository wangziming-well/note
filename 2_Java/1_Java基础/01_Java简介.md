## Java 语言的由来

* 1995年诞生
* 1996年发布第一版jdk
* 2008年：SUM公司收购Mysql公司
* 2009年：Oracle公司收购了SUM公司 

## Java 语言的特点

* 面向对象(封装，继承，多态)
* 跨平台性（可移植性）
* 支持多线程，网络编程
* 编译与解释并存
* 垃圾回收机制

## 跨平台性的原理

JVM ：Java 虚拟机

## JDK JRE JVM

* JVM（Java Virtual Machine）：是运行 Java 字节码的虚拟机。JVM 有针对不同系统的特定实现（Windows，Linux，macOS）

* JRE(Java Runtime Environment):Java 运行时环境：它是运行已编译 Java 程序所需的所有内容的集合，包括 Java 虚拟机（JVM），Java 类库，java 命令和其他的一些基础构件。但是，它不能用于创建新程序。

* JDK(Java Development Kit):Java开发工具包 ：它是功能齐全的 Java SDK。它拥有 JRE 所拥有的一切，还有编译器（javac）和工具（如 javadoc 和 jdb）。它能够创建和编译程序。

如果仅仅想要Java程序，安装JRE即可

如果想进行Java开发，需要JDK

![JVM JER JDK关系图](https://gitee.com/wangziming707/note-pic/raw/master/img/JVM%20JER%20JDK%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

## 编译与解释并存

==TODO==





## 第一个Java程序

执行Java程序需要经历2个阶段：

* 阶段一：编译 通过javac.exe将程序语言(.java称为java源文件)编译成机器语言（.class又被称为类文件、字节码文件、二进制文件、机器码文件）
* 阶段二：运行 通过java.exe执行.class文件（执行命令时不能加后缀.class）

~~~java
class Demo{
    public static void main(String[] args){
        System.out.println("Hello World");
    }
}
~~~

注意：

1. Java语言严格区分大小写
2. Java程序绝大多数情况下不是以大括号结尾就是以分号结尾
3. Java程序内容一旦进行改动，必须重新进行编译操作（javac的行为）

4. Java程序中的标点符号都是英文输入法下的

5. 在一个Java源文件中，public可以修饰类，但只能修饰一个类，而且类名必须和文件名一致



