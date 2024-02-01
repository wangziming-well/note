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

如果没有明确给出`.mjs/.cjs`扩展名文件，Node会在同级目录以及所有包含目录中查找一个名为`package.json`的文件。一旦找到最近的`package.json`文件，Node会检查其中JSON对象的顶级type属性

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

~~~js
const stream = require("stream");

class GrepStream extends stream.Transform {
    constructor(pattern) {
        super({decodeStrings:false}); //不把字符串转换回缓冲区
        this.pattern = pattern; // 要匹配的正则表达式
        this.incompleteLine =  ""; //最后一个数据块的剩余数据
    }
    //当一个字符串准备好可以转换时会调用这个方法，它应该把转换后的数据传给指定的回调函数
    _transform(chunk,encoding,callback){
        //这个方法期待一个字符流
        if (typeof chunk !== "string" ){
            callback(new Error("Expected a string but got a buffer"))
            return;
        }
        //把chunk添加到之前不完整的行，并将所有内容按换行符分隔为数组
        let lines = (this.incompleteLine + chunk).split("\n");
        //数组的最后一个元素是新的不玩增行
        this.incompleteLine = lines.pop();
        //过滤筛选匹配的行
        let output = lines.filter(l => this.pattern.test(l)).join("\n");
        //如果有匹配，在最后加上一个换行符
        if (output)
            output += "\n";

        callback(null,output);
    }
    //这个方法在流关闭前会被调用，在方法里可以把最后的数据写出
    _flush(callback){
        if (this.pattern.test(this.incompleteLine))
            callback(null,this.incompleteLine + "\n")
    }
}

let pattern = /^\s*?\d*?\s*?$/g;
process.stdin
    .setEncoding('utf8')
    .pipe(new GrepStream(pattern))
    .pipe(process.stdout)
    .on("error",() => process.exit());
~~~

## 异步迭代

在Node12及以后，可读流是异步迭代器，所以可以在async函数中使用`for/await`循环从流中读取字符串或者Buffer块，我们用这种方式来重写上面的grep程序：

~~~js
async function grep(source , destination, pattern , encoding="utf-8"){
    source.setEncoding(encoding);

    destination.on("error" ,err => process.exit());
    let incompleteLine = "";
    for await(let chunk of source){
        let lines = (incompleteLine + chunk).split("\n");
        incompleteLine = lines.pop();
        for(let line of lines){
            if (pattern.test(line))
                destination.write(line + "\n",encoding);
        }

    }
    if (pattern.test(incompleteLine))
        destination.write(incompleteLine + "\n",encoding);
}


let pattern = /^\s*?\d*?\s*?$/g;
grep(process.stdin,process.stdout,pattern)
    .catch(err => {
        console.error(err);
        process.exit();
    })
~~~

## 写入流及背压处理

一个写入流有方法`write()`，这个方法以一个缓冲区或字符串作为第一个参数。

如果传入一个缓冲区，则该缓冲区的字节会被直接写入。

如果传入了一个字符串，则字符串在被写入前会被编码成字节缓冲区。编码格式通过第二个字符串参数指定。

可写流有默认编码，在给`write()`方法只传一个字符串参数时使用。这个默认编码通常是`utf8`，但可以通过`setDefaultEncoding()`方法显示地设置。

`write()`可选地接收一个回调函数作为第三个参数。这个回调会在数据已经实际写入，不再 存在于可写流内部的缓冲区时被调用。

`write()`返回一个布尔值，在调用时，如果内部缓冲区未满，则返回true，如果已满或太满，则返回false。这个返回值是建议性的，可以忽略。

`write()`方法返回false值是一种背压(backpressure)的表现。背压是一种消息，表示你向流中写入数据的速度超过了它的处理能力。对这种背压的正确反应是停止调用`write()`，直到流发出`drain`（耗尽）事件，表示缓冲区又有空间了。

手工处理背压要面临一些问题，因为可能的情况很多，很难编写通用的算法。所以我们通常使用`pipe()`方法，这个方法会自动处理背压。

## 通过事件读取流

Node可读流有两种模式，每种模式都有自己的API。如果不使用管道或异步迭代处理可读流，那么就需要用这两种基于事件的API中的一种来处理流。注意不能两种API混用

### 流动模式

在流动模式(flowing mode)下，当可读数据到达时，会立即以`data`事件的形式发送。所以在这种模式下读取流，只需要为data事件注册一个事件处理程序。

新创建的流并非一开始旧处于流动模式，注册`data`事件处理程序会把流切换到流动模式。这意味着流在注册data事件监听器前不会发送`data`事件。

如果使用流动模式从一个可读流读取数据、处理数据，然后再把数据写入要给可写流，那么可能需要处理可写流的背压。

如果此时发送背压，可以调用可读流的`pause()`方法暂时停止`data`事件。然后，当从可写流接收到“耗尽”事件时，可以调用可读流的`resume()`方法，再次启动`data`事件。

处于流动模式的流会在到达流末尾时发出一个`end`事件。如果错误发送，会发出一个`error`事件。

接下来演示一个用流动模式实现的文件复制函数：

~~~js
const fs = require("fs");

function copyFile(sourceFilename,destinationFilename,callback){
    let input = fs.createReadStream(sourceFilename);
    let output = fs.createWriteStream(destinationFilename);
    //注册data事件，将可读流中的内容写入到可写流中
    input.on("data",chunk => {
        let hasRoom = output.write(chunk);
        //如果可写流已满，则暂停可读流的读取
        if (!hasRoom)
            input.pause();
    })
    //可读流读取完毕时，同时关闭可写流
    input.on("end",()=>{
        output.end();
    })
    //可读流发生错误时，调用回调并退出程序
    input.on("error",err =>{
        callback(err);
        process.exit();
    })
    //可写流有空间时，通知可读流继续发送data事件
    output.on("drain",()=>{
        input.resume();
    })
    //可写流发送错误时，调用回调并退出程序
    output.on("error",err => {
        callback(err);
        process.exit();
    })
    //可写流结束是，调用回调并退出
    output.on("finish",()=>{
        callback(null);
    })

}

copyFile("./picture.png","./picture-copy.png",err =>{
        if (err)
            console.error(err);
        else
            console.log("done.")
    })
~~~

### 暂停模式

可读流的另一种模式是暂停模式，这个模式是流开始时所处的模式。

如果不注册`data`事件，也不调用`pipe()`方法，那么可读流就一直处于暂停模式。

在暂停模式下，需要显示调用`read()`方法从流中拉取数据。这个方法不是阻塞方法，如果流中没有可读数据，它就返回null。

暂停模式也是基于事件的，可读流会在暂停模式下发送`readable`事件，表示流中有可读数据。此时，可以调用`read()`方法读取该数据，并且需要在循环中反复调用`read()`直到它返回null，这样才能完全耗尽流中的缓冲区，从而在将来再次触发新的`readable`事件。

如果只调用了一次`read()`方法，导致缓冲区中仍有数据，那么之后就不会收到`readable`事件，程序就可能被挂起

处于暂停模式的流也会发送`end`和`error`事件。

如果是从一个可读流读数据，处理数据后写入可写流，那么暂停模式并非好的选择。为了正确处理背压，我们只希望在输入流可读且输出流未满时读取数据。而在暂停模式中，因为要保证循环调用`read()`方法直到返回null，要同时达到这两个目标处理起来很麻烦。

下面示例演示了使用暂停模式为指定文件的内容计算SHA26散列值。

~~~js
const fs = require("fs");
const crypto = require("crypto");

function sha256(filename,callback){
    let input = fs.createReadStream(filename);
    let hasher = crypto.createHash("sha256");

    input.on("readable",() =>{
        let chunk;
        while (chunk = input.read())
        hasher.update(chunk);
    })

    input.on("end",() =>{
        let hash = hasher.digest("hex");
        callback(null,hash);
    })

    input.on("error",callback);
}

sha256("./picture.png",(err,hash) =>{
    if (err)
        console.error(err);
    else
        console.log(hash)
})
~~~

# 全局Process对象

全局Process对象有很多有用的属性和函数，通常与当前运行的Node进程的状态有关。我们介绍一些常用的：

~~~js
process.argv //包含命令行参数的数组
process.arch //cpu架构：如x64
process.cwd() //返回当前工作目录
process.chdir() //设置当前工作目录
process.cpuUsage() //报告cpu使用情况
process.env //环境变量对象
process.execPath //Node可执行文件的绝对路径
process.exit() //终止当前程序
process.exitCode //程序退出时报告的整数编码
process.getuid() //返回当前用户的Unix用户ID
process.hrtime.bigint() //返回纳秒级时间戳
process.kill() //向另一个进程发送信息
process.memoryUsage() //返回一个包含内存占用细节的对象
process.nextTick() //类似于setImmediate(),立即调用一个函数
process.pid //当前进程的进程ID
process.ppid //父进程ID
process.platform // 操作系统，如Linux、Win32等
process.resourceUsage() // 返回包含资源占用细节的对象
process.setuid() //通过ID或名字设置当前用户
process.title // 出现在ps列表中的进程名
process.uptime() // 返回Node正常运行的时间
process.version //Node的版本字符串
process.versions // Node依赖库的版本字符串
~~~

os模块提供了与process类似的Node所在计算机和操作系统的细节。

~~~js
const os = require("os");
os.arch(); //cpu架构：如x64
os.constants //一些系统相关的有用的常量
os.cpus() //关于cpu核心的数据
os.endianness() // CPU的原生字节序：BE或则和LE(大端序，小端序)
os.EOL // 操作系统原生的行终止符: \n 或者 \r\n
os.freemem() //自由RAM数量(字节)
os.getPriority() //进程优先级
os.homedir() // 当前用户的主目录
os.hostname() // 当前计算机的主机名
os.loadavg() // 返回1、5、15分钟的平均负载
os.networkInterfaces() // 可用网络连接的细节
os.platform() //操作系统如Linux
os.release() // 操作系统的版本号
os.setPriority() //尝试设置进程的优先级
os.tmpdir() //默认的临时目录
os.totalmem() // 返回RAM的总数(字节)
os.type() // 返回操作系统
os.uptime() // 返回系统正常运行的时间(s)
os.userInfo() //返回当前用户的信息对象
~~~

# 操作文件

Node的fs模块是用于操作文件和目录的综合性API。path模块是fs模块的补充，定义了操作文件和目录名的常用函数。

fs模块定义了大量API，主要因为每种基本操作都有很多变体。很多函数都是非阻塞的、基于回调的、异步的。不过，通常这样的函数都会有一个同步阻塞的变体。如`fs.readFile()`和`fs.readFileSync()`。在Node10之后，很多这样的函数还会有一个基于期约的异步变体，如`fs.promise.readFile()`

多数fs中的函数都以一个字符串作为第一个参数，用于指定要操作的文件的路径。但是其中某些函数也有变体支持一个整数“文件描述符”而非字符串路径作为第一个参数。这样的变体会以字母`f`开头。例如`fs.truncate()`和`fs.ftruncate()`

fs模块中还有少量函数有名字前面加`l`的变体，这个变体与基本函数类似，但不会跟踪文件系统中的符号链接，而是直接操作符号链接本身。

## 路径、文件描述符和FileHandle

文件通常时通过路径来指定的，也就是文件本身的名字再加上文件所在的目录层次

* 绝对路径是从文件系统的根目录开始的
* 相对路径是相对于其他某个路径(通常是当前工作路径)才有意义

不同操作系统使用不同的字符来分隔目录名，另外`../`父目录部分也需要特殊处理。

Node的path模块和一些Node特性可以帮助我们处理路径：

~~~js
//一些重要路径
process.cwd() //当前工作目录的绝对路径
__filename //保存当前代码的文件的绝对路径
__dirname //保存__filename的目录的绝对路径
os.homedir() // 用户的主目录

const path = require("path");

path.sep //路径分隔符 /或者 \ ，取决于操作系统
//提供了简单的解析函数
let p = "src/pkg/test.js"
path.basename(p) // test.js
path.extname(p) // .js
path.dirname(p) // src/pkg
path.basename(path.dirname(p)) //pkg
path.dirname(path.dirname(p)) // src

//清理路径、规范化路径
path.normalize("a/b/c/../d") // a/b/d/ :去掉../
path.normalize("a/./b") // a/b : 去掉 ./
path.normalize("//a//b//") // /a/b/ :去掉重复的/

//组合路径片段，添加分隔符，并规范化
path.join("src","pkg","t.js") // src/pkg/t.js

//由多个路径片段，返回一个绝对路径，从最后一个参数开始，反向处理，直到构建起绝对路径或者相对于process.cwd()解析得到绝对路径
path.resolve() // process.cwd()
path.resolve("t.js") // path.join(process.cwd(),"t.js")
path.resolve("/tmp","t.js") // /tmp/t.js
path.resolve("/a","/b","t.js") // /b/t.js
~~~

除了文件名，很多fs函数接收文件描述符。文件描述符是操作系统级的整数引用，可以用来访问文件。

调用`fs.pen()/fs.openSync()`可以的大宋一个指定文件的文件描述符。进程依次只能打开有限个数的文件，所以再操作完成后，一定要在文件描述符上调用`fs.close()`

最后，`fs.promises`定义的基于期约的API中，与`fs.open()`对应的是`fs.promises.open()`，它返回一个期约，解决为一个`FileHandle`对象。

## 读文件

可以通过流来处理文件内容，也可以通过低级API来一次性读取文件的内容。

如果要打开的文件很小，或者内存的占用或性能并非首要考虑因素，可以通过一次调用读取文件的全部内容。

~~~js
const fs = require("fs");
//同步读取文件
let buffer = fs.readFileSync("picture.png");
let text = fs.readFileSync("package.json","utf8");
//异步读取文件
fs.readFile("picture.png",(err ,buffer) =>{
    if (err)
        ; //处理错误
    else
        ;//处理文件字节
})
//基于期约的异步读取
fs.promises
    .readFile("package.json","utf-8")
    .then(processFileText)
    .catch(handleReadError);
//或者在async函数中使用await和期约API
async function processText(filename,encoding){
    let text = await fs.promises.readFile(filename,encoding);
    //处理文本
}
~~~

如果可以顺序处理文件内容，同时不需要把文件内容全部放到内存中，那么可以使用流

可以直接使用流和`pipe()`方法将一个文件内容输出到其他流中：
~~~js
function printFile(filename,encoding){
    fs.createReadStream(filename,encoding).pipe(process.stdout);
}
~~~

如果需要更精细的控制，可以打开文件或者文件描述符，然后再使用`fs.read()/fs.readSync()/fs.promises.read()`从文件中指定的来源将指定数量的字节读取到指定目标位置的指定缓冲区：

~~~js
const fs = require("fs");

fs.open("picture.png",(err,fd) =>{
    if (err){
        //报告错误
        return;
    }

    try{
        fs.read(fd,Buffer.alloc(400),0,400,20,(err,n,b) =>{
            //err是错误
            //n是实际读取的字节数
            //b是读入字节的缓冲区
        })
    } finally {
        fs.close(fd); //关闭文件描述符
    }

})
~~~

如果要从文件中读取多个数据块，那么基于回调的`read()`API使用起来会很繁琐，此时我们可以使用同步API或者基于期约的API来读取：

~~~js
const fs = require("fs");

function readData(filename){
    let fd = fs.openSync(filename);
    try {
        let header = Buffer.alloc(12);
        fs.readSync(fd,header,0,12,0);
        let magic = header.readInt32LE(0);
        if (magic !== 0xDADAFEED)
            throw  new Error("File id of wrong type")
        let offset = header.readInt32LE(4);
        let length = header.readInt32LE(8);
        let data = Buffer.alloc(length);

        fs.readSync(fd,data,0,length,offset);
        return data;

    } finally {
        fs.closeSync(fd);
    }
}
~~~

## 写文件

Node中写文件和读文件类型，但也有不同之处，如通过写入一个不存在的文件名可以创建一个新文件。

和读文件一样，Node中有三种写文件的基本方式，如果需要一次性写入，可以调用`fs.writeFile()`、`fs.writeFileSync()`和`fs.promises.writeFile()`一次性写入全部内容：

~~~js
fs.writeFileSync(path.resolve(__dirname,"settings.json"),JSON.stringify(settings),'utf-8');
~~~

相干函数`fs.appendFile()/appendFileSync()/promises.appendFile()`也类似，但是它们会再指定文件存在时，将数据追加到已有数据的末尾，而不是重写已有的文件内容。

如果要写入文件的数据并不全部都在内存中，可以使用可写流：

~~~js
const fs = require("fs");

let output = fs.createWriteStream("numbers.txt");
for(let i = 0 ;i<100;i++)
    output.write(`${i}\n`)
output.end();
~~~

如果想要以多个块的形式将数据写入文件，并想精细控制每个块写入文件中的位置。那么可以使用`fs.open()`或其变体来打开文件，然后把生成的文件描述符传给`fs.write()`或`fs.writeSync()`函数。这两个函数对字符串和缓冲区有不同的变体。

* 字符串变体接收一个文件描述符、字符串和文本中写入该字符串的位置
* 缓冲区变体接收一个文件描述符、缓冲区、偏移量、缓冲区中数据块的长度，以及再文件中写入该数据库字节的位置。

如果要写入一组Buffer对象的树，可以调用一次`fs.writev()/writevSync()`来完成操作。

### 文件模式

在写读文件时，我们使用`fs.open()/openSync()`时之传入了文件名。但在写文件时，我们必须同时传入第二个字符串参数，用于指定打开这个文件描述符的模式。有几个可用的标志字符串：

* `w`:只读
* `w+`:读写
* `wx`：只创建新文件，如果指定的文件存在则失败
* `wx+`：为创建新文件，并且可以读取，如果指定的文件存在则失败
* `a`:追加模式，原有数据不会被覆盖
* `a+`追加模式，也允许读取

如果不传第二个参数，则默认使用`r`标志位，即只读。

注意，将这些标志传给其他写入方法同样有效：

~~~js
fs.writeFileSync("message.log","hello",{flag:"a"});
~~~

### truncate

可以通过`fs.truncate()`和其变体方法截断文件后面的内容

这几个函数以路径为第一个参数，以一个长度作为第二个参数，将文件修改为指定的长度。如果省略长度，则使用默认值0，结果文件会变空。

如果指定的程度超过了当前文件大小，那么文件会以0字节扩展到新大小。

## 文件操作

### 复制文件

fs定义了文件复制的方法`fs.copyFlie()`。以及它的两个变体。

这几个函数都接收原始文件的名字和副本的名字作为前两个参数。这两个参数可以时字符串、URL或Buffer对象。

它们还接收一个可选的第三个整数参数，用于指定标志，以控制复制操作的细节。

下面是例子：

~~~js
const fs = require("fs");

fs.copyFileSync("./picture.png","./picture1.png");
//COPYFILE_EXCL参数表示只在新文件不存在时复制
fs.copyFile("./picture.png","./picture3.png" ,fs.constants.COPYFILE_EXCL,err=>{
    //这个回调在复制完成时被调用。如果出错，err将非空
})

//第三个参数是按位编码的，所以可以使用按位与同时设置多个参数
//COPYFILE_FICLONE 表示如果系统支持，新的副本将是原始文件的一个写时复制副本 copyOnWrite
fs.promises.copyFile("./picture.png","./picture2.png",
    fs.constants.COPYFILE_EXCL| fs.constants.COPYFILE_FICLONE)
    .then(() => console.log("copy complete"))
    .catch(err => console.error("copy failed",err))
~~~

### 移动或重命名文件

`fs.rename()`函数(以及相应的变体)可以移动或重命名文件。调用它要传入当前文件路径和期望的新文件路径。

~~~js
fs.renameSync("./picture-copy.png","./back/picture.png")
~~~

注意，新文件路径中的文件夹必须已经存在。

### 创建文件链接

函数`fs.link()`和`fs.symlink()`和其变体与`fs.rename()`有相同的签名，其行为类似于文件复制，但是它们只分别创建硬链接和符号链接。

~~~js
fs.linkSync("./picture.png","./picture-link.png")
fs.symlinkSync("./picture.png","./picture-symlink.png")
~~~

### 删除文件

`fs.unlink()`和其变体方法用来删除文件：

~~~js
fs.unlinkSync("./picture-link.png")
~~~

## 文件元数据

`fs.stat()`和其变体方法可以取得指定文件或目录的元数据，例如：

~~~js
let stats = fs.statSync("./picture.png");
stats.size //文件大小
~~~

该方法返回一个`Stats`对象，记录了如文件大小、名称、是否是文件等文件元数据。

`fs.lstat()`和`fs.stat()`类似，只是在指定文件为符号链接时，Node会返回链接本身的元数据，而不会追踪链接。

## 操作目录

在Node中要创建新目录，可以是哟共`fs.mkdir()`和其变体。

第一个参数时要创建的目录的路径，第二个参数是可选的整数，表示新目录的模式，或者也可以传入一个包含可选mode和recursive属性的对象。如果recursive属性为true，这个函数会创建路径中所有不存在的目录：

~~~js
fs.mkdirSync("dir1/dir2",{recursive:true})
~~~

`fs.mkdtemp()`及其变体可以创建临时目录，它接收一个传入的路径前缀，然后在后面追加一些随机字符(这对安全很重要),然后以该名字创建一个目录，最后返回这个目录的路径

要删除一个目录，可以使用`fs.rmdir()`和其变体。注意要删除的目录必须是空目录。

~~~js
let tempDirName
try {
    tempDirName = fs.mkdtempSync("./temp");
    //在这里对临时目录执行一些操作
} finally {
    fs.rmdirSync(tempDirName)
}
~~~

fs模块提供两组不同的API用于列出目录的内容：

`fs.readdir()`和其变体一次性读取整个目录，然后返回要给字符串数组或者一个指定了名字和类型的Dirent对象的数组。此时返回的文件名并非完整路径，而是单指文件的本地名。

~~~js
let tempFiles = fs.readdirSync("../");

fs.promises.readdir("../",{withFileTypes:true})
    .then(entries =>{
        entries.filter(entry => entry.isDirectory())
            .map(entry => entry.name)
            .forEach(name => console.log(name))
    }).catch(console.error);
~~~

如果要列出的目录很多，可以hi哟弄个基于流的`fs.opendir()`和其变体。这个函数返回一个Dir对象，表示指定的目录。

可以使用这个对象的`read()`和`readSync()`方法读取一个`Dirent`对象。如果给`read()`传了回调，那么它会调用这个回调，如果省略了回调，那么它返回一个期约。

使用Dir对象最简单的方式是将其作为异步迭代器，配合`for/await`循环：

~~~js
const fs = require("fs");
const path = require("path");

async function listDirectory(dirpath){
    let dir = await fs.promises.opendir(dirpath);

    for await( let entry of dir){
        let name = entry.name;
        if (entry.isDirectory())
            name += "/";
        let stats = await fs.promises.stat(path.join(dirpath,name));
        let size = stats.size;
        console.log(String(size).padStart(10),name);
    }
}

listDirectory("../");
~~~

# 网络连接

## HTTP



Node中的http、https和http2模块是功能完整但相对低级的HTTP协议实现。这些模块定义了实现HTTP客户端和服务器的所有API。我们只用示例演示编写简单的客户端和服务器

发送HTTP GET请求的最简单方式是使用`http.get()`或者`https.get()`。这两个函数的第一个参数是要获取的URL。第二个参数是一个回调，当服务器响应开始到达时这个回调会以一个`IncomeingMessage`对象被调用。调用对象时，HTTP状态和头部已经可以读取，但是响应体还未就绪。IncomingMessage是一个可读流。一个示例如下：

~~~js
const https = require("https");

https.get("https://www.baidu.com" ,response=>{
    if (response.statusCode !== 200){
        console.log("error,Status Code:" + response.statusCode)
        response.resume(); //防止内存泄漏
    } else {
        let body = "";
        response.setEncoding("utf-8");
        response.on("data" ,chunk => body += chunk);
        response.on("end",()=>console.log(body))
    }
})
~~~

更加通用的是`http.request()`和`https.request()`函数。下面演示它的使用方式：

~~~js
const https = require("https");

let requestOptions = {
    method: "POST",
    host: 8080,
    path: "http://localhost",
    headers: {
        "Content-Type": "application/json"
    }
}
let request = https.request(requestOptions);

request.write(JSON.stringify({key:"Hello"}))
request.end();

request.on("error", console.error);
request.on("response" , response => {
    if (response.statusCode !==200){
        console.log("Error,Status Code:" + response.statusCode);
        response.resume();
        return;
    }
    let body = "";
    response.setEncoding("utf-8");
    response.on("data" ,chunk => body += chunk);
    response.on("end",()=>console.log(body))
})
~~~

除了发送HTTP七个球，Node也允许编写响应这些HTTP请求的服务器。基本流程如下：

* 创建一个Server对象
* 调用它的`listen()`方法，开始监听指定端口的请求
* 为`request`事件注册处理程序，读取请求，然后写入响应

下面演示一个简单的HTTP服务器：

~~~js
const http = require("http");
let url = require("url");
let path = require("path");
let fs = require("fs");

let server = new http.Server();
server.listen(8080);

server.on("request",(request,response) =>{
    let endpoint = url.parse(request.url).pathname;
    if ("/pic" === endpoint){
        let stream = fs.createReadStream("D:\\Data\\Temporary\\picture.png");
        stream.once("readable",() =>{
            response.setHeader("Content-Type","application/octet-stream");
            response.writeHead(200)
            stream.pipe(response);
        })
        server.on("error",(err) =>{
            response.setHeader("Content-Type","text/plain; charset=UTF-8");
            response.writeHead(404);
            response.end(err.message);

        })
    } else {
        response.setHeader("Content-Type","text/plain; charset=UTF-8");
        response.writeHead(404);
        response.end("找不到资源");
    }
})
~~~

## Socket

net模块提供网络socket的API。net模块定义了Server和Socket类，

要创建服务器，可以调用`net.createServer()`，然后调用返回的对象的`listen()`方法监听指定端口的连接。Server对象会在客户端连接到该端口时生成`connnect`事件。而传给事件监听器的就是一个Socket对象，这个Socket对象时一个双工流，可以使用它从客户端读取数据和向客户端写入数据。在这个Socket对象删调用`end()`可以断开连接

通过`net.createConnection()`创建客户端，传一个端口号和主机名。返回一个Socket对象，通过这个对象向服务器交换数据。

下面是一个服务端的演示：

~~~js
const net = require("net");

let server = net.createServer();
server.listen(6789,() => console.log("Delivering laughs on port 6789"));

server.on("connection",socket => {
    socket.write("Hello");
    socket.end();
})
~~~

然后是客户端：

~~~js
const net = require("net");

let socket = net.createConnection(6789,"localhost");
socket.pipe(process.stdout);
process.stdin.pipe(socket);
socket.on("close",() => process.exit());
~~~

# 子进程

Node中的`child_process`模块定义了一些函数，用于在子进程中运行其他程序。

## `execSync()/execFileSync()`

运行其他程序的最简单方式是使用`child_process.execSync()`。这个函数的第一个参数是要运行的命令，它会创建一个子进程，在该进程中运行一个命令行解释器`shell`，并使用该解释器执行传入的命令。命令执行期间会阻塞，直到命令退出。

如果命令中有错误，则`execSync()`会抛出异常。否则，函数将返回该命令写入器标准输出流的任何内容。如果命令向标准错误流写入了任何输出，则该输出只会传给父进程的标准错误流。

默认情况下，这个返回值是一个缓冲区，可以在可选的第二个参数中设置一个编码，从而得到一个字符串。例如：

~~~mysql
const childProcess = require("child_process");
const iconv = require('iconv-lite');
let buffer = childProcess.execSync("ipconfig");
str = iconv.decode(buffer, 'gbk');
~~~

因为window平台shell使用gbk编码，但是node.js本身不支持gbk编码。所以这里用了三方库`iconv-lite`来进行编码

直接给`eecSync()`传递命令行字符传可以利用命令行的一些特性：如可以包含多个分号分隔的命令，可以使用文件名通配符、管道和输出重定向。

如果不需要这样的命令行特性，可以使用`child_process.execFileSync()`来避免启动命令行，这个函数直接执行程序，不调用命令行。它的第一个参数传入可执行文件，第二个参数传入命令行参数数组，例如：

~~~js
let buffer = childProcess.execFileSync("ping",["baidu.com"]);
let s = iconv.decode(buffer,"gbk");
~~~

## 子进程选项

`execSync()`和其他很多`child_process`函数都有可选的参数对象，用于指定子进程如何运行，下面列出比较重要的属性(注意，并非所有选线都适用于所有子进程函数)

* `cwd`指定子进程的当前工作目录。默认为`process.cwd()`
* `env`指定子进程有权访问的 环境标量。默认为`process.env`
* `input`指定作为子进程标准输入数据的字符串或缓冲区。这个选项只能用于不返回`ChildProcess`对象的同步函数
* `maxBuffer`指定exec函数可以手机的最大输出字节数(不适用于`spawn()`和`fork()`,它们使用流)。如果子进程产生的输出超过了这俄格值，那么会杀死子进程并以错误退出。
* `shell`：指定命令行解释器可执行文件的路径或true。
  * 对正常执行命令行程序的子进程函数，可以指定shell可执行文件以指定使用哪个命令行
  * 对正常不使用命令行的函数，将这个选项置为true或者指定shell路径可以让其使用命令行
* `timeout`：指定允许子进程运行的最长毫秒数。如果到了时间没有退出，程序将被杀死并以错误退出(不适用于`spawn()`或f`ork()`)

* `uid`：可以指定哪个用户ID来运行程序。

## `exec()/execFile()`

`execSync()`和`execFileSync()`是同步执行的，它们是`exec()/execFile()`的变体，而后者是异步执行的。

`exec()/execFile()`会立即返回一个`ChildProcess`对象，表示证字啊运行的子进程，并且接受一个错误在先的回调作为最后的参数。这个参数会在子进程退出时被调用。回调被调用时传入三个参数：如果发送了错误，第一个参数就是错误对象，否则第一个参数为null；第二个参数时子进程标准输出流的输出。第三个参数是子进程标准错误流的输出。

其返回的`ChildProcess`对象允许终止子进程，向子进程写入数据。后续我们会详细介绍

如果想要同时执行多个子进程，那么可以使用`exec()`的期约版。这个期约对象会解决为一个包含`stdout`和`stderr`对象的属性。例如：

~~~js
const childProcess = require("child_process");
const util = require("util");
const execP = util.promisify(childProcess.exec);

function parallelExec(commands){
    let promises = commands.map(command=> execP(command));
    return Promise.all(promises)
        .then(outputs => outputs.map(out => out.stdout))
}
~~~

## `spawn()`

`child_process.spawn()`函数允许在子进程允许期间流式访问子进程的输出。同时，也允许向子进程写入数据。这意味着可以动态于子进程进行交互。

`spawn()`默认不使用命令行解释器，因此必须像`execFile()`一样传入可执行文件和命令行参数来调用它，并且`spawn()`同样返回一个`ChildProcess`对象，但它不接收回调参数。

这个`spawn()`返回的`ChildProcess`是一个事件发送器(event emitter),可以监听子进程退出时发送的exit事件。这个`ChildProcess`还有三个流属性

* `stdout`和`stderr`是可读流，读取子程序的标准输出和标准错误流。
* `stdin`属性是可写流，可以将任何数据写入子进程的标准输入。

`ChildProcess`对象也定义了一个pid属性，用于指定子进程的进程ID。它还定义了`kill()`方法，用于终止子进程。

## `fork()`

`child_process.fork()`用于在一个Node子进程中运行一段JS代码。`fork()`和`spawn()`接收相同的参数，但第一个参数应该是JS代码文件的路径，而非可执行二进制文件的路径。

与`spawn()`一样，使用`fork()`创建的子进程可以通过标准输出和输入和父进程通信。此外`fork()`还提供了一种更简单的通信方式。

在使用`fork()`创建子进程后，可以使用它返回的`ChildProcess`对象的`send()`方法向子进程发送一个对象的副本。可以监听这个`ChildProcess`的`message`事件，从子进程中接收消息。

在子进程中运行的代码可以通过`proceess.send()`向父进程发送消息，也可以监听`process`的`message`事件，从父进程接收消息

例如，下面代码用`fork()`创建一个子进程，然后向子进程发送了一条消息并等待子进程回应：

~~~js
const childProcess = require("child_process");
let child = childProcess.fork("./child.js");

child.send({x:4,y:3});

child.on("message",message => {
    console.log(message.hypotenuse);
    child.disconnect(); //终止父进程和子进程的连接。这样两个进程都可以明确退出
})
~~~

然后`child.js`文件中子进程代码如下：

~~~js
process.on("message" ,message => {
    //计算，并把结果发给父进程
    process.send({hypotenuse:Math.hypot(message.x,message.y)})
})
~~~

启动子进程的开销巨大，如果子进程不能完成几个大数量级的运算，那么不值得使用`fork()`来开启子进程。

`send()`的第一个参数会被`JSON.stringify()`序列化，而在子进程中会被`JSON.parse()`反序列化。所以传参是对象中的值必须是JSON格式允许的。

`send()`有第二个特殊的参数，通过这个参数可以把`net`模块的`Socket`和`Server`对象转移给子进程。

# 工作线程

我们已经知道了Node的并发模型是单线程的，基于事件的。但是Node10开始支持真正的多线程编程，提供了类似于浏览器Web Workers API的一套API。

Node多线程和Web JS多线程一样，线程之间不共享内存，所以避免了多线程编程中的很多风险和困难。

没有共享内存，JS的工作线程只能通过消息传递来通信。

* 主线程可以调用代表工作线程的`Worker`对象的`postMessage()`方法向工作线程发送消息；通过`Worker`对象的`message`事件接收子线程消息
* 工作线程可以通过监听全局的`message`事件从父线程接收消息；可以通过自己的`postMessage()`方法向父线程发送消息

在Node中使用工作线程的原因有：

* 更好的利用计算机的多核心能力
* 将计算分给工作线程，可以更快的响应请求
* 将阻塞的同步操作转换为异步操作，可以通过工作线程将同步遗留代码转换为异步的

## Worker

工作线程的Node模块叫做`worker_threads`，我们使用`threads`来代指它：

~~~js
const threads = require("worker_threads");
~~~

这个模块定义了`Worker`类来表示工作线程，可以使用`threads.Worker()`构造函数创建新线程。下面代码给出示例，并且演示了一个技巧，可以把主线程代码和工作线程代码放在同一个文件中：

~~~js 
const threads = require("worker_threads");

if (threads.isMainThread){
    module.exports = function reticulateSplines(splines) {
        return new Promise((resolve, reject)=>{
            let reticulator = new threads.Worker(__filename);
            reticulator.postMessage(splines);
            reticulator.on("message",resolve);
            reticulator.on("error",reject)
        })
    }
} else {
    threads.parentPort.once("message",splines =>{
        for (let i = 0; i < splines.length; i++) {
            splines[i] +=1; //这里正常需要进行大量的计算
        }
        threads.parentPort.postMessage(splines); //将计算后的结果传递给主线程
    })
}
~~~

然后在其他模块导入这个模块并使用：

~~~js
const reticulateSplines = require("./demo.js");

reticulateSplines([1,2,3]).then(results =>{
    console.log(results)
})
~~~

`Worker()`构造函数的第一个参数是线程中运行的JS代码文件的路径。如果是相对路径，它是相对于`process.cwd()`的。

如果想让它相对于当前模块，可以使用这样的方式：

~~~js
path.resolve(__dirname,'xxx/demo.js');
~~~

`Worker()`接收一个可选的配置对象作为第二个参数，如果第二个参数传入了 `{eval:true}`作为第二个参数，那么`Worker()`的第一个参数将被作为要进行求值的JS代码字符串而不是一个文件名来解释：

~~~js
new threads.Worker(`
	const threads = require("worker_threads");
    threads.parentPort.postMessage('hello'); 
`,{eval: true}).on("message",console.log);
~~~

Node会将传递个`postMessage()`的对象复制一个副本，然后传递给目标线程，而不是让它在线程间共享。这里副本的制作不是通过JSON的序列化、反序列化，而是通过结构化克隆算法的技术。这个技术Node 和Web JS都有用到。

结构化克隆算法可以序列化多种JS类型，包括Map、Set、Date和RegExp对象以及定型数组。而且，它还能处理包含循环引用的数据结构。不过它不能序列化函数和类。也不能复制宿主环境定义的类型，如`process`。

## 工作线程的执行环境

Node工作线程的环境基本上和主线程一样，但是有有一下区别：

* `thread.isMainThread`在主线程中是true，在任何工作线程都是false

* 在工作线程中，可以使用`threads.parentPort.postMessage()`向父线程发送消息，使用`threads.parentPort.on`接收父线程的消息。而在主线程中`threads.parentPort`始终为null

* 在工作线程中，`thread.workerData`被设置为`Worker()`构造函数第二个参数workerData属性的一个副本。在主线程中，这个属性为null。可以通过它向工作线程传递一个初始数据。

* 默认情况下，`process.env`在工作线程中式父线程的`process.env`的一个副本。但是父线程可以通过设置`Worker()`构造函数的options对象的env属性指定一组自定义的环境变量

  在特殊的情况下，也可以将options对象的env属性设置为`threads.SHARE_ENV`，这样两个线程就共享一组环境变量，因此一个线程中的修改对另一个线程就是可见的。

* 默认情况下，工作线程中的`process.stdin`永远不会有可读数据。可以通过将`Worker()`构造函数的options对象的stdin属性设置为true改变这个默认设置。此时，`Worker`对象的stdin属性就是一个可写的流。父线程写入`worker.stdin`的数据会传入工作线程的`process.stdin`中

* 默认情况下,工作线程的`process.stdout`和`process.stderr`会被重定向到父线程中对应的流中。通过将`Worker()`构造函数的options对象的stdout和stderr属性设置为true。可以将工作线程中的标准输出流和错误流定向到`worker.stdout`和`worker.stderr`中。

* 工作线程调用`process.exit()`只会退出当前线程。

* 工作线程不能改变它们所属进程的共享状态。在工作线程中调用`process.chdir()`和`process.setuid()`等函数会抛出异常

## MessageChannel

和Web JS中的`Worker.postMessage()`一样，Node中的`postMessage()`同样接收可选的第二个数组参数，这个数组的元素不是被复制到信道的另一端，而是被转移到信道的另一端。但不一样的是这个可选的第二个数组元素的类型：

* 在Web中，必须是 `ArrayBuffer | MessagePort | ImageBitmap`
* 而在Node中，必须是`ArrayBuffer | MessagePort | FileHandle | X509Certificate | Blob`

所以Node中的`postMessage()`同样可以转移定型数组和`MessagePort`

具体代码示例请参照之前Web JS中的示例。

## 线程间共享定型数组

