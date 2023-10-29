# 引用类型

## 数组

和java不同JS的数组成员可以是不同的数据类型

创建数组语法：

~~~js
var array_name =[item1,item2,...];
var array_name = new Array(item1,...)
~~~

访问数组：

通过下标取值，赋值，添加元素

### 属性

* length 返回数组中元素的数量

### 静态方法

* `isArray(object)`判断对象是否为数组

### 非静态方法

* `toString()`返回数组元素用`,`拼接的字符串

* `join([separator])`用指定的分隔符将数组元素拼接成字符串，

    如果没有指定separator，将默认用`,`拼接

* `pop()`弹栈，删除数组的最后一个元素,并返回该元素
* `push(object)`进栈，在数组末尾添加一个新元素，并返回新数组长度

* `shift()`弹栈，删除数组第一个元素，并返回该元素值
* `unshift(object)`进栈，在数组首位添加一个新元素，返回新数组长度

* `delete`关键字可以删除数组的指定位置元素，将其变为`undefined`
* `concat(array1,array2,...)`拼接数组
* `sort()`以字典顺序对数组排序，可指定排序函数
* `reverse()`反转数组
* `forEach(function(item[,index,arr]))`遍历数组，对于每个元素，执行一次参数中的函数

* `map(function(item[,index,arr]))`为每个数组元素调用函数的结果创建新数组。返回新数组，不改变原数组
* `filter(function(item[,index,arr]))`，返回一个新数组，由通过测试的数组元素组成，测试由指定函数提供，对原数组的每个元素，执行测试函数，返回true时通过测试
* `indexOf(item)`，返回指定值在数组中首次出现的位置下标，如果没有返回`-1`
* `lastIndexOf(item)`,返回指定值在数组中最后一次出现的位置下标，如果没有返回`-1`

## 函数

定义：

~~~js
function 函数名(形参){
    //函数体
    return ;
}
~~~

调用与引用：

~~~js
function demo(){
    alert("deom")
    return 0;
}
//调用
let a = demo();
//引用
let b = demo;
~~~

### 函数参数

* 如果调用函数时实参少于形参，则丢失的值会被设置为undefined

* JavaScript 函数有一个名为 arguments 对象的内置对象。（array-like）

    该对象将传入的所有实参保存在对象中



### call()、 apply()

TODO

### 闭包

TODO





## 对象

所有 JavaScript 值，除了原始值，都是对象。

JavaScript 对象是无序属性的集合

定义格式

~~~js
var objectName = {
    propertyName = value,
    functionName = function(){},
    ....
}
~~~



* this 在对象中函数的定义中，`this`指向该对象的引用

* 访问对象属性 （获取，添加，修改对象属性）：
    * `objectName.propertyName`
    * `objectName["propertyName"]`
    * `objectName[propertyName]`
* 删除对象属性：`delete`关键字

* 访问对象方法：

    `objectName.methodName()`

* 显示对象
    * `Object.values()`
    * `JSON.stringify()` 不会转化方法

### 对象访问器

通过get关键字，set关键字设置getter和setter函数

直接通过`.访问器名`调用，不需`()`

### 对象构造器

实例：

~~~js
function Person(first, last, age, eye) {
    this.firstName = first;
    this.lastName = last;
    this.age = age;
    this.eyeColor = eye;
    this.name = function() {return this.firstName + " " + this.lastName;};
    this.changeName = function (name) {
        this.lastName = name;
    };
}
~~~

JS用于原始对象的构造器

~~~js
var x1 = new Object();    // 一个新的 Object 对象
var x2 = new String();    // 一个新的 String 对象
var x3 = new Number();    // 一个新的 Number 对象
var x4 = new Boolean();   // 一个新的 Boolean 对象
var x5 = new Array();     // 一个新的 Array 对象
var x6 = new RegExp();    // 一个新的 RegExp 对象
var x7 = new Function();  // 一个新的 Function 对象
var x8 = new Date();      // 一个新的 Date 对象
~~~



### 对象原型

所有 JavaScript 对象都从原型继承属性和方法。

日期对象继承自 Date.prototype。数组对象继承自 Array.prototype。Person 对象继承自 Person.prototype。

Object.prototype 位于原型继承链的顶端：

日期对象、数组对象和 Person 对象都继承自 Object.prototype。

JavaScript prototype 属性允许您为对象构造器添加新属性，新方法：

~~~js
Person.prototype.nationality = "English";
Person.prototype.name = function() {
    return this.firstName + " " + this.lastName;
};
~~~



### 全局对象

在 JavaScript 中，始终存在一种默认的全局对象。

在 HTML 中，默认全局对象是 HTML 页面本身，所有上面的函数“属于”HTML 页面。

在浏览器中，这个页面对象就是浏览器窗口。上面的函数自动成为一个窗口函数。

myFunction() 和 window.myFunction() 是同一个函数：

也就说在全局作用域中创建的函数，默认属于全局对象

# JS特性

## Hoisting

Hoisting 是 JavaScript 将所有声明提升到当前作用域顶部的默认行为（提升到当前脚本或当前函数的顶部）。

* 只有声明被提升，初始化不会
* let,const关键字声明的变量不会被提示
* 函数也会被提升

## this关键字

JS中this指的是它所属的对象

它拥有不同的值，具体取决于它的使用位置：

- 在方法中，`this` 指的是所有者对象。
- 单独的情况下，`this` 指的是全局对象。
- 在函数中，`this` 指的是全局对象。
- 在函数中，严格模式下，`this` 是 undefined。
- 在事件中，`this` 指的是接收事件的元素。



## 类

JS中类是对象的模板

语法

~~~js
class ClassName{
    constructor(){}
    method(){}
    
}
~~~

构造方法是一种特殊的方法：

- 它必须拥有确切名称的“构造函数”
- 创建新对象时自动执行
- 用于初始化对象属性
- 如果未定义构造函数方法，JavaScript 会添加空的构造函数方法。

### 继承

和python类似

super调用父类构造器



### 静态

和java类似

用static声明的变量，函数，只能由类调用，对象实例无法调用



# DOM

## DOM简介

DOM ：Document Object Model

当页面被加载时，浏览器会创建DOM

JS通过访问DOM对象，能动态的创建、改变HTML元素

### 什么是 DOM

DOM 是一项 W3C (World Wide Web Consortium) 标准。

DOM 定义了访问文档的标准：

> “W3C 文档对象模型（DOM）是中立于平台和语言的接口，它允许程序和脚本动态地访问、更新文档的内容、结构和样式。”

W3C DOM 标准被分为 3 个不同的部分：

- Core DOM - 所有文档类型的标准模型
- XML DOM - XML 文档的标准模型
- HTML DOM - HTML 文档的标准模型

### 什么是 HTML DOM？

HTML DOM 是 HTML 的标准*对象*模型和*编程接口*。它定义了：

- 作为*对象*的 HTML 元素
- 所有 HTML 元素的*属性*
- 访问所有 HTML 元素的*方法*
- 所有 HTML 元素的*事件*

JS通过访问document对象访问和操作 HTML 的实例。



## DOM对象

Document 对象使我们可以从脚本中对 HTML 页面中的所有元素进行访问。

**提示：**Document 对象是 Window 对象的一部分，可通过 window.document 属性对其进行访问。

### 属性

* `body` 访问body元素
* `cookie`返回或设置当前文档有关的所有cookie
* `domain`返回当前文档的域名
* `lastModified`返回文档的最后修改日期和时间
* `referrer`返回载入当前文档的超链接的文档的URL
* `title`返回文档标题
* `URL`返回文档URL



### 方法

* `getElementById()`返回对拥有指定 id 的第一个对象的引用。
* `getElementsByName()`返回带有指定name的对象集合。
* `getElementsByTagName()`返回带有指定标签名的对象集合。

* `getElementsByClassName()` 返回带有指定class的对象集合
* `querySelectorAll() `查找指定CSS选择器的所有HTML元素对象的引用

这些方法返回HTML元素的对应对象的引用也就是Element对象



## Element对象

在 HTML DOM 中，Element 对象表示 HTML 元素。

dom提供了属性方法以获取，改变HTML元素的内容

下面提供一些常用的属性方法

### 属性

* `id`
* `innerHTML`

### 方法

* `setAttribute(attributename,attributevalue)`

* `removeAttribute(attributename)`



## Event

事件能够支持页面实现当用户进行某些操作时，触发对应的JS函数



### 向HTML元素分配事件

以click事件为例：

* 通过元素的事件属性

    ~~~html
    <button onclick="demo()">试一试</button>
    ~~~

* 通过element对象：

    ~~~js
    document.getElementById("btn").onclick = method
    ~~~

    

### 常用事件

* `click`：当用户点击指定元素时，发生此事件
* `focus`：当指定元素得焦时，发生此事件
* `blur`：当指定元素失焦时，发生此事件
* `input`：当\<input> 或 \<textarea> 元素的值发生改变时，会发生此事件。
* `submit`：当表单提交时，发生此事件

* `load`在对象已加载时，发生此事件。

## HTMLCollection

是HTML元素的类似数组的列表

由诸如`getElementsByTagName()`之类的方法会返回HTMLCollection

###  属性/方法

* `length `返回HTMLCollection中的元素数
* `item(index)`获取指定索引的HTML元素对象
* `namedItem(name)`获取指定ID或名称 的HTML元素对象 

# BOM

允许JS与浏览器窗口对话

## Window

加载到页面时浏览器会生成window对象，供JS使用

所有全局JS对象，函数，变量自动成为window对象的成员

全局变量时window对象的属性

全局函数是window对象的方法

document也是window对象的属性

调用当前窗口的方法和属性时，`window.`可以省略

`window.document <=> document`

### 属性

* `closed`返回窗口是否被关闭
* `name`设置或返回窗口的名称

* `opener`返回对创建该窗口的窗口的引用
* `parent`返回父窗口
* `frames[]`返回窗口中所有命名的框架。
* `document`对Document对象的只读引用
* `history`对History对象的只读引用
* `location`用于窗口或框架的 Location 对象
* `Navigator`对 Navigator 对象的只读引用
* `Screen`对 Screen 对象的只读引用

### 方法

#### open()

`open([URL[,name[,features[,replace]]]])`打开一个新的浏览器窗口或查找一个已命名的窗口，并返回该窗口的window对象的引用

* URL：字符串，声明了要在新窗口中显示的文档的 URL。默认为空字符串

* name：字符串，声明了新窗口的名称

    如果该参数指定了一个已经存在的窗口，那么 open() 方法就不再创建一个新窗口，而只是返回对指定窗口的引用。此时features 将被忽略。

* features：字符串，声明了新窗口要显示的标准浏览器的特征

    字符串中的窗口特征用`,`隔开下面列举了窗口特征：

    | 窗口特征                  | 描述                                                         |
    | ------------------------- | ------------------------------------------------------------ |
    | channelmode=yes\|no\|1\|0 | 是否使用剧院模式显示窗口。默认为 no。                        |
    | directories=yes\|no\|1\|0 | 是否添加目录按钮。默认为 yes。                               |
    | fullscreen=yes\|no\|1\|0  | 是否使用全屏模式显示浏览器。默认是 no。处于全屏模式的窗口必须同时处于剧院模式。 |
    | height=pixels             | 窗口文档显示区的高度。以像素计。                             |
    | left=pixels               | 窗口的 x 坐标。以像素计。                                    |
    | location=yes\|no\|1\|0    | 是否显示地址字段。默认是 yes。                               |
    | menubar=yes\|no\|1\|0     | 是否显示菜单栏。默认是 yes。                                 |
    | resizable=yes\|no\|1\|0   | 窗口是否可调节尺寸。默认是 yes。                             |
    | scrollbars=yes\|no\|1\|0  | 是否显示滚动条。默认是 yes。                                 |
    | status=yes\|no\|1\|0      | 是否添加状态栏。默认是 yes。                                 |
    | titlebar=yes\|no\|1\|0    | 是否显示标题栏。默认是 yes。                                 |
    | toolbar=yes\|no\|1\|0     | 是否显示浏览器的工具栏。默认是 yes。                         |
    | top=pixels                | 窗口的 y 坐标。                                              |
    | width=pixels              | 窗口的文档显示区的宽度。以像素计。                           |

* `replace`：布尔值，规定了载到窗口的 URL 是在窗口的浏览历史中创建一个新条目，还是替换浏览历史中的当前条目。

#### 对话框

* `alert(message)`

    显示带有一条指定消息和一个 OK 按钮的警告框。

* `confirm(message)`

    显示一个带有指定消息和 OK 及取消按钮的对话框。

    如果用户点击确定按钮，则 confirm() 返回 true。如果点击取消按钮，则 confirm() 返回 false。

    是一个就绪方法，调用 confirm() 时，将暂停对 JavaScript 代码的执行，在用户作出响应之前，不会执行下一条语句。

    在用户点击确定按钮或取消按钮把对话框关闭之前，它将阻止用户对浏览器的所有输入

* `prompt(text,defaultText)`

    显示可提示用户进行输入的对话框

    text：提示信息

    defaultText：输入框默认文本

    如果用户单击提示框的取消按钮，则返回 null。如果用户单击确认按钮，则返回输入字段当前显示的文本。

    在用户点击确定按钮或取消按钮把对话框关闭之前，它将阻止用户对浏览器的所有输入。在调用 prompt() 时，将暂停对 JavaScript 代码的执行，在用户作出响应之前，不会执行下一条语句。

#### close()

`close()`

用于关闭浏览器方法

只有通过 JavaScript 代码打开的窗口才能够由 JavaScript 代码关闭。这阻止了恶意的脚本终止用户的浏览器。

## History

History对象包含用户在浏览器窗口中访问过的URL

## 属性

* `length`返回浏览器历史列表中的URL数量

## 方法

* `back()`加载history列表中当前一个URL
* `forward()`加载history列表中的下一个URL
* `go(number|URL)`加载history列表中某个具体页面
    * URL：history中的url
    * number：要访问的 URL 在 History 的 URL 列表中的相对位置。

## Location对象

Location对象包含相关当前URL的信息

### 属性

* `hash`	设置或返回从井号 (#) 开始的 URL（锚）。
* `host`	设置或返回主机名和当前 URL 的端口号。
* `hostname`	设置或返回当前 URL 的主机名。
* `href`	设置或返回完整的 URL。
* `pathname`	设置或返回当前 URL 的路径部分。
* `port`	设置或返回当前 URL 的端口号。
* `protocol`	设置或返回当前 URL 的协议。
* `search`	设置或返回从问号 (?) 开始的 URL（查询部分）

### 方法

* `assign(URL)`	加载新的文档。

* `reload()`	重新加载当前文档。

* `replace()`	用新的文档替换当前文档。

    replace不会在 History 对象中生成一个新的记录。当使用该方法时，新的 URL 将覆盖 History 对象中的当前记录。

## Navigator

Navigator 对象包含的属性描述了正在使用的浏览器。可以使用这些属性进行平台专用的配置。

虽然这个对象的名称显而易见的是 Netscape 的 Navigator 浏览器，但其他实现了 JavaScript 的浏览器也支持这个对象。

Navigator 对象的实例是唯一的，可以用 Window 对象的 navigator 属性来引用它

### 属性

* `appCodeName	`返回浏览器的代码名。
* `appMinorVersion	`返回浏览器的次级版本。
* `appName	`返回浏览器的名称。
* `appVersion	`返回浏览器的平台和版本信息。
* `browserLanguage	`返回当前浏览器的语言。
* `cookieEnabled	`返回指明浏览器中是否启用 cookie 的布尔值。
* `cpuClass	`返回浏览器系统的 CPU 等级。
* `onLine	`返回指明系统是否处于脱机模式的布尔值。
* `platform	`返回运行浏览器的操作系统平台。
* `systemLanguage	`返回 OS 使用的默认语言。
* `userAgent	`返回由客户机发送服务器的 user-agent 头部的值。
* `userLanguage`	返回 OS 的自然语言设置。

### 方法

* `javaEnabled()`	规定浏览器是否启用 Java。
* `taintEnabled()`	规定浏览器是否启用数据污点 (data tainting)。

## Screen

Screen 对象包含有关客户端显示屏幕的信息。

### 属性

* `availHeight	`返回显示屏幕的高度 (除 Windows 任务栏之外)。
* `availWidth	`返回显示屏幕的宽度 (除 Windows 任务栏之外)。
* `bufferDepth	`设置或返回调色板的比特深度。
* `colorDepth	`返回目标设备或缓冲器上的调色板的比特深度。
* `deviceXDPI	`返回显示屏幕的每英寸水平点数。
* `deviceYDPI	`返回显示屏幕的每英寸垂直点数。
* `fontSmoothingEnabled	`返回用户是否在显示控制面板中启用了字体平滑。
* `height	`返回显示屏幕的高度。
* `logicalXDPI	`返回显示屏幕每英寸的水平方向的常规点数。
* `logicalYDPI	`返回显示屏幕每英寸的垂直方向的常规点数。
* `pixelDepth	`返回显示屏幕的颜色分辨率（比特每像素）。
* `updateInterval	`设置或返回屏幕的刷新率。
* `width	`返回显示器屏幕的宽度。

## Console

Console 对象提供对浏览器调试控制台的访问。

### 方法

* log()	将消息输出到控制台。

* warn()	将警告消息输出到控制台。

* time()	启动计时器（可跟踪操作需要多长时间）。
* timeEnd()	停止以前由 console.time() 启动的计时器。



