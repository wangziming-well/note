JDK1.4的`java.nio`包中欧引入新的JavaI/O类库。

nio使用通道和缓冲器来进行数据传输。数据不会直接与通道交互，而是先和缓冲器交互，并把缓冲器派送到通道。

通道要么从缓冲器获得数据，要么从缓冲器发送数据。

# 使用通道和缓冲器

唯一直接与通道交互的缓冲器是`ByteBuffer`

旧I/O类库中有三个类被修改以产生`FileChannel`：`FileInputStream`、`FileOutputStream`以及`RandomAccessFile`

下面示例演示获取通道并通过缓冲器与通道交互以读取或者写入数据：

* 只写通道：

  ~~~java
  FileChannel channel = new FileOutputStream("data.txt").getChannel();
  channel.write(ByteBuffer.wrap("Some text".getBytes()));
  ~~~

* 只读通道：

  ~~~java
  FileChannel channel = new FileInputStream("data.txt").getChannel();
  ByteBuffer buffer = ByteBuffer.allocate(1024);
  channel.read(buffer);
  buffer.flip();
  while (buffer.hasRemaining())
      System.out.print((char) buffer.get());
  ~~~

* 读写通道

  ~~~java
  FileChannel channel = new RandomAccessFile("data.txt", "rw").getChannel();
  channel.position(channel.size());
  channel.write(ByteBuffer.wrap("Some more".getBytes()));
  channel.position(0);
  ByteBuffer buffer = ByteBuffer.allocate(1024);
  channel.read(buffer);
  buffer.flip();
  while (buffer.hasRemaining())
      System.out.print((char) buffer.get());
  ~~~

我们可以通过`ByteBuffer`提供的静态方法`ByteBuffer.wrap(byte[])`获取一个用指定byte数组填充好内容的`ByteBuffer`实例，以将数据传递到通道中，写入数据。

可以通过`ByteBuffer`提供的静态方法`ByteBuffer.allocate(int)`获取一个指定容量大小的空`ByteBuffer`实例，以获取通道中的数据。

# Buffer

`Buffer`由数据和可以高效访问以及操纵这些数据的四个索引组成：

~~~java
// Invariants: mark <= position <= limit <= capacity
private int mark = -1; //标记，可以记录position位置
private int position = 0; //位置:起到游标的作用，客户端只能访问Buffer在position处的数据
private int limit; //界限:限制游标的活动范围
private int capacity; //容量,指示Buffer的容量。
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
Buffer clear(); //清除此缓冲区。position设置为0，limit设置为当前capacity，mark重置为-1。此方法不会真正删除Buffer存储的数据，而是通过重置索引，指示通道可以重新记录数据。只有当通道重新记录数据时，旧的记录才会被覆盖。
Buffer flip(); //翻转此缓冲区。limit设置为当前position，然后将position设置为0。mark重置为-1。此方法用于写入完数据后，准备从缓冲区读取数据。
Buffer rewind(); //倒带此缓冲区。position设置为0，mark重置为 -1.
int remaining(); //返回剩余可操作的元素数: (limit - position )值
boolean hasRemaining(); //判断是否有剩余: 当 position < limit 时返回true
~~~

## 继承体系

`Buffer`的继承体系如下：

![Buffer](https://gitee.com/wangziming707/note-pic/raw/master/img/Buffer.png)

除了布尔类型，八大数据类型的其他七种类型都有对应的`Buffer`:`ByteBuffer`、`CharBuffer`等。但是他们都是抽象类，他们还有具体的实现类，如`HeapByteBuffer`、`DirectByteBuffer`等

其中`HeapXXXBuffer`的数据维护在JVM堆内存中；而`BirectXXXBuffer`的数据维护在操作系统的内存中，是堆外内存，不受GC控制。

* `HeapXXXBuffer`：由于内容维护在jvm里，所以把内容写进buffer里速度会快些；并且，可以更容易回收。

* `DirectXXXBuffer`：跟IO设备打交道时会快很多，因为IO设备读取jvm堆里的数据时，不是直接读取的，而是把jvm里的数据读到一个内存块里，再在这个块里读取的，如果使用DirectByteBuffer，则可以省去这一步，实现零拷贝。

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

初次之外`ByteBuffer`作为最基本的缓冲区也可以转换为其他类型的`Buffer`

~~~java
CharBuffer asCharBuffer();
ShortBuffer asShortBuffer();
IntBuffer asIntBuffer();
LongBuffer asLongBuffer();
FloatBuffer asFloatBuffer();
DoubleBuffer asDoubleBuffer();
~~~





