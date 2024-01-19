# Node编程基础

首先简单介绍一下Node程序的构成，看它是如何和操作系统交互的

## 控制台输出

Node.js中，我们仍然用`console.log()`向控制台输出，但是不同于浏览器的客户端JS，它在Node中是作为标准输出流进行输出的。

而Node程序的`console.error()`写入的是标准错误流。

Node程序可以把这些标准流重定向到一个文件或者管道。

## 命令行参数和环境变量

Node程序的输入首先从命令行参数获取，其次是从环境变量中获取。

Node程序可以从全局变量`process`的`process.argv`字符串数组中读取其命令行参数。这个数组的第一个元素始终是Node可执行文件的路径。第二个参数是Node执行的JS代码文件的路径。数组中剩下的所有元素都是在调用Node时，通过命令行传给它的空格分隔的参数

假设把如下Node程序保存到文件`argv.js`中：

~~~js
console.log(process.argv);
~~~

然后在对应文件夹下执行下面命令行：

~~~cmd
node argv.js Hello World
~~~

 将进行下面输出：

~~~js
[
  'D:\\Program Files\\nodejs\\node.exe',
  'D:\\Data\\Learning\\learn-project\\js-learn\\argv.js',
  'Hello',
  'World'
]
~~~

Node程序也会从环境变量中获取输入，Node把这些变量保存在`process.env`对象中使用

## 程序生命周期

node命令期待的参数是要执行的JS文件。这个JS文件通常会导入其他JS代码的模块，也可能定义自己的类和函数。不过Node基本上是自顶向下执行指定文件中的JSdiamagnetic。如果没有回调和事件处理。那么Node程序会在执行文件的所有代码时退出。

Node程序在运行完初始文件、调用完所有回调、不再有未决事件之前不会退出。基于Node的服务器监听网络连接，理论上会永远运行。

程序可以通过调用`process.exit()`可以强制退出。

如果程序中的代码抛出异常，也没有`catch`子句捕获该异常， 程序会打印栈追踪信息并退出。

Node中，发生在回调或事件监听器中的异常必须局部处理，否则异常不会传播到主程序上。

如果不想这些异常导致程序崩溃，可以注册一个全局处理程序，以备调用，防止崩溃：

~~~js
process.setUncaughtExceptionCaptureCallback(e=>{
    console.error("Uncaught exception:" , e)
})
~~~

类似的，如果Node程序的期约被拒绝，而且没有`.catch()`调用处理它，那么也会遇到类似的问题，到Node13为止，这不会导致程序退出的致命错误，但会在控制台打印出大量错误信息。未来Node的版本中，未处理的期约拒绝可能会变成致命错误。所以建议同样注册一个全局处理程序：

~~~js
process.on("unhandledRejection",(reason,promise) =>{
    //reason时传给.catch()函数的拒绝理由
    //promise是被拒绝的期约对象
})
~~~

## Node模块

Node的模块系统使用`require()`函数向模块中导入值，使用`exports`对象或者`module.exports`属性从模块中导出值。这些在之前已经介绍过。

Node 13 增加了对ES6模块的支持，同样支持基于`require()`的模块(Node称其为 CommonJS模块)，但这两者并不兼容。

所以Node在加载模块前，需要知道该模块是会使用`require()`和`module.exports`还是`import`和`export`。

告诉Node它要加载的是什么模块的最简单方式，就是将信息编码到不同的扩展名中：

* `Node`会将`.mjs`结尾的文件当作ES6模块来加载，假设其中使用了`import`和`export`，并不再提供`require()`函数
* `Node`会将`.cjs`结尾的文件到左一个CommonJS模块来加载，会提供`require()`函数，如果其中使用了`import`或者`export`声明，则抛出SyntaxError异常

如果没有明确给出`.mjs/.cjs`扩展名文件，Node会在同级目录以及所有包含目录中查找一个名为`package.json`的文件。一旦找到最近的`package.json`文件，Node会检查其中JSON兑现的顶级type属性

* 如果type属性值为module，Node会将文件按照ES6模块来加载
* 如果type属性值为commonjs，那么Node就按照CommonJS模块来加载该文件

如果没找找到`package.json`，那么Node默认会使用CommonJS模块。

## Node包管理器

在安装Node时，会同时安装一个`npm`的程序，这个程序时Node的包管理器，它可以下载和管理程序的依赖库。npm通过其程序根目录下的`package.json`文件跟踪依赖(以及与程序相关的其他信息),这个`package.json`是由npm创建的，如果想要在使用npm管理依赖的项目中使用ES6模块，需要同样在这个文件中添加`"type":"module"`

可以运行`npm install`命令下载安装需要的库及其所有依赖，并把所有包都安装到本地的`node_modules`目录下：

~~~cmd
npm install express
~~~

在通过npm安装一个包时，npm会在`package.json`文件中记录这个依赖。

# Node 默认异步

Node针对I/O密集型程序进行涉及和优化。Node的设计让实现高并发服务器非常容易。

Node采用了Web 使用的单线程JS编程模型，让其API默认异步和非阻塞实现了高层次的并发，同时保持了单线程的编程模型。

一般来说，传给异步Node函数的最后一个参数始终是一个回调。这个回调有两个参数:

* 第一个参数表示错误
  * 如果是null表示没有错误发生
  * 如果是Error对象或者是一个整数错误码或者字符串错误消息,那么说明一定出错了，此时回调函数的第二个参数可能就是null
* 第二个参数是最初调用的异步函数产生的数据或者返回的响应

下面代码演示了非阻塞的`readFile()`函数读取一个文件：

~~~js
const  fs = require("fs");

function readConfigFile(path,callback){
    fs.readFile(path,"utf-8",(err,text) =>{
        if (err){
            console.error(err);
            callback(null);
            return;
        }
        let data = null;
        try {
            data = JSON.parse(text);
        } catch (e){
            console.error(e);
        }
        callback(data);
    })
}

readConfigFile("./package.json",console.log)
~~~

Node出现在期约之前，但是由于它错误在先的回调相当一致，所以可以使用`util.promisify()`包装函数创建按其基于回调API的期约版。下面演示将`readConfigFile`重写返回一个期约对象：

~~~js
const  fs = require("fs");
const util = require("util");
const pfs = {
    readFile: util.promisify(fs.readFile)
}

function readConfigFile(path){
    return pfs.readFile(path,"utf-8").then(text => {
        return JSON.parse(text);
    })
}

readConfigFile("./package.json").then(json =>{
    console.log(json)
})
~~~

可以使用`async`和`await`简化这个基于期约的函数：

~~~js
async function readConfigFile(path) {
     let text = await pfs.readFile(path, "utf-8");
     return text;
}
~~~

虽然Node的编程模型默认是异步的，但是考虑到程序员的方便，Node也为其很多函数定义了阻塞、同步的版本，特别是文件系统模块中的函数。这些函数的名字最后通常都带有明确的Sync字样，例如：

~~~js
const  fs = require("fs");
function readConfigFileSync(path){
    return JSON.parse(fs.readFileSync(path,"utf-8"))
}

~~~

`Node`内置的非阻塞函数使用了操作系统的回调和事件处理程序。在Node中调用一个异步函数是，Node会在操作系统中注册某种事件处理程序。当操作完成时，它就能收到通知。你传给Node函数的回调会被保存在内部，当操作系统向Node发送相应事件时，Node就可以调用它的回调。

这种基于事件的并发，其核心时Node用单线程运行一个事件循环，当Node程序启动时，它会运行js文件中的代码，这些代码可能会至少调用一个非阻塞函数，导致一个回调或事件处理程序被注册到操作系统中。当Node执行到程序的末尾时，它会一直阻塞直到有事件发送，此时操作系统会再启动并运行它。

Node把操作系统事件映射到你注册的JS回调，然后调用该函数，这个回调函数可能会调用更多非阻塞的Node函数，导致注册更多的事件处理程序。当回调函数运行完毕，Node又会进入睡眠状态，如此循环。

# 缓冲区

Node有一个比较常用的数据类型就是Buffer，常用于从文件或者网络读取数据。Node再JS支持定型数组之前诞生，因此没有表示无符号字节的`Uint8Array`。Node的Buffer类就是为了满足这个需求而设计的。在JS语言支持定型数组后，Node的Buffer就成为了`Uint8Array`的子类

Buffer与其超类`Uint8Array`的区别在于，它是用来操作JS字符串的。因此Buffer中的字节可以从字符串初始化而来，也可以转换为字符串。在这个过程中需要指定Buffer类的编码类型例如：

~~~js
let b = Buffer.from([0x41,0x42,0x43]);
console.log(b.toString()) //ABC
console.log(b.toString("hex")) //414243
~~~

# 事件和EventEmitter

所有Node API都是默认异步的，对于一些API，异步性表现为回调。但一些更复杂的API则是基于事件的。

在Node中，发送事件的对象都是`EventEmitter`或其子类的实例：

~~~js
const EventEmitter = require("events");
const net = require("net");
let server = new net.Server();
console.log(server instanceof EventEmitter) // true
~~~

`EventEmitter`的主要功能是允许我们使用`on()`方法注册事件处理程序。要事件的回调会传哪些参数，需要阅读对应的文档。

~~~js
const net = require("net");
let server = new net.Server();
server.on("connection",socket =>{
    socket.end("Hello World","utf8")
})
~~~

除了`on()`还可以使用`addListener()`注册事件监听器。对应的，可以使用`off()`或者`removeListener()`去掉之前注册的事件监听器。

还有一个特殊的`once()`方法，它注册的事件监听器，被触发一次后就会被自动清除。

事实上，对事件监听器的调用仍然是同步的，如果在同一时间发送了多个事件，那么这写对应的监听器会依次被顺序执行。也就是说，如果某个事件监听器执行的事件过长，就会阻塞其他的事件监听器的执行。所以，如果某个事件发送时需要执行大量计算，那么最好在处理程序中使用`setTimeout()`将该计算调度为异步执行。

`EventEmitter`类定义了一个`emit()`方法，可以导致其注册的事件监听器被调用。这个函数的第一个参数传入事件类型的名字。后续的所有参数都会称为注册的事件处理程序回调的参数。

事件处理程序被调用时，其this值也会被设置为`EventEmitter`对象。

事件处理程序返回的任何值都会被忽略。但是如果某个事件处理程序抛出异常，则该异常会从`emit()`调用中传播出来。

# 流

在实现处理数据的算法时，最简单的方法通常是先把所有数据读取到内存中，进行处理，然后再把数据写到某个地方。但是，如果要读取的数据非常大，就会占用大量的内存，甚至导致内存溢出。

针对这个问题的解决方案就是使用基于流的算法，其本质是将数据分割成小块，内存不会保存全部数据，而是依次读取部分数据，处理完这部分数据后再读取下一份数据。这样内存利用率更高，处理速度也更快。

Node的网络API是基于流的，Node的文件系统模块也定义了流API用于读写文件。

Node支持4种基本流：

* 可读流(readable)，是数据源
* 可写流(writable),是数据的接受地或目的地
* 双工流(duplex),把可读流和可写流组合成一个对象。
* 转换流(transform):转换流也是可读和可写的，但是和双工流有一个重要区别：写入转换流的数据在同一个流会变成可读的(通常是某种转换后的形式)

可读流必须从某个地方读取数据，而可写流必须把数据写到某个地方。因此每个流都有两端：输入端和输出端。

流的实现种总会包含一个内部缓冲区，用于保存已经写入但尚未读取的数据。缓冲有助于保证在读取时有数据，而在写入有空间。但这两点都无法绝对保证。这种情况下，读取端或写入段就必须等待。

在基于线程并发的编程环境，流API通常存在阻塞调用(例如java 的Stream API)。但在基于事件的并发模型种，阻塞调用就没有意义了，所以Node的流API是基于事件和回调的。

## 管道

有时候，我们需要把从流中读取的数据写入另一个流。可以通过`pipe()`方法将两个流连接为管道来实现。例如：

~~~js
const fs = require("fs");
function pipeFileToSocket(filename,socket){
    fs.createReadStream(filename).pipe(socket)
}
~~~

转换流特别适合于管道一起使用，可以创建多个流的传输管道。下面示例实现了文件压缩：

~~~js
const fs = require("fs");
const zlib = require("zlib");


function gzip(filename,callback){
    let source = fs.createReadStream(filename);
    let destination = fs.createWriteStream(filename+".gz");
    let gzipper = zlib.createGzip();
    source.on("error",callback)
        .pipe(gzipper)
        .pipe(destination)
        .on("error",callback)
        .on("finish",callback);
}


gzip("./picture.png",err =>{
    if (err)
        console.error(err.stack)
    else
        console.log("压缩完成")
})
~~~

## 自定义Transform流

有时候我们需要对流的数据做自定义的处理，我们可以读取流，处理完后再写入流。但页可以实现自己的Transform流来完成相应的处理

例如，下面函数类似于Unix 的grep命令：
