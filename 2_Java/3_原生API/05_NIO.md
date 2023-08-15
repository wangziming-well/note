# NIO概述

很多时候应用程序的吞吐率受制于I/O。程序读/写一个数据所花费的时间和计算一个数据所花费的时间甚至不是一个数量级。所以有的时候优化程序的I/O可能比优化计算带来的性能提示大得多。

但是JVM自身没能发挥操作系统的I/O性能。操作系统与Java基于流的I/O模型有些不匹配。操作系统I/O时移动的是大块数据(缓冲区)，但JVM的I/O类往往操作小块数据:单个字节、文本等。

所以一在操作系统索尼过来整个缓冲区数据后，`java.io`的流数据类会花费大量时间将它们拆成小块，往往拷贝一个小块就要经过几层对象。在最终使用的时候，我们可能还需要将这些小块数据拼起来。

所以JDK1.4的`java.nio`包中引入新的Java I/O类库。NIO使用的I/O模型与操作系统I/O更接近，使用连续的大块缓冲区作为数据传输的载体。这样NIO就能充分发挥操作系统I/O的性能。

todo：简述Buffer 、Channel、Selector之间的作用和关系

# Buffer

Buffer是`java.nio`的构造基础，它是对缓冲区的抽象。

一个`Buffer`对象是固定数量的数据的容器。作为缓冲区。同一个`Buffer`对象可以反复写满和释放(读取)数据。

`Buffer`由数据和可以高效访问以及操纵这些数据的四个索引组成：

~~~java
// Invariants: mark <= position <= limit <= capacity
private int mark = -1; //标记，可以记录一个position位置
private int position = 0; //位置:起到游标的作用，客户端只能访问Buffer在position处的数据。即下一个要被读/写的元素的索引
private int limit; //界限:限制游标的活动范围的最大上界。缓冲区第一个不饿能被读/写的索引位置
private int capacity; //容量，缓冲区能够容纳的数据元素的最大数量。这一容量在缓冲区创建时被设定，且永远不变。
~~~

下面是`Buffer`与这四个索引相关的方法：

~~~java
int capacity(); //返回capacity值
int position(); //返回position值
Buffer position(int newPosition); // 设置position值
int limit(); //返回limit值
Buffer limit(int newLimit); //设置limit值
Buffer mark(); //将当前position值记录为mark值
Buffer reset(); //将当前位置position为先前标记的位置(mark值)。
Buffer clear(); //清除此缓冲区。position设置为0，limit设置为当前capacity，mark重置为-1。此方法不会真正删除Buffer存储的数据，而是通过重置索引，指示通道可以重新记录数据。只有当通道重新记录数据时，旧的记录才会被覆盖。将缓冲区且切换为写模式
Buffer flip(); //翻转此缓冲区。limit设置为当前position，然后将position设置为0。mark重置为-1。此方法用于写入完数据后，准备从缓冲区读取数据。即将缓冲区切换为读模式
Buffer rewind(); //倒带此缓冲区。position设置为0，mark重置为 -1.重新读/写该缓冲区
int remaining(); //返回剩余可操作的元素数: (limit - position )值
boolean hasRemaining(); //判断是否有剩余: 当 position < limit 时返回true
~~~

除此之外，它还提供了一些缺省方法，以供其子类在写入和获取缓冲区数据时使用：

~~~java
final int nextGetIndex();
final int nextGetIndex(int nb);
final int nextPutIndex();
final int nextPutIndex(int nb);
//返回当前position，并使position的值增加1或者指定的值
~~~

## 继承体系

`Buffer`的继承体系如下：

![Buffer](https://gitee.com/wangziming707/note-pic/raw/master/img/Buffer.png)

除了布尔类型，八大数据类型的其他七种类型都有对应的`Buffer`:`ByteBuffer`、`CharBuffer`等。但是他们都是抽象类，他们还有具体的实现类，如`HeapByteBuffer`、`DirectByteBuffer`等

其中`HeapXXXBuffer`的数据维护在JVM堆内存中；而`BirectXXXBuffer`的数据维护在操作系统的内存中，是堆外内存，不受GC控制。

* `HeapXXXBuffer`：由于内容维护在jvm里，所以把内容写进buffer里速度会快些；并且，可以更容易回收。

* `DirectXXXBuffer`：是真正与通道交互数据的缓冲区

## 获取Buffer实例

Buffer体系中的所有类的构造器的访问权限都是缺省的，即只有在`java.nio`包中能直接创建`Buffer`的实例。

所以客户端没办法通过直接创建获取`Buffer`实例，只能通过`XXXXBuffer`提供的静态方法来获取`XXXXBuffer`实例，以`ByteBuffer`为例：

~~~java
public static ByteBuffer wrap(byte[] array);
public static ByteBuffer wrap(byte[] array,int offset, int length);
//获取以指定byte数组为缓冲区的HeapByteBuffer，可以指定要作为缓冲区的array的偏移量和长度
public static ByteBuffer allocate(int capacity);
//获取指定大小的空HeapByteBuffer
ByteBuffer allocateDirect(int capacity);
//获取指定大小的空DirectByteBuffer
~~~

## ByteBuffer

`Buffer`对外提供了操纵游标相关的方法

`ByteBuffer`等直接继承`Buffer`的抽象类，在`Buffer`的基础上，对外提供了向缓冲区写入、读取数据和操纵缓冲区的方法。

我们以`ByteBuffer`方法为例，了解这些方法：

* 在当前缓冲区基础上创建新缓冲区：

~~~java
public abstract ByteBuffer slice();
/**
创建一个新的字节缓冲区，其内容是此缓冲区内容的共享子序列。
新缓冲区的内容将从此缓冲区的当前position开始。
对此缓冲区内容的更改将在新缓冲区中可见，反之亦然;两个缓冲区的position、limit和mark值将是独立的。
新缓冲区的position将为零，其capacity和limit将是此缓冲区中剩余的字节数，并且其mark为-1。
当且仅当此缓冲区是直接缓冲区时，新缓冲区将是直接的，并且当且仅当此缓冲区是只读时，新缓冲区将是只读的。
*/
public abstract ByteBuffer duplicate();
/**
创建一个共享此缓冲区内容的新字节缓冲区。新缓冲区的内容将共享此缓冲区的内容。
对此缓冲区内容的更改将在新缓冲区中可见，反之亦然;
两个缓冲区的位置、限制和标记值将是独立的。新缓冲区的容量、限制、位置和标记值将与此缓冲区的容量、限制、位置和标记值相同。
当且仅当此缓冲区是直接缓冲区时，新缓冲区将是直接的，并且当且仅当此缓冲区是只读时，新缓冲区将是只读的。
*/
public abstract ByteBuffer asReadOnlyBuffer();
/**
创建一个共享此缓冲区内容的新只读字节缓冲区。
新缓冲区的内容将是此缓冲区的内容。对此缓冲区内容的更改将在新缓冲区中可见;但是，新缓冲区本身将是只读的，不允许修改共享内容。
新缓冲区的capacity、limit、position和mark将与此缓冲区的相同。
*/
~~~

* 读、写数据：

~~~java
public abstract byte get();
//读取缓冲区当前position位置的字节数据，并使position增加1
public abstract ByteBuffer put(byte b);
//向缓冲区的当前position位置写入指定字节数据，并使position增加1
public abstract byte get(int index);
//获取指定索引处的字节数据
public abstract ByteBuffer put(int index, byte b);
//向缓冲区的指定索引处写入指定字节数据
public ByteBuffer get(byte[] dst, int offset, int length);
public ByteBuffer get(byte[] dst);//相当于get(dst,0,dst.length)
//从缓冲区向指定的字节数组dst写入字节数据。offset和length指定写入到dst中数据的偏移量和长度。并且此缓冲区的position将增加length。
//如果缓冲区中剩余的字节数少于满足请求所需的字节数，也就是说，如果length>remaining(), 则不会传输任何字节，并且会抛出 BufferUnderflowException。
public ByteBuffer put(ByteBuffer src);
//将给定源缓冲区src中剩余的字节传输到此缓冲区中
//如果src.remaining() > this.remaining(),抛出BufferOverflowException异常
public ByteBuffer put(byte[] src, int offset, int length);
ByteBuffer put(byte[] src); //相当于put(src,0,src.length)
//把指定源字节数组src中从offset到(offset+length)的数据写入缓冲区
//如果length>reemaining()则抛出BufferOverflowException
~~~

* 访问字节数组

~~~java
public final boolean hasArray();
//告知此缓冲区是否由可访问的字节数组支持。如果此方法返回 true，则可以安全地调用array()和arrayOffset()方法。
public final byte[] array();
//返回支持此缓冲区的字节数组。对返回数组的修改将导致缓冲区的修改，反之亦然
public final int arrayOffset();
//返回此缓冲区后备数组中缓冲区第一个元素的偏移量。如果此缓冲区由数组支持，则缓冲区位置 p 对应于数组索引p+arrayOffset()
~~~

* 切换到写模式

~~~java
public abstract ByteBuffer compact();
//将缓冲区的剩余部分(当前position到limit的数据)复制到缓冲区开头
//防止在读取该缓冲区数据还没有读完的情况，就要就向该缓冲区写入数据的情况下，数据的丢失:
//即更加安全的将缓冲区从读模式切换到写模式使用示例：
buf.clear();          // Prepare buffer for use  
while (in.read(buf) >= 0 || buf.position != 0) {      
    buf.flip();      
    out.write(buf);      
    buf.compact();    // In case of partial write  
}
~~~

* 直接读/写其他基本数据类型

其中Xxxx/xxxx 为除byte、boolean外的其他6中基本数据类型

n取决于xxxx的类型：比如xxxx是char时n为2，xxxx为int时n为4

~~~java
public abstract xxxx getXxxx();
//从缓冲区读取n个字节，然后把这n个字节转换为xxxx类型返回，并且缓冲区position增加n。
public abstract ByteBuffer putXxxx(xxxx value);
//将xxxx类型的value转换为n个字节，以当前字节顺序写入的当前position，并且缓冲区position增加n。
public abstract char getXxxx(int index);
//在给定索引处读取n个字节，并转换为xxxx类型返回
public abstract ByteBuffer putXxxx(int index, xxxx value);
//将给定value转换为n个字节，以当前字节顺序写入到给定索引处
public abstract CharBuffer asXxxxBuffer();
//创建当前字节缓冲器的xxxx缓冲器视图，新视图从当前buffer的position处开始。对当前buffer的修改对新视图可见。
//视图的position为0，capacity和limit为当前buffer的remaining值
//当且仅当此缓冲区是直接缓冲区时，新缓冲区将是直接的，并且当且仅当此缓冲区是只读时，新缓冲区将是只读的。
~~~

## 字节顺序

非byte类型的基本类型，除了布尔型都是由几个字节组合而成的。如`char`由两个字节组合而成，`long`由8个字节组合而成。它们是以连续字节序列的形式存储在内存中的。

而多个字节值被存储在内存中的方式就被称为字节顺序,字节顺序有两种：

* 大端字节顺序：最高字节存储在低位地址
* 小端字节顺序：最低字节存储在低位地址

假设内存地址从左往右增加，32位的int值为58700999的数据对应的字节值为 ： 0x037fb4c7

如果以上字节按照03 ->7f ->b4 ->c7的顺序存储在内存中，那么它就是按照大端字节顺序存储

如果以上字节按照c7->b4 ->7f ->03的顺序存储在内存中，那么它就是按照小端字节顺序存储

大端字节顺序符合我们人类的阅读顺序，而小端字节存储符合计算机程序读取内存的顺序



在`java.nio`中，字节顺序由`ByteOrder`类封装：

~~~java
public final class ByteOrder {
    public static final ByteOrder BIG_ENDIAN = new ByteOrder("BIG_ENDIAN");
    public static final ByteOrder LITTLE_ENDIAN = new ByteOrder("LITTLE_ENDIAN");
    public static ByteOrder nativeOrder();
    public String toString();
}
~~~

`ByteOrder`定义了决定从缓冲区存储或者检索多字节数据时使用哪一种字节顺序的常量：大端字节`ByteOrder.BIG_ENDIAN`和小端字节`ByteOrder.LITTLE_ENDIAN`

`Buffer`的实现提供了以下方法访问其`ByteOrder`：

~~~Java
ByteOrder order();
//获取当前缓冲区的字节顺序
~~~

`ByteBuffer`默认的字节顺序是大端字节顺序，其他基本类型的缓冲区默认为

但只有`ByteBuffer`可以设置其字节顺序：

~~~JAVA
ByteBuffer order(ByteOrder bo);
//设置当前缓冲区的字节顺序
~~~

## 直接缓冲区

操作系统进行I/O操作的目标内存区域必须是连续的字节序列。而在JVM中，字节数组可能不是在内存中连续存储的。所以`HeapByteBuffer`并不能直接与操作系统进行I/O交互，处于这样的原因，引入了直接缓冲区的概念。直接缓冲区被用于与通道和本地I/O交互。直接缓冲区在JVM堆外划出一块连续的字节序列作为缓冲区。告知操作系统来直接释放或者填充该内存区域。

可以通过各基本类型对应的缓冲区的静态方法`allocateDirect(int capacity)`获取其对应的直接缓冲区。

如`ByteBuffer.allocateDirect()`方法将返回一个`ByteBuffer`,其实际类型为`DirectByteBuffer`

直接字节缓冲区`DirectByteBuffer`支持JVM可用的最高效I/O机制。通道只于直接字节缓冲区进行直接交互。

如果向通道中传递一个非直接的缓冲区`HeapByteBuffer`用于写入，那么通道在每次调用中会隐含下面操作：

* 创建一个临时的`DirectByteBuffer`对象
* 将`HeadpByteBuffer`的内容复制到`DirectByteBuffer`对象中
* 使用`DirectByteBuffer`执行底层的I/O操作
* 临时的直接缓冲区离开作用域，被GC回收

所以这可能导致通道在每次读写时复制并产生大量对象。所以要使用的缓冲区可能需要进行高频读写时，应该尽量使用直接缓冲区

但是使用直接缓冲区的成本会比堆内缓冲区高得多，直接缓冲区使用的内存时调用本地方法的代码分配的，绕过了JVM堆栈。建立和销毁直接缓冲区会比堆内缓冲区更耗时。

# Channel

Channel通道时`java.nio`的重要组件，它提供于I/O服务的直接连接。用于在字节缓冲区和通道另一侧的实体(通常为文件或者socket)直接有效地传输数据。

借助通道，可以用最小的总开销来访问操作系统本身的I/O服务。缓冲区则是通道内部用来发送和接收数据的端点

其Channel抽象类和接口的继承体系如下：

![Channel](https://gitee.com/wangziming707/note-pic/raw/master/img/Channel.png)

`Channel`有四个主要的实现，其签名如下：

~~~java
public abstract class FileChannel extends AbstractInterruptibleChannel implements SeekableByteChannel, GatheringByteChannel, ScatteringByteChannel;
public abstract class DatagramChannel extends AbstractSelectableChannel implements ByteChannel, ScatteringByteChannel, GatheringByteChannel, MulticastChannel;
public abstract class SocketChannel extends AbstractSelectableChannel implements ByteChannel, ScatteringByteChannel, GatheringByteChannel, NetworkChannel;
public abstract class ServerSocketChannel extends AbstractSelectableChannel implements NetworkChannel;
~~~

通道的顶级接口`Channel`定义如下：

~~~java
public interface Channel extends Closeable {
    public boolean isOpen(); //检查通道是否打开
    public void close() throws IOException; //关闭一个打开的通道
}
~~~

而通道的具体行为由`Channel`的子接口和抽象类来规范

* `InterruptibleChannel`是一个标记接口，指示当前通道使用时该通道是可以可以中断的
* `WritableByteChannel&ReadableByteChannel`支持读写字节缓冲区
* `AbstractInterruptibleChannel`为可中断的通道实现提供所需的常用方法
* `AbstractSelectableChannel`为可选择的通道提供所需的常用方法

## 获取通道

I/O可以分为广义的两大类别：File I/O和Stream I/O 对应到Channel通道就分为文件通道和套接字通道

`java.nio`提供一个FileChannel类和三个socket通道：`SocketChannel`、`DatagramChannel`、`ServerSocketChannel`

`FileChannel`不能通过创建来获取`FileChannel`对象只能通过在一个打开的`FileInputStream`、`FileOutputStream`以及`RandomAccessFile`对象上调用`getChannel()`方法来获取:

~~~java
FileChannel channel = new FileOutputStream("data.txt").getChannel();//只写通道
FileChannel channel = new FileInputStream("data.txt").getChannel();//只读通道
FileChannel channel = new RandomAccessFile("data.txt", "rw").getChannel();//读写通道
~~~

而Socket通道可以通过各自的工厂方法创建：

~~~java
SocketChannel sc = SocketChannel.open();
sc.connect(new InetSocketAddress("127.0.0.1", 3306));
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.socket().bind(new InetSocketAddress(3307));
DatagramChannel dc = DatagramChannel.open();
~~~

## 使用通道

`WritableByteChannel`和`ReadableByteChannel`提供了write()和read()方法支持通道读写字节缓冲区：

~~~java
public interface WritableByteChannel extends Channel{
    public int write(ByteBuffer src) throws IOException;
    //从缓冲区读取字节序列写入到通道中，
}
public interface ReadableByteChannel extends Channel {
    public int read(ByteBuffer dst) throws IOException;
    //从此通道读取字节序列写入到给定的缓冲区,尝试从通道读取r个字节，其中r是缓冲区中剩余的字节数，即 dst.remaining()。
}
~~~

如果一个通道实现了`WritableByteChannel`和`ReadableByteChannel`中的一个，那么通道就是单向的。如果一个通道同时实现这两个接口，那么它就是双向的。

因为`ByteChannel`接口本身只继承了`ReadableByteChannel`和`WritableByteChannel`,聚集这两个接口的功能，它本身没有提供任何功能。所以实现了`ByteChannel`接口的通道就是双向通道。

通过Channel的继承体系我们发现，一个file通道和三个socket通道都是双向通道。这对socket通道来说没什么问题，因此网络通道一直是双向的。

但对file通道却是个问题，因为一个文件可以在不同的时候以不同的权限打开：

从`FileInputStream.getChannel()`方法获取到的`FileChannel`对象是只读的，但是从接口声明的角度来看它是双向的。在这样一个通道上调用`write()`方法会抛出异常

下面演示使用通道和缓冲区将一个通道的数据移动到另一个通道：

~~~java
ReadableByteChannel in = Channels.newChannel(System.in);
WritableByteChannel out = Channels.newChannel(System.out);
ByteBuffer buffer = ByteBuffer.allocate(16 * 1024);
while (in.read(buffer) != -1) { //操作1
    buffer.flip(); //操作2
    out.write(buffer); //操作3
    buffer.compact(); //操作4
}
~~~

我们来看下在操作1到操作4之间，缓冲区的数据和索引是怎样变化以完成反复读写数据的：

* 初始状态：buffer中没有数据，position为0，limit等于cap：

  ![BufferIndex0](https://gitee.com/wangziming707/note-pic/raw/master/img/BufferIndex0.png)

* 操作1：in通道将读入的数据写入buffer中，position变为写入的数据的长度：

  ![BufferIndex1](https://gitee.com/wangziming707/note-pic/raw/master/img/BufferIndex1.png)

* 操作2：翻转缓冲区，将其变为可读的状态，limit变为position，position变为0：

  ![BufferIndex2](https://gitee.com/wangziming707/note-pic/raw/master/img/BufferIndex2.png)

* 操作3：out通道读取buffer中的数据到通道中，假设本次读取没有读取buffer中的所有数据，假设只读取了1个字节：

  ![BufferIndex3](https://gitee.com/wangziming707/note-pic/raw/master/img/BufferIndex3.png)

* 操作4：压缩此缓冲区，使其变为可写状态，同时保留未读取的数据：将未读数据复制到数组开头，position置为未读字节个数，limit置为capacity：

  ![BufferIndex4](https://gitee.com/wangziming707/note-pic/raw/master/img/BufferIndex4.png)

当然我们也可以使用如下方式保障每次从缓冲区中读取数据时，一定读取完缓冲区所有的数据才继续写入：

~~~java
ReadableByteChannel in = Channels.newChannel(System.in);
WritableByteChannel out = Channels.newChannel(System.out);
ByteBuffer buffer = ByteBuffer.allocate(16 * 1024);
while (in.read(buffer) != -1){
    buffer.flip();
    while (buffer.hasRemaining()) //确保缓冲区数据被完全读取
        out.write(buffer);
    buffer.clear();
}
~~~



# Path和Files

todo
