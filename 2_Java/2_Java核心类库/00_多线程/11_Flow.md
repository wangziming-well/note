# Reactive Streams规范

Reactive Streams是具有非阻塞背压的异步流的标准规范。

`java.util.concurrent.Flow`提供的接口在语义上实现了Reactive Streams规范。

在异步系统中处理数据(尤其是其数量未预先确定的“实时”数据)流需要特别小心。最突出的问题是需要控制资源的消耗，以防止快速的数据源让流的目的地过载。

为了在网络主机上或者单个计算机的多个cpu上并行计算数据源，异步是必要的。

Reactive Streams的主要目标是管理流数据在异步边界上的交换(将元素传递个另一个线程或者线程池)，同时确保数据接收方不会被迫缓存任意大量的数据。即不会发生数据堆积。也就是说，Reactive Streams中背压是必须的，以保证两个线程间的调节队列(缓存未消费数据)是有界的。如果背压的通信是同步的，那么异步进程的好处将被抵消。因此，必须注意Reactive Streams实现的所有方面都必须强制要求完全非阻塞和异步。

# Flow

`Flow`类提供了几个相互关联的接口和静态方法以用于建立流控制组件。其中`Publisher`生产数据项，这些数据项会被一个或者多个`Subscriber`消费。生产的每个数据项由`Subscription`管理。

这些接口对应了`Reactive Streams`规范，并适用于分布式和并发的异步环境，这些组件的所有方法都没有返回值，这是一种单向消息传递的风格。

通信依赖于一种简单的流控制形式（通过 `Flow.Subscription.request()` 方法）,生产者可以依据消费者的处理能力来控制数据的推送速度，从而避免资源过度消耗或阻塞，实现更有效的资源管理。

## 示例

一个`Flow.Publisher`通常会定义其自己的`Flow.Subscription`实现。并在方法`Publisher.subscribe()`中构建该实现并将其发给调用该方法的`Flow.Subscriber`（方法入参）它通常通过`Executor`异步地向`Subscriber`发布数据项。

例如，下面是一个简单的示例`Pulbisher`，它只向一个单独的`Subscriber`发送一个单独的`true`数据项.因为`Subscriber`只接受一个数据项，所以这个类就没有大部分实现都需要的使用缓存和顺序控制。

~~~java
class OneShotPublisher implements Flow.Publisher<Boolean> {
    private final ExecutorService executor = ForkJoinPool.commonPool(); // daemon-based
    private boolean subscribed; // true after first subscribe

    public synchronized void subscribe(Flow.Subscriber<? super Boolean> subscriber) {
        if (subscribed)
            subscriber.onError(new IllegalStateException()); // only one allowed
        else {
            subscribed = true;
            subscriber.onSubscribe(new OneShotSubscription(subscriber, executor));
        }
    }

    static class OneShotSubscription implements Flow.Subscription {
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
}
~~~

`Flow.Subscriber`负责数据项的请求和处理。`Flow. Subscriber. onNext()`调用中的数据项只有在被请求时才会被发布。但可以一次同时请求多个数据项。多数`Subscriber`的实现会按照下面的形式安排请求：缓冲区大小为1时逐个处理数据项。更大的缓冲区则允许更高效的重叠处理并减少通信次数。

由于针对给定 `Flow.Subscription `的 `Subscriber` 方法调用是严格按顺序进行的，所以在方法内部不需要考虑并发的问题。



