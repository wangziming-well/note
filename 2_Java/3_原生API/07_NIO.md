# NIO概述

很多时候应用程序的吞吐率受制于I/O。程序读/写一个数据所花费的时间和计算一个数据所花费的时间甚至不是一个数量级。所以有的时候优化程序的I/O可能比优化计算带来的性能提示大得多。

但是JVM自身没能发挥操作系统的I/O性能。操作系统与Java基于流的I/O模型有些不匹配。操作系统I/O时移动的是大块数据(缓冲区)，但JVM的I/O类往往操作小块数据:单个字节、文本等。

所以一在操作系统索尼过来整个缓冲区数据后，`java.io`的流数据类会花费大量时间将它们拆成小块，往往拷贝一个小块就要经过几层对象。在最终使用的时候，我们可能还需要将这些小块数据拼起来。

所以JDK1.4的`java.nio`包中引入新的Java I/O类库。NIO使用的I/O模型与操作系统I/O更接近，使用连续的大块缓冲区作为数据传输的载体。这样NIO就能充分发挥操作系统I/O的性能。

Java的NIO体系主要有三个主体：`Buffer`、`Channel`、`Selector`

* `Buffer`缓冲区是数据的载体的封装，使得nio能够按份传递数据，提高数据传输的效率

* `Channel`是对数据传输操作的抽象，相比于传统I/O，它还提供了非阻塞式的I/O的功能

* `Selector`实现了对非阻塞I/O的多路复用，可以大大提高I/O系统的吞吐率和性能。

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

JDK1.7`FileChannel`添加了两个静态方法，以直接创建文件对应的通道实例：

~~~java
public static FileChannel open(Path path, Set<? extends OpenOption> options,FileAttribute<?>... attrs);
public static FileChannel open(Path path, OpenOption... options);
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

## 关闭通道

与缓冲区不同，通道不能被反复使用。一个打开的通道就代表与一个特定I/O服务的特定连接。当通道关闭时，这个连接会丢失，然后通道将不再连接任何东西。

调用通道的`close()`方法时，可能会导致线程阻塞，哪怕该通道处于非阻塞模式。通道关闭时的阻塞行为取决于操作系统或者文件系统。

可以在一个通道上多次调用`close()`方法：如果第一个线程在`close()`方法中阻塞，那么在它完成关闭通道前，其他任何调用这个通道的`close()`方法的线程都会阻塞。后续在该已关闭的通道上调用`close()`不会产生任何操作，只会立即返回。

### InterruptibleChannel的中断关闭

`InterruptibleChannel`通道的中断行为可能会导致通道关闭：如果一个线程在一个`InterruptibleChannel`上被阻塞的同时被中断，那么该通道将被关闭，该被阻塞线程也会产生一个`ClosedByInterruptException`

此外如果一个线程的中断状态被设置并且该线程试图访问访问一个`InterruptibleChannel`通道，那么这个通道将被立即关闭

## Scatter/Gather

操作系统支持本地矢量I/O，进程只需要一个系统调用，就可以把一连串缓冲区地址传递给操作系统。然后，内核就可以顺序填充或者排干多个缓冲区，读的时候就把数据发散到多个用户空间缓冲区，写的时候再从多个缓冲区汇聚起来。这样用户进程就不必多次执行系统调用，这样开销很大。

`ScatteringByteChannel`和`GatheringByteChannel`就是基于这样技术的接口，它们的定义如下：

~~~java
public interface ScatteringByteChannel extends ReadableByteChannel{
    public long read(ByteBuffer[] dsts, int offset, int length) throws IOException;
    public long read(ByteBuffer[] dsts) throws IOException;
}
public interface GatheringByteChannel extends WritableByteChannel{
    public long write(ByteBuffer[] srcs, int offset, int length)
        throws IOException;
    public long write(ByteBuffer[] srcs) throws IOException;
}
~~~

这两个接口为通道提供了在I/O时读取并写入多个缓冲区；将多个缓冲区的数据组合发送出去。

这使我们可以委托操作系统来完成将读取到的数据分开存放到多个缓冲区，或者将不同的数据合并成一个整体的操作。

## FileChannel

`FileChannel`类实现常用的`read/write`和`scatter/gather`操作，同时也提供了专用于文件的方法。

### 文件和位置

它除了实现`GatheringByteChannel`, `ScatteringByteChannel`,还实现了`SeekableByteChannel`接口，

`SeekableByteChannel`是一个维护了当前位置并且允许位置变动的字节通道，其API如下：

~~~Java
public interface SeekableByteChannel
    extends ByteChannel{
    int read(ByteBuffer dst);//从通道中读取字节序列到给定的buffer，读取是从通道的当前位置开始，读取后通道位置position值将增加读取的字节数。当position值达到文件大小的值(size())，方法返回-1;
    int write(ByteBuffer src);//从给定的buffer中读取字节序列写入到通道中，写入是从通道的当前位置开始的，写入后通道位置position值将增加实际写入的字节数. position前进到超过文件大小的值时，该文件会扩展以容纳新写入的字节。
    long position() throws IOException;	//返回通道的当前位置
    SeekableByteChannel position(long newPosition); //设置此通道的位置
    long size() throws IOException; //返回此通道连接到的实体的当前大小
    SeekableByteChannel truncate(long size); //将此通道连接到的实体截断为给定大小，size为截断后新实体的大小，position会被设置为size的值
}
~~~

除此之外，`FileChannel`还提供如下操作文件和位置API：

~~~java
public abstract void force(boolean metaData);//通知通道强制将对文件的修改缓冲都应用到磁盘的文件上，如果文件不在本地机器上，则不能保证方法生效。metaData表示在方法返回值前文件的元数据（metadata）是否也要被同步更新到磁盘
public abstract int read(ByteBuffer dst, long position);
public abstract int write(ByteBuffer src, long position);
//在指定的position处读写，不会改变当前文件的position;也被称为绝对读写操作
~~~

在文件末尾之外的position进行一个绝对读操作，`read()`会返回-1。在超出文件大小的position上进行一个绝对写操作会导致文件增加以容纳正在被写入的新字节。

### 文件锁

`FileChannel`提供文件锁定的功能，其注意，文件锁的对象是文件，文件锁的持有者是JVM进程，而不是线程和通道。

这意味着，同一个JVM中一个线程申请了文件锁，另一个线程仍然可以访问到该文件。

~~~java
public abstract FileLock lock(long position, long size, boolean shared);
public final FileLock lock(); // lock(0L, Long.MAX_VALUE, false);
public abstract FileLock tryLock(long position, long size, boolean shared);
public final FileLock tryLock(); // tryLock(0L, Long.MAX_VALUE, false)

~~~

参数说明：

* `position`:锁定区域开始的位置
* `size`:锁定区域的大小
* `shared`:如果为 true，则请求共享锁,false 请求独占锁

`position`和`size`参数决定了文件锁锁定的区域，锁定区域不一定要限制在文件的size值内，锁可以拓展超出文件尾。所以我们可以把待写入数据的区域锁定，也可以锁定一个不包含任何文件内容的区域，比如从文件最后一个字节以外的区域。如果以后文件增长到达这块区域，那么文件锁能作用到文件新增部分的区域。

`lock()`方法是一个阻塞方法，如果请求的文件锁的范围已经被其他程序/应用占用。那么该方法会阻塞，直到文件锁的范围数据被释放。

该阻塞方法受中断语义控制：

* 如果通道被另外一个线程关闭，该暂停线程将被唤醒并产生一个`AsynchronousCloseException`异常
* 如果该暂停线程被直接中断，它将唤醒并产生一个`FileLockInterruptionException`异常
* 如果在调用`lock()`方法时线程的中断状态已经被设置，也会产生`FileLockInterruptionException`异常

而`tryLock()`方法是`lock()`方法的非阻塞变体。它们作用相同，不过`tryLock()`请求锁如果不能立刻申请到会返回一个null，而不是阻塞。

调用`tryLock()`和`lock()`方法返回一个`FileLock`对象，以下是其完整API：

~~~Java
public final FileChannel channel();
public Channel acquiredBy();
public final long position();
public final long size();
public final boolean isShared();
public final boolean overlaps(long position, long size);
public abstract boolean isValid();
public abstract void release();
public final void close();
~~~

`FileLock`类封装一个锁定的文件区域。`FileLock`对象由`FileChannel`创建并且总是关联到那个特定的通道实例。可以通过`channel()`、`acquiredBy`获取创建它的通道。

一个`FileLock`对象创建之后即有效，直到它的`release()`或者`close()`方法被调用或者它关联的通道被关闭或者JVM关闭时才会失效。可以通过调用它的`isValid()`方法测试有效性。

一个锁的有效性会随时间改变，但是它的其他属性：position、size和exclusivity，在创建时就被确定，不会随着时间而改变。

可以通过调用`isShared()`方法来判断一个锁是共享的还是独占的。需要注意：调用`FileChannel.lock()`时，`isShared`传true，并不意味着获得的`FileLock`实例的`isShared()`一定返回true。因为操作系统或者文件系统可能不支持共享锁。

所以我们在申请一个共享锁后，最好再调用`FlieLock.isShared()`方法进行判断。

我们可以通过调用`overlaps()`方法来查询当前锁是否与指定的文件区域有重叠。可以用来判断你拥有的锁和指定的区域是否有交叉。

### 内存映射文件

传统的文件I/O是通过用户进程发布`read()`和`write()`系统调用来传输数据的。为了在内核空间的文件系统页与用户空间的内存区之间移动数据，一次以上的拷贝操作总是免不了的。这是因为，在文件系统页与用户缓冲区之间没有一一对应关系。

操作系统提供了一种特殊类型的I/O操作，允许用户进程最大限度地利用面向页的系统I/O特性，并完全摒弃缓冲区拷贝，这就是内存映射I/O。

内存映射I/O使用文件系统建立从用户空间到可用文件系统页的虚拟内存映射。这样：

* 用户进程把文件数据当作内存，无需发布`read()/write()`系统调用
* 用户进程读取映射内存空间，页错误会自动产生，从而将文件数据从磁盘读取进内存。如果用户修改了映射内存空间，相关页会自动标记为脏，随后刷新到磁盘，文件得到更新。
* 操作系统的虚拟内存子系统会对页进行高速缓冲，自动根据系统负载进行内存管理
* 大型文件使用映射，无需消耗大量内存，即课进行数据拷贝。

所以内存映射I/O会比常规I/O更高效，因为不需要做明确的系统调用。并且操作系统的虚拟内存可以自动缓冲内存页。这些页是由系统内存来缓存的，所以不会消耗JVM内存堆。

`FileChannel`支持对内存映射I/O的应用，它提供的`map()`方法如下：

~~~java
public abstract MappedByteBuffer map(MapMode mode,long position, long size);
~~~

该方法会创建一个由磁盘文件支持的虚拟内存映射，将这个虚拟内存空间封装为`MappedByteBuffer`对象返回。它的参数说明如下：

* mode:指定虚拟映射内存打开的模式，可以指定一下模式：
  * `FileChannel.MapMode.READ_ONLY`:只读模式，对应的通道可以是读写通道、只写通道。
  * `FileChannel.MapMode.READ_WRITE`:读写模式，对应的通道必须是读写通道
  * `FileChannel.MapMode.PRIVATE`:写时拷贝模式，对应的通道必须是读写通道
* position:指定映射文件的起始位置
* size:指定映射文件的大小

例如，要映射100到299位置的字节，可以使用如下代码：

~~~java
buffer = fileChannel.map(FileChannel.MapMode.READ_ONLY,100,300);
~~~

要映射整个文件，可以用：

~~~java
buffer = fileChannel.map(FileChannel.MapMode.READ_ONLY,0,fileChannel.size());
~~~

与文件锁的范围机制不一样，映射文件的范围不应超过文件的实际大小。如果请求一个超出文件大小的映射。文件会被增大以匹配映射的大小。

#### 写时拷贝

COW(copy-on-write)是文件系统中常用的一种优化策略

常规情况下，如果有多个进程读取同一个文件，那么文件系统会为每个系统拷贝一份内存，但是并不是每一个进程都会修改文件的，可能大部分进程都是共用一份文件，那么为每个进程都复制一份内存就会很浪费，特别是访问的文件很大时。并且使用这种全部拷贝的策略，在并发访问的情况下是线程不安全的，会发生脏读和写覆盖。所以使用这种策略并发访问文件必须加文件锁。

使用有了COW的策略后，文件系统只会为多个文件访问者分配一个共享的拷贝内存，多个访问者在读取文件内容时，访问的是同一个内存，只有在访问者打算修改文件时，文件系统才会为访问者拷贝一份访问者独享的文件内存供其修改。这样够延迟必要的拷贝，能够减少内存的开销，特别是大部分访问者只读的情况下。并且能够在不加锁的情况下保证了访问的线程安全。

在请求映射文件时，mode可以传`FileChannel.MapMode.PRIVATE`,以获得一个写时拷贝文件映射，如果以`REIVATE`模式创建多个对相同文件的`MappedByteBuffer`映射内存。

#### MappedByteBuffer

`MappedByteBuffer`的行为很类似基于内存的缓冲区，只不过该对象的数据元素存储在磁盘上的一个文件中

调用`get()`方法会从磁盘f文件中获取数据，`get()`方法实时反应文件的当前内容。

调用`put()`方法会更新磁盘上的文件，这样的更新对其他文件的访问方可见。

除此之外，它还有其独有的方法：

~~~java
public final boolean isLoaded();
public final MappedByteBuffer load();
public final MappedByteBuffer force();
~~~

当我们为一个文件建立了虚拟内存映射之后，文件数据通常不会立即被从磁盘读取到内存，只有当访问者执行访问操作时，文件系统才会把相应位置的文件数据加载到内存(通常是文件的若干页)

我们可以选择先把文件的所有页都读进内存，以实现最低的访问延迟，此时它的访问速度就和访问一个基于内存的缓冲区一样了。

`load()`方法会加载整个文件以使它常驻内存。在一个映射缓冲区上调用`load()`方法会是一个代价高的操作，它会导致大量的页调用。

对那些要求近乎实时访问的程序，就可以使用`load()`进行预加载。

可以使用`isLoad()`方法判断当前映射缓冲区对应的文件是否完全常驻内存。

`force()`方法和`FileChannel`类的同名方法作用相似，会强制将缓冲区上的更改从内存冲刷到磁盘上。

当用 `MappedByteBuffer `对象来更新一个文件，应使用` MappedByteBuffer.force()`而非 `FileChannel.force()`，因为通道对象可能不清楚通过映射缓冲区做出的文件的全部更改。   

`force()`方法只有在`MapMode.READ_WRITE`模式下起作用。

### 传输

由于经常需要从一个位置将文件数据传输到另一个位置。`FileChannel`提供了一些方法来提高传输过程的效率：

~~~java
public abstract long transferTo(long position, long count, WritableByteChannel target);
//将当前文件数据传输到给定可写通道，传输的数据从position开始，不超过count个字节。
public abstract long transferFrom(ReadableByteChannel src,long position, long count);
//从可读通道中读取数据到当前文件通道，读取的数据从src的指定position开始，不超过count个字节
~~~

示例

文件复制：

~~~java
FileChannel src = new RandomAccessFile("srcPath","rw").getChannel();
FileChannel target = new RandomAccessFile("targetPath","rw").getChannel();
src.transferTo(0,src.size(),target);
target.close();
src.close();
~~~

输出文件内容到打印台：

~~~java
FileChannel src = new RandomAccessFile("srcPath","rw").getChannel();
WritableByteChannel target = Channels.newChannel(System.out);
src.transferTo(0,src.size(),target);
~~~

键盘输入键入到文件中：

~~~java
FileChannel target = new RandomAccessFile("ccc.txt","rw").getChannel();
ReadableByteChannel src = Channels.newChannel(System.in);
target.transferFrom(src,0,100);
~~~

## Socket通道

Socket通道是模拟网络Socket的通道类。Socket通道可以以非阻塞模式允许并且是可选择的。这可以为程序提供巨大的可伸缩性和灵活性。

全部socket通道类(`DatagramChannel `,`SocketChannel `,`ServerSocketChannel `)都拓展于`AbstractSelectableChannel`，这意味着我们可以用一个Selector对象来执行socket通道的有条件的选择。

`DatagramChannel`和`SocketChannel`实现定义了读写功能，但`ServerSocketChannel`不实现。`ServerSocketChannel`负责监听传入的连接和创建新的`SocketChannel`队形，它本身不传输数据。

全部socke通道类在被实例化时都会创建一个对等的socket对象，这些对象是`java.net`的类(`Socket`、`ServerSocket`、`DatagramSocket`)可以通过通道的`socket()`方法获取通道关联的socket对象。

此外，这三个`java.net`类现在都有`getChannel()`方法。

### 非阻塞模式

Socket通道可以在非阻塞 模式下运行。在所有socket通道的父类`SelectableChannel`中，提供了关于阻塞模式的相关API：

~~~java
public abstract SelectableChannel configureBlocking(boolean block)throws IOException;
public abstract boolean isBlocking();
public abstract Object blockingLock();
~~~

非阻塞I/O于可选择性紧密相连，这也是管理阻塞的API定义在`SelectableChannel`中的原因。

可以调用`configureBlocking()`来设置通道的阻塞模式，通过`isBlocking()`方法判断当前通道的阻塞模式。

`blockingLock()`方法返回一个锁，这个锁是通道内部在执行`configureBlocking()`时使用同步代码块时所使用的锁。

所以如果我们希望在一小段时间内保证通道的阻塞模式不被修改，那么就可以使用`blockingLock()`方法的返回值作为锁，

这样在执行该锁引导的临界区时，通道的阻塞模式不会被修改，因为在这个临界区执行时，`configureBlocking()`对锁的申请不会成功。

### ServerSocketChannel

`ServerSocketChannel`部分API如下：

~~~java
public static ServerSocketChannel open() throws IOException;
public abstract ServerSocket socket();
public final ServerSocketChannel bind(SocketAddress local);
public abstract ServerSocketChannel bind(SocketAddress local, int backlog);
public abstract SocketChannel accept();
public final int validOps()

~~~

`ServerSocketChannel`是一个基于通道的socket监听器。和`java.net.ServerSocket`执行相同的基本任务，不过它增加了通道语义，能在非阻塞模式下运行。

用静态的`open()`方法可以创建一个新的`ServerSocketChannel`对象，其对应一个未绑定的`ServerSocket`对象。可以通过`socket()`方法获取通道对应的`ServerSocket`

`socket()`方法返回这个通道对应的`ServerSocket`实例

可以直接使用`bind()`方法将管道对应的`ServerSocket`绑定到指定端口也可以取出对应的socket并将其绑定到一个端口以开始监听连接

`accept()`方法和`ServerSocket`的同名方法不同，它返回的是一个socket管道，并且这个方法可以在非阻塞模式下运行，此时，如果没有传入连接在等待，它会立即返回null

下面是一个非阻塞模式的`accept()`的使用模板：

~~~java
ByteBuffer buffer = ByteBuffer.wrap("收到信息".getBytes());
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.socket().bind(new InetSocketAddress(6666));
ssc.configureBlocking(false);
while (true) {
    System.out.println("等待连接");
    SocketChannel sc = ssc.accept();
    if (sc == null) {
        Thread.sleep(2000);
    } else {
        System.out.println("收到连接请求:" + sc.getRemoteAddress());
        buffer.rewind();
        sc.write(buffer);
        sc.close();
    }
}
~~~

`validOps() `配合选择器使用

### SocketChannel

`SocketChannel`提供类似`Socket`的服务。每个 `SocketChannel `对象创建时都是同一个对等的` java.net.Socket `对象串联的。

除了管道通用的`read/write`方法外，它还提供了于`Socket`方法类似的API：

~~~java
public static SocketChannel open();
public static SocketChannel open(SocketAddress remote);
public final int validOps();
public abstract SocketChannel bind(SocketAddress local);
public abstract <T> SocketChannel setOption(SocketOption<T> name, T value);
public abstract SocketChannel shutdownInput() throws IOException;
public abstract SocketChannel shutdownOutput() throws IOException;
public abstract Socket socket();
public abstract boolean isConnected();
public abstract boolean isConnectionPending();
public abstract boolean connect(SocketAddress remote) throws IOException;
public abstract boolean finishConnect() throws IOException;
public abstract SocketAddress getRemoteAddress() throws IOException;
~~~

静态的`open()`方法可以创建一个新的`SocketChannel`对象，在新创建的`SocketChannel`上调用`socket()`方法能够返回它对应的`Socket`对象；在该`Socket`上调用`getChannel()`方法则能返回其对应的`SocketChannel`。

直接创建的`Socket`对象不会关联`SocketChannel`对象，它们的`getChannel()`方法只返回null

在阻塞模式和非阻塞模式下`connect()`有不同的行为：

* 阻塞模式下：和`Socket`的连接行为相同，此方法的调用将阻塞，直到建立连接或发生 IO 错误
* 非阻塞模式下：此方法将启动非阻塞连接操作。
  * 如果立即建立连接，则此方法返回 true。
  * 如果没有立即建立连接，此方法返回 false，此时socket通道将变为可连接状态，稍后必须通过调用 `finishConnect() `方法完成连接操作

`finishConnect()`用于在非阻塞模式下，调用了`connect()`方法返回false，socket通道变为可连接状态后调用。调用该方法可能出现以下情况：

* 如果`connect()`方法尚未被调用，那么将抛出一个`NoConnectionPendingException`异常
* 如果此通道处于非阻塞模式，且建立连接的过程正在进行，此方法立即返回 false。
* 如果此通道处于阻塞模式(在非阻塞模式下调用`connect()`后，socket通道又被切换回了阻塞模式)，则此方法将阻塞，直到连接完成或失败，并且始终返回 true 或引发描述失败的已检查异常
* 若连接建立过程已经完成。但`SocketChannel`内部状态仍然为连接中，那么该方法将更新内部状态为已连接，并返回true。

* 如果连接已经建立，将立即返回 true。

如果socket通道打开并连接时，`isConnected()`返回true

如果socket通道还在连接中，`isConnectionPending()`返回true，即当且仅当连接操作已在此通道上启动但尚未通过调用 finishConnect 方法完成时，才为 true

示例代码：

~~~Java
SocketChannel channel = SocketChannel.open();
channel.configureBlocking(false);
channel.connect(new InetSocketAddress("www.baidu.com", 80));
while (!channel.finishConnect()) {
    //连接未建立
}
//连接已建立
channel.write(ByteBuffer.wrap("你好".getBytes()));
channel.close();
~~~

### DatagramChannel

`DatargramChannel`对应`java.net`中的`DatagramSocket`,实现无连接协议UDP

~~~java
public static DatagramChannel open();
public static DatagramChannel open(ProtocolFamily family);
public abstract DatagramChannel bind(SocketAddress local);
public abstract DatagramChannel connect(SocketAddress remote);
public abstract DatagramChannel disconnect() throws IOException;
public abstract SocketAddress getRemoteAddress() throws IOException;
public abstract SocketAddress getLocalAddress() throws IOException;
public final int validOps() ;
public abstract SocketAddress receive(ByteBuffer dst);
public abstract int send(ByteBuffer src, SocketAddress target);
~~~

提供的api方法与`DatagramSocket`类似,但是注意其`send()/receive()`方法的底层实现仍是`write()/read()`

要使用`read()/write()`方法，该`DatagramChannel`必须指定了远程连接地址。

在使用`read()/receive()`时，如果提供的 ByteBuffer 没有足够的剩余空间来存放正在接收的数据包， 没有被填充的字节都会被悄悄地丢弃。

如果处于阻塞模式，`receive()`的行为和`DatagramSocket`类似，如果当前socket未收到数据包，将阻塞直至收到数据包或者异常。

如果处于非阻塞模式,`receive()`方法将返回0，或者buffer的字节数。

`DatagramChannel`的`connnet()`语义和`DatargramSocket`的同名方法语义类似，都是将对话限制在双方，而不是建立实际的连接。处于已连接状态的`DatargramChannel`将忽略非指定地址以外的地址发送的数据包，并且发送数据包时的地址也必须是指定地址。

示例：

服务端

~~~java
DatagramChannel channel = DatagramChannel.open();
channel.configureBlocking(false);
channel.bind(new InetSocketAddress(6666));
System.out.println("Listening on port 6666 for time requests");
ByteBuffer buffer = ByteBuffer.wrap("好的。".getBytes());
ByteBuffer receiveBuf = ByteBuffer.allocate(16);
while (true) {
    SocketAddress sa = channel.receive(receiveBuf);
    if (sa == null) {
        Thread.sleep(1000);
        continue;
    }
    System.out.println("收到來自" + sa+"的信息:"+new String(receiveBuf.array(),0,receiveBuf.position(), StandardCharsets.UTF_8));
    receiveBuf.clear();
    channel.send(buffer, sa);
    buffer.rewind();
}
~~~

客户端：

~~~Java
DatagramChannel channel = DatagramChannel.open();
channel.connect(new InetSocketAddress("localhost",6666));
channel.write(ByteBuffer.wrap("请求".getBytes()));
ByteBuffer buffer = ByteBuffer.allocate(16);
int read = channel.read(buffer);
System.out.println("received:"+ new String(buffer.array(),0,read, StandardCharsets.UTF_8));
~~~

## 管道

`java.nio.channels.Pipe`提供传统io中类似`PipedInputStream/PipedOutputstream`的功能，提供连接两个channel的功能。其定义如下：

~~~java
public abstract class Pipe {
    public final int validOps()
    public abstract SourceChannel source();
    public abstract SinkChannel sink();
    public static Pipe open();
    public static abstract class SourceChannel extends AbstractSelectableChannel implements ReadableByteChannel, ScatteringByteChannel;
    public static abstract class SinkChannel extends AbstractSelectableChannel implements WritableByteChannel, GatheringByteChannel;
}
~~~

通过调用`open()`方法创建其实例

`Pipe`类定义了两个内部类来实现管路分别是：`Piped.SourceChannel`(负责读的一端),和`Pipe.SinkChannel`(负责写的一端)

这两个通道实例在`Piped`对象创建的同时被创建，可以通过其`source()`和`sink()`方法来获取。

`Pipe`的source通道和sink通道提供类似`PipedInputStream`和`PipedOutputStream`所提供的功能，不过它们有全部的通道语义。

`Pipe`用来在同一JVM内部传输数据。使用如下：

~~~java
public class PipeChannelDemo {
    public static void main(String[] args) throws IOException {
        Pipe pipe = Pipe.open();
        new Thread(() -> read(pipe.source())).start();
        new Thread(() -> write(pipe.sink())).start();
    }

    public static void write(WritableByteChannel out){
        ReadableByteChannel in = Channels.newChannel(System.in);
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        try(in;out) {
            while (in.read(buffer) > -1){
                buffer.flip();
                out.write(buffer);
                buffer.clear();
            }
        } catch (IOException e){
            throw new RuntimeException(e);
        }

    }

    public static void read(ReadableByteChannel in) {
        WritableByteChannel out = Channels.newChannel(System.out);
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        try (in;out){
            while (in.read(buffer) > -1){
                buffer.flip();
                out.write(buffer);
                buffer.clear();
            }
        } catch (IOException e){
            throw new RuntimeException(e);
        }
    }
}
~~~

可以在不同线程间使用以实现线程间的数据传输

## Channels

`java.nio.channels.Channels`工具类定义了几个静态的工厂方法以使通道可以更加容易地同流和读写器互联：

~~~java
public static InputStream newInputStream(ReadableByteChannel ch);
public static OutputStream newOutputStream(WritableByteChannel ch);
public static InputStream newInputStream(AsynchronousByteChannel ch);
public static OutputStream newOutputStream(AsynchronousByteChannel ch);
public static ReadableByteChannel newChannel(InputStream in);
public static WritableByteChannel newChannel(OutputStream out);
public static Reader newReader(ReadableByteChannel ch, Charset charset);
public static Writer newWriter(WritableByteChannel ch, Charset charset);
~~~

# 选择器

选择器能够高效管理非阻塞模式的通道。选择器提供选择执行已经就绪的任务的能力，这使单线程能够有效地同时管理多个I/O通道

选择器涉及三个主要的类：`Selector`、`SelectableChannel`、`SelectionKey`，它们会同时参与到整个过程中。

使用选择器首先需要创建一个或者多个`SelectableChannel`注册到`Selector`对象中，一个表示通道和选择器的键`SelectionKey`将被返回。`SelectionKey`将记录通道和选择器的关联关系。

当调用`Selector.select()`方法时，相关的`SelectionKey`状态将被更新，用来检查所有被注册到该选择器的通道。

然后可以通过相关方法获取`SelectionKey`集合，通过遍历这些键，可以选择出从上次调用`select()`方法开始到现在，已经就绪的通道。

**`Selector`:**选择器类管理着一个被注册的通道集合的信息和它们的就绪状态。

**`SelectableChannel`:**提供了实现通道的可选择性所需要的方法。它时所有支持就绪检查的通道类的父类。

**`SelectionKey`:**封装了特定的通道和特定的选择器的注册关系。

一个使用Selector注册通道并进行就绪判断的实例如下：

~~~java
SelectableChannel channel1 = ServerSocketChannel.open().bind(new InetSocketAddress("localhost", 3366)).configureBlocking(false);
SelectableChannel channel2 = ServerSocketChannel.open().bind(new InetSocketAddress("localhost", 3367)).configureBlocking(false);
SelectableChannel channel3 = ServerSocketChannel.open().bind(new InetSocketAddress("localhost", 3368)).configureBlocking(false);
Selector selector = Selector.open();
channel1.register(selector, SelectionKey.OP_ACCEPT);
channel2.register(selector,SelectionKey.OP_ACCEPT);
channel3.register(selector,SelectionKey.OP_ACCEPT);
int select = selector.select(10000);//等待10s，让通道就绪
System.out.println(select);
~~~

## SelectableChannel

~~~java
public abstract SelectorProvider provider();
public abstract int validOps();
public abstract boolean isRegistered();
public abstract SelectionKey keyFor(Selector sel);
public abstract SelectionKey register(Selector sel, int ops, Object att);
public final SelectionKey register(Selector sel, int ops);
public abstract SelectableChannel configureBlocking(boolean block);
public abstract boolean isBlocking();
public abstract Object blockingLock();
~~~

`register()`位于`SelectableChannel`类，尽管通道实际上是被注册到选择器上的。并产生一个`SelectionKey`它的参数说明：

* 一个`Selector`对象，表示想要注册到的选择器
* att:可以传递对象引用，在新生成的`SelectionKey`的`attachment()`方法将返回该参数。

* ops整数参数表示选择器所关心的通道操作。这是一个表示选择器在检查通道就绪状态时需要关系的操作的比特掩码。特定的操作比特值在`SelectionKey`中有定义为静态字段，有四种可选操作：读、写、连接(connect)和接收(accept)

一个可选的通道并不支持所有的操作，比如`SocketChannel`不支持accept。可以通过调用`validOps()`方法获取当前通道支持的操作集合。

一个通道可以注册到多个选择器上，可以调用`isRegistered()`来检查一个通道是否被注册到任何一个选择器上。

任何一个通道和选择器的注册关系都被封装在一个`SelectionKey`对象中，`keyFor()`方法将返回当前通道于指定的选择器相关的`SelectionKey`。

## SelectionKey

~~~java
public static final int OP_READ = 1 << 0;
public static final int OP_WRITE = 1 << 2;
public static final int OP_CONNECT = 1 << 3;
public static final int OP_ACCEPT = 1 << 4;
public abstract SelectableChannel channel();
public abstract Selector selector();
public abstract boolean isValid();
public abstract void cancel();
public abstract int interestOps();
public abstract SelectionKey interestOps(int ops);
public abstract int readyOps();
public final boolean isReadable();
public final boolean isWritable();
public final boolean isConnectable();
public final boolean isAcceptable();
public final Object attach(Object ob);
public final Object attachment();
~~~

一个`SelectionKey`实例表示一个特定的通道对象和一个特定的选择器对象之间的注册关系。`channel()`返回于该键相关的`SelectableChannel`对象，而`selector()`则返回相关的Selector对象。

`SelectionKey`对象标识了一种特定的注册关系。可以调用你`cancel()`方法来结束这种注册关系。当键被取消时，它将被放到相关的选贼强的已取消的键的集合中。注册不会立即被取消，但键会立即失效。

可以使用`isValid()`方法检测当前键是否仍然表示其注册关系。

当通道关闭时，所有相关的键会自动取消。当选择器关闭时，所有被注册到该选择器的通道都会被注销，并且相关的键会立即无效化。

一个SelectionKey对象包含两个以整数形式进行编码的比特掩码：一个用于指示通道/选择器组合体所关系的操作(interest集合)，另一个表示通道准备准备好要执行的操作(ready集合)，其中ready集合时interest集合的子集，因为在这个注册关系中，选择器只关系关注操作的就绪情况。

当前`interest`集合可以通过`interestOps()`方法来获取。最初，这是通道被注册时传进来的值。可以调用`interestOps(int)`方法来改变这个集合。但是不能在一个通道上注册一个它不支持的操作。

ready集合表示了interest集合中从上次调用`Selector.select()`以来已经就绪的那些操作。可以通过调用`readyOps()`来获取相关的通道的已经就绪的操作。`readyOps()`方法只是一个提示，不是保证

还可以单独调用`isReadable()/isWritable()/isConnectable()/isAcceptable()`方法来测试那些操作已经就绪。

`attach()/attachment`允许在`SelectionKey`上放置一个"附件",并在后面获取它。这是一个允许将任意对象于`SelectionKey`关联的方法。可以关联任何有用的对象，如业务对象，会话句柄、其他通道等。

## Selector

~~~java
public static Selector open() throws IOException
public abstract Set<SelectionKey> keys();
public abstract Set<SelectionKey> selectedKeys();
public abstract int selectNow() throws IOException;
public abstract int select(long timeout)throws IOException;
public abstract int select() throws IOException;
public abstract Selector wakeup();
public abstract void close() throws IOException;
public abstract boolean isOpen();
~~~

一个`Selector`对象内部维护三个集合：

* 已注册的键的集合(Registered Key Set)：与选择器关联的已经注册的键的集合。并不是注册过的键都是有效的。可以通过`keys()`方法访问。获取的集合不允许修改。
* 已选择的键的集合(Selected Key Set)：已注册的键的集合的子集。这个集合的每个成员都是相关的通道被选择器(在前一个select操作中)判断为interest操作中有已经就绪的操作。可以通过`selectedKeys()`方法获取。获取的集合只能移出元素，不能添加元素。
* 已取消的键的集合(Canelled Key Set)：已注册的键的集合的子集。这个集合暂存了`cancel()`方法被调用过的键(这个键已经被无效化)，但它们还没有被注销。该集合无法直接访问。

### 选择过程

`Selecotr`的核心方法选择过程，选择操作时当三种形式的`select()`中的任意一种被调用时，由选择器执行的。将执行下面步骤：

* 第一步：已取消的键的集合将会被检查。如果它是非空的，每个已取消的键的集合中的键将从另外两个集合中移除，取消该键关联的通道和选择器之间的注册关系。此时如果通道是关闭的，并且没有别注册，将注销该通道。 这个步骤结束后，已取消的键的集合将是空的
* 第二步：遍历已注册的键的集合中的键，检查键的interest集合指示的对应通道操作的就绪状态。
  * 如果没有通道就绪，线程可能阻塞，可以设置一个阻塞超时值。
  * 如果有通道的interest操作就绪，那么对于这样的通道
    * 如果该通道对应的键还没有处于已选择的键的集合中，那么将重新设置键的ready集合对应的比特掩码。
    * 如果该通道对应的键已经在已选择的键的集合中了，那么键的ready集合将被更新。所有之前已经不再是就绪状态的操作表示的比特位不会被清除。事实上，所有的的比特位都不会被清理。一旦键被放置于选择器的已选择的键的集合中，它的ready集合将是累积的。比特位只会被设置，不会被清理。

* 第三步：重复第一步。因为第二步可能会持续很长时间(特别是线程在休眠时),这样即使在第二步时，又有新的键被取消(`SelectionKey.cancel()`),这些键在`select()`方法结束前，仍然被检查到。

* 第四步：`select()`操作的返回的值是ready集合在第二步中被修改的键的数量，而不是已选择的键的集合中的通道的总数。

  返回值不是已准备好的通道的总数，而是从上一次`select()`调用之后进入就绪状态的通道的总数。

三种形式的`select()`方法区别如下：

* `select()`如果通道一直没有就绪的，就会一直阻塞下去
* `select(int)`可以为阻塞设置一个超时时间
* `selectNow()`完全不阻塞，如果通道就绪就直接返回0

### 停止选择过程

可以调用下面方法唤醒在`select()`操作中休眠的线程：

* `wakeup()`使选择器上的第一个还没有返回的选择操作立即返回。如果当前没有在进行中的选择，那么下一次对 select( )方法的一种形式的调用将立即返回。  

  如果只想唤醒一个睡眠中的线程，而使得后续的选择继续正常地进行。您可以通过在调用 wakeup( )方法后调用 selectNow( )方法来绕过这个问题。  

* `close()`方法：内部调用了一次`wakeup()`方法，并且与选择器相关的键都将被取消，如果与选择器相关的通道是关闭的，且没有被注册的，那么通道将被注销。

* 调用`Thread.interrupt()`:如果`select()`方法收到中断请求，将捕获`InterruptedException`异常并调用`wakeup()`方法

### 使用选择器

使用selector用单线程实现一个简单的服务器。

~~~java
public class SelectSockets {
    public static void main(String[] args) throws IOException {
        ServerSocketChannel channel = ServerSocketChannel.open();
        Selector selector = Selector.open();
        channel.bind(new InetSocketAddress("localhost", 6666));
        channel.configureBlocking(false);
        channel.register(selector, SelectionKey.OP_ACCEPT);
        while (true) {
            int select = selector.select();
            if (select == 0)
                continue;
            Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
            while (iter.hasNext()){
                SelectionKey key = iter.next();
                if (key.isAcceptable()){
                    ServerSocketChannel server =(ServerSocketChannel) key.channel();
                    SocketChannel accept = server.accept();
                    registerChannel(selector,accept,SelectionKey.OP_READ);
                    sendMessage(accept);
                }
                if (key.isReadable())
                    readDataFromRemoteSocket(key);
                iter.remove();
            }
        }

    }
    
    private static final WritableByteChannel printChannel = Channels.newChannel(System.out);

    private static void readDataFromRemoteSocket(SelectionKey key) throws IOException {
        SocketChannel channel =(SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        int len;
        while ((len = channel.read(buffer)) != -1){
            buffer.flip();
            while (buffer.hasRemaining())
                printChannel.write(buffer);
            buffer.clear();
        }
    }

    private static void sendMessage(SocketChannel accept) throws IOException {
        accept.write(ByteBuffer.wrap("你好!".getBytes(StandardCharsets.UTF_8)));
    }

    private static void registerChannel(Selector selector, SelectableChannel channel, int ops) throws IOException {
        if (channel == null)
            return;
        channel.configureBlocking(false);
        channel.register(selector,ops);
    }
}
~~~

### 并发处理

Selector是线程安全的，但是`Selector.selectedKeys()`获取的已选择的键的集合并不是线程安全的。在使用该集合的`Iterator`访问元素时，如果有其他线程同时操作该集合，那么会快速失败，抛出`ConcurrentModificationException  `异常，所以在访问键集合时，可以使用同步方法保证一次只有一个线程在访问。

如果对就绪的管道的处理响应时间比较长，那么可以考虑使用线程池异步执行响应任务。使用一个线程持有选择器监控就绪状态，对于就绪状态管道的处理，则交给线程池的工作线程。

# 字符集

Java应用程序需要处理多种语言以及组成这些语言的多个字符。与字符相关的有如下概念:

* Character set 字符集：字符的集合，带有特殊语义的符号。
* Coded Character Set 编码字符集：与数值有映射关系的字符的集合。
* Character Encoding Scheme字符编码方案：如何把字符编码的序列表达为字节序列，是字符和数值的具体映射关系

`java.nio.charset`包组成的类实现了处理字符集编码的解决方式。

常见的编码字符集有：ASCII  、ISO-8859-1、UTF-8、UTF-16、UTF-16BE、UTF-16BE

## Charset

~~~java
public static boolean isSupported (String charsetName);
public static Charset forName (String charsetName);
public static SortedMap<String,Charset> availableCharsets()
public static Charset defaultCharset();
public final String name();
public final Set aliases();
public String displayName();
public String displayName (Locale locale);
public final boolean isRegistered();
public boolean canEncode();
public abstract CharsetEncoder newEncoder();
public final ByteBuffer encode (CharBuffer cb);
public final ByteBuffer encode (String str);
public abstract CharsetDecoder newDecoder();
public final CharBuffer decode (ByteBuffer bb);
public abstract boolean contains (Charset cs);
public final boolean equals (Object ob);
public final int compareTo (Object ob);
public final int hashCode();
public final String toString();
~~~

Charset类封装特定字符集的信息。通过静态方法`forName()`获取具体实例。

所有的`Charset`方法都是线程安全的。

可以调用`isSupported()`方法来确定指定的字符集是否可用

一个字符集可以有多个名称，通常它有一个规范名称但也可能有多个别名。规范名和别名都可以通过`forName()`和`isSupported()`进行使用

`availableCharsets()`返回在JVM中当前有效的字符集的SortedMap。调用它时，它会实例化所有已知的Charset对象。

`defaultCharset()`返回JVM默认的字符集

`name()`方法返回当前字符集的规范名称，`aliases()`返回其别名的集合

`displayName()`返回指定Locale参数的本地化名称，无参版本返回默认的本地化名称。这两个方法默认返回规范名称。需要实现类重写方法。java自带的字符集目前没有实现类重写该方法。

