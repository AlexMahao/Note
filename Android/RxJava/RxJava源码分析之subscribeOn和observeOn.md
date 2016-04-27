## RxJava源码分析之subscribeOn和observeOn


RxJava的特色就是可以改变他的任务线程，可以很优雅的在子线程和主线程中切换，而切换用到的两个主要方法是subscribeOn()和observeOn().

> 备注：因本人水平有限，以下分析只代表本人所见，如有不当，请见谅并指出。

### subscribeOn()和observeOn()的区别
- subscribeOn()主要改变的是订阅的线程。即call()执行的线程
- ObserveOn()主要改变的是发送的线程。即onNext()执行的线程。

### subscribeOn()分析

在分析之前，首先要明确一点，不然会产生思路混乱。

当我们调用了Observable.subscribe()方法，发生了什么？？？？

总结性的说：创建Subscriber对象，调用OnSubscribe对象的call方法。注意，Subscriber什么都没做，Subscriber什么都没做，Subscriber什么都没做。重要的事情说三遍，下面开始分析。

首先，我们写一个例子：
```java 
 Observable
                .create(new Observable.OnSubscribe<String>() {
                    @Override
                    public void call(Subscriber<? super String> subscriber) {
                        subscriber.onNext("a");
                        subscriber.onNext("b");

                        subscriber.onCompleted();
                    }
                })
                .subscribeOn(Schedulers.io())

                .subscribe(new Observer<String>() {
                    @Override
                    public void onCompleted() {
                       
                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(String integer) {
                        System.out.println(integer);
                    }
                });

```

这个例子很简单，但在intellij 中运行，不打印任何东西，但在ecplise中确没问题。打印如下：
```java 
a
b
```

这个问题我也搞不清为什么，如果有知道的欢迎留言。

老套路，我们看一下subscribeOn()中，都干了什么
```java
  public final Observable<T> subscribeOn(Scheduler scheduler) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return create(new OperatorSubscribeOn<T>(this, scheduler));
    }
```

很明显，会走if之外的方法。

在这里我们可以看到，我们又创建了一个Observable对象，但创建时传入的参数为`OperatorSubscribeOn(this,scheduler)`，我们看一下此对象以及其对应的构造方法

```java 
public final class OperatorSubscribeOn<T> implements OnSubscribe<T> {

    final Scheduler scheduler;
    final Observable<T> source;

    public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler) {
        this.scheduler = scheduler;
        this.source = source;
    }

    @Override
    public void call(final Subscriber<? super T> subscriber) {
		//......暂时省略
    }
}
```

可以看到，`OperatorSubscribeOn`实现`Onsubscribe`，并且由其构造方法可知，他保存了上一个`Observable`对象，并保存了`Scheduler`对象。

在这里，必须做一个总结。

把Observable.create()创建的称之为Observable_1,OnSubscribe_1。把subscribeOn()创建的称之为Observable_2,OnSubscribe_2(= OperatorSubscribeOn)。

那么,前两步就是创建了两个的observable,和OnSubscribe，并且OnSubscribe_2中保存了Observable_1的应用，即source。

调用Observable_2.subscribe()方法，那么根据我们之前blog[RxJava 源码走读之Observable.create()和subscribe()](http://blog.csdn.net/lisdye2/article/details/51068499)，可知，此时会调用OnSubscibe_2的call方法，即OperatorSubscribeOn的call()。

那么我们看一下call()的源码
```java 

    @Override
    public void call(final Subscriber<? super T> subscriber) {

        //worker是一个Subscription对象，同时也是一个线程循环worker
        final Worker inner = scheduler.createWorker();


        subscriber.add(inner);
        //改变线程
        inner.schedule(new Action0() {
            @Override
            public void call() {
                final Thread t = Thread.currentThread();
                
                Subscriber<T> s = new Subscriber<T>(subscriber) {
                    @Override
                    public void onNext(T t) {
                        subscriber.onNext(t);
                    }
                    
                    @Override
                    public void onError(Throwable e) {
                        try {
                            subscriber.onError(e);
                        } finally {
                            inner.unsubscribe();
                        }
                    }
                    
                    @Override
                    public void onCompleted() {
                        try {
                            subscriber.onCompleted();
                        } finally {
                            inner.unsubscribe();
                        }
                    }
                    
                    @Override
                    public void setProducer(final Producer p) {
                        subscriber.setProducer(new Producer() {
                            @Override
                            public void request(final long n) {
                                if (t == Thread.currentThread()) {
                                    p.request(n);
                                } else {
                                    inner.schedule(new Action0() {
                                        @Override
                                        public void call() {
                                            p.request(n);
                                        }
                                    });
                                }
                            }
                        });
                    }
                };
                
                source.unsafeSubscribe(s);
            }
        });
    }

```

这么长的一大段逻辑，瞬间懵了。

挑重点看：

- `inner.schedule()`改变了线程，此时`Action`的`call()`在我们指定的线程中运行。
- `Subscriber`被包装了一层。
- ` source.unsafeSubscribe(s);`，注意`source`是`Observable_1`对象。那么看一下此方法

```java 
public final Subscription unsafeSubscribe(Subscriber<? super T> subscriber) {
        try {
            // new Subscriber so onStart it
            subscriber.onStart();
            // allow the hook to intercept and/or decorate
            hook.onSubscribeStart(this, onSubscribe).call(subscriber);
            return hook.onSubscribeReturn(subscriber);
        } catch (Throwable e) {
            // special handling for certain Throwable/Error/Exception types
            Exceptions.throwIfFatal(e);
            // if an unhandled error occurs executing the onSubscribe we will propagate it
            try {
                subscriber.onError(hook.onSubscribeError(e));
            } catch (Throwable e2) {
                Exceptions.throwIfFatal(e2);
                // if this happens it means the onError itself failed (perhaps an invalid function implementation)
                // so we are unable to propagate the error correctly and will just throw
                RuntimeException r = new RuntimeException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
                // TODO could the hook be the cause of the error in the on error handling.
                hook.onSubscribeError(r);
                // TODO why aren't we throwing the hook's return value.
                throw r;
            }
            return Subscriptions.unsubscribed();
        }
    }

```

代码很多，但很重要的一句：`hook.onSubscribeStart(this, onSubscribe).call(subscriber);`，该方法即调用了OnSubscribe_1.call()方法。注意，此时的call()方法在我们指定的线程中运行。那么就起到了改变线程的作用。

对于以上线程，我们可以总结，其有如下流程：

- Observable.create() : 创建了Observable_1和OnSubscribe_1;
- subscribeOn()： 创建Observable_2和OperatorSubscribeOn(OnSubscribe_2)，同时`OperatorSubscribeOn`保存了`Observable_1`的引用。
- observable_2.subscribe(Observer):
	- 调用`OperatorSubscribeOn`的`call()`。`call()`改变了线程的运行，并且调用了`Observable_1.unsafeSubscribe(s);`
	- `Observable_1.unsafeSubscribe(s);`，该方法的实现中调用了`OnSubscribe_1`的`call()`。



从这个可以了解，无论我们的`subscribeOn()`放在哪里,他改变的是subscribe()的过程，而不是onNext()的过程。

那么如果有多个subscribeOn()，那么线程会怎样执行呢。如果按照我们的逻辑，有以下程序
```java 
   Observable.just("ss") 
                .subscribeOn(Schedulers.io())   // ----1---
                .subscribeOn(Schedulers.newThread()) //----2----
                .subscribe(new Action1<String>() {
                    @Override
                    public void call(String s) {

                    }
                });
```

那么，我们根据之前的源码分析其执行逻辑。

-  `Observable.just("ss") `,创建`Observable_1`，`OnSubscribe_1`
-  `Observable_1.subscribeOn(Schedulers.io())`:创建`observable_2`,`OperatorSubscribeOn_2`并保存`Observable_1`的引用。
-  `observable_2.subscribeOn(Schedulers.newThread())`：创建`Observable_3`,`OperatorSubscribeOn_3`并保存`Observable_2`的引用。
-  `Observable_3.subscribe()`:
	-  调用`OperatorSubscribeOn_3.call()`,改变线程为`Schedulers.newThread()`。
	-  调用`OperatorSubscribeOn_2.call()`,改变线程为`Schedulers.io()`。
	-  调用`OnSubscribe_1.call()`,此时`call()`运行在`Schedulers.io()`。

根据以上逻辑分析，会按照1的线程进行执行。


### ObserveOn() 分析

看一下`observeOn()`源码：

```java 
 public final Observable<T> observeOn(Scheduler scheduler) {
        return observeOn(scheduler, RxRingBuffer.SIZE);
    }

 public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return lift(new OperatorObserveOn<T>(scheduler, delayError, bufferSize));
    }

```

`lift()`，我在[RxJava源码走读之map](http://blog.csdn.net/lisdye2/article/details/51075503)已经说过此方法。

在`lift()`中，有如下关键代码
```java 
 Subscriber<? super T> st = hook.onLift(operator).call(o);
```

那么关键点就在`OperatorObserveOn.call()`。

```java 
 @Override
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        if (scheduler instanceof ImmediateScheduler) {
            // avoid overhead, execute directly
            return child;
        } else if (scheduler instanceof TrampolineScheduler) {
            // avoid overhead, execute directly
            return child;
        } else {
            ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError, bufferSize);
            parent.init();
            return parent;
        }
    }

```

我们看到其返回了`ObserveOnSubscriber<T>`，注意：此时只调用了`call()`方法，但`call()`方法中并没有改变线程的操作，此时为`subscribe()`过程。

我们直奔重点，因为，我们了解到其改变的是onNext()过程，那么我们肯定要看一下`ObserveOnSubscriber.onNext()`找找在哪改变线程

```java 
@Override
        public void onNext(final T t) {
            if (isUnsubscribed() || finished) {
                return;
            }
            if (!queue.offer(on.next(t))) {
                onError(new MissingBackpressureException());
                return;
            }
            schedule();
        }
```

可以看到，此时调用了`schedule();`，从方法名可以看到，其肯定是改变线程用的，并且该方法经过一番循环之后，调用了该类的`call()`方法。

```java 
// only execute this from schedule()
        @Override
        public void call() {
           //......


            for (;;) {
                long requestAmount = requested.get();
                long currentEmission = 0L;
                
                while (requestAmount != currentEmission) {
                    boolean done = finished;
                    Object v = q.poll();
                    boolean empty = v == null;
                    
                    if (checkTerminated(done, empty, localChild, q)) {
                        return;
                    }
                    
                    if (empty) {
                        break;
                    }
                    
                    localChild.onNext(localOn.getValue(v));

                    currentEmission++;
                    emitted++;
                }
                
           //......
        }

````
`call()`中有` localChild.onNext(localOn.getValue(v));`调用。

可以看到，其`observeOn`改变的是`onNext()`调用。


### observableOn和observeOn 对比分析

```java 
Observable
.map                    // 操作1
.flatMap                // 操作2
.subscribeOn(io)
.map                    //操作3
.flatMap                //操作4
.observeOn(main)
.map                    //操作5
.flatMap                //操作6
.subscribeOn(io)        //!!特别注意
.subscribe(handleData)
```

有如上逻辑，则我们对其运行进行分析。

首先，我们需要先明白其内部执行的逻辑。

在调用`subscribe`之后，逻辑开始运行。分别调用每一步`OnSubscribe.call()`，注意：自下往上。当运行到最上，即`Observable.create()`后，我们在其中调用了`subscriber.onNext()`,于是程序开始自上往下执行每一个对象的`subscriber.onNext()`方法。最终，直到`subscribe()`中的回调。

其次，从上面对`subscribeOn()`和`observeOn()`的分析中可以明白，`subscribeOn()`是在`call()`方法中起作用，而`observeOn()`实在`onNext()`中作用。

那么对于以上的逻辑，我们可以得出如下结论：

- 操作1,2,3,4在`io`线程中，因为在如果没有`observeOn()`影响，他们的回调操作默认在订阅的线程中。而我们的订阅线程在`subscribeOn(io)`发生了改变。注意他们执行的先后顺序。
- 操作5,6在`main`线程中运行。因为`observeOn()`改变了`onNext()`.
- 特别注意那一个逻辑没起到作用


### subscribeOn和observeOn 总结

- subscribeOn()改变的是订阅的线程，即call()执行的线程。
- observeOn()改变的是发送的线程，即onNext()的过程，注意，即我们在逻辑中通过操作符对数据进行修改都是在发送的过程中。除了最初`Observable`创建的过程。

