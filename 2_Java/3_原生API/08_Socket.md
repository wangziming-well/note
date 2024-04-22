# 计算机网络概述

计算机网络是由一组通过通信信道相互连接的机器组成。这些机器是主机(hosts)和路由器(routers)

* **主机**是指运行应用程序的计算机，这些应用程序如网络浏览器，即时通讯代理等是计算机网络的真正用户

* **路由器**的作用是将信息从一个通信信道转递或转发到另一个通信信道
* **通信信道**是将字节序列从一个主机传输到另一个主机的手段，如有线电缆、WIFI等

计算机网络中主机和主机之间一般不是直接相连的，通常主机先连接到路由器，路由器再连接到其他路由器。这样使每个主机只需要用到数量相对较少的通信信道，大部分主机仅需要一条通道。而网络上相互转递信息的程序并不直接与路由器交互。

网络间程序互相传递的信息是由程序创建和解释的字节序列，在计算机网络中，这些字节序列被称为**分组报文**

**协议**(protocol):是相互通信的程序间达成的约定，它规定了分组报文的交换方式和它们包含的信息。一个协议规定了分组报文的结构以及怎样对报文中所包含的信息进行解析。

## TCP/IP协议族

人们设计了不同的协议用来解决不同类型的问题。TCP/IP协议就是这样一组的解决方案，也被称为协议族

TCP/IP协议族将网络分为链路层、网络层、传输层、应用层

其主要协议有：

* IP协议：互联网协议(Internet Protocol)
* TCP协议：传输控制协议(Transmission Control Protocol)
* UDP协议：用户数据报协议(User Datagram Protocol)

IP协议位于网络层，它使两个主机间的一系列通信信道和路由器看起来像是一条单一的主机到主机的信道。它为每个主机标记了一个地址，这就是IP地址。

TCP和UDP协议都位于传输层，建立在IP协议所提供的服务基础之上。但IP地址只能帮助定位到主机，无法区分寻址到具体的应用程序。

TCP和UDP在IP地址的基础上使用端口号来区分同一主机中的不同应用程序。它们也被称为端到端传输协议，因为它们将数据从一个应用程序传输到另一个应用程序。

TCP协议提供一个可信赖的字节流通道，通过三次握手建立起稳定的TCP连接

UDP协议并不可靠，可能发生报文丢失、重复以及其他错误。

## 地址

在TCP/IP协议中，IP地址和端口号可以定位网络中一个指定的程序。其中IP地址由IP协议使用，端口号由传输协议解析。

IP地址分为IPv4和IPv6两种形式。

IPv4地址占4个字节：被表示为一组4个十进制数，每两个数字之间由原点隔开，每个数字的范围是0到255，如10.1.2.3

IPv6地址占16个字节：由几组16进制的数组表示，这些16进制数之间由分号隔开，如：2000:fdb8:0000:0000:0001:00ab:853c:39a1

每个IP地址代表了一个主机与底层的链路层的连接，也就是一个网络接口。主机可以有多个接口。

TCP或UDP协议种的端口号总与一个IP地址相关联。端口号范围：1-65535，其中1-1024被系统应用占用

IP地址有一个特殊的回环地址。回环接口是一种虚拟设备，它的功能是将发送给它的报文直接回发给发送者。在测试种很有用，因为发送给这个地址的报文能够立即返回到目标地址。每台主机上都有回环地址。IPv4的回环地址是127.0.0.1。IPv6的回环地址是0:0:0:0:0:0:0:1

## Socket套接字

Socket事一种抽象层，应用程序通过它来发送和接收数据。一个Socket允许应用程序添加到网络种，并于处于同一网络种的其他应用程序进行通信。



# InetAddress

是对IP地址的抽象封装。包括IP地址和端口号。它有两个实现类`Inet4Address`和`Inet6Address`，分别对应了目前IP地址的两个版本。

## 获取InetAddress实例

可以通过`NetworkInterface.getNetworkInterfaces()`获取当前主机的网络接口实例，可以通过`NetworkInterface.getInetAdresses()`获取网络接口下的地址信息。

可以获取本机的地址信息：

~~~java
Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();
while (interfaces.hasMoreElements()){
    NetworkInterface inter = interfaces.nextElement();
    System.out.println("接口名:"+inter.getName());
    Enumeration<InetAddress> addresses = inter.getInetAddresses();
    while (addresses.hasMoreElements()){
        InetAddress inetAddress = addresses.nextElement();
        System.out.println("地址信息："+inetAddress);
    }
}
~~~

可以通过`InetAddress  `提供的静态工厂方法获取IP地址实例：

~~~java
public static InetAddress getByAddress(String host, byte[] addr);
//根据提供的主机名和 IP 地址创建 InetAddress。不检查名称服务地址的有效性
public static InetAddress getByAddress(byte[] addr);
//返回给定 InetAddress 原始 IP 地址的对象。
public static InetAddress getByName(String host);
//根据主机的名称确定主机的IP 地址。可以是域名也可以是其 IP 地址的文本表示形式。如果提供了文本 IP 地址，则仅检查地址格式的有效性。
public static InetAddress[] getAllByName(String host);
//给定主机的名称，根据系统上配置的名称服务返回其 IP 地址的数组。可以是域名也可以是其 IP 地址的文本表示形式。如果提供了文本 IP 地址，则仅检查地址格式的有效性。
public static InetAddress getLocalHost();
//返回本地主机的地址。这是通过从系统中检索主机的名称，然后将该名称解析为 InetAddress 来实现的。
public static InetAddress getLoopbackAddress();
//返回环回地址
~~~

例：

~~~java
InetAddress[] inetAddresses = InetAddress.getAllByName("www.baidu.com");
for (InetAddress address : inetAddresses)
    System.out.println(address);
//输出：www.baidu.com/112.80.248.75 和 www.baidu.com/112.80.248.76
~~~

## 访问属性

提供获取当前实例的主机名或者IP地址：

~~~java
public String getHostName();
public String getCanonicalHostName();
public byte[] getAddress();
public String getHostAddress();
~~~

`getHostName()`和`getCanonicalHostName()`都是返回主机名，如果实例最初通过主机名创建，`getHostName()`直接返回这个主机名，而`getCanonicalHostName()`会尝试对地址进行解析，以获取主机域名全程。如果主机名解析失败，这两个方法都将返回数字型地址。

`InetAddress`提供对地址属性的检查：

~~~java
public boolean isAnyLocalAddress();
//一个布尔值，指示 Inetaddress 是否为通配符地址。
public boolean isLinkLocalAddress();
//指示 InetAddress 是否为链接本地地址
public boolean isLoopbackAddress();
//指示 InetAddress 是否为环回地址
public boolean isSiteLocalAddress();
//示 InetAddress 是否为站点本地地址
public boolean isMulticastAddress();
//指示 InetAddress 是否为 IP 多播地址
public boolean isMCGlobal();
public boolean isMCNodeLocal();
public boolean isMCLinkLocal();
public boolean isMCSiteLocal();
public boolean isMCOrgLocal();
public boolean isReachable(int timeout);
public boolean isReachable(NetworkInterface netif, int ttl,int timeout);
~~~

前三个方法检查地址实例是否属于任意本地地址，本地链接地址，回环地址。

`isMulticastAddress()`检查是否为一个多播地址

`isMC...()`形式的方法检查多播地址的各种范围。

`isReachable()`检查是否真的和实例代表的地址对应的主机进行数据报文交换。



# InetSocketAddress

`InetSocketAddress`类为主机地址和端口号提供了一个不可变的组合，用以定位网络上指定的`Socket`应用程序。在使用`Socket`和`ServerSocket`进行连接和监听时，将用到它。

可以直接创建对象，也可以使用提供的工厂方法:

~~~Java
public InetSocketAddress(int port);
public InetSocketAddress(InetAddress addr, int port);
public InetSocketAddress(String hostname, int port);
public static InetSocketAddress createUnresolved(String host, int port);
~~~

如果使用第一个构造器，不指定主机地址，那么将使用特殊的任何地址来创建实例，即`0.0.0.0`，它代表主机上所有接口的地址。这对服务端的监听非常有用：

~~~java
ServerSocket server = new ServerSocket();
server.bind(new InetSocketAddress(6666));
~~~

以上`server`将监听本机上所有接口对应地址的6666接口。

接受字符串主机名hostname的构造函数将尝试将其解析成相应的IP地址。

而`createUnresolved()`允许在没有对主机名进行解析的情况下创建实例。



# TCP Socket

Java为TCP协议提供了两个类：`Socket`类和`ServerSocket`类。

一个TCP连接是一个抽象的双向通道，在通讯之前，要建立一个TCP连接，这需要客户端向服务端发送请求。

一个`Socket`实例代表了TCP连接的一端。

`ServerSocket`实例则监听TCP连接请求，并为每个请求创建新的`Socket`实例。

所以客户端只要使用`Socket`实例，而服务端要同时处理`ServerSocket`实例和`Socket`实例。

## TCP客户端

客户端向服务器发起连接请求后，就被动地等待服务器的响应。典型的TCP客户端需要经过下面步骤：

* 创建一个Socket实例，并向指定远程主机和端口建立一个TCP连接。可以通过构造器指定地址，也可以通过`bind()`方法
* 通过`Socket`的输入输出流进行通信
* 使用Socket类的`close()`方法关闭连接

以下是一个使用Socket实例实现客户端的示例代码：

~~~java
public class TCPClient {
    public static void main(String[] args) throws IOException {
        Socket client = new Socket("localhost", 6666);
        System.out.println("建立TCP连接成功");
        OutputStream out = client.getOutputStream();
        InputStream in = client.getInputStream();
        new Thread(() -> receive(in)).start();
        new Thread(() -> send(out)).start();
    }

    public static void receive(InputStream in) {
        Scanner scanner = new Scanner(in);
        while (true) {
            System.out.println("Received: " + scanner.nextLine());
        }

    }

    public static void send(OutputStream out) {
        Scanner scanner = new Scanner(System.in);
        String buf;
        try {
            while ((buf = scanner.nextLine()) != null) {
                out.write((buf + "\n").getBytes(StandardCharsets.UTF_8));
                System.out.println("Send: " + buf);
            }
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

}
~~~

## TCP服务端

典型的TCP服务器有如下两个步骤：

* 创建一个`ServerSocket`实例并指定本地端口，监听指定端口收到的链接
* 重复执行：
  * 调用`ServerSocket`的`accept()`方法以获取下一个客户端连接。
  * 使用返回的`Socket`实例的输入输出流与客户端进行通信
  * 通信完成后，使用`Socket.close()`方法关闭客户端Socket连接。

以下是能同时接收和响应信息的服务端的实例代码：
~~~java
public class TCPServer {

    public static void main(String[] args) throws IOException {
        ServerSocket server = new ServerSocket();
        server.bind(new InetSocketAddress(6666));
        System.out.println("监听6666端口");
        Socket accept = server.accept();
        System.out.println("监听到连接");
        OutputStream out = accept.getOutputStream();
        InputStream in = accept.getInputStream();
        new Thread(() -> receive(in)).start();
        new Thread(() -> send(out)).start();
    }

    public static void receive(InputStream in) {
        Scanner scanner = new Scanner(in);
        while (true) {
            System.out.println("Received: " + scanner.nextLine());
        }

    }

    public static void send(OutputStream out) {
        Scanner scanner = new Scanner(System.in);
        String buf;
        try {
            while ((buf = scanner.nextLine()) != null) {
                out.write((buf + "\n").getBytes(StandardCharsets.UTF_8));
                System.out.println("Send: " + buf);
            }
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
~~~



## Socket

刚才我们已经介绍了`Socket`的简单用法，现在继续深入了解它的API和使用

### 创建Socket

Socket提供多种构造器以创建其实例：

~~~java
public Socket(String host, int port);
public Socket(InetAddress address, int port);
public Socket(String host, int port, InetAddress localAddr,int localPort);
public Socket(InetAddress address, int port, InetAddress localAddr,int localPort);
public Socket(Proxy proxy);
public Socket();
~~~

前四个构造器在创建了一个TCP Socket后，先连接到指定的远程地址和端口号，再返回实例。

前两个构造器没有指定本地地址和端口号，所以采用默认地址和可用的端口号。在有多个接口的主机上指定本地地址是有用的。

也可以使用代理来创建Socket连接。

最后一个构造器创建一个没有连接的Socket，在使用它进行通信前，必须进行显示连接(通过调用`connect()`)

### 操作Socket

~~~java
public void connect(SocketAddress endpoint);
public void connect(SocketAddress endpoint, int timeout);
public void bind(SocketAddress bindpoint);
public synchronized void close();
public void shutdownInput();
public void shutdownOutput();
public InputStream getInputStream();
public OutputStream getOutputStream();
~~~

`connect()`使指定的终端打开一个TCP连接

`close()`方法关闭套接字及其相关联的输入输出流

`shutdownInput()`方法关闭TCP流的输入端。还在流中未读取的数据，将被舍弃。

`shutdownOutput()`方法关闭TCP流的输出端，已经写入流的数据，将被尽量发送到另一端。

`getInputStream()/getOutputStream()`获取对应的输入输出流

`bind()`将该Socket绑定到指定的本地地址

### 获取/检测属性

~~~java
public InetAddress getInetAddress(); //返回socket连接到的地址
public InetAddress getLocalAddress(); //获取socket绑定到的本地地址
public int getPort(); // 返回此socket连接到的远程端口号
public int getLocalPort(); //返回此socket绑定到的本地端口号
public SocketAddress getRemoteSocketAddress(); //返回此socket连接到的远程地址
public SocketAddress getLocalSocketAddress();//返回此socket绑定到的本地地址
~~~





## ServerSocket

`ServerSocket`用于服务端监听来自客户端的`Socket`请求

### 创建

~~~java
public ServerSocket();
public ServerSocket(int port);
public ServerSocket(int port, int backlog);
public ServerSocket(int port, int backlog, InetAddress bindAddr);
~~~

以上后三个构造函数指定了`ServerSocket`监听的端口。

`backlog`:是socket上请求的最大挂起连接数

`bindAddr`:监听的主机地址，在多接口的主机上可以指定监听的接口地址，如果没有指定，默认监听该主机的所有地址。

而第一个构造函数创建的实例没有关联本地端口，在使用该实例前，必须使用`bind()`方法绑定一个端口号。

### 操作

~~~java
public void bind(SocketAddress endpoint);
public void bind(SocketAddress endpoint, int backlog);
public Socket accept();
public void close();
~~~

`bind()`方法为Socket关联一个本地端口

`accept()`方法为下一个传入的连接请求创建`Socket`实例，并将已成功连接的Socket实例返回。如果没有请求连接,方法将阻塞等待，直到有新的连接请求或者超时

`close()`方法关闭socket。

### 访问属性

~~~java
public int getLocalPort();
public InetAddress getInetAddress();
public SocketAddress getLocalSocketAddress();
~~~

以上方法返回`ServerSocket`绑定的本地地址和端口。

# UDP Socket

UDP协议提供了一种不同于TCP协议的端到端服务。实际上UDP协议只实现两个功能：

* 在IP协议的基础上添加了另一层地址(端口)
* 对在数据传输过程中可能产生的数据错误进行检查，并抛弃已损坏的数据

UDP Socket在使用时不需要进行连接，只需要将数据投递到指定地址即可，也不关心指定地址是否成功接受。使用它数据会出现丢失和重排。

Java提供`DatagramPacket`和`DatagramSocket`类以支持UDP Socket

客户端和服务器都使用`DatagramSocket`、`DatagramPacket`来发送和接受数据

## DatagramPacket

与TCP协议发送和接受字节流不同，UDP终端交换的是一种称为数据报文的自包含信息。这种信息在Java中表示为`DatagramPacket`实例。

发送信息时，需要创建一个包含了待发送信息的`DatagramPacket`实例，然后传递给`DatagramSocket.send()`方法。

接受信息时，需要创建一个分配了一些空间的`DatagramPacket`实例，然后传递给`DatagramSocket.receive()`方法以接受数据。

除了传输的信息本身外，每个`DatagramPacket`实例还附加了地址和端口信息，其具体含义取决于该数据报文是被发送还是被接收：

* 若是被接收的数据报文，`DatagramPacket`实例的地址则指明了目的地址和端口号，需显式指定
* 若是接收到的数据报文，`DatagramPacket`实例中的地址则指明了所收信息的源地址，由`receive()`方法填充

因此，服务器端可以修改接受到的`DatagramPacket`实例的缓存区内容，然后直接将这个实例发送回它的源地址。

### 创建

~~~java
public DatagramPacket(byte buf[], int offset, int length);
public DatagramPacket(byte buf[], int length);
public DatagramPacket(byte buf[], int offset, int length,InetAddress address, int port);
public DatagramPacket(byte buf[], int offset, int length, SocketAddress address);
public DatagramPacket(byte buf[], int length,InetAddress address, int port);
public DatagramPacket(byte buf[], int length, SocketAddress address);
~~~

创建构造器必须传入用于传递/接受数据的字节数组buf,同时可以指定数据的偏移量和有效大小。

地址和端口号的指定在创建`DatagramPacket`实例时是可选的，创建时没有指定，后续在使用实例前必须先指定地址。

### 处理地址

~~~java
public synchronized InetAddress getAddress();
public synchronized int getPort();
public synchronized void setAddress(InetAddress iaddr);
public synchronized void setPort(int iport);
public synchronized void setSocketAddress(SocketAddress address);
public synchronized SocketAddress getSocketAddress();
~~~

以上方法可以访问和修改`DatagramPacket`实例的地址信息。

### 处理数据

~~~java
public synchronized int getLength();
public synchronized void setLength(int length);
public synchronized int getOffset();
public synchronized byte[] getData();
public synchronized void setData(byte[] buf, int offset, int length);
public synchronized void setData(byte[] buf);
~~~

可以通过前两个方法访问和设置数据报文中数据部分的内部长度。

可以通过`getData()/setData()`访问和设置数据报文关联的字节数组。

## DatagramSocket

### 创建

~~~java
public DatagramSocket();
public DatagramSocket(SocketAddress bindaddr);
public DatagramSocket(int port);
public DatagramSocket(int port, InetAddress laddr);
~~~

创建`DatagramSocket()`实例时，需要将其于某个本地地址端口绑定，以监听该接口接收到的数据报文,或者从该本地接口发送数据

如果没有指定本地地址，那么该实例将监听所有的本地地址。

如果没有指定本地端口，或者将其设置为0，那么实例将于某个可用的本地端口绑定。

### 连接和关闭

~~~java
public void connect(InetAddress address, int port);
public void connect(SocketAddress addr);
public void disconnect();
public void close();
~~~

`connect()`方法用来设置socket实例的远程地址和接口。连接成功后，该socket只能和指定的地址和端口进行通信，任何向其他地址和端口发送数据报文的尝试都将抛出一个异常。

注意：该连接仅仅是本地操作，并不会在网络上实际建议一个端和端的连接。

`disconnet()`方法用来清空远程地址和端口号。

`close()`方法关闭此socket

### 访问地址信息

~~~Java
public InetAddress getInetAddress();
public int getPort();
public SocketAddress getRemoteSocketAddress();
public InetAddress getLocalAddress();
public int getLocalPort();
public SocketAddress getLocalSocketAddress();
~~~

前三个方法访问socket连接的远程地址信息

后三个方法访问socket绑定的本地地址信息

### 发送和接收

~~~java
public void send(DatagramPacket p);
public synchronized void receive(DatagramPacket p);
~~~

`send()`方法用来发送`DatagramPacket`实例，该方法不阻塞。

如果该socket已经建立了连接：

* 若发送的`DatagramPacket`实例中已经指明了地址，该地址必须和连接地址相同，否则将抛出异常
* 若发送的`DatagramPacket`实例中未指明地址，那么该数据包将发送到连接地址

`receive()`方法将阻塞等待，直到接收到数据报文，并将报文中的数据复制到指定的`DatagramPacket`实例中

## 使用模板

客户端

~~~java
DatagramSocket socket = new DatagramSocket();
socket.connect(new InetSocketAddress("localhost",6666));
byte[] bytes = "你好.".getBytes(StandardCharsets.UTF_8);
System.out.println("发送信息。");
DatagramPacket packet = new DatagramPacket(bytes, bytes.length);
System.out.println("发送成功。");
socket.send(packet);
socket.close();
~~~

服务端

~~~java
DatagramSocket socket = new DatagramSocket(6666);
byte[] bytes = new byte[256];
DatagramPacket packet = new DatagramPacket(bytes, bytes.length);
while (true){
    socket.receive(packet);
    String message = new String(bytes, 0, packet.getLength(), StandardCharsets.UTF_8);
    SocketAddress address = packet.getSocketAddress();
    System.out.println("接收到来自"+address+"的信息:"+ message);
}
~~~

在使用UDP数据报接收数据时，如果指定的字节数组长度不够，那么接收数据包时将截取并自动丢弃超过字节数组长度的信息

而一个UPD数据包最多能够负载65507个字节。

所以为了保障能够接收到UDP数据包的所有数据，作为服务端使用的`DatagramPacket`实例关联的字节数组长度最好为65507

## 广播和多播

到目前为止，socket处理的都是两个实体之间的通信，这种一对一的通信方法称为单播(unicast)

而在一些场景下，一个信息可能需要发送给多个接收者。这种情况下，我们可以为每个接收者单播一个数据副本。但是这样的效率可能非常低。同样的数据发送了多次，且浪费带宽。

我们可以将复制数据包的工作交给网络来做，而不是发送者负责。这就是网络提供的一对多服务

* 广播：本地网络中的所有主机都会接收到一份数据副本
* 多播：消息只发送给一个多播地址，网络只将数据分发给那些表示想要接收发送到该多播地址的数据的主机。

### 广播

使用`DatagramSocket`进行广播和单播数据类似，唯一区别是广播使用的是一个广播地址而不是一个常规的IP地址。

IPv4的广播地址(255.255.255.255)将消息发送到同一广播网络上的所有主机。本地广播信息不会被路由器转发。

### 多播

多播和单播之间的主要区别仍然是地址的形式。一个多播的地址指示了一组接收者。

* IPv4中的多播地址范围是224.0.0.0到239.255.255.255

* IPv6中的多播地址是任何由FF开头的地址

Java中使用`MulticastSocket`实例进行通信，它是`DatagramSocket`的子类，并在父类基础上提供了控制多播特定属性的api

