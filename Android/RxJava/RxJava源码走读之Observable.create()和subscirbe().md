## RxJava 源码剖析

### 最简单的Observable.subscribe(Observable)

看一下我们的例子
```java 
 Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {

            }
        }).subscribe(new Observer<String>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(String s) {

            }
        });
```

在这个方法中，我们通过Observable.create()创建一个Observable对象，然后通过subscribe()方法订阅。
#### Obsevable.create(OnSubscribe<T>)

那么，我们先看一下`Observable.create()`的源码
```java 
  public static <T> Observable<T> create(OnSubscribe<T> f) {
        return new Observable<T>(hook.onCreate(f));
    }
```
`hook.onCreate(f)`源码

```java 
  public <T> OnSubscribe<T> onCreate(OnSubscribe<T> f) {
        return f;
    }
```
`onCreate`中并没有做什么，只是将其传入的参数再返回，感觉好无聊啊，什么都没有做，那么`hook`在`Observable`中`hook`的生命
```java

    static final RxJavaObservableExecutionHook hook = RxJavaPlugins.getInstance().getObservableExecutionHook();

``` 
可以看到`hook`对象是一个静态的final对象，暂时我们无需管他是干什么的。

然后调用了Observable的构造函数并返回，看一下其构造方法
```java 
  protected Observable(OnSubscribe<T> f) {
        this.onSubscribe = f;
    }

```
Observable中就干了一件事情，他将传入的OnSubscribe赋值当前类的字段中保存。

那么我们可以总结Observable.create（）就是调用Observable的构造方法，保存传入的Obsubscribe对象。

#### Observable.subscribe(Observable)

看完了create的方法，下一步紧接着就是订阅了，我们看一下其方法。
```java 
    public final Subscription subscribe(final Observer<? super T> observer) {
        if (observer instanceof Subscriber) {
            return subscribe((Subscriber<? super T>)observer);
        }
        return subscribe(new Subscriber<T>() {

            @Override
            public void onCompleted() {
                observer.onCompleted();
            }

            @Override
            public void onError(Throwable e) {
                observer.onError(e);
            }

            @Override
            public void onNext(T t) {
                observer.onNext(t);
            }

        });
    }
```
在该方法中，先判断当前`Observable`是否属于`Subscriber`，如果属于，则调用`subscribe(Subscribe)`方法，如果不属于，则依然调用`subscribe(Subscribe)`方法，不过会重新new 一个`Observer`对象，并将新创建的Observer的`onNext`，`onComplete`，`onError`方法的结果与传入的`Observer`的三个方法所绑定。

在这里，我们可以思考，为什么非要多此一个判定呢，我们看一下Observer，发现其只是一个接口，并不是Subscriber的子类
```java 
public interface Observer<T> {
    void onCompleted();
    void onError(Throwable e);
    void onNext(T t);

}
```

可以看到其只是定义了三个方法，并没有干什么。我真日了狗了，这是几个意思呢。那么又有一个问题，我们的subscriber中，走了if里面的逻辑，还是走的if之外的逻辑，直接创建一个新的进行返回了呢，我为此测试了一下，在`onComplete`中，添加了一个打印
```java 
        @Override
        public void onCompleted() {
            System.out.print("sssss");
            observer.onCompleted();
        }


```

好吧，他果然走的是if之外的，直接创建了一个新的Subscriber对象,那么，为什么要加这个判断呢，`Subscriber`又和`Observer`是什么关系呢。。。。。。然而，我并不知道，真的。

我们唯一知道的是，当我们调用`subscribe（）`方法时，返回的是一个`Subscriber`方法，而且`subscribe(Observer)`方法的实质是调用了`subscribe(Subscriber)`方法。我们暂时先带着疑问往下看，看一下`subscribe(Subscriber)`方法的实现。

该方法我们按照实现顺序走

```java 
 	if (subscriber == null) {
            throw new IllegalArgumentException("observer can not be null");
        }
        if (observable.onSubscribe == null) {
            throw new IllegalStateException("onSubscribe function can not be null.");
            /*
             * the subscribe function can also be overridden but generally that's not the appropriate approach
             * so I won't mention that in the exception
             */
        }

```
判断`subscriber`是否为null，这个不用解释，`Observable.onSubscriber`是否为null，这个在之前`Observable.create()`源码的分析中，知道该方法在构造方法中进行赋值。

```java 
 // new Subscriber so onStart it
        subscriber.onStart();
```

调用了`subscriber.onStart`的方法，难道，这又是除了三个基本的回调方法之外的有一个回调，于是，我赶紧回到之前的`new Observer()`的地方，添加`onStart()`方法，发现并没有该重载的方法，哦，对，`Observer`是一个接口，就定义了三个方法，我们这时候传入的值在subscribe（Observable）中new 了一个`Subscriber`对象传入的，那么我们赶紧看一下`Subscriber`这个类。

```java 
public abstract class Subscriber<T> implements Observer<T>,Subscription

```
那么他俩的关系已经明白了，`Subscriber`是`Observer`的实现类。

而`onStart()`方法是`Subscriber`中的一个方法。它也属于回调级别的。

我们继续往下看
```java 
 // if not already wrapped   包裹一层
        if (!(subscriber instanceof SafeSubscriber)) {
            // assign to `observer` so we return the protected version
            subscriber = new SafeSubscriber<T>(subscriber);
        }
```

他将`subscriber`包装起来，这个具体什么意思有待研究，但不影响后续的，我们继续往下看。

```java 
   hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
   return hook.onSubscribeReturn(subscriber);
```

有牵扯到`hook`这个类了，我们看一下这个方法
```java 

 public <T> OnSubscribe<T> onSubscribeStart(Observable<? extends T> observableInstance, final OnSubscribe<T> onSubscribe) {
        // pass through by default
        return onSubscribe;
    }
```
可以看到该方法就是把当前方法的第二个参数，即`onSubscribe`返回并调用该对象的`call()`方法。

在之前说了那么多的`OnSuscribe`，那么`OnSubscribe`具体的实现是什么呢。
```java 
public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {
        // cover for generics insanity
    }

```
他继承了`Action1`的接口
```java 
public interface Action1<T> extends Action {
    void call(T t);
}
```

ok,Call方法终于出现了。也就是说在这个地方会回调OnSubscribe的call()方法，并且call方法中传入的是我们之前订阅的Observer对象。

就差最后一步了，返回值，我们可以看到subscribe的返回值为`hook.onSubscribeReturn(subscriber);`
```java 
 public <T> Subscription onSubscribeReturn(Subscription subscription) {
        // pass through by default
        return subscription;
    }
```
说白了，就是返回订阅的Observer对象。


#### 思路整理归纳

那么我们现在通过之前的源码分析，再把思路理一下。
- 在`Observable.create(OnSubscribe)`中，我们会调用`Observable`的构造方法，并保存当前我们传入的对象`OnSubscribe`，而`OnSubscribe`的实质是继承了`Action1`接口，而`Action1`接口就定义了一个call方法。
- 在`Observable.subscribe(Observer)`中，虽然传入的是`Observer`，但会通过包装将其转为`Subscriber`对象，并调用`Observable.subscribe(Subscriber)`方法，而该方法的实质，是会调用之前传入的`OnSubscribe`的Call（T）方法，而call(T)方法的参数就为当前`Subscribe`对象，这也也就理解了为什么在`Observer.create()`方法中，new 出的`Onsubcribe`对象中实现的call方法中能够操作`Subscribe`对象，同时在下面订阅时产生回调。而且在`subscribe()`方法调用时，最后的返回值就是我们订阅的Subscriber对象。
- 最后，其实我们在订阅Observer时，操作的实质是Subscriber对象，只不过在订阅时会有一侧转换和过渡。


#### 最后的疑问
OK,关于`Observable.create`和`Observable.subscribe()`有了简单的分析，但仍然还有两个疑问。
- hook在这里充当着什么角色，为什么其总是做一些无用的操作。
- SafeSubscriber对象又是一个什么东西，为什么我们在subscribe()中会对Subscriber对象做一层封装。

对于第一个疑问，我也暂时不清楚。

对于第二个疑问，我们看一下SafeSubscriber类应该能够找到原因。
```java 
   public SafeSubscriber(Subscriber<? super T> actual) {
        super(actual);
        this.actual = actual;
    }
```
actual就是Subscriber对象，调用父类构造方法，保存当前引用。
```java 
public class SafeSubscriber<T> extends Subscriber<T>
```
很显然，就是一层包装，添加一些附属的功能。那么根据思路，我们看一下`onNext()`

```java 
  @Override
    public void onNext(T args) {
        try {
            if (!done) {
                actual.onNext(args);
            }
        } catch (Throwable e) {
            // we handle here instead of another method so we don't add stacks to the frame
            // which can prevent it from being able to handle StackOverflow
            Exceptions.throwOrReport(e, this);
        }
    }

```

很明显，捕获异常，同时抛出一个异常，我们深入到`Exceptions.throwOrReport(e, this);`方法中看一下
```java 
@Experimental
    public static void throwOrReport(Throwable t, Observer<?> o) {
        Exceptions.throwIfFatal(t);
        o.onError(t);
    }
```

第一个参数是一场信息，第二个参数是我们当前的订阅对象Subscriber，那么`o.onError(t)`就不用再多解释了。

OK，大致基本的问题已经解决，就剩一个hook的问题，仍然存在，我们带着疑问，继续探析。