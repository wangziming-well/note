# 高速缓存

现代处理器处理速率远胜于主内存(DRAM)访问速率，主内存执行一次内存读、写操作所需的时间可能足够处理器执行上百条的命令。为了弥补处理器与主内存处理能力之间的鸿沟，硬件设计者在主内存和处理器之间引入了高速缓存(Cache)

高速缓存时一种存取速率远比主内存大而容量远比主内存下的存储部件，每个处理器都有其高速缓存。

引入高速缓存后，处理器在执行内存读写操作的时候并不直接与主内存打交道，而是通过高速缓存进行的。

变量名相当于内存地址，而变量值相当于相应内存空间所存储的数据。从这个角度看，高速缓存相当于为程序所访问的每个变量保存了一份相应内存空间所存储数据的副本。由于高速缓存的存储容量远小于主内存，因此高速缓存并不是每时每刻保留着所有变量值的副本。

## 高速缓存结构

高速缓存相当于一个由硬件实现的内存极小的散列表，其键(Key)是一个内存地址，其值(Value)是内存数据的副本或者准备写入内存的数据。

从内部结构来看，高速缓存相当于一个拉链散列表(Chained Hash Table)，它包含若干桶(Bucket)，每个桶又包含若干缓存条目(Cache Entry)

![image-20230804103629417](https://gitee.com/wangziming707/note-pic/raw/master/img/%E9%AB%98%E9%80%9F%E7%BC%93%E5%AD%98%E5%86%85%E9%83%A8%E7%BB%93%E6%9E%84%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

缓存条目可被进一步划分为Tag、Data Block和Flag三部分：

![image-20230804103800395](https://gitee.com/wangziming707/note-pic/raw/master/img/%E7%BC%93%E5%AD%98%E6%9D%A1%E7%9B%AE%E7%BB%93%E6%9E%84.png)

* Data Block 也被称为缓存行(Cache Line)，它是高速缓存与主内存之间的数据交换最小单元，用于存储从内存中读取的或者准备写往内存的数据。缓存行的容量(缓存行宽度)通常是2的倍数，其大小在16~256字节(Byte)之间。从代码角度看，一个缓存行可以存储若干变量的值，而多个变量的值可能被存储在同一个缓存行之中。

* Tag则包含了与缓存行中数据相应的内存地址的部分信息(内存地址的高位部分比特)
* Flag用于表示相应缓存行的状态信息

## 内存地址解码和缓存命中

处理器在执行内存访问操作时会将相应的内存地址解码(具体的解码动作是由高速缓存控制器负责执行)。内存地址的解码结果包括tag、index以及offset这三部分数据。其中index相当于桶编号，它可以用来定位内存地址对应的桶，tag相当于缓存条目的相对编号，其作用是定位一个具体的缓存条目(即缓存条目的Tag);offset是缓存行内的位置偏移，其作用是确定一个变量在一个缓存行中的存储起始位置。

根据这个内存地址的解码结果，如果高速缓存子系统能够找到相应的缓存行并且缓存行所在的缓存条目的Flag表示相应缓存条目是有效的，那么就称相应的内存操作产生了缓存命中(Cache Hit)；否则，我们就称相应的内存操作产生了缓存未命中(Cache Miss)

缓存未命中包括读未命中和写未命中，分别对应内存的读和写操作。当读未命中产生是，处理器所需要读取的数据会从主内存中加载并被存入相应的缓存行之中，这个过程会导致处理器停顿而不能执行其他指令，这不利于发挥处理器的处理能力。所以从性能角度来看我们应该尽可能减少缓存未命中。而由于高速缓存的总容量远小于主内存的总容量，因此缓存未命中不可避免。

## 多级缓存

现代处理器一般具有多个层次的高速缓存。在这个层次中，相应的高速缓存通常被称为一级缓存(L1 Cacha)、二级缓存(L2 Cache)、三级缓存(L3 Cache)等。

一级缓存可能被集成在处理器的内核里，因此其访问效率非常高，一般来说一级缓存的访问操作可以在2~4个处理器时钟循环完成。一级缓存通常包括两部分，其中一部分用于存储指令(L1i),另外一部分用于存储数据(L1d)。

距离处理器越近的高速缓存，器存取速率越快，制造成本越高，因此其容量也越小。

![image-20230804112552333](https://gitee.com/wangziming707/note-pic/raw/master/img/%E5%A4%9A%E7%BA%A7%E7%BC%93%E5%AD%98%E7%BB%93%E6%9E%84.png)

# 缓存一致性协议

多个线程访问同一个共享变量的时候，这些线程的执行处理器上的高速缓存各自都会保留一份该共享变量的副本，这带来的新的问题：一个处理器对其副本数据进行更新之后，其他处理器如果不同步更新数据，那么其他处理器将无法读取到这个处理器对变量的更新。这就是缓存一致性问题，其实质就是如何防止读脏数据和丢失更新的问题。

为了解决这个问题，处理器之间需要一种通信机制：缓存一致性协议。

## MESI协议

MESI(Modified-Exclusive-Shared-Invalid)协议是常用的缓存一致性协议，x86处理器所使用的缓存一致性协议就是基于MESI协议的。MESI协议对内存数据访问的控制类似于读写锁，它使得针对同一地址的读内存操作时并发的，而针对同一内存地址的写操作是独占的，即针对同一内存地址进行的写操作在任意一个时刻只能由一个处理器执行。在MESI协议中，一个处理器往内存中写数据时必须持有该数据的所有权。

为了保障数据的一致性，MESI将缓存条目的状态划分为：Modified、Exclusive、Shared、Invalid四种，并在此基础上定义了一组消息用于协调各个处理器的读、写内存操作

## MESI协议的Flag值

MESI协议中一个缓存条目的Flag值有以下四种可能：

* Invalid(无效的，记为I)：该状态表示相应缓存行中不包含任何内存地址对应的有效副本数据。该状态是缓存条目的初始状态。
* Shared(共享的，记为S)：该状态表示相应缓存行包含相应内存地址所对应的副本数据。并且，其他处理器上的高速缓存中也可能包含相同内存地址对应的副本数据。一个缓存条目的状态如果为Shared，并且其他处理器上也存在Tag值与该缓存条目Tag值相同的缓存条目，那么这些缓存条目的状态也为Shared。处于该状态的缓存条目，其缓存行中包含的数据与主内存中包含的数据一致。
* Exclusive(独占的，记为E)：该状态表示相应缓存行包含相应内存地址所对应的副本数据。并且，该缓存行以独占的方式保留了相应内存地址的副本数据，即其他所有处理器上的高速缓存当前都不保存该数据的有效副本。处于该状态的缓存条目，其缓存行中包含的数据与主内存中包含的数据一致。
* Modified(更改过的，记为M)：该状态标识相应缓存行包含对相应内存地址所做的更新结果数据。由于MESI协议中的任意一个时刻只能够有一个处理器对同一内存地址对应的数据进行更新，因此在多个处理器上的高速缓存中Tag值相同的缓存条目中，任意一个时刻只能够有一个缓存条目处于该状态。处于该状态的缓存条目，其缓存行中包含的数据与主内存中包含的数据不一致。

## MESI协议的消息

MESI协议定义了一组消息(Message)用于协调各个处理器的读、写内存操作，我们可以将MESI协议中的消息分为请求消息和响应消息。

处理器在执行内存读、写操作时在必要的时候会往总线中发送特定的请求消息，同时每个处理器还拦截总线中其他处理器发出的请求消息并在特定条件下往总线中回复相应的响应消息。

| 消息名                 | 消息类型 | 描述                                                         |
| ---------------------- | -------- | ------------------------------------------------------------ |
| Read                   | 请求     | 通知其他处理器、主内存当前处理器准备读取某个数据。该消息包含待读取数据的内存地址 |
| Read Response          | 响应     | 该消息包含被请求读取的数据。该消息可能是主内存提供的，也可能是拦截Read消息的其他高速缓存提供的 |
| Invalidate             | 请求     | 通知其他处理器将其高速缓存中指定内存地址对应的缓存条目状态置为I，即通知这些处理器删除指定内存地址的副本数据 |
| Invalidate Acknowledge | 响应     | 接收到Invalidate消息的处理器必须回复该消息，以表示删除了其高速缓存上的相应副本数据 |
| Read Invalidate        | 请求     | 该消息是由Read消息和Invalidate消息组合而成的复合消息。其作用在于通知其他处理器当前处理器准备更新(读后写更新)一个数据，并请求其他处理器删除其高速缓存中相应的副本数据。接收到该消息的处理器必须回复Read Response消息和Invalidate Acknowledge消息 |
| Writeback              | 请求     | 该消息包含需要写入主内存的数据以及对应的内存地址             |

## MESI协议协调下的内存访问操作

假设内存地址A上的数据S是处理器P0和P1可能共享的数据

### 内存读操作

P0读取数据S的过程如下：

P0会根据地址A找到对应的缓存条目，读取该缓存条目的Flag值：

如果Flag的值为M、E或者S，那么当前缓存条目的缓存行就是有效的，该处理器可以直接从相应的缓存行中读取地址A所对应的数据，而不需往总线中发送任何消息。

如果Flag的值为I，那么当前缓存条目的缓存行是无效的，P0需要往总线发送Read消息以读取地址A对应的数据。而其他处理器P1或者主内存需要回复Read Response以提供相应的数据。P0接收到Read Response消息时，会将其中携带的数据S存入相应的缓存行并将相应缓存条目状态更新为S

P0收到的Read Response消息可能来自主内存或者其他处理器P1：P1拦截到P0发送的Read消息时，会从该消息中取出待读取的内存地址，并根据该地址在P1自己的高速缓存中查找相应的缓存条目。如果P1找到的缓存条目的状态不为I，说明P1的高速缓存中有带读取数据的副本，此时P1会构造相应的Read Response消息并将相应缓存行锁存储的整块数据(而不仅仅是P0锁请求的数据S)放入消息。

* 如果P1找到的相应缓存条目的状态为M，那么P1可能在往总线发送Read Response消息前将相应缓存行中的数据写入主内存。P1往总线发送Read Response之后，相应的缓存条目的状态会被更新为S
* 如果P1找到的相应缓存条目的状态为I,那么P1将不会相应P0的Read消息。如果所有的处理器都没有数据S的副本，那么最终主内存将相应P0的Read请求。

由上面的过程可见，当P0读取内存时，即使P1对相应的内存数据进行了更新且这种更新还停留在P1的高速缓存中而造成高速缓存和主内存中的数据不一致，在MESI消息的协调下，这种不一致也不会导致P0读到脏数据。

### 写内存操作

P0往地址A写数据的过程如下：

任何一个处理器执行内存写操作时必须拥有相应数据的所有权。在执行内存写操作时，P0会先根据内存地址A找到相应的缓存条目。

若P0所找到的缓存条目的状态若为E或者M，则说明该处理器已经拥有相应数据的所有权，此时该处理器可以直接将数据写入相应的缓存行并将相应缓存条目的状态更新为M。

若P0所找到的缓存条目的状态不为E或者M，则该处理器需要往总线发送Invalidate消息以获取数据的所有权。其他处理器接受到Invalidate消息后会将其高速缓存中相应的缓存条目更新为I(相当于删除相应的副本数据)并回复Invalidate Acknowledge消息。P0发送完Invalidate消息后必须在接受到其他所有处理器所回复的所有Invalidate Acknowledge消息之后再将其数据更新到相应的缓存中。

P0在接受到其他所有处理器所回复的Invalidate Acknowledge 消息之后会将相应的缓存条目的状态更新为E，此时P0获得了地址A上数据的所有权。接着，P0便可以将数据写入相应的缓存行，并将相应的缓存条目的状态更新为M。

如果P0找到的缓存条目状态为I，此时P0需要往总线发送Read invalidate消息。P0在接收到Read Response消息和其他所有处理器回复的Invalidate Acknowledge 消息之后，会将相应缓存条目的状态更新为E，表示获取到相应数据的所有权。接着P0便可以将数据写入相应的缓存行，并将相应的缓存条目的状态更新为M。

由上面的过程可见，处理器只能向状态为E，M的缓存条目写数据，P0在写数据前会发起Invalidate 请求，其他处理器必须将相应的缓存条目状态置为I并答复相应后，P0才会将更改相应的缓存条目状态为E并开始写入数据。这保证了针对同一个内存地址的写操作在任意一个时刻只能由一个处理器执行，从而避免了多个处理器同时更新同一个数据可能导致的数据不一致问题。

# 写缓冲器和无效化队列

MESI协议解决了缓存一致性问题，但其本身也存在性能弱点：处理器执行写内存操作时，必须等待其他所有处理器将其高速缓存中的相应副本数据删除并接受到这些处理器回复的Invalidate Acknowledge/Read Response消息之后才能将数据写入高速缓存。为了避免和减少这种等待造成的写操作的延迟，硬件设计者引入了写缓冲器和无效化队列。

写缓冲器(Store Buffer/Write Buffer)是处理器内部一个容量比高速缓存还小的高速存储部件，每个处理器都有其写缓冲器，写缓冲器内部可包含若干条目。一个处理器无法读取到另外一个处理器上的写缓冲器中的内容。

引入写缓冲器之后，处理器在执行写操作时会进行以下步骤：

如果相应的缓存条目状态为E或者M，那么处理器可能会直接将数据写入相应的缓存行而无须发送任何消息(取决于处理器的实现。有的处理器不管相应缓存条目状态如何，直接将每一个写操作的结果都存入写缓冲器)

如果相应的缓存条目状态为S，那么处理器会先将写操作的相关数据(包括数据和待操作的内存地址)写入写缓冲器的条目之中，并发送Invalidate消息。

如果相应的缓存条目状态为I，那么写操作就遇到了写未命中，此时处理器会先将写操作相关数据存入写缓冲器的条目之中，并发送Read Invalidate消息。

内存写操作的执行处理器在将写操作的相关数据写入写缓冲器之后便认为该写操作已经完成，即该处理器并不等待其他处理器返回Invalidate Acknowledge /Read Response   消息而是继续执行其他命令。一个处理器接受到其他处理器所回复的针对同一个缓存条目的所有Invalidate Acknowledge消息的时候，该处理器会将写缓冲器中针对地址的写操作的结果写入相应的缓存行中，此时写操作对于其执行处理器之外的其他处理器来说才算完成。

所以写缓冲器的引入使得处理器在执行写操作时不必等待Invalidate Acknowledge 消息，从而减少了写操作的延时，这使得写操作的执行处理器在其他处理器回复Invalidate Acknowledge/Read Response 消息这段时间内能够执行其他命令，从而提高了处理器的指令执行效率。



引入无效化队列(Invalidate Queue)后，处理器在接收到Invalidate消息之后并不删除消息中指定地址对应的副本数据，而是将消息存入无效化队列之后就回复Invalidate Acknowledge消息。从而减少了写操作执行处理器所需的等待时间。有些处理器可能没有无效化队列。

写缓冲器和无效化队列的引入又会带来新的问题：内存重排序和可见性问题。

### 存储转发

引入写缓冲器之后，处理器在执行读操作的时候不能根据相应的内存地址直接读取相应缓存行中的数据作为该操作的结果。

因为一个处理器在更新一个变量之后紧接着又读取该变量的时候，由于该处理器先前对该变量的更新结果可能仍然停留在写缓冲器之中，因此该变量相应的内存地址所对应的缓冲行中存储的值是该变量的旧值。

这种情况下为了避免读操作所返回的结果是一个旧值，处理器在执行读操作的时候会根据相应的内存地址先查询写缓冲器。如果写缓冲器存在相应的条目，那么该条目所代表的写操作的结果数据就会直接作为该读操作的结果返回；否则，处理器才从高速缓存中读取数据。

这种处理器直接从写缓冲器中读取数据来实现内存读操作的技术被称为存储转发。

存储转发使得写操作的执行处理器能在不影响该处理器执行读操作的情况下将写操作的结果存入写缓冲器。

### 导致内存重排序

以下是写缓冲器和无效化队列可能导致的重排序

#### StoreLoad重排序

写缓冲器可能导致StoreLoad重排序。StoreLoad重排序是绝大部分处理器都运行的一种内存重排序。

假设P0，P1上的两个线程为使用任何同步措施各自按照程序顺序执行，并依照下表的程序交错顺序执行。其中变量X，Y为共享变量，初始值均为0，r1，r2为局部变量：

| P0        | P1        |
| --------- | --------- |
| X=1;//S1  | Y=1;//S3  |
| r1=Y;//L2 |           |
|           | r2=X;//L4 |

当P0上的线程执行到L2时，虽然在此前S3已经被P1执行完毕，但由于S3的执行结果仍然还停留在P1的写缓冲器中，而一个处理器无法读取另外一个处理器的写缓冲器的内存，因此P0此刻读取到的Y值仍然是其高速缓存中存储的该变量的初始值0.所以在P0的角度来看，P1先执行了L4，才执行了S3，即P1执行的两个操作的感知顺序是L4->S3;进行了StoreLoad重排序。

同样的。因为写缓冲器的存在，P0的两个操作的感知顺序也可能是：L2->S1

#### StoreStore重排序

写缓冲器可能导致StoreStore重排序。

假设P0，P1上的两个线程为使用任何同步措施各自按照程序顺序执行，并按照下表的程序交错顺序执行。

其中变量data，ready为共享变量，初始值为0，false。

| P0              | P1                         |
| --------------- | -------------------------- |
| data=1;//S1     |                            |
| ready=ture;//S2 |                            |
|                 | while(!ready)continue;//L3 |
|                 | print(data);//L4           |

假设P0执行S1，S2时该处理器的高速缓存中包含变量ready的副本但是不包含data的副本。那么S1的执行结果会被先存入写缓冲器而S2的执行结果会直接被存入高速缓存。

L3被执行时，S2对ready的更新通过缓存一致性协议可以被P1读取到，于是ready值为变为了true，P1继续执行L4。L4被执行时，由于S1对data的更新结果可能仍然停留在P0的写缓冲器中，因此P1此时读取到的变量data值可能仍然是其初始值0.

从P1的角度看，P0的操作S2先于S1被执行了，即P0的感知顺序为：S2->S1，发生了StoreStore重排序。



另外，某些处理器为了充分利用总线带宽以提高将写缓冲器中的内容冲刷(写入) 到高速缓冲的效率，会将针对连续内存地址的写操作并入同一个写缓冲器条目之中。这种处理就被称为写合并。这就导致了写缓冲器实际上并不是先进先出的公平队列。也可能导致StoreStore重排序。

#### LoadLoad重排序

无效化队列可能导致LoadLoad重排序。

假设P0，P1上的两个线程为使用任何同步措施各自按照程序顺序执行，并按照下表的程序交错顺序执行。

其中变量data，ready为共享变量，初始值为0，false。

| P0              | P1                         |
| --------------- | -------------------------- |
| data=1;//S1     |                            |
| ready=ture;//S2 |                            |
|                 | while(!ready)continue;//L3 |
|                 | print(data);//L4           |

假设P0的高速缓存中存在变量data和ready的副本，P1仅有变量data的副本而没有ready的副本。那么P0，P1可能按照如下顺序执行操作：

* P0执行S1。由于P1上也有变量data的副本，所以变量data对应的缓存条目状态一定为S，因此P0会发出Invalidate消息并将S1的操作结果存入写缓冲器。
* P1接收到P0发出的Invalidate消息将消息存入其无效化队列并回复Invalidate Acknowledge消息
* P0接受到Invalidate Acknowledge消息，随即将S1的操作结果写入高速缓存。
* 然后P0执行S2，此时由于只有P0上存有变量ready的副本，那么ready对应的缓存条目的状态就可能为E或者M，此时P0无须发送任何消息，直接将S2的操作结果存入高速缓存中
* P1执行L3。由于此时P1的高速缓存中没有ready的副本，所以P1发送一个Read消息，P0拦截到该消息后会回复一个Read Response消息，其中包含已经修改国有的ready值，为true。这样P1读到的ready值就为true，所以P1就可以继续执行L4
* P1执行L4.此时由于P0为了更新变量data而发出的Invalidate消息可能仍然还停留在P1的无效化队列中，因此P1从其高速缓存中读取的变量data的值仍然是其初始值。因此L4锁打印的变量值可能是一个旧值。

从P0的视角来看，L4读操作被重排序到L3之前，可见LoadLoad重排序会导致类似StoreStore重排序的效果。

### 导致可见性问题

**写缓冲器导致的可见性问题**

写缓冲器是处理器内部的私有存储部件，一个处理器中的写缓冲器所存储的内容是无法被其他处理器所读取的。因此一个处理器上运行的线程更新了一个共享变量之后，如果这个更新还停留在处理器的写缓冲器之中。那么其他处理器上运行的线程就无法读取到之前线程对这个变量的更新。

所以写缓冲器是可见性问题的硬件根源。

为了使处理器上运行的线程对共享变量所做的更新被其他处理器上运行的其他线程所读取，我们必须将写缓冲器中的内容写入其所在的处理器上的高速缓存之中，从而使该更新在缓存一致性协议的作用下可以被其他处理器读取到。

处理器在一些特定条件下(如写缓冲器满、I/O指令被执行)会将写缓冲器排空或者冲刷，即将写缓存器中的内容写入高速缓存，但是从程序对一个或者一组变量更新的角度来看，处理器本身无法保证这种冲刷对程序来说是及时的。因此，为了保证一个处理器对共享变量所做的更新可以及时被其他处理器同步，编译器等底层系统需要借助一类被称为内存屏障的特殊指令。内存屏障中的存储屏障(Store Barrier)可以使执行该命令的处理器冲刷其写缓冲器。

**无效化队列导致的可见性问题**

无效化队列也可能导致可见性问题：处理器在执行内存读取操作前如果没有根据无效化队列中的内容将该处理器上的高速缓存中的相关副本数据删除，那么就可能导致该处理器读取到的数据是过时的旧数据，从而使其他处理器所做的更新丢失。

因此，为了使一个处理器上运行的线程能够读取到另外一个处理器上运行的线程对共享变量所做的更新，该处理器必须先根据无效化队列中存储的Invalidate消息删除其高速缓存中相应的副本数据，从而使其他处理器上运行的线程对共享变量所做的更新在缓存一致性协议的作用下能够被同步到该处理器的高速缓存中。

内存屏障中的加载屏障(Load Barrier)就是用来解决这个问题的。加载屏障会根据无效化队列内容所指定的内存地址，将相应处理器上的高速缓存中相应的缓存条目的状态都记为I，从而使该处理器后续执行针对相应地址的读内存操作时必须发送Read消息，以将其他处理器对相关共享变量的更新同步到该处理器的高速缓存中。

**解决可见性问题**

解决可见性问题首先要使写线程对共享变量所做的更新能够达到高速缓存，从而使该更新对其他处理器是可同步的。其次，读线程所在的处理器要先将其无效化队列中的内容应用到其高速缓存上，这样才能够将其他处理器对该共享变量的更新同步到该处理器的高速缓存中。

而这两点是通过存储屏障和加载屏障的成对使用实现的：写线程的执行处理器所执行的存储屏障保障了该线程对共享变量所做的更新对读线程是可同步的；读线程的执行处理器所执行的加载屏障将写线程对共享变量所做的更新同步到该处理器的高速缓存中。

**存储转发导致的可见性问题**

存储转发技术也可能导致可见性问题。

假设处理器P0在t1时刻更新了某个共享变量，随后又在t2时刻读取了该变量。在t1时刻到t2时刻之间的这段时间内其他处理器可能已经更新了该共享变量，并且这个更新的结果已经到达了该处理器的高速缓存。但是如果P0在t1时刻所做的更新仍然停留在该处理器的写缓冲器中，那么存储转发技术会使P0直接从其写缓冲器中读取该共享变量的值。此时P0根本不从高速缓存中读取该变量的值，这使得其他处理器对该共享变量所做的更新无法被该处理器读取。从而导致P0在t2时刻读取到的变量值是一个旧值。

所以为了避免存储转发技术导致的可见性问题，读线程在读取共享变量前需要清空该处理器上的写缓冲器以及无效化队列。即读线程执行内存读取操作前需要先执行加载屏障和存储屏障。

# 基本内存屏障

硬件方面处理器支持下面屏障：

* Load屏障，是x86上的”ifence“指令，在其他指令前插入ifence指令，可以让CPU寄存器中的数据失效，强制从主内存里面加载数据到CPU寄存器中。

*  Store屏障，是x86的”sfence“指令，在其他指令后插入sfence指令，能让当前线程写入CPU寄存器中的最新数据，写入到主内存，并让其他线程可见。

处理器支持哪种内存重排序，就会提供能够禁止相应重排序的指令，这些指令就被称为内存屏障：

LoadLoad屏障、LoadStore屏障、StoreStore屏障、StoreLoad屏障。

基本内存屏障可以统一用XY来表示，其中X和Y可以代表Load或者Store。XY内存屏障的作用是禁止该指令前的所有X操作与该指令后的Y操作之间进行重排序。

例如：StoreLoad屏障指令禁止该指令前的Store指令和该指令后的Load指令进行重排序。即该指令保障了该指令之前的写操作的结果在该指令之后的任何读操作的数据被加载之前对其他处理器来说可同步，即这些写操作的结果会在该屏障指令时写入高速缓存或者主内存。

| 屏障名称   | 具体作用                                                     |
| ---------- | ------------------------------------------------------------ |
| StoreLoad  | 禁止StoreLoad重排序，确保该屏障之前的任意一个写操作的结果都会在该屏障之后的任意一个读操作的数据被加载之前对其他处理器来说是可同步的 |
| StoreStore | 禁止StoreStore重排序，确保该屏障之前的任何一个写操作的结果都会在该屏障之后的任意一个写操作之前对其他处理器来说是可同步的 |
| LoadLoad   | 禁止LoadLoad重排序，确保该屏障之前的任何一个读操作的数据都会在该屏障之后的任何一个读操作之前被加载 |
| LoadStore  | 禁止LoadStore重排序，确保该屏障之前的任何一个读操作的数据都会在该屏障之后的任何一个写操作的结果被冲刷到高速缓存之前被加载 |

基本内存屏障的作用只是保障其左侧的X操作先于其右侧的Y操作被提交，它并不全面禁止重排序。XY屏障两侧的内存操作仍然可以在不越过内存屏障 本身的情况下载各自的阀内内进行重排序。并且XY屏障左侧的非X操作与屏障右侧的非Y操作之间仍然可以进行重排序，如下图所示：

![image-20230805222336132](https://gitee.com/wangziming707/note-pic/raw/master/img/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C%E5%85%81%E8%AE%B8%E7%9A%84%E9%87%8D%E6%8E%92%E5%BA%8F%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

编译器(JIT编译器)、JVM和处理器都会尊重内存屏障，从而保障其作用得以落实。因为不止处理器可以发生重排序，JIT和JVM都可以造成重排序。

LoadLoad屏障是通过清空无效化队列来实现LoadLoad重排序的。LoadLoad屏障会使其执行处理器根据无效化队列中的Invalidate消息删除其高速缓存中相应的副本。这个过程被称为无效化队列应用到高速缓存，也被称为清空无效化队列。它使处理器有机会将其他处理器对共享变量的所做的更新同步到该处理器的高速缓存中，从而消除了LoadLoad重排序的根源而实现了LoadLoad重排序

StoreStore屏障可以通过对写缓冲器中的条目进行标记来实现禁止StoreStore重排序。StoreStore屏障会将写缓冲器中的现有条目做一个标记，以表示这些条目代表的写操作需要先于该屏障之后的写操作被提交。处理器在执行写操作的时候如果发现写缓存器找那个存在被标记的条目，那么即使这个写操作对应的高速缓存条目的状态为E或者M，此时处理器也不会将写操作的数据写入高速缓存，从而使得StoreStore屏障之前的任何写操作先于该屏障之后的写操作被提交。

StoreLoad屏障会清空无效化队列，并将写缓冲器中的条目冲刷到高速缓存。所以它 既可以将其他处理器对共享变量所做个更新同步到该处理器的高速缓存中，又可以使其执行处理器对共享变量所做的更新对其他处理器来说可同步。

# Java同步机制与内存屏障

JVM对synchronized、volatile和final关键字的语义的实现就是借助内存屏障的。

获得屏障和释放屏障相当于由基本内存屏障组合而成的复合屏障。

获得屏障相当于LoadLoad屏障和LoadStore屏障的组合，它能禁止该屏障之前的任何读操作与该屏障之后的任何读、写操作之间进行重排序。

释放屏障相当于LoadStore屏障和StoreStore屏障的组合，它能禁止该屏障之前的任何读、写操作和屏障之后的任何写操作之间进行重排序。

## volatile关键字的实现

JVM在volatile变量写操作之前插入的释放屏障使得该屏障之前的任何读写操作都先于这个volatile变量写操作被提交。

JVM在volatile变量读操作之后插入的获得屏障使得这个volatile变量读操作先于该屏障之后的任何读写操作被提交。

### 保障有序性

写线程和读线程通过各自执行释放屏障和获得屏障保障了有序性。

假设读写线程按照下表的线程交错顺序执行，A、B是普通共享变量，V是volatile变量：

| 写线程                                                       | 读线程                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| A=1;<br />B=2;<br />[LoadStore+StoreStore]//释放屏障<br />V=true; |                                                              |
|                                                              | `if(V){`<br />    `[LoadLoad+LoadStore]//获得屏障`<br />    `sum=A+B;`<br />`}` |

释放屏障确保了写线程对共享变量A，B的更新会先于对V的更新被提交，这意味着读线程在读到写线程对V的更新的情况下也能读到写线程对A和B的更新。

为了保障读线程对写线程所执行的写操作的感知顺序和程序顺序一致，读线程必须依照与写线程的程序顺序相反顺序即先读取V再读取A或者B来执行读操作。由于读线程中的读操作也可能被重排序。因此JVM会在读线程中的volatile读操作之后插入一个获得屏障，以保证该线程对变量V的读取操作是先于对A、B的读取的。

写线程、读线程通过释放屏障和获得屏障这种配对使用保障了读线程对写线程执行的写操作的感知顺序和程序顺序一致，即保障了有序性。

### 保障可见性

JVM会在volatile变量写操作之后插入一个StoreLoad屏障。该屏障不仅禁止该屏障之后的任何读操作和该屏障之前的任何写操作之间进行重排序，它还起到一下两个作用：

* 充当存储屏障：StoreLoad屏障通过清空写缓冲器使得该屏障前的所有写操作得以到达高速缓存，从而使这些更新对其他处理器是可同步的
* 充当加载屏障，以消除存储转发的副作用。存储转发可能使一个处理器无法及时读取到其他处理器对共享变量的更新。而StoreLoad屏障既能够清空写缓冲器还能够清空无效化队列，从而使其他处理器对volatile变量所做的更新能够被同步到volatile变量读线程的执行处理器上。

JVM在volatile变量读操作前会插入一个加载屏障相当于LoadLoad屏障，它通过清空无效化队列来使得后续的读操作有机会读取到其他处理器对共享变量的更新。

所以volatile对可见性的保障是通过写线程、读线程配对使用存储屏障和加载屏障实现的。

## synchronized关键字的实现

JVM对synchronized关键字的实现方式与对volatile的实现方式类似，在获得锁和释放锁前后，JVM执行如下命令：

获得锁monitorenter->加载屏障->获得屏障->临界区->释放屏障->释放锁monitorexit->存储屏障

JVM会在monitorenter指令后、临界区前插入一个获得屏障；在临界区结束后，monitorexit指令前插入一个释放屏障。这样获得屏障和释放屏障一起保障了临界区内的任何读写操作都无法被重排序到临界区外，再加上锁的排他性，这使得临界区内的操作具有原子性。

JVM会在monitorexit指令(相当于写操作)之后插入一个StoreLoad屏障。相当于存储屏障，确保锁的持有现场在释放锁之前所执行的所有操作的结果都能达到高速缓存，并消除了存储转发的副作用。

## final关键字的实现

在调用构造器对final字段初始化时，JVM会在执行final字段初始化后插入一个StoreStore；以禁止对final字段的写操作与之后的任何写操作(包括对象的发布)重排序

这保证了对象发布时，其final字段一定是初始化完毕的。





















