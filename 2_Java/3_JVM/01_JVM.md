# JVM概述

JVM :Java虚拟机 是Java 的核心和基础。它运行在操作系统之上，不与硬件直接交互，通过虚拟化技术为Java 程序提供运行环境。是Java语言实现跨平台特性的关键。

JVM 可以执行编译器生成的 Java字节码文件 来运行Java程序

## JVM结构

![JVM结构](https://gitee.com/wangziming707/note-pic/raw/master/img/JVM%E7%BB%93%E6%9E%84.png)

主要组件有

* 类加载器：负责将.class字节码文件加载进JVM空间
* 内存空间：包括方法区，堆，栈，本地方法栈，程序计数器
* 垃圾收集器：负责回收程序生产的垃圾数据

## Java运行原理

运行一段Java代码，需要经过编译和执行两个阶段:

* Java源代码需要经过Java编译器编译成JVM能识别的字节码文件(.class)

* JVM 执行字节码文件以运行Java程序

### 编译

由Java源码编译器执行编译，将源码编译成字节码文件,主要有下面三个过程：

* 分析和输入到符号表
* 注解处理
* 语义分析和生成 class 文件

最后生成的class文件由以下部分组成：

* 结构信息：包括 class 文件格式版本号及各部分的数量与大小的信息。  

* 元数据：对应于 Java 源码中声明与常量的信息。包含类/继承的超类/实现的接口的声明信息、域与方法声明
    信息和常量池。  
* 方法信息：对应 Java 源码中语句和表达式对应的信息。包含字节码、异常处理器表、求值栈与局部变量区
    大小、求值栈的类型记录、调试符号信息。  

### 执行

JVM执行class文件大致分为两个部分：

* 类加载器将class文件加载到JVM中
* 基于栈执行class字节码



# JVM内存区域

Java 虚拟机在执行 Java 程序的过程中会把他所管理的内存划分为若干个不同的数据区域。Java 虚拟机规范将
JVM 所管理的内存分为以下几个运行时数据区：程序计数器、Java 虚拟机栈、本地方法栈、Java 堆、方法

它们可以分为线程共享的和线程私有的：

* 线程共享：
    * 方法区
    * 堆
* 线程私有：
    * 栈 
    * 本地方法栈
    * 程序计数器

## 方法区

是线程共享的内存区域，用于存储已经被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据

运行时常量池是方法区的一部分， Class文件常量池中字面量和符号引用 将在类加载后存放到运行时常量池

运行时常量池是动态的，也就是说运行期间允许有新的常量加入常量池。

## 堆

Java Heap 是JVM内存区域中最大的一块，它是所有线程贡献的一块内存区域几乎所有的对象实例和数组都在这类分配内存。Java Heap 是垃圾收集器管理的主要区域，因此很多时候也被称为“GC堆”

## Java虚拟机栈

简称为栈，它的生命周期和线程相同，是描述Java方法执行的内存模型

每个方法被执行的时候都会创建一个栈帧，它是用于支持虚拟机进行方法调用和方法执行的数据结构

它的结构如下：

![JVM栈结构](https://gitee.com/wangziming707/note-pic/raw/master/img/JVM%E6%A0%88%E7%BB%93%E6%9E%84.png)

在活动线程中，只有栈顶的栈帧是有效的，称为当前栈帧，这个栈帧所关联的方法称为当前方法，执行引擎所运行的所有字节码指令都只针对当前栈帧进行操作

一个方法被调用，会生成对应的栈帧，并压栈，当前方法运行完成后，当前栈帧会弹栈

栈帧用于存储局部变量表、操作数栈、动态链接、方法返回地址和一些额外的附加信息。

* 局部变量表：是一组变量值存储空间，用于存放方法参数和方法内部定义的局部变量

    局部变量表所需的内存空间在编译期间完成分配，即在 Java 程序被编译成 Class 文件时，就确定了所需分配的最大局部变量表的容量

    存放变量的数据类型有基本数据类型，对象引用，和returnAddress  类型(Java 不会直接和该类型打交道，它指向了一条字节码指令的地址)

* 操作数栈：用于存放方法执行过程中产生的中间计算结果。另外，计算过程中产生的临时变量也会放在操作数栈中。在方法的执行过程中，会有各种字节码指令（比如：加操作、赋值元算等）向操作栈中写入和提取内容，也就是入栈和出栈操作。

* 动态链接：是指向运行时常量池的方法引用，用以调用其他方法

    ![JVM栈的动态链接](https://gitee.com/wangziming707/note-pic/raw/master/img/JVM%E6%A0%88%E7%9A%84%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5.png)

* 方法返回地址：存放调用该方法的PC计数器的值

     当一个方法被执行后，总要退出该方法(要么执行return ，要么遇到未处理的异常)

    无论采用何种退出方式，在方法退出之后，都需要返回到方法被调用的位置，程序才能继续执行  

## 本地方法栈

该区域与虚拟机栈所发挥的作用非常相似，只是虚拟机栈为虚拟机执行 Java 方法服务，而本地方法栈则为使用到的本地操作系统（Native）方法服务。

## 程序计数器

一块较小的内存空间，它是当前线程所执行的字节码的行号指示器。字节码解释器工作时通过改变该计数器的值来选择下一条需要执行的字节码指令，分支、跳转、循环等基础功能都要依赖它来实现当线程在执行一个 Java 方法时，该计数器记录的是正在执行的虚拟机字节码指令的地址

* 每条线程都有一个独立的的程序计数器，各线程间的计数器互不影响，因此该区域是线程私有的  

* 当线程在执行的是 Native 方法（调用本地操作系统方法）时，该计数器的值为空

* 程序计数器只会改变自己的值而不是扩大自己的空间，所以该内存区域是唯一一个在 Java 虚拟机规范中么有规定任何 OOM（内存溢出：OutOfMemoryError）情况的区域

## 直接内存

直接内存并不是虚拟机运行时数据区的一部分，也不是 Java 虚拟机规范中定义的内存区域，它直接从操作系统中分配，因此不受 Java 堆大小的限制，但是会受到本机总内存的大小及处理器寻址空间的限制，因此它也可能导致 OutOfMemoryError 异常出现。



## 引用类型访问对象实例方式

对象实例存放在堆中，而引用类型在栈中，用来访问堆中实例，而具体的访问方式分为两种：使用句柄池和直接使用指针

* 句柄池访问:

    reference 中存放的是稳定的句柄地址，当对象被移动时(垃圾收集时移动对象是非常普遍的行为  )只需要改变句柄中的实例数据指针，而不需要改变reference本身。

    ![引用变量访问实例-句柄池](https://gitee.com/wangziming707/note-pic/raw/master/img/%E5%BC%95%E7%94%A8%E5%8F%98%E9%87%8F%E8%AE%BF%E9%97%AE%E5%AE%9E%E4%BE%8B-%E5%8F%A5%E6%9F%84%E6%B1%A0.png)

* 直接指针访问：速度快，它节省了一次指针定位的时间开销  

    ![引用变量访问实例-直接指针](https://gitee.com/wangziming707/note-pic/raw/master/img/%E5%BC%95%E7%94%A8%E5%8F%98%E9%87%8F%E8%AE%BF%E9%97%AE%E5%AE%9E%E4%BE%8B-%E7%9B%B4%E6%8E%A5%E6%8C%87%E9%92%88.png)

## 内存溢出 内存泄漏 栈溢出

内存溢出是指程序所需的内存超过了系统所能分配的内存的上线

内存泄漏是指分配出的内存没有被回收回来，由于失去了对该内存区域的控制，因而造成了资源的浪费

栈溢出是指JVM栈帧的数量超过了JVM规定的上限

# 类加载机制

类加载过程包括了加载、验证、准备、解析、初始化五个阶段

验证、准备、解析可以并称为链接

在这五个阶段中，加载、验证、准备、初始化四个阶段发生的顺序是确定的，但解析阶段不一定，它在某些情况下可以在初始化阶段之后开始，这是为了支持Java 语言的运行时绑定(动态绑定或者晚绑定)

这几个阶段是按顺序开始，但不是按顺序进行或完成的。因为这些阶段通常都是互相交叉地混合进行的，通常在一个阶段执行的过程中调用或激活另一个阶段

## 加载

加载是类加载过程的第一个阶段，在加载阶段，虚拟机需要完成以下三件事情：

* 通过类的全限定名来获取其定义的二进制字节流
* 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构

* 在Java堆中生成一个代表这个类的 Class对象，作为对方法区中这些数据的访问入口

大部分类的加载是通过类加载器来完成 ；数组类型不通过类加载器创建，它由 Java 虚拟机直接创建

### 类加载器

类加载器虽然只用于实现类的加载动作，但它在Java 程序中起到的作用却远远不限于类的加载阶段。

对于任意一个类，都需要由它的类加载器和这个类本身一通确定其在Java虚拟机中的唯一性，也就是说，即使两个类来源于同一个Class文件，只要加载它们的类加载器不同，两个类就必定不相等。这里的“相等”包括了代表类的 Class 对象的 equals（）、isAssignableFrom（）、isInstance（）等方法的返回结果，也包括了使用 instanceof 关键字对对象所属关系的判定结果。  

从Java 虚拟机的角度来看，有两种不同的类加载器：

* 启动类加载器：它使用 C++ 实现（这里仅限于 Hotspot，也就是 JDK1.5 之后默认的虚拟机，有很多其他的虚拟机是用 Java 语言实现的），是虚拟机自身的一部分。

* 所有其他的类加载器：这些类加载器都由 Java 语言实现，独立于虚拟机之外，并且全部继承自抽象类 `java.lang.ClassLoader`，这些类加载器需要由启动类加载器加载到内存中之后才能去加载其他的类。

从 Java 开发人员的角度来看，类加载器可以大致划分为以下三类：

* 启动类加载器： Bootstrap ClassLoader ，跟上面相同，最顶层的加载类，由 C++实现，负责加载 `%JAVA_HOME%/lib`目录下的 jar 包和类或，或被` -Xbootclasspath `参数指定的路径中的，并且能被虚拟机识别的类库（如 rt.jar，所有的 java.* 开头的类均被 Bootstrap ClassLoader 加载）。启动类加载器是无法被 Java 程序直接引用的。

* 扩展类加载器：Extension ClassLoader，该加载器由 sun.misc.Launcher$ExtClassLoader 实现，它负责加载 `%JRE_HOME%/lib/ext`目录中，或者由 `java.ext.dirs `系统变量指定的路径中的所有类库（如 javax.* 开头的类），开发者可以直接使用扩展类加载器。  

* 应用程序类加载器：Application ClassLoader，该类加载器由 sun.misc.Launcher$AppClassLoader来实现，它负责加载用户类路径（ClassPath）所指定的类，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。  负责加载当前应用 classpath 下的所有 jar 包和类。

* 自定义加载器:因为 JVM 自带的 ClassLoader 只是懂得从本地文件系统加载标准的 java class 文件，因此如果编写了自己的 ClassLoader，便可以做到如下几点：
    * 在执行非置信代码之前，自动验证数字签名。  
    * 动态地创建符合用户特定需要的定制化构建类。  
    * 从特定的场所取得 java class，例如数据库中和网络中  

### 双亲委派模型

上面介绍的类加载器有层次关系:

![双亲委派模型](https://gitee.com/wangziming707/note-pic/raw/master/img/%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE%E6%A8%A1%E5%9E%8B.png)

我们把每一层上面的类加载器叫做当前层类加载器的父加载器，它们之间的继承关系不是通过继承关系实现的，而是使用组合关系来复用父加载器中的代码。

双亲委派模型的工作流程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上，因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中，只有当父加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。

这样的模型能够保证， Java 类随着它的类加载器（说白了，就是它所在的目录）一起具备了一种带有优先级的层次关系，这对于保证 Java 程序的稳定运作很重要。

无论那个类加载器加载 Object类，都会是启动类加载器加载它，这样保证了Object 类在程序中的各种类加载器中都是同一个类  

## 验证

验证是为了确保 Class 文件中的字节流包含的信息符合当前虚拟机的要求，且不会危害虚拟机自身的安全。

验证大致分为四个部分：

* 文件格式验证：验证字节流是否符合 Class 文件格式的规范，并且能被当前版本的虚拟机处理，该验证的主要目的是保证输入的字节流能正确地解析并存储于方法区之内 经过该阶段的验证后，字节流才会进入内存的方法区中进行存储，后面的三个验证都是基于方法区的存储结构进行的。  
* 元数据验证：对类的元数据信息进行语义校验（其实就是对类中的各数据类型进行语法校验），保证不存在不符合 Java 语法规范的元数据信息。
* 字节码验证：该阶段验证的主要工作是进行数据流和控制流分析，对类的方法体进行校验分析，以保证被校验的类的方法在运行时不会做出危害虚拟机安全的行为。
* 符号引用验证：主要是对类自身以外的信息（常量池中的各种符号引用）进行匹配性的校验

## 准备

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些内存都将在方法区中分配。

在该阶段中：

* 这时候进行内存分配的仅包括类变量（static），而不包括实例变量，实例变量会在对象实例化时随着对象一块分配在 Java 堆中  

* 这里所设置的初始值通常情况下是数据类型默认的零值（如 0、0L、null、false 等），而不是被在 Java 代码中被显式地赋予的值

## 解析

虚拟机将常量池中的符号引用转化为直接引用的过程，解析动作主要针对类或接口、字段、类方法、接口方法四类符号引用进行

* 符号引用：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标并不一定已经加载到了内存中  

* 直接引用：让JVM能直接访问到对象的引用，通常是指针地址，相对偏移量或是一个能间接定位到目标的句柄  

    如果有了直接引用，那说明引用的目标必定已经存在于内存之中了  

解析阶段可能开始于初始化之前，也可能在初始化之后开始

该机制可以实现Java 面向对象的多态，在解析类或接口时：

* 如果上层代码没有多态， 在类被加载器加载时就对常量池中的符号引用进行解析（初始化之前）
* 如果上层代码有多态，那么在加载阶段是JVM是不知道这个变量的具体引用的。 那么会等到符号引用将要被使用前才去解析它（初始化之后）。

在解析类方法，接口方法时，实现了Java 的绑定：

Java中的绑定：一个方法的调用与方法所在的类(方法主体)关联起来，就是让执行方法调用的时候，jvm知道调用了哪个类的方法，绑定分为：

* 静态绑定，即前期绑定。在程序执行前方法已经被绑定，此时由编译器或其它连接程序实现。针对 Java，简单的可以理解为程序编译期的绑定

    Java 当中的方法只有 final，static，private 和构造方法是前期绑定的(都是无法被继承 或者被覆盖的方法)

* 动态绑定，即晚期绑定，也叫运行时绑定 。 在运行时根据具体对象的类型进行绑定。在 Java 中，几乎所有的方法都是后期绑定的

## 初始化

初始化时类加载的最后一个阶段 ，是执行类构造器 `<clinit>()`方法的过程

clinit的执行规则如下：

* clinit方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块合并产生的

    编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句中可以赋值，但是不能访问。  

* 类构造器`clinit()`和实例构造器`init()`不同，它不需要显式地调用父类构造器。虚拟机会保证在子类的clinit方法执行之前，父类的clinit方法已经执行完毕。因此，在虚拟机中第一个被执行的clinit方法的类肯定是`java.lang.Object`

* `clinit()`方法对于类或接口来说并不是必须的，如果一个类中没有静态语句块，也没有对类变量的赋值操作，那么编译器可以不为这个类生成`clinit()`方法。  

* 接口中不能使用`static`代码块 ，但仍然有类变量(final static) 初始化的复制操作，所有接口和类一样会生成`clinit()`方法。

    但接口和类不同的是：执行接口的`clinit()`方法不需要先执行父接口的`clinit()`方法 ，只有当父接口中定义的类变量被使用是，父接口才会被初始化。

    另外，接口的实现类在初始化时也一样不会执行接口的`clinit()`方法。

* JVM会保证一个类的`clinit()`方法在多线程环境中被正确的加锁和同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的`clinit()`方法，其他线程都需要阻塞等待，直到活动线程执行`clinit()`方法完毕。  

    如果在一个类的`clinit()`方法中有耗时很长的操作，那就可能造成多个线程阻塞，在实际应用中这种阻塞往往是很隐蔽的。  



# Java 垃圾收集

Java 程序员不需要手动释放内存。

垃圾回收（Garbage Collection，GC）：在程序的运行环境中，JVM 提供了一个系统级的垃圾回收器线程，它负责自动回收那些无用对象所占用的堆内存。

## 对象引用

针对引用和垃圾回收的关系，Java 划分了四种引用：

* 强引用：直接 new出来的对象引用。只要强引用还存在，垃圾收集器就永远不会回收掉被引用的对象  

* 软引用：它用来描述一些可能还有用，但并非必须的对象。在系统内存不够用时，这类引用关联的对象将被垃圾收集器回收。可以通过SoftReference 类来实现软引用。

    软引用可以用于实现对内存敏感的高速缓存。

* 弱引用：它也是用来描述非需对象的，但它的强度比软引用更弱些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。可以通过 WeakReference 类来实现弱引用。  

* 虚引用：最弱的一种引用关系，就和没有引用一样，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。可以通过 PhantomReference 类来实现虚引用。  

    将一个对象设置虚引用关联的唯一作用是让系统在这个对象在被销毁之前接受到一个通知。程序员可以借此在对象销毁前做一些措施。

    虚引用必须配合引用队列(ReferenceQueue)使用。当垃圾回收器准备回收一个对象时，如果发现它还有引用，那么就会在回收对象之前，把这个引用加入到与之关联的引用队列中去。程序可以通过判断引用队列中是否已经加入了引用，来判断被引用的对象是否将要被垃圾回收，这样就可以在对象被回收之前采取一些必要的措施。

## 垃圾对象的判定

垃圾收集器在回收对象前，需要判定对象是否为垃圾对象，有如下算法：

### 引用计数算法

给对象添加一个引用计数器，每当一个地方引用它时，计数器值加一，当引用失效时，计数器就减一，任何时刻计数器都为0的对象就是不可能再被使用的

这种方法实现简单，判定效率也高，但Java语言并没有选择这种算法来进行垃圾回收，因为它很难解决对象之间循环引用的问题

### 根搜索算法(可达性分析算法)

#### 算法描述

通过一系列必须活跃的引用对象 (这样的引用就叫做 GC Roots)作为起始点，从这些节点向下搜索，搜索走过的路径称为引用链，如果一个对象没有与到GC Roots的引用链相连时，这个对象就是可回收的。

在Java中，可作为GC Roots 的对象包括：

* 栈中引用的对象
* 方法区中静态变量属性引用的对象
* 方法区中常量引用的对象
* 本地方法栈中 JNI(Native 方法)的引用对象
* 所有被同步锁 synchronized 持有的对象

也就是说，如果一个指针，它保存了堆内存的对象地址，但自己本身不保存在堆内存中，那么它就是一个GC Root

#### 根节点一致性

使用这种算法需要保证根节点集合的一致性

也就是说，在执行根节点枚举的过程中，根节点不能改变。否则会影响算法结果的准确定

这种算法在枚举跟节点时必须暂停用户线程，或者生成根节点快照，以保证根节点的一致性

#### 两次标记

事实上，在根搜索算法中，要宣告一个对象的死亡，需要至少经过两次标记：

对象在进行根搜索后发现没有与GC Roots 相连的引用链时，并不会直接被垃圾回收，它会被第一次标记并且进行一次筛选：筛选的条件是此对象是否有必要执行finalize()方法。当对象没有覆盖finalize()方法或者finalize()方法已经被虚拟机调用过一次，那么它就不需要执行finalize()方法。

如果该对象被判定为有必要执行finalize()方法，那么这个对象将被放置在一个叫 F-Queue队列中，并在稍后由一条由虚拟机自动建立的，低优先级的 Finalizer线程 去执行 finalize()方法。

如果这个对象的finalize()方法中有让该对象与任意一个引用链上的对象建立关联。那么它就可以逃脱死亡。否则它就会被回收。

## 垃圾收集算法

判定出了垃圾对象后，就可以进行垃圾回收了。

### 分代收集理论

#### 强弱分代假说

现在大部分的垃圾收集器，大部分都遵循了分代收集的理论进行设计的，它建立在两个假说之上：

* 弱分代假说：绝大部分对象的生命周期都是很短暂的
* 强分代假说：经过越多次垃圾收集过程且没消亡的对象就越难以消亡

基于这两个假说，垃圾收集器应该有这样的设计原则：将Java 堆划分出不同的区域，然后将回收对象依据其年龄(即熬过垃圾收集过程的次数)分配到不同的区域之中存储，这样：

* 新生代：如果一个区域中大多数对象都是朝生夕灭，难以熬过垃圾收集过程的话，那么把它们集中放在一起，每次回收时只关注如何保留少量存活而不是去标记那些大量将要被回收的对象，就能以较低代价回收到大量的空间
* 老年代：如果剩下的都是难以消亡的对象，那把它们集中放在一块，虚拟机便可以使用较低的频率来回收这个区域，这就同时兼顾了垃圾收集的时间开销和内存的空间有效利用  

有了这样的区域划分，垃圾收集器就可以每次只回收其中某一个或者某些部分的区域，这样就有了划分：

* Minor GC 新生代收集：  指目标只是新生代的垃圾收集  

* Major GC 老年代收集  ：指目标只是老年代的垃圾收集  
* Mixed GC  混合收集  ：指目标是收集整个新生代以及部分老年代的垃圾收集  

* Full GC 整堆收集  ：收集整个 Java 堆和方法区的垃圾收集  

而针对不同区域对象的生命特征，可以匹配合适的收集算法，就有了“标记-复制算法”“标记-清除算法”“标记-整理算法”等针对性的垃圾收集算法。  

#### 跨代引用假说

这样的划分有这明显的问题，对象不是孤立的，对象之间会存在跨代引用：新生代对象是完全有可能被老年代所引用的，这样的对象不应该被回收，这样在一次 Minor GC过程中，除了 GC Roots之外，还需要遍历整个老年代中所有对象来保证跟算法的正确性，这样对垃圾回收来说是很大的性能负担。

为了解决这样的问题就需要对分代收集理论添加第三条经验法则 :

* 跨代引用假说:  跨代引用相对于同代引用来说仅占极少数  

这条假说也可以由前两条假说推理得出:存在互相引用关系的两个对象，是应该倾向于同时生存或者同时消亡的  

根据这条假说，我们不需要为了少量的跨代引用去扫描整个老年代，也不需要浪费空间去专门记录，每一个对象是否存在及存在哪些跨代引用  

只需在新生代上建立一个全局的数据结构  (该结构被称为“记忆集”， Remembered Set  )

这个结构把老年代划分成若干小块，标识出老年代的哪一块内存会存在跨代引用。  

此后当发生 Minor GC 时，只有包含了跨代引用的小块内存里的对象才会被加入到 GCRoots 进行扫描。  

### 标记-清除算法

最早出现和最基础的垃圾收集算法，算法分为标记和清除两个阶段：

* 首先标记出所有需要回收的对象
* 然后统一回收所有被标记的对象

它主要有两个缺点：

* 执行效率不稳定：如果 Java 堆中包含大量对象，而且其中大部分是需要被回收的，这时必须进行大量标记和清除的动作，导致标记和清除两个过程的执行效率都随对象数量增长而降低  
* 内存空间碎片化：标记、清除之后会产生大量不连续的内存碎片 ，空间碎片太多可能会导致当以后在程序运行过程中需要分配较大对象时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作  

### 标记-复制算法

#### 基本思路

简称为复制算法， 可以解决标记-清除算法面对大量可回收对象时执行效率不稳定和内存空间碎片化的问题  

它将内存按容量划分为大小相等的两块，每次只使用其中的一块，进行垃圾回收时，标记不需回收的对象，将其复制到另外一块上面，再将已用过的内存空间一次清理掉。

这种算法针对新生代十分有效，因为新生代的多数对象都是可回收的

但缺点显而易见，这种算法将可用内存缩小为了原来的一半，浪费了大量内存空间

#### 优化

因为新生代的大部分对象都熬不过第一轮收集，所以不需要按1:1的比例划分新生代空间，针对此特点可以做出优化：

将新生代分为一块较大的Eden空间和两块较小的Survivor空间。每次分配都只使用Eden和其中一块Survivor空间。发送垃圾收集时，将Eden和Survivor中标记存活的对象一次性复制到另外一块Survivor空间上，然后直接清理掉 Eden 和已用过的那块 Survivor空间  

HptSpot虚拟机默认Eden和Survivor的大小比例是8:1这样新生代的可用空间为新生代容量的90%

为了应付一些罕见情况(比如某次回收后存活对象比例很大，超过了survivor的大小)，它提供了一种安全设计：当Survivor 空间不足以容纳一次 Minor GC 之后存活的对象时，就需要依赖其他内存区域（实际上大多就是老年代）进行分配担保（Handle Promotion）

如果另外一块 Survivor 空间没有足够空间存放上一次新生代收集下来的存活对象，这些对象便将通过分配担保机制直接进入老年代，这对虚拟机来说就是安全的。   



复制算法在对象存活率较高时要进行较多的赋值操作，效率会降低。而且如果不想浪费一般的内存空间，那么就需要额外的空间进行分配担保，所以这种算法不适合老年代的垃圾清理。

### 标记-整理算法

标记整理算法是针对老年代对象的生命周期特征的算法

该算法在对对象标记后，并不会直接清理需要回收的对象，而是让所有存活的对象都向内存空间一端移动，然后直接清理掉边界以外的内存

标记-清除算法与标记-整理算法的本质差异在于前者是一种非移动式的回收算法，而后者是移动式的  

是否移动回收后的存活对象是一项优缺点并存的风险决策：  

* 如果移动对象，尤其是在老年代这种每次回收都有大量对象存活区域，移动存活对象并更新所有引用这些对象的地方将会是一种极为负重的操作，而且对象的移动操作必须全程暂停用户应用程序才能进行

* 如果不移动对象，那么就会导致空间碎片化问题，需要依赖更为复杂的内存分配器和内存访问器来解决  

    势必会直接影响应用程序的吞吐量  

还有一种同时运用两种方法的解决方案：

* 平时多数时间采用标记-清除算法，当内存空间的碎片化程度开始影响内存分配时，再采用标记-整理算法

## 垃圾收集器

以上是垃圾收集的理论，但实际执行垃圾收集的是垃圾收集器，这里介绍一些常用或者经典的垃圾收集器

这些垃圾收集器有不同的特点和各自的适用场景

有的垃圾收集器之间可以搭配使用

### Serial 收集器

串行收集器是最基础、历史最长的新生代收集器

这是一个单线程收集器，而且在进行垃圾收集时，必须暂停其他所有工作线程，直到它收集结束

但相比于其他收集器，它简单而高效，是在收集器里对额外内存消耗最少的

对于用户桌面和部分微服务应用的场景中，分配给虚拟机的内存一般不会特别大，而Serial收集器收集几十或者几百兆新生代内存可以控制在十几毫秒之内，只要不是频繁发送，这样的停顿时间是可以接受的

### ParNew 收集器

ParNew 收集器实质上是 Serial 收集器的多线程并行版本  

除了同时使用多条线程进行垃圾收集之外，其余的行为几乎和Serial收集器一致

它是除了Serial收集器外，唯一能和CMS收集器配合工作的

ParNew收集器 在单核处理器环境中绝对不会有比Serial收集器更好的效果，它更适用于多核处理器环境

### Parallel Scavenge收集器

是一款基于标记-复制算法的多线程新生代收集器

与其他收集器致力于缩短垃圾回收时用户线程的停顿时间不同，这个收集器的目标是达到一个可控制的吞吐量

吞吐量就是处理器用户运行用户代码的时间和处理器总消耗时间的比值，即：
$$
吞吐量= 运行用户代码时间 /(运行用户代码时间+运行垃圾收集时间)
$$
也就是说想要提高吞吐量，就需要减少垃圾收集的时间

停顿时间越短就越适合需要与用户交互或需要保证服务响应质量的程序，良好的响应速度能提升用户体验  

高吞吐量则可以最高效率地利用处理器资源

由于与吞吐量关系密切， Parallel Scavenge 收集器也经常被称作“吞吐量优先收集器”  
