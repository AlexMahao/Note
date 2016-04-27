## RxJava源码走读之map


map的作用主要是将不同的对象进行变换，比如我们有一个需求，对于我们输入"a","b"，如果是"a"，则返回0，如果是"b"返回1，如果都不是则返回-1；

如果有基础的可以很简单的写出代码。
```java 
  Observable.create(new Observable.OnSubscribe<String>() {

			// -------1-----------
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("a");
                subscriber.onNext("b");
                subscriber.onCompleted();
            }
        }).map(new Func1<String, Integer>() {

			// -------2-----------
            @Override
            public Integer call(String s) {
                if ("a".equals(s)) {
                    return 0;
                } else if ("b".equals(s)) {
                    return 1;
                }
                return -1;
            }
        }).subscribe(new Observer<Integer>() {


			// -------3-----------
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(Integer s) {

            }
        });

```

下面我们就进入到源码探究一下

在我的上一篇博客[RxJava 源码走读之Observable.create()和subscribe()](http://blog.csdn.net/lisdye2/article/details/51068499)中，已经介绍了`create()`和`subscribe()`方法，所以我们直接进入call方法中瞅瞅。

```java 
  public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
        return lift(new OperatorMap<T, R>(func));
    }
```
`map()`中调用了`lift`方法，并传入了`OperatorMap`对象，注意这个`OperatorMap`很重要。

看一下`lift（）`,

```java 
  public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
        return new Observable<R>(new OnSubscribe<R>() {
            @Override
            public void call(Subscriber<? super R> o) {
         	 ...
        });
    }

```
我们可以看到，`lift()`返回了一个新的Observable对象，了和之前最初的`Observable`区分，这里称之为`Observable_1`,同时新创建的`OnSubscribe`对象为`OnSubscribe_1`。

那么我们的`map()`返回的是一个新的对象`Observable_1`。那么后面的`subscribe()`订阅的就是新的`Observable_1`对象。

根据上一篇博客的分析，他的执行逻辑是
> Observable_1.subscribe() =》OnSubscribe_1.call()方法。

那么看一下`OnSubscribe_1.call()`的逻辑，

```java 
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
        return new Observable<R>(new OnSubscribe<R>() {
            @Override
            public void call(Subscriber<? super R> o) {
                try {
                    Subscriber<? super T> st = hook.onLift(operator).call(o);
                    try {
                        // new Subscriber created and being subscribed with so 'onStart' it
                        st.onStart();

                        //外部类的onSubscriber
                        onSubscribe.call(st);
                    } catch (Throwable e) {
                        // localized capture of errors rather than it skipping all operators 
                        // and ending up in the try/catch of the subscribe method which then
                        // prevents onErrorResumeNext and other similar approaches to error handling
                        Exceptions.throwIfFatal(e);
                        st.onError(e);
                    }
                } catch (Throwable e) {
                    Exceptions.throwIfFatal(e);
                    // if the lift function failed all we can do is pass the error to the final Subscriber
            // as we don't have the operator available to us
            o.onError(e);
        }
    }
        });
    }
```

我们按照逻辑，看一下都是做了什么。

` Subscriber<? super T> st = hook.onLift(operator).call(o);`又是hook，我一猜就知道，还是直接返回参数。

```java 
 public <T, R> Operator<? extends R, ? super T> onLift(final Operator<? extends R, ? super T> lift) {
        return lift;
    }
```
果不然，那么我们还记得`operator`是什么对象吗，如果我们只看当前方法，`Operator`，而该对象又是`Func1`的子类，那么
```java 
public interface Func1<T, R> extends Function {
    R call(T t);
}
```
看到这里，大写的懵逼。。。。

我们在回头想一想，在`map()`中，我们调用`lift()`中，参数传入的是什么，`OperatorMap`，是该对象，很明显，该类是`Operator`的子类，我们看一下他`call()`方法，
```java 
@Override
    public Subscriber<? super T> call(final Subscriber<? super R> o) {
        return new Subscriber<T>(o) {

            @Override
            public void onCompleted() {
                o.onCompleted();
            }

            @Override
            public void onError(Throwable e) {
                o.onError(e);
            }

            @Override
            public void onNext(T t) {
                try {
                    o.onNext(transformer.call(t));
                } catch (Throwable e) {
                    Exceptions.throwOrReport(e, this, t);
                }
            }

        };
    }
```
`call()`返回了Subscriber对象。在探讨之前，我们分析一下到现在为止，我们总共创建了几个`Subscriber`对象

- `subscribe(Subscriber)`中传入的`subscriber`对象，


- 在上面的`call()`方法中，对我们传入的`subscriber`中进行了一次包装，创建了`Subscriber_1`对象，我们看一下他的三个实现方法，重点看一下`onNext()`方法。

```java 
 @Override
            public void onNext(T t) {
                try {
                    o.onNext(transformer.call(t));
                } catch (Throwable e) {
                    Exceptions.throwOrReport(e, this, t);
                }
            }
```
当`Subscriber_1`对象回调`onNext()`方法时，会调用我们初创`Subscriber`的`onNext()`，传入的参数`transformer.call(t)`,很明显，就是我们在map中传入的`Func1`对象。

我们该行代码分析完之后，在回到`OnSubscribe_1`的`call()`.

主要代码就两句，
```java 
  st.onStart();

  //外部类的onSubscriber
  onSubscribe.call(st);
```

调用`st.onStart()`方法，在这里需要区分的两点，st为`Subscriber_1`,而`onSubcribe`是在`Observable.create()`保存的`OnSubscribe`对象，那么逻辑就解开了，调用此时的call方法，会执行在实例代码中1，2，3的顺序执行。

`map()`的整体逻辑我们已经走了一遍。下面该总结了

根据我们上面的推论，他的逻辑应该如下。

- `Observable.create(OnSubscribe)`创建`Observable`,并保存`OnSubscribe`对象的引用到当前`Observable`。


- `map(Func)`，创建新的`Observable_1`和`OnSubscribe_1`.


- `subscribe(Observer)`，订阅。注意此时是调用的是`Observable_1`的`subscibe()`。


- 根据我们上一篇博客的分析，`Observer`在订阅时，会创建`Subscriber`，此为目标Subscriber，然后调用`OnSubscribe_1`的`call()`方法。

- 在`OnSubscribe_1`的`call()`中，会对目标Subscriber进行包装，创建出`Subscriber_1`对象。调用`OnSubscribe`的`call`方法，传入的参数为`Subscriber_1`。


- 那么我们此时在实例中的第一个`call()`被回调，参数为`Subscriber_1`,我们在该方法中调用了`subscriber_1.onNext("a")`.
 
- 在`Subscriber_1`的`onNext()`中，调用目标Subscriber的`onNext(Func.call())`方法，注意参数，所以，此时`map`中的`call()`被回调，同时作为参数，传入到了最终的onNext()中 。

借用一下扔物线大神的图
> lift返回的Observable中的对象类比Observable_1,OnSubscribe_1,Subscriber_1.


 
但在这幅图中，存在着一个问题即lift返回的Observable中，Subscriber的构建时间要迟于目标Subscriber.