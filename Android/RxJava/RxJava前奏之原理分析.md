## RxJava 之前奏：原理分析

首先我们进入一个例子，关于猫的例子。


>我们有个 Web API，能根据给定的查询请求搜索到整个互联网上猫的图片。每个图片包含可爱指数的参数（描述图片可爱度的整型值）。我们的任务将会下载到一个猫列表的集合，选择最可爱的那个，然后把它保存到本地。

首先定义实体类
```java 
public class Cat implements Comparable<Cat> {
	/**
	 * 图片
	 */
	String image;

	/**
	 * 喜爱度
	 */
	int cuteness;

	@Override
	public int compareTo(Cat another) {
		return Integer.compare(cuteness, another.cuteness);
	}

}

```

定义一个Uri类，用来定义图片保存的地址
```java 
public class Uri {

}
```
定义Api接口类，主要查询猫和保存猫的方法。
```java 
public interface Api {

	/**
	 * 查询所有猫
	 * @param query
	 * @return
	 */
	List<Cat> queryCats(String query);
		
	/**
	 * 保存
	 * @param cat
	 * @return
	 */
	Uri store(Cat cat);
	
}
```

定义业务类CastsHelp
```java 
public class CastsHelp {
	Api api;

	//保存猫
	public Uri saveTheCutestCat(String query) {
		List<Cat> queryCats = api.queryCats(query);
		Cat cat = findCustest(queryCats);
		
		Uri savedUri = api.store(cat);
		return savedUri;
	}

	//查询最受喜爱的猫
	private Cat findCustest(List<Cat> queryCats) {
		return Collections.max(queryCats);
	}
}
```
ok,我们的第一个例子写完了，让我们回过头来再仔细看一下这个例子，该业务逻辑类CastsHelp，里面包含了两个方法，其中saveTheCutestCat方法里面，包含了三个组合的方法，最终获取到保存猫的地址，并返回。这种组合方式即简单，又直白。一眼看去，清晰，明了。但我们往下看。

### 异步执行
我们知道，对于一些网络请求，我们通常都需要进行异步去执行，那么，对于刚才说saveTheCutestCat方法，我们就不能如此写，对于api的中的方法，我们需要利用接口回调的方式回传结果。于是我们修改我们的代码。
```java 
public interface Api {

	
	interface CatsQueryCallback{
		void onCatListReceived(List<Cat> cats);
		void onCatsQueryError(Exception e);
	}
	
	
	interface StoreCallback{
		void onCateStored(Uri uri);
		void onStoteFailed(Exception e);
	}
	/**
	 * 查询所有猫
	 * @param query
	 * @return
	 */
	List<Cat> queryCats(String query,CatsQueryCallback castQueryCallback);
	
	
	/**
	 * 保存
	 * @param cat
	 * @return
	 */
	Uri store(Cat cat,StoreCallback storeCallback);
	
}

```
修改了一下api中的方法，因为对于猫和保存猫，分别为网络请求，和I/O操作，我们不能进行同步操纵，所以添加了两个接口，分别查询猫的和保存猫地址的结果的回调。

同时我们需要修改一下我们的业务逻辑类CastsHelp，也需要添加接口回调。
```java 
public interface CutestCatCallback{
		void onCutestCatSaved(Uri uri);
		
		void onCutestCatFail(Exception e);
	}
	
	Api api;

	public void saveTheCutestCat(String query,final CutestCatCallback custestCatCallback) {
		List<Cat> queryCats = api.queryCats(query,new CatsQueryCallback() {
			
			@Override
			public void onCatsQueryError(Exception e) {
				//查询猫失败
				custestCatCallback.onCutestCatFail(e);
			}
			
			
			@Override
			public void onCatListReceived(List<Cat> cats) {
				Cat cat = findCustest(cats);
				
				api.store(cat, new StoreCallback() {
					
					@Override
					public void onStoteFailed(Exception e) {
						//保存猫失败
						custestCatCallback.onCutestCatFail(e);
						
					}
					
					@Override
					public void onCateStored(Uri uri) {
						custestCatCallback.onCutestCatSaved(uri);
					}
				});
			}
		});
		
	}

	private Cat findCustest(List<Cat> queryCats) {
		return Collections.max(queryCats);
	}
```
代码开始复杂了，我们再来分析一下代码，在CastsHelp的saveTheCutestCat方法，我们传入了一个url，以及结果的回调，及该方法调用者用来实现的接口回调。
 - 首先，调用api的queryCats方法查询所有的猫，因为该方法是异步的，所以我们传入一个接口的实现类，该接口包含了两个回调方法，onCatsQueryError（）查询失败的方法，在此方法中，我们直接调用调用者实现的接口告诉调用者获取信息出现异常，onCatListReceived（List<Cat> cats）方法，表明查询猫成功，并返回猫列表。
 - 我们在onCatListReceived（List<Cat> cats）方法内，此时查询猫列表的数据已经返回，我们继续调用了获取最喜爱的猫方法findCustest，然后保存猫。store（）方法又包含了两个回调方法，在onStoteFailed（）方法中，依然如上。在成功的方法onCateStored（）中，我们将获取到的值通过调用者的实现接口进行回传给调用者。
 
好了，我们完成了异步任务的处理，代码复杂度大大的提升了，而且对于这么多接口回调，看着都心理难受。

### 接口回调的封装

我们加入异步任务之后，整个程序在逻辑上的复杂度，代码的复杂度上都大大的提高了，但我们发现该接口有些共同点，我们可以进行封装。
首先，先把之前添加的接口放在一起类比一下：
```java 
	//业务逻辑类
	public interface CutestCatCallback{
		void onCutestCatSaved(Uri uri);
		
		void onCutestCatFail(Exception e);
	}


	//查询猫
	interface CatsQueryCallback{
		void onCatListReceived(List<Cat> cats);
		void onCatsQueryError(Exception e);
	}

	//保存猫
    interface StoreCallback{
		void onCateStored(Uri uri);
		void onStoteFailed(Exception e);
	}

```
看到三个接口，我们会发现，他们都有一个成功的回调和一个失败的回调，失败的回调参数一样，而成功的回调接口不一样。那么我们来抽取一下，抽取了一个如下的新接口回调：
```java 
public interface Callback<T> {
	
	void onResult(T result);
	
	void onError(Exception e);
	
}

```
新的接口回调，比较简单，成功和失败的方法，添加了泛型，那么我们把它移入项目中，这样关于我们的api接口又有所变动：
```java
public interface Api {
	/**
	 * 查询所有猫
	 * @param query
	 * @return
	 */
	void queryCats(String query,Callback castQueryCallback);
	
	
	/**
	 * 保存
	 * @param cat
	 * @return
	 */
	void store(Cat cat,Callback storeCallback);
	
}
```
在看一下我们的业务逻辑类
```java 
public class CastsHelp {

	Api api;

	public void saveTheCutestCat(String query,final Callback<Uri> custestCatCallback) {
		api.queryCats(query, new Callback<List<Cat>>() {

			@Override
			public void onResult(List<Cat> result) {
				Cat cat = findCustest(result);
				api.store(cat, new Callback<Uri>() {

					@Override
					public void onResult(Uri result) {
						custestCatCallback.onResult(result);
					}

					@Override
					public void onError(Exception e) {
						custestCatCallback.onError(e);
					}
				});
			}

			@Override
			public void onError(Exception e) {
				custestCatCallback.onError(e);
				
			}
		});
		
	}

	private Cat findCustest(List<Cat> queryCats) {
		return Collections.max(queryCats);
	}

```

逻辑上本有太大的变动，但在代码上，我们将之前定义的三个接口，统一改成了我们新定义的Callback接口。代码简洁了一些，同时也便于理解了一些。

### 再次进阶

在上面我们的Demo中，我们通过定义一个泛型回调，大大降低了代码量，但还是存在一个问题，及我们每次调用这个方法，都需要传入两个参数，分别为url和接口回调。如果，我们能不能只请求一个参数url，同时返回一个对象，该对象包含了我们所需要的回调信息。

那么我们在思考一下，我们需要的是一个对象，该对象包含了一个只有接口回调为参数的方法，我们称之为AsyncJob。
```java 
public abstract class AsyncJob<T> {

	public abstract void start(Callback<T> callback);

}

```
我们让查询的方法等的返回值为该对象，则返回的该对象中调用start方法即可开始执行回调。

因为我们的Api中的两个方法是通过接口回掉来进行的，所以我们需要对Api类进行包装。ApiWrapper.

```java 
/**
 * 接口的包装类 备注人： Alex_MaHao
 * 
 * @date 创建时间：2016年2月26日 上午10:11:28
 */
public class ApiWrapper {
	Api api;

	public AsyncJob<List<Cat>> queryCats(final String query) {
		return new AsyncJob<List<Cat>>() {

			@Override
			public void start(final Callback<List<Cat>> callback) {
				api.queryCats(query, new Callback<List<Cat>>() {

					@Override
					public void onResult(List<Cat> result) {
						callback.onResult(result);
						
					}

					@Override
					public void onError(Exception e) {
						callback.onError(e);
						
					}
				});
			}
		};

	}
	

	public AsyncJob<Uri> store(final Cat cat) {
		return new AsyncJob<Uri>() {

			@Override
			public void start(final Callback<Uri> callback) {
				api.store(cat, new Callback<Uri>() {

					@Override
					public void onResult(Uri result) {
						callback.onResult(result);
					}

					@Override
					public void onError(Exception e) {
						callback.onError(e);
					}
				});
			}

		};
	}

}

```
我们看一下我们的ApiWrapper中的方法，因为两个方法比较类似，我们只看queryCats方法，理解一下即可。如果单纯的看逻辑，会发现比较绕，我们看一下第一个方法应该怎么使用。
```java 
		//调用方法，返回包含我们回调信息的asyncJob类
		AsyncJob<List<Cat>> async = new ApiWrapper().queryCats("query");
		
		//启动回调方法，获取回调结果
		async.start(new Callback<List<Cat>>() {
			
			@Override
			public void onResult(List<Cat> result) {
				//所有猫的集合
			}
			
			@Override
			public void onError(Exception e) {
				//错误回调
			}
		});
```
可以看到，我们的queryCats()传入一个url地址，返回一个包含我们回调接口的对象，这时候，我们只需要调用start()方法，即可获取到我们的回调结果。

那么，同时我们也需要修改一下CastsHelp中的代码
```java 
public class CastsHelp {
	
	ApiWrapper apiWrapper;

	public AsyncJob<Uri> saveTheCutestCat(final String query) {
		return new AsyncJob<Uri>() {
			
			@Override
			public void start(final Callback<Uri> callback) {
				apiWrapper.queryCats(query).start(new Callback<List<Cat>>() {
					
					@Override
					public void onResult(List<Cat> result) {
						Cat cat = findCustest(result);
						
						apiWrapper.store(cat).start(new Callback<Uri>() {
							
							@Override
							public void onResult(Uri result) {
								callback.onResult(result);
							}
							
							@Override
							public void onError(Exception e) {
								callback.onError(e);
							}
						});
					}
					
					@Override
					public void onError(Exception e) {
						callback.onError(e);
					}
				});;
			}
		};
		
	}

	private Cat findCustest(List<Cat> queryCats) {
		return Collections.max(queryCats);
	}
}
```

该方法也只是把我们的方法的返回值对象做了一下转换，修改了其返回值对象为我们的AsyncJob。

我们把该方法在进行整理，于是就有了如下代码
```java 
public class CastsHelp {

	ApiWrapper apiWrapper;

	public AsyncJob<Uri> saveTheCutestCat(final String query) {

		//猫列表数据的接口回调
		final AsyncJob<List<Cat>> catsListAsyncJob = apiWrapper
				.queryCats(query);
		
		//查询最喜爱的猫
		final AsyncJob<Cat> cutestCatAsyncJob = new AsyncJob<Cat>() {

			@Override
			public void start(final Callback<Cat> callback) {
				catsListAsyncJob.start(new Callback<List<Cat>>() {

					@Override
					public void onResult(List<Cat> result) {
						callback.onResult(findCustest(result));

					}

					@Override
					public void onError(Exception e) {
						callback.onError(e);
					}
				});
			}
		};

		//获取保存的url地址的回掉对象并返回
		AsyncJob<Uri> storedUriAsyncJob = new AsyncJob<Uri>() {

			@Override
			public void start(final Callback<Uri> callback) {
				cutestCatAsyncJob.start(new Callback<Cat>() {

					@Override
					public void onResult(Cat result) {
						apiWrapper.store(result).start(callback);
					}

					@Override
					public void onError(Exception e) {
						callback.onError(e);
					}
				});
			}
		};

		return storedUriAsyncJob;

	}

	private Cat findCustest(List<Cat> queryCats) {
		return Collections.max(queryCats);
	}
}

```

### map和flatMap

在我们的CastsHelp的saveTheCutestCat()方法中，有一个查询喜爱都最高的猫的列表，那么我们能不能这样理解呢，从输入和输出来看，我们期望将一个AsyncJob<List<Cat>>的回调对象转化为一个AsyncJob<Cat>对象,中间的具体转换方式，我们并不去管他怎么实现。而是定义一个接口，让其调用者来实现。

我们首先定义一个接口，该接口主要目的是让起调用者实现其接口，来完成转换操作。这种设计模式类属于模板模式，即我们不清楚该方法的具体实现时，我们将其定义为虚函数，确定输入和输出，让其调用者实现输出到输入的转换，我们只需要在我们需要调用此方法时，调用即可，不过我们这里使用的是定义了一个接口。

```java 
public interface Func<T,R> {
	R call(T t);
}
```
该接口很简单，传入T，通过call方法返回R。

我们在AsyncJob中添加map方法，先看代码
```java 
public <R> AsyncJob<R> map(final Func<T,R> func){
		final AsyncJob<T> source = this;
		
		return new AsyncJob<R>(){

			@Override
			public void start(final Callback<R> callback) {
				source.start(new Callback<T>() {

					@Override
					public void onResult(T result) {
						//调用者实现的call方法
						R r = func.call(result);

						callback.onResult(r);
					}

					@Override
					public void onError(Exception e) {
						callback.onError(e);
					}
				});
			}
			
		};
		
	}
```
我们看一下这个方法，在之前我们这样分析，我们需要的是将AsyncJob<List<Cat>>的回调对象转化为一个AsyncJob<Cat>对象,其实质就是将List<cat>数据转化为Cat并包装成AsyncJob<Cat>对象进行返回，那么我们将我们的需要带入到map中，map的声明就如下
```java
public  AsyncJob<Cat> map(final Func<List<Cat>,Cat> func){
}
```

ok,我们在深入的看一下map方法，可以看到我们返回的对象，无非就是将当前的AsyncJob对象的回调信息与我们返回的信息所绑定，同时在onResult方法中将数据类型进行转换。

那么，在CastsHelp中的查询最喜爱的猫的方法，可改成如下所示：
```java
	final AsyncJob<Cat> cutestCatAsyncJob = catsListAsyncJob.map(new Func<List<Cat>, Cat>() {

			@Override
			public Cat call(List<Cat> t) {
				return findCustest(t);
			}
		});

```

map对象，在转换的时候是将一个对象转换为另一个对象，通过的是返回值的方式，业绩是同步的方式进行的，那么我们会不会有另一种情况呢，即我们的call方法返回的也是一个回调信息，及也是异步的呢。

这种情况也是存在的，我们看一下我们的保存猫的方法，该方法是一个异步的任务，他不能直接返回值，而是返回一个回调的信息，我们看一下保存猫的方法，`public AsyncJob<Uri> store(final Cat cat)`，很明显，他不能向我们之前查询喜爱都最高的猫时直接获取Cat对象，并调用onResult方法并返回，他是一个回调对象，我们需要做的是将返回的回调对象和我们转换过后的相绑定。这时候我们定义了flatMap（）；

```java
public <R> AsyncJob<R> flatMap(final Func<T, AsyncJob<R>> func ){
		final AsyncJob<T> source = this;
		return new AsyncJob<R>(){

			@Override
			public void start(final Callback<R> callback) {
				source.start(new Callback<T>() {

					@Override
					public void onResult(T result) {
						AsyncJob<R> mapped = func.call(result);
						
						mapped.start(callback);
					}

					@Override
					public void onError(Exception e) {
						callback.onError(e);
						
					}
				});
				
			}
			
		};
	}
```

该方法将返回的Func回调对象与我们的flatMap对象绑定。那么，保存猫的方式也可以修改如下：

```java 
	AsyncJob<Uri> storedUriAsyncJob = cutestCatAsyncJob.flatMap(new Func<Cat, AsyncJob<Uri>>() {

			@Override
			public AsyncJob<Uri> call(Cat t) {
				// TODO Auto-generated method stub
				return apiWrapper.store(t);
			}
		});
	
```

最后，贴上CastHelps的完整代码，
```java 

	public AsyncJob<Uri> saveTheCutestCat(final String query) {

		//查询猫
		final AsyncJob<List<Cat>> catsListAsyncJob = apiWrapper
				.queryCats(query);

		//获取最喜爱的猫
		final AsyncJob<Cat> cutestCatAsyncJob = catsListAsyncJob.map(new Func<List<Cat>, Cat>() {

			@Override
			public Cat call(List<Cat> t) {
				return findCustest(t);
			}
		});

		//获取地址
		AsyncJob<Uri> storedUriAsyncJob = cutestCatAsyncJob.flatMap(new Func<Cat, AsyncJob<Uri>>() {

			@Override
			public AsyncJob<Uri> call(Cat t) {
				return apiWrapper.store(t);
			}
		});
	
		return storedUriAsyncJob;

	}
```

- map和flatMap的总结
   - 相同点：都是通过Func对象的Call方法，让调用者来实现不同数据的对象转换
   - 不同点：Call方法转换的目标结果不同，导致对结果的处理不同。如果是同步的转换，不涉及接口回掉，异步等，map处理。如果Call方法返回的是一个回调对象，那么我们需要去绑定，所以使用flatMap.

细心的可以看到我们saveTheCutestCat方法还可以精简
```java 

	public AsyncJob<Uri> saveTheCutestCat(final String query) {

		return apiWrapper
				.queryCats(query)
				.map(new Func<List<Cat>, Cat>() {

					@Override
					public Cat call(List<Cat> t) {
						
						
						return findCustest(t);
					}
				}).flatMap(new Func<Cat, AsyncJob<Uri>>() {

					@Override
					public AsyncJob<Uri> call(Cat t) {
						
						return apiWrapper.store(t);
					}
				});

	}
```

### RxJava 的引入

在引入RxJava 之前，我们之前写了那么多的代码势必要使用一下。那么怎么使用呢：
```java 
public static void main(String[] args) {
		CastsHelp castsHelp = new CastsHelp();
		castsHelp.saveTheCutestCat("ss").start(new Callback<Uri>() {
			
			@Override
			public void onResult(Uri result) {
				//Uri 地址对象
			}
			
			@Override
			public void onError(Exception e) {
				//异常
			}
		});
	}
```

ok,开始和RxJava进行类比。

AsyncJob类比Observable
CallBack类比Observer
start()类比  subscribe();


那么我们开始对我的类进行修改,在这之前需要导入rxjava的jar包
```java 
public class ApiWrapper {
	Api api;

	public Observable<List<Cat>> queryCats(final String query) {
		
		return Observable.create(new OnSubscribe<List<Cat>>() {

			@Override
			public void call(final Subscriber<? super List<Cat>> arg0) {
				api.queryCats(query, new CatsQueryCallback() {
					
					@Override
					public void onCatsQueryError(Exception e) {
						arg0.onError(e);
					}
					
					@Override
					public void onCatListReceived(List<Cat> cats) {
						arg0.onNext(cats);
					}
				});
			}
		});
	}

	public Observable<Uri> store(final Cat cat) {
		return Observable.create(new OnSubscribe<Uri>() {

			@Override
			public void call(final Subscriber<? super Uri> subscriber) {
				api.store(cat, new StoreCallback() {
					
					@Override
					public void onStoteFailed(Exception e) {
						subscriber.onError(e);
					}
					
					@Override
					public void onCateStored(Uri uri) {
						subscriber.onNext(uri);
					}
				});
			}
		});
	}

}


```

```java
public class CastsHelp {

	ApiWrapper apiWrapper;

	public Observable<Uri> saveTheCutestCat(final String query) {
		return apiWrapper
				.queryCats(query)
				.map(new Func1<List<Cat>, Cat>() {

						@Override
						public Cat call(List<Cat> cats) {
			
							return findCustest(cats);
						}
					})
				.flatMap(new Func1<Cat, Observable<Uri>>() {
			
						@Override
						public Observable<Uri> call(Cat cat) {
			
							return apiWrapper.store(cat);
						}
					});

	}

	private Cat findCustest(List<Cat> queryCats) {
		return Collections.max(queryCats);
	}
}
```

```java 
public class Test {
	public static void main(String[] args) {
		CastsHelp castsHelp = new CastsHelp();
		castsHelp.saveTheCutestCat("ss").subscribe(new Observer<Uri>() {

			@Override
			public void onCompleted() {
				
			}

			@Override
			public void onError(Throwable arg0) {
				
			}

			@Override
			public void onNext(Uri arg0) {
				
			}
		});
	}
}

```


RxJava的原理已经理解和整理完毕。下面就开进入到了RxJava的整理与总结了。没多少代码，就不共享了。

长路漫漫啊。加油！！！

研究资料
[NotRxJava懒人专用指南](http://www.devtf.cn/?p=323)

