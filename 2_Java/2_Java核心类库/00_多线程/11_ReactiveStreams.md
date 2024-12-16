# Reactive Streams规范

Reactive Streams是具有非阻塞背压的异步流的标准规范。

`java.util.concurrent.Flow`提供的接口在语义上实现了Reactive Streams规范。

在异步系统中处理数据(尤其是其数量未预先确定的“实时”数据)流需要特别小心。最突出的问题是需要控制资源的消耗，以防止快速的数据源让流的目的地过载。

为了在网络主机上或者单个计算机的多个cpu上并行计算数据源，异步是必要的。

Reactive Streams的主要目标是管理流数据在异步边界上的交换(将元素传递个另一个线程或者线程池)，同时确保数据接收方不会被迫缓存任意大量的数据。即不会发生数据堆积。也就是说，Reactive Streams中背压是必须的，以保证两个线程间的调节队列(缓存未消费数据)是有界的。如果背压的通信是同步的，那么异步进程的好处将被抵消。因此，必须注意Reactive Streams实现的所有方面都必须强制要求完全非阻塞和异步。

# Flow

`Flow`类提供了几个相互关联的接口和静态方法以用于建立流控制组件。其中：`Publisher`生产数据项，这些数据项会被一个或者多个`Subscriber`消费。生产的每个数据项由`Subscription`管理。

这些接口对应了`Reactive Streams`规范，并适用于分布式和并发的异步环境，这些组件的所有方法都没有返回值，这是一种单向消息传递的风格。

通信依赖于一种简单的流控制形式（通过 `Flow.Subscription.request()` 方法）,生产者可以依据消费者的处理能力来控制数据的推送速度，从而避免资源过度消耗或阻塞，实现更有效的资源管理。

## `Publisher`

`Flow.Publisher`是数据项和相关控制信息的生产者。除非数据项丢失或者遇到错误，每个当前的`Flow.Subscriber`会通过`onNext()`方法按顺序接受到其生产的数据项。

如果`Publisher`发生错误以至于无法发送数据项给`Subscriber`，那么`Susbcriber`会收到一个`onError`通知，并且之后不会再收到更多的消息。

另外，如果决定不再生产新的消息时，`Subscriber`会收到`onComplete`通知。

`Publisher`保证每个订阅的`Subscriber`的方法调用严格按照happen-before的顺序进行，这意味着如果事件A在事件B前发生，那么在订阅者的角度来看，对应的订阅方法也会按照这个顺序被调用。这种保证有助于维护数据一致性，特别是在处理并发和异步操作时。

不同的`Publisher`可以根据实际情况决定下面策略：

* 是否将数据项的丢弃(由于资源限制导致未能发布资源项的情况)视为不可恢复的错误
* 是否让订阅者收到在它们订阅前发布的数据项

下面是`Publisher`的定义：

~~~java
@FunctionalInterface
public static interface Publisher<T> {
    public void subscribe(Subscriber<? super T> subscriber);
}
~~~

`subscribe()`方法添加/绑定一个`Subscriber`，该方法需要保证：

* 如果已经添加过，或者由于违反订阅策略或者发生错误导致订阅失败，调用 `Subscriber`的`onError()`方法，并出传入一个`IllegalStateException`。通知订阅者订阅失败
* 否则，调用`Subscriber`的`onSubscribe()`方法，并传入一个新的`Subscription`。通知订阅者订阅成功

以上逻辑需要在`Publisher`的具体实现类在`subscribe()`中主动实现。

订阅完成后`Subscriber`可以调用`Subscription.request()`方法来开启接受数据项，调用`Subscription.cancel()`方法来取消订阅。

一个`Publisher`实现的示例：

该`Publisher`发布布尔值的数据项，且只能被订阅一次：

~~~java
class OneShotPublisher implements Flow.Publisher<Boolean> {
    private final ExecutorService executor = ForkJoinPool.commonPool(); // 使用ExecutorService实现消息发布的异步
    private boolean subscribed; // 是否首次订阅

    public synchronized void subscribe(Flow.Subscriber<? super Boolean> subscriber) {
        if (subscribed) // 如果不是首次订阅，订阅失败，调用onError方法通知订阅失败
            subscriber.onError(new IllegalStateException()); 
        else { //否则，调用onSubscribe方法通知订阅成功
            subscribed = true;
            subscriber.onSubscribe(new OneShotSubscription(subscriber, executor)); //OneShotSubscription是Subscription的一个实现
        }
    }
}
~~~

## `Subscriber`

`Subscriber`用于订阅并接受消息。此接口中的方法对于每个 `Flow.Subscription` 都会严格按照顺序依次调用。其定义如下：

~~~java
public static interface Subscriber<T> {
    public void onSubscribe(Subscription subscription);	
    public void onError(Throwable throwable);
    public void onNext(T item);
    public void onComplete();
}
~~~

* `onSubscribe()`：订阅成功回调。对给定的`Subscription`,在调用在其他所有`Subscriber`方法前，会先调用该方法。

  如果该方法抛出异常，将无法确保后续的行为。可能会导致订阅构建失败或者被取消

  通常，此方法的实现会调用 `Subscription.request()`方法来启用接收数据项

* `onError()`:调阅失败回调。当`Publisher`或者`Subscription`遇到不可恢复的错误时，该调用该方法。此后`Subscription`将不再调用当前`Subscriber`的任意其他方法。

* `onNext()`:消息回调。在`Subscription`订阅的下一个数据项到达时该方法将被调用。如果该方法抛出异常，行为将不可预测，也可能导致订阅被取消(`Subscription.cancel()`)

* `onComplete()`:订阅结束回调。当已知`Subscription`不会再调用当前`Subscriber`的其他方法，且该订阅没有因异常而终止时，该方法将被调用。

一个`Subscriber`的示例：

~~~java
class SampleSubscriber<T> implements Flow.Subscriber<T> {
    final Consumer<? super T> consumer;
    Flow.Subscription subscription;
    final long bufferSize;
    long count; //表示当前

    SampleSubscriber(long bufferSize, Consumer<? super T> consumer) {
        this.bufferSize = bufferSize;
        this.consumer = consumer; //用于消费消息/数据项
    }
	//当订阅成功后，立即调用Subscription.request()方法请求数据项
    public void onSubscribe(Flow.Subscription subscription) {
        long initialRequestSize = bufferSize;
        count = bufferSize - bufferSize / 2; 
        (this.subscription = subscription).request(initialRequestSize);
    }

    public void onNext(T item) {
        // 当缓存的数据项数目小于缓存容量的一半时，将在此发起请求
        if (--count <= 0) subscription.request(count = bufferSize - bufferSize / 2);
        consumer.accept(item);
    }

    public void onError(Throwable ex) {
        ex.printStackTrace();
    }

    public void onComplete() {
    }
}
~~~

`onNext()`只有在被请求时才会被调用(即在`Subscription.request()`中)。但可以一次同时请求多个数据项。多数`Subscriber`的实现会按照下面的形式安排请求：缓冲区大小为1时逐个处理数据项。更大的缓冲区则允许更高效的重叠处理并减少通信次数。

由于针对给定 `Flow.Subscription `的 `Subscriber` 方法调用是严格按顺序进行的，所以在方法内部不需要考虑并发的问题。

## `Subscription`

链接`Publisher`和`Subscriber`的消息控制接口。

`Subscriber`只有在发起请求时才会收到数据项，即`Subscriber.onNext()`方法只会在`Subscription.request()`方法调用时被调用。但可以随时取消订阅。

`Subscription`中的方法只仅供其`Subscriber`调用。

`Subscription`定义为：

~~~java
public static interface Subscription {
    public void request(long n);
    public void cancel();
}
~~~

* `request()`将给定数量`n`的数据项添加到订阅中。
  * 如果`n`小于或等于0，则`Subscriber`将收到`IllegalArgumentException`异常在`onError`回调中。
  * 否则，`Subscriber`将收到最多`n`次的`onNext()`调用
* `cancel()`通知`Subscriber`取消接受消息。该方法不是并不完全确保效果，在调用此方法后，`Subscriber`可能仍会接收到一些消息。被取消的订阅不一定会接收到 `onComplete` 或 `onError` 信号。

示例`OneShotSubscription`被请求是只会发送一个数据项，不论请求的量`n`为多少。并且请求只会有一次，首次请求后就会调用`Subscriber.onComplete()`通知订阅完成。

~~~java
class OneShotSubscription implements Flow.Subscription {
    private final Flow.Subscriber<? super Boolean> subscriber;
    private final ExecutorService executor;
    private Future<?> future; // to allow cancellation
    private boolean completed;

    OneShotSubscription(Flow.Subscriber<? super Boolean> subscriber, ExecutorService executor) {
        this.subscriber = subscriber;
        this.executor = executor;
    }

    public synchronized void request(long n) {
        if (!completed) {
            completed = true;
            if (n <= 0) {
                IllegalArgumentException ex = new IllegalArgumentException();
                executor.execute(() -> subscriber.onError(ex));
            } else {
                future = executor.submit(() -> {
                    subscriber.onNext(Boolean.TRUE);
                    subscriber.onComplete();
                });
            }
        }
    }

    public synchronized void cancel() {
        completed = true;
        if (future != null) future.cancel(false);
    }
}
~~~

## `Processor`

定义如下：

~~~java
public static interface Processor<T,R> extends Subscriber<T>, Publisher<R> {
}
~~~

表示即是消息的发布者也是消息的订阅者。用于在消费消息后产生新的消息给另外的订阅者。

## 使用示例

在前面介绍中我们使用了`SampleSubscriber`、`OneShotSubscription`、`SampleSubscriber`作为示例，现在我们来使用它启动一次订阅：

~~~java
OneShotPublisher oneShotPublisher = new OneShotPublisher();
SampleSubscriber<Boolean> sampleSubscriber = new SampleSubscriber<>(10, System.out::println);
oneShotPublisher.subscribe(sampleSubscriber);
~~~

# Reactor

Reactor是JVM的完全的非阻塞反应式编程基础，以背压的方式实现高效的数据需求管理。

它直接整合了Java8的函数式编程API，特别是`CompletableFuture`, `Stream`和`Duration`

Reactor提供可组合的异步序列API:`Flue`（处理多个数据项）和`Momo`（处理单个数据项）并且全面实现了Reactive Streams规范。

Reactor还支持通过`reactor-netty`项目来实现进程间通信。Reactor Netty 适用于微服务架构，为 HTTP（包括 Websockets）、TCP 和 UDP 提供背压就绪网络支持。全面支持反应式编码和解码。

## 获取Reactor

Reactor使用BOM式版本管理，将整个Reactor分为不同的组件，使用BOM统合不同组件的版本管理。

可以在Maven中引入其BOM，然后声明相关组件的依赖，此时组件就不需要声明版本，版本由BOM管理：

~~~xml
<dependencyManagement> 
    <dependencies>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-bom</artifactId>
            <version>2023.0.8</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
~~~

然后声明相关组件：

~~~xml
<dependencies>
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-core</artifactId> 
    </dependency>
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-test</artifactId> 
        <scope>test</scope>
    </dependency>
</dependencies>
~~~

## 反应式编程

Ractor是反应式编程范式的实现，所谓反应式编程范式即：

 反应式编程是一种异步编程范式，其关注数据流以及数据变化的传播。通过这种方式可以更轻松的表达静态或者动态的数据流

作为反应式编程的先行者，微软首先在.net生态中创造了Reactive Extensions (Rx)库。然后RxJava在JVM中实现了反应式编程。

后来`Reactive Streams`提供了Java反应式编程的规范，该规范为JVM上的反应式编程库定义了一系列接口和交互规则。这些接口被整合到了Java9的`Flow`类下。

在面向对象语言中，反应式编程通常被解释为观察者设计模式的一个拓展。

可以将反应式编程和迭代器设计模式作为对比。迭代器模式是`Iterable`和`Iterator`（即迭代器和可迭代对象）之间的交互。

反应式编程和迭代器设计模式的一个主要区别就是：

* 迭代器设计模式是基于`pull`的，即迭代器会主动获取可迭代对象的数据项
* 反应式编程是基于`push`，即`Subscriber`首先会发起数据项的获取请求，然后`Publisher`才会推送数据项给订阅者

迭代器模式是典型的命令式编程范式，尽管访问值的方法完全由可迭代对象负责，但是是由开发者决定合适访问序列中的下一个值(通过`next()`方法)。

在Reactive Streams中，与迭代器模式中的`Iterable`和`Iterator`实体对应的是`Publisher`和`Subscriber`。不同之处在于，是由`Publisher`来通知`Subscriber`新值何实到达，这种`push`的推送机制正在响应式编程的关键。

对推送的值的操作是表达为声明式的：程序员只需要表达对值的计算逻辑。而不需要描述对数据的精确控制流程。(即不需要关注这些数据何时产生，何时消费)

除了推送值，`Reactive Stream`也提供了异常处理和通知推送完成。

一个`Publisher`通过`onNext()`方法推送数据给其`Susbcriber`，通过`onError()`方法来通知一个错误，通过`onComplete()`方法通知通信结束。`onError`和`onComplete`都会终结操作。

这种方式提供了非常灵活的通知策略。可以支持空值、单个值、多个值的情况。

### 阻塞的缺点

现代应用可能会同时服务大量用户，尽管当前硬件性能仍然在进步，但是软件的性能仍然是关键问题。

通常有两种方式提升程序性能：

* 并行使用更多的线程和更大的硬件资源
* 在目前的条件下寻找更高效的利用硬件资源的方式

同时Java使用阻塞的代码。这种实践很好但是它有性能瓶颈。因此引入了多线程来并行运行相同的代码块。但这同时造成了资源的竞争和线程并发问题。

更有甚者，阻塞会造成资源的浪费：当程序涉及一些延迟操作时(例如数据库请求和网络IO)资源会被浪费，因为此时线程会闲置，等待资源。

所以并不是万能的解决方案。需要进一步全面利用硬件资源，

### 异步方案

异步时第二种更简单，更高效的方法来解决等待资源造成的浪费问题。

通过异步非阻塞代码，可以让代码的执行切换到另一个使用相同的底层资源的活动任务上。当异步执行结束后，会返回结果给当前进程。

Java提供两种异步编程模型：

* `Callbacks`：没有具有额外callback参数的异步方法。一般回调callback为lambda表达式或者匿名类。回调会在何时的时候被调用。一个典型的例子时Swing的`EventListener`体系
* `Future`:会立即返回一个`Future<T>`的异步方法。匿名的进程会计算`T`的值，`Future`对象包装了对异步计算结果的访问。当异步计算还为结束时，`Future`代表的结果不可用。可以轮询知道计算完成，结果可用。

以上方法并不适用于所有用例，并且都有各自的限制。

#### Callback

多个`Callback`回调很难组合在一起，多个回调嵌套在一起会让代码难以阅读和调试。

通过一个示例就可以直观的看到当回调多起来后会多糟糕：

下面示例向用户展示其5个收藏夹，如果没有收藏夹，则展示推荐：

~~~java
//获取收藏夹
userService.getFavorites(userId, new Callback<List<String>>() { 
  public void onSuccess(List<String> list) { 
    //如果没有收藏夹则展示推荐
    if (list.isEmpty()) { 
      suggestionService.getSuggestions(new Callback<List<Favorite>>() {
        public void onSuccess(List<Favorite> list) { 
          UiUtils.submitOnUiThread(() -> { 
            list.stream()
                .limit(5)
                .forEach(uiList::show); 
            });
        }

        public void onError(Throwable error) { 
          UiUtils.errorPopup(error);
        }
      });
    } else {
      list.stream() 
          .limit(5)
          .forEach(favId -> favoriteService.getDetails(favId, 
            new Callback<Favorite>() {
              public void onSuccess(Favorite details) {
                UiUtils.submitOnUiThread(() -> uiList.show(details));
              }

              public void onError(Throwable error) {
                UiUtils.errorPopup(error);
              }
            }
          ));
    }
  }

  public void onError(Throwable error) {
    UiUtils.errorPopup(error);
  }
});
~~~

以上代码繁琐且难以理解且有可复用的部分。

相同的功能如果用`Reactor`来实现将十分清晰简单：

~~~java
userService.getFavorites(userId) //获取一个生产收藏夹id的Flow
           .flatMap(favoriteService::getDetails) //异步地根据id获取收藏夹详情
           .switchIfEmpty(suggestionService.getSuggestions()) //如果收藏夹为空，则使用推荐 
           .take(5) 
           .publishOn(UiUtils.uiThreadScheduler()) //在Ui线程中处理每条数据
           .subscribe(uiList::show, UiUtils::errorPopup); //订阅数据
~~~

#### Future

使用`Future`会比Callback回调好点，但不考虑`CompletableFuture`时，直接组合使用`Future`时仍然表现不佳。将多个`Future`组合使用是可行的但并不简单，并且`Future`还有其他问题：

* 通过调用`Future.get()`方法，很容易就在其他地方进入阻塞。
* 不支持惰性计算
* 缺乏对多值和高级异常处理的支持

考虑如下示例：根据一个ID列表，异步地获取对应的名称和统计数据，并将它们组合。下例使用`CompletableFuture`来实现：

~~~java
CompletableFuture<List<String>> ids = ifhIds(); 

CompletableFuture<List<String>> result = ids.thenComposeAsync(l -> { 
	Stream<CompletableFuture<String>> zip =
			l.stream().map(i -> { 
				CompletableFuture<String> nameTask = ifhName(i); 
				CompletableFuture<Integer> statTask = ifhStat(i); 

				return nameTask.thenCombineAsync(statTask, (name, stat) -> "Name " + name + " has stats " + stat); 
			});
	List<CompletableFuture<String>> combinationList = zip.collect(Collectors.toList()); 
	CompletableFuture<String>[] combinationArray = combinationList.toArray(new CompletableFuture[combinationList.size()]);

	CompletableFuture<Void> allDone = CompletableFuture.allOf(combinationArray); 
	return allDone.thenApply(v -> combinationList.stream()
			.map(CompletableFuture::join) 
			.collect(Collectors.toList()));
});

List<String> results = result.join(); 
assertThat(results).contains(
		"Name NameJoe has stats 103",
		"Name NameBart has stats 104",
		"Name NameHenry has stats 105",
		"Name NameNicole has stats 106",
		"Name NameABSLAJNFOAJNFOANFANSF has stats 121");
~~~

`Reactor `有很多开箱即用的组合运算符，所以可以用它简化上面的实现:

~~~java
Flux<String> ids = ifhrIds(); 

Flux<String> combinations =
		ids.flatMap(id -> { 
			Mono<String> nameTask = ifhrName(id); 
			Mono<Integer> statTask = ifhrStat(id); 

			return nameTask.zipWith(statTask, 
					(name, stat) -> "Name " + name + " has stats " + stat);
		});

Mono<List<String>> result = combinations.collectList(); 

List<String> results = result.block(); 
assertThat(results).containsExactly( 
		"Name NameJoe has stats 103",
		"Name NameBart has stats 104",
		"Name NameHenry has stats 105",
		"Name NameNicole has stats 106",
		"Name NameABSLAJNFOAJNFOANFANSF has stats 121"
);
~~~

## 从命令式到反应式编程



















