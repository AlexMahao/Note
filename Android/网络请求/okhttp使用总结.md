## okhttp 使用总结


`okhttp` 的使用越来越火，有必要对其进行研究。以下博客中的例子为了简单，在`Eclipse`中通过`JAVA`工程进行验证。

### **导入OkHttp**

**AndroidStudio**

添加如下代码即可

```java 
 compile 'com.squareup.okhttp3:okhttp:3.3.1'
 compile 'com.squareup.okio:okio:1.8.0'

```
其中`okio`是`okhttp`中关于流的操作。必须导入。

**Eclipse**

导入`okhttp`与`okio`的jar包即可。

下载地址[okhttp3.3.1与okio-1.8.0  ](http://download.csdn.net/detail/lisdye2/9563067)



### **同步的GET请求**

主要分为以下几步：

- 创建`OkHttpClient`对象
- 根据需求创建`Request`对象，`Request`中主要包含了请求的url，参数等信息。
- 通过`request`对象创建`Call`对象
- 使用`call`对象进行网络请求。返回封装好的`Response`
- 解析`Response`对象，获取响应信息 

```java 
/**
*  同步 Get 请求
*/
public static void syncGet() throws Exception {

		// 1. 创建`OkHttpClient`对象
		final OkHttpClient client = new OkHttpClient();

		// 2.根据需求创建`Request`对象.在此只是添加了最基本的url
		Request request = new Request.Builder().url(BAIDU_MP3_PATH).build();

		// 3.通过`request`对象创建`Call`对象
		Call call = client.newCall(request);
		
		// 4.使用`call`对象进行网络请求。返回封装好的`Response`
		Response response = call.execute();

		// 5.解析`Response`对象，获取响应信息
		if (!response.isSuccessful())
			// code >= 200 && code < 300;
			System.out.println(response);

		Headers responseHeaders = response.headers();

		for (int i = 0; i < responseHeaders.size(); i++) {
			System.out.println(responseHeaders.name(i) + ":"
					+ responseHeaders.value(i));
		}
		
		// 打印响应的内容
		System.out.println(response.body().string());

	}

```

其中使用的url为百度MP3的接口：
```java 
http://tingapi.ting.baidu.com/v1/restserver/ting?from=qianqian&version=2.1.0&method=baidu.ting.billboard.billList&format=json&type=1&offset=0&size=1
```


看一下打印结果：

```java 

//---- response 的toString（） 默认打印的信息  响应行
Response{protocol=http/1.1, code=200, message=OK, url=http://tingapi.ting.baidu.com/v1/restserver/ting?from=qianqian&version=2.1.0&method=baidu.ting.billboard.billList&format=json&type=1&offset=0&size=1}


//----- 响应的头
Date:Wed, 29 Jun 2016 01:51:39 GMT
Content-Type:application/json
Transfer-Encoding:chunked
Connection:keep-alive
X-LIGHTTPD-LOGID:427084050
RT:427084050_d41d8cd98f00b204e9800998ecf8427e_d41d8cd98f00b204e9800998ecf8427e
Server:Apache1.0
SC:MISS.0.0
tracecode:30991485713674384576062909
Set-Cookie:BAIDUID=B343E34EB3618E7E2E867A6E1D4F04EB:FG=1; expires=Thu, 29-Jun-17 01:51:39 GMT; max-age=31536000; path=/; domain=.baidu.com; version=1
P3P:CP=" OTI DSP COR IVA OUR IND COM "


// ---- 响应的内容  body
{"song_list":[{"artist_id":"247010469","language":"\u56fd\u8bed","pic_big":"http:\/\/musicdata.baidu.com\/data2\/pic\/0c87cb20fd6b543cd1c90d1d82f1f6c5\/266782982\/266782982.jpg","pic_small":"http:\/\/musicdata.baidu.com\/data2\/pic\/169cbb0d82522568dea198363f7028ec\/266782985\/266782985.jpg","country":"\u5185\u5730","area":"0","publishtime":"2016-06-20","album_no":"1","lrclink":"http:\/\/musicdata.baidu.com\/data2\/lrc\/1f75306d7f1e25f723d3d175f5ef3d46\/266791084\/266791084.lrc","copy_type":"1","hot":"276927","all_artist_ting_uid":"232954914","resource_type":"0","is_new":"1","rank_change":"0","rank":"1","all_artist_id":"247010469","style":"\u6d41\u884c","del_status":"0","relate_status":"0","toneid":"0","all_rate":"64,128,256","sound_effect":"0","file_duration":0,"has_mv_mobile":1,"versions":"","bitrate_fee":"{\"0\":\"0|0\",\"1\":\"-1|-1\"}","song_id":"266784254","title":"\u7ea2","ting_uid":"232954914","author":"\u51af\u5efa\u5b87","album_id":"266784256","album_title":"\u7ea2","is_first_publish":0,"havehigh":0,"charge":0,"has_mv":1,"learn":0,"song_source":"web","piao_id":"0","korean_bb_song":"1","resource_type_ext":"1","mv_provider":"0100000000","artist_name":"\u51af\u5efa\u5b87"}],"billboard":{"billboard_type":"1","billboard_no":"1871","update_date":"2016-06-28","billboard_songnum":"190","havemore":1,"name":"\u65b0\u6b4c\u699c","comment":"\u8be5\u699c\u5355\u662f\u6839\u636e\u767e\u5ea6\u97f3\u4e50\u5e73\u53f0\u6b4c\u66f2\u6bcf\u65e5\u64ad\u653e\u91cf\u81ea\u52a8\u751f\u6210\u7684\u6570\u636e\u699c\u5355\uff0c\u7edf\u8ba1\u8303\u56f4\u4e3a\u8fd1\u671f\u53d1\u884c\u7684\u6b4c\u66f2\uff0c\u6bcf\u65e5\u66f4\u65b0\u4e00\u6b21","pic_s640":"http:\/\/c.hiphotos.baidu.com\/ting\/pic\/item\/f7246b600c33874495c4d089530fd9f9d62aa0c6.jpg","pic_s444":"http:\/\/d.hiphotos.baidu.com\/ting\/pic\/item\/78310a55b319ebc4845c84eb8026cffc1e17169f.jpg","pic_s260":"http:\/\/b.hiphotos.baidu.com\/ting\/pic\/item\/e850352ac65c1038cb0f3cb0b0119313b07e894b.jpg","pic_s210":"http:\/\/business.cdn.qianqian.com\/qianqian\/pic\/bos_client_c49310115801d43d42a98fdc357f6057.jpg","web_url":"http:\/\/music.baidu.com\/top\/new"},"error_code":22000}


```

如果熟悉HTTP 协议的可知，作为Http的响应，分为响应行，响应头，响应体。

如上打印结果就分别对应了响应行，响应头，响应体。


注释很清楚，按照步骤即可，着重提一下解析`Response`的内容。

- `response.isSuccessful()`，该方法在响应code为[200,300)时，返回true，其余返回false。注意区间为前闭后开。
- `response.headers()`：获取响应行信息的所有参数的集合`headers`。`headers`的使用方式类似于`List`。
- `response.body().string()`：该方法返回的是响应的内容。但是，官方给出的指导是，如果响应的内容大于1MB 时，不建议使用该方法。因为该方法会将内容全部加载到内存中。推荐使用流的方式获取加载的数据。
	- `response.body().byteStream();`：字节输入流
	- `response.body().charStream();`：字符输入流



### **异步的GET 请求**


异步GET的请求方式几乎和上面的步骤类似，唯一的区别是通过`Call`对象请求网络时，不在直接返回`Response`对象，而是通过接口回调的方式获取数据。当然，表面上是这样，其内部肯定有对应的线程切换。

```java 
/**
	 * 异步的get方法
	 */
	public static void asyncGet() throws Exception {

		// 1. 创建`OkHttpClient`对象
		final OkHttpClient client = new OkHttpClient();
		
		// 2.根据需求创建`Request`对象.在此只是添加了最基本的url
		Request request = new Request.Builder().url(BAIDU_MP3_PATH).build();
		
		// 3.通过`request`对象创建`Call`对象
		Call call = client.newCall(request);
		
		// 4.使用`call`对象进行网络请求。通过Callback 监听网络请求
		call.enqueue(new Callback() {

			public void onResponse(Call call, Response response)
					throws IOException {
				//5， 获取响应结果并解析
				if (!response.isSuccessful()) {
					System.out.println(response);
					return;
				}

				Headers responseHeaders = response.headers();
				for (int i = 0, size = responseHeaders.size(); i < size; i++) {
					System.out.println(responseHeaders.name(i) + ": "
							+ responseHeaders.value(i));
				}

				System.out.println(response.body().string());
			}

			public void onFailure(Call call, IOException e) {
				System.out.println(e);
			}

		});
	}

```

此时我们使用了`call.enqueue()`方法，并通过`Callback`对象监听回调，其中`Callback`对象中包含了两个回调方法，成功`onResponse`和失败`onFailure`。

肯定会有人说，我擦~~`onResponse()`中为什么还要判断`response.isSuccessful()`，不是已经成功了吗。

在这里这两个方法的含义和其他框架的有区别。


在这贴上该方法的注释就明白了
```java 
	
	/**
   * Called when the request could not be executed due to cancellation, a connectivity problem or
   * timeout. Because networks can fail during an exchange, it is possible that the remote server
   * accepted the request before the failure.
   */
  void onFailure(Call call, IOException e);

  /**
   * Called when the HTTP response was successfully returned by the remote server. The callback may
   * proceed to read the response body with {@link Response#body}. The response is still live until
   * its response body is {@linkplain ResponseBody closed}. The recipient of the callback may
   * consume the response body on another thread.
   *
   * <p>Note that transport-layer success (receiving a HTTP response code, headers and body) does
   * not necessarily indicate application-layer success: {@code response} may still indicate an
   * unhappy HTTP response code like 404 or 500.
   */
  void onResponse(Call call, Response response) throws IOException;

```

有能力的自己翻译，我是这样理解的。

- `onResponse()` ：连接上了服务器，不管服务器是否有资源（404）或者服务器出bug了（500）。
- `onFailure()`: 没有连接上服务器。（域名不存在，无网络。。。）



### **设置和获取响应行参数（Header）**

HTTP 协议中，无论对请求和响应，都有相应的头参数。一些特别的参数能够完成一些特别的任务。类似缓存，传输编码，重定向等等。


我们知道头信息中的参数类似于`Map<String,String>`，键值对的形式，但是有些特殊的头，不是一对一的，而是一对多。那么此时`okHttp`肯定要有特别的方法获取这些信息。

```java 
public void accessHeader() throws Exception {
		final OkHttpClient client = new OkHttpClient();
		
		Request request = new Request.Builder()
				.url("https://api.github.com/repos/square/okhttp/issues")
				// 只有一个参数的头  ，一对一，如果在添加会覆盖
				.header("User-Agent", "OkHttp Headers.java")
				// 一对多，添加的关系
				.addHeader("Accept", "application/json; q=0.5")
				.addHeader("Accept", "application/vnd.github.v3+json").build();

		Response response = client.newCall(request).execute();
		if (!response.isSuccessful())
			throw new IOException("Unexpected code " + response);

		System.out.println("Server: " + response.header("Server"));
		System.out.println("Date: " + response.header("Date"));
		// 一对多的获取
		System.out.println("Vary: " + response.headers("Vary"));
	}

```

添加请求头信息时分为两种情况：
- 一对一： 使用`header(String key,String value)`方法，相同的key值调用此方法，会覆盖之前的value。
- 一对多： 使用`addHeader(String key,String value)`方法，相同的key值调用此方法，执行追加操作。


获取响应头信息时：

- 一对一： 使用`header(String key)`方法。
- 一对多： 使用`headers(String key)`方法。



### **POST 请求，传入一个字符串**

Post请求，在构造`Request`时，传入`POST`的参数即可。


```java
	public static void postString() throws Exception {

		// post 传参类型 编码
		final MediaType MEDIA_TYPE_MARKDOWN = MediaType
				.parse("text/x-markdown; charset=utf-8");
		// post 参数值
		String params = "### 三级标题";
		
		final OkHttpClient client = new OkHttpClient();
		Request request = new Request.Builder()
				.url("https://api.github.com/markdown/raw")
				.post(RequestBody.create(MEDIA_TYPE_MARKDOWN, params)).build();

		Response response = client.newCall(request).execute();
		if (!response.isSuccessful())
			throw new IOException("Unexpected code " + response);

		System.out.println(response.body().string());

	}

 ```
在`Request.post`中传入的是`RequestBody`对象，该对象的创建需要两个参数

- `MediaType contentType`：传入参数的类型。通常使用的是`text/html; chareset=utf-8`。明白此值的含义，其实就是理解HTTP 协议中`content-type`的作用：决定浏览器/服务器将以什么形式、什么编码读取这个文件。
- `String content`：post发送的内容。


在此，参数类型使用的是`text/x-markdown; charset=utf-8`，因为是调用github 的markdown的解析服务。所以传入了特定的类型。

返回结果

```java
<h3>
<a id="user-content-三级标题" class="anchor" href="#%E4%B8%89%E7%BA%A7%E6%A0%87%E9%A2%98" aria-hidden="true"><span aria-hidden="true" class="octicon octicon-link"></span></a>三级标题</h3>
```

可见已经解析成了html 语言。



### **POST 文件**

传入文件就是在`RequestBody.creat()`中，第二个参数传入一个`file`对象。

```java 
 /**
     * 传入文件
     */
    public static void postFile() throws Exception{
        final MediaType MEDIA_TYPE_MARKDOWN = MediaType
                .parse("text/x-markdown; charset=utf-8");

        final OkHttpClient client = new OkHttpClient();

        Request request = new Request.Builder()
                .url("https://api.github.com/markdown/raw")
                .post(RequestBody.create(MEDIA_TYPE_MARKDOWN,new File("README.MD")))
                .build();

        // ....
    }

```

看看`create`方法，对于文件干了什么。

```java 
	new RequestBody() {
      @Override public MediaType contentType() {
        return contentType;
      }

      @Override public long contentLength() {
        return file.length();
      }

      @Override public void writeTo(BufferedSink sink) throws IOException {
        Source source = null;
        try {
			// ---- 读取文件
          source = Okio.source(file);
          sink.writeAll(source);
        } finally {
          Util.closeQuietly(source);
        }
      }
    };

```

该方法中，根据传入的参数创建了一个`RequestBody`对象，其中`writeTo`中读取了文件，`source = Okio.source(file);`是`square`封装的io操作库。继续深入

```java 
public static Source source(File file) throws FileNotFoundException {
    if (file == null) throw new IllegalArgumentException("file == null");
    // --- 字节输入流，读取文件
	return source(new FileInputStream(file));
  }
```

看到这里，就明白了，其实就是通过字节输入流，去读取文件。中间有加了基层流的转换，暂时不管。



### **POST form 表单（日常使用）**

如上的例子中，使用`RequestBody.create()`方法作为post的参数，很是麻烦。如果对于日常使用的post请求，需要些很多代码。那么理所当然的会有其子类替我们封装了这些方法。

`FormBody`为`ReqeustBody`的子类，主要用于常见的表单操作。

```java 
	/**
     * post form 表单
     */
    public static void postForm(){

        RequestBody formBody = new FormBody.Builder()
                .add("name","1234")
                .build();

        final OkHttpClient client = new OkHttpClient();

        Request request = new Request.Builder()
                .url("https://api.github.com/markdown/raw")
                .post(formBody)
                .build();

        // .....
    }

```

通过`add(String key,String value)`方法添加参数即可。


### **POST 实现复杂的form表单提交（上传文件）**

在开发，往往会有上传用户头像等的需求，即复杂的表单。此时可以使用`RequestBody`的子类`MultipartBody`实现。

```java 
 /**
     * 提交复杂的form 表单 附带文件
     *
     *  MultiBody
     */
    public static void postMultipart(){

        /**
         * 上传的文件类型
         */
        final MediaType MEDIA_TYPE_PNG = MediaType.parse("image/png");

        /**
         * 参数
         */
        RequestBody requestBody = new MultipartBody.Builder()
                .setType(MultipartBody.FORM)
                .addFormDataPart("title","标题")
                .addFormDataPart("image","logo.png",RequestBody.create(MEDIA_TYPE_PNG,new File("xx.png")))
                .build();

        Request request = new Request.Builder()
                .url("xxx")
                .post(requestBody)
                .build();

        // ......
    }

```



### **Cache 缓存的实现**

`OkHttpClient`可以实现缓存，但如果我们不做任何配置（设置缓存的目录），则是没有缓存的。

看下面的例子

```java 
		OkHttpClient client = new OkHttpClient.Builder().build();

		Request request = new Request.Builder().url(
				"http://xxxx/login.txt")
				//.cacheControl(new CacheControl.Builder().noCache().build())
				.build();

		Response response1 = client.newCall(request).execute();
		
		String response1Body = response1.body().string();
		//　响应的结果信息
		System.out.println("Response 1 response:          " + response1);
		
		// 缓存的响应信息
		System.out.println("Response 1 cache response:    "
				+ response1.cacheResponse());
		// 网络请求的响应信息
		System.out.println("Response 1 network response:  "
				+ response1.networkResponse());

		// ************第二次请求******************	
		Response response2 = client.newCall(request).execute();
		
		String response2Body = response2.body().string();
		System.out.println("Response 2 response:          " + response2);
		System.out.println("Response 2 cache response:    "
				+ response2.cacheResponse());
		System.out.println("Response 2 network response:  "
				+ response2.networkResponse());

		System.out.println("Response 2 equals Response 1? "
				+ response1Body.equals(response2Body));

```


进行了两次请求，在请求之后，分别打印三个结果
- `response`：最终的响应
- `cacheResponse`： 缓存的响应，如果是从缓冲中取的，则该值不为null。
- `networkResponse`：网络请求的响应，如果是从网络中请求的，则该值不为null。

```java 
Response 1 response:          Response{protocol=http/1.1, code=200, message=OK, url=http://xxx/login.txt}
Response 1 cache response:    null
Response 1 network response:  Response{protocol=http/1.1, code=200, message=OK, url=http://xxx/login.txt}
Response 2 response:          Response{protocol=http/1.1, code=200, message=OK, url=http://xxx/login.txt}
Response 2 cache response:    null
Response 2 network response:  Response{protocol=http/1.1, code=200, message=OK, url=http://xxx/login.txt}
Response 2 equals Response 1? true
```

会发现`cache response`一直为null，而`network response` 一直不为null。可见使用了缓存。


修改代码，添加缓存

```java 
		// 缓存的大小
		int cacheSize = 10 * 1024 * 1024; // 10 MiB
		// 缓存对象，第一个参数为缓存的目录
		Cache cache = new Cache(new File("cache"), cacheSize);
		// 设置缓存
		OkHttpClient client = new OkHttpClient.Builder().cache(cache).build();

		//.....
```

通过构造`Cache`对象并对`OkHttpClient`设置。

```java 
Response 1 response:          Response{protocol=http/1.1, code=200, message=OK, url=http://xxx/login.txt}
Response 1 cache response:    null
Response 1 network response:  Response{protocol=http/1.1, code=200, message=OK, url=http://xxx/login.txt}
Response 2 response:          Response{protocol=http/1.1, code=200, message=OK, url=http://xxx/login.txt}
Response 2 cache response:    Response{protocol=http/1.1, code=200, message=OK, url=http://xxx/login.txt}
Response 2 network response:  null
Response 2 equals Response 1? true
```

第一次请求时，缓存为null，网络请求不为null。因为初次加载，并没有缓存。

第二次时，网络请求为null。缓存不为null。直接从之前保存的缓存中获取值。


当然。此时是对所有的`request`设置是否缓存，同时我们可以对每一个`request`设置独属于自己的缓存。

```java 
		// 不缓存
		Request request = new Request.Builder().url(
				"http://xxx/login.txt")
				.cacheControl(new CacheControl.Builder().noCache().build())
				.build();
```

通过`cacheControl()`传入一个缓存的配置对象，

- 不使用缓存：`new CacheControl.Builder().noCache().build()`
- 设置缓存的时长：`new CacheControl.Builder().maxAge().build()`
	- `maxAge(int maxAge, TimeUnit timeUnit)`: 时间数值，单位。



### **取消网络请求**

当一个页面结束时，往往要结束网络请求，此时可以通过`Call.cancel()`取消一个网络请求。这样做有利于节省系统资源。如果结束时，当前正在请求或响应，则会抛出一个`IOException`。

也可以取消多个网络请求。

当构建请求时，使用`RequestBuilder.tag(tag)`对当前请求设置一个标签。之后通过`OkHttpClient.cancel(tag)`取消对应标签的请求。


### **超时的设置**

连接超时，写入超时，读取超时是网络访问中常见的超时。

```java 
OkHttpClient  client = new OkHttpClient.Builder()
                .connectTimeout(10, TimeUnit.SECONDS)// 连接超时 10s
                .writeTimeout(10, TimeUnit.SECONDS) // 输入超时 10s
                .readTimeout(30, TimeUnit.SECONDS) // 读取超时  30s
                .build();

```


该方式是设置全局的超时规则，同样我们也可以对于不同的情况设置不同的超时选项。

关键点： 通过`OkHttpClient.newBuilder()`构造一个client的复制体。注意，此复制类似于从client 扣下来一块设置不同的规则，同时复制体上设置的规则不会影响原`client`。


```java 

    /**
     * 设置不同的超时时间
     */
    public static void singleTimeOut() throws Exception {

        OkHttpClient client = new OkHttpClient();

        Request request = new Request.Builder().url("http://xxxx").build();

        try {
            // 从client 中抠出一小块，用于单独的设置
            // 设置读取时间为1ms,肯定会超时
            OkHttpClient copy = client.newBuilder()
                    .readTimeout(1, TimeUnit.MILLISECONDS).build();

            Response response = copy.newCall(request).execute();
            System.out.println("Response 1 succeeded: " + response);
        } catch (IOException e) {
            System.out.println("Response 1 failed: " + e);
        }

        try {
            // 从client 中抠出一小块，用于单独的设置
            // 设置读取时间为3000ms
            OkHttpClient copy = client.newBuilder()
                    .readTimeout(3000, TimeUnit.MILLISECONDS).build();

            Response response = copy.newCall(request).execute();
            System.out.println("Response 2 succeeded: " + response);
        } catch (IOException e) {
            System.out.println("Response 2 failed: " + e);
        }

    }

```

结果

```java 
Response 1 failed: java.net.SocketTimeoutException: Read timed out

Response 2 succeeded: Response{protocol=http/1.1, code=200, message=OK, url=http://xxx/}

```

### **Interceptor(拦截器)**

`Interceptor`：监听网络的请求与变化。

实现方式：

-  自定义类实现`Interceptor`接口
-  `OkHttpClient.Builder().addInterceptor(自定义Interceptor).build()`;


作用：

- 打印网络请求的请求和响应的信息。
- 对网络请求做统一处理，列入添加共同的头信息。


**打印网络请求的信息**

为了方便，使用`Eclipse`进行编写。


```java 
static class LoggingInterceptor implements Interceptor {

		public Response intercept(Chain chain) throws IOException {
			// 固定写法
			Request request = chain.request();
			// 固定写法
			Response response = chain.proceed(request);
			
			// --------Log 信息---在android 中改成Log.i()即可
			System.out.println("=====请求========");

			// 请求行
			System.out.println(request);
			// 请求头信息
			System.out.println(request.headers());

			// form 表单信息。注意，此时只对form 表单适用
			FormBody form = (FormBody) request.body();
			for (int i = 0; i < form.size(); i++) {
				System.out.println(form.encodedName(i) + "="
						+ form.encodedValue(i));
			}

			System.out.println("=====响应========");
			// 响应行
			System.out.println(response);
			// 响应头
			System.out.println(response.headers());
			// 响应体
			System.out.println(response.body().string());
			return response;
		}

	}


```

添加`Interceptor`

```java 
		final OkHttpClient client = new OkHttpClient.Builder()
				// 添加Interceptor
				.addNetworkInterceptor(new LoggingInterceptor()).build();
		Request request = new Request.Builder().url(BAIDU_MP3_PATH)
				.post(new FormBody.Builder().add("name", "123").build())
				.build();
		....

```

注意，使用的POST 表单提交。

看一下结果

```java 
=====请求========
Request{method=POST, url=http://tingapi.ting.baidu.com/v1/restserver/ting?from=qianqian&version=2.1.0&method=baidu.ting.billboard.billList&format=json&type=1&offset=0&size=1, tag=Request{method=POST, url=http://tingapi.ting.baidu.com/v1/restserver/ting?from=qianqian&version=2.1.0&method=baidu.ting.billboard.billList&format=json&type=1&offset=0&size=1, tag=null}}
Content-Type: application/x-www-form-urlencoded
Content-Length: 8
Host: tingapi.ting.baidu.com
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.3.1

name=123
=====响应========
Response{protocol=http/1.1, code=200, message=OK, url=http://tingapi.ting.baidu.com/v1/restserver/ting?from=qianqian&version=2.1.0&method=baidu.ting.billboard.billList&format=json&type=1&offset=0&size=1}
Date: Wed, 29 Jun 2016 08:22:28 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
X-LIGHTTPD-LOGID: 624562847
RT: 624562847_d41d8cd98f00b204e9800998ecf8427e_d41d8cd98f00b204e9800998ecf8427e
Server: Apache1.0
SC: MISS.0.0
tracecode: 13487736192701306048062916
Set-Cookie: BAIDUID=8184D1B9CC4C45EB85C3AB62E8441688:FG=1; expires=Thu, 29-Jun-17 08:22:28 GMT; max-age=31536000; path=/; domain=.baidu.com; version=1
P3P: CP=" OTI DSP COR IVA OUR IND COM "

{"song_list":[{"artist_id":"247010469","language":"\u56fd\u8bed","pic_big":"http:\/\/musicdata.baidu.com\/data2\/pic\/0c87cb20fd6b543cd1c90d1d82f1f6c5\/266782982\/266782982.jpg","pic_small":"http:\/\/musicdata.baidu.com\/data2\/pic\/169cbb0d82522568dea198363f7028ec\/266782985\/266782985.jpg","country":"\u5185\u5730","area":"0","publishtime":"2016-06-20","album_no":"1","lrclink":"http:\/\/musicdata.baidu.com\/data2\/lrc\/1f75306d7f1e25f723d3d175f5ef3d46\/266791084\/266791084.lrc","copy_type":"1","hot":"276927","all_artist_ting_uid":"232954914","resource_type":"0","is_new":"1","rank_change":"0","rank":"1","all_artist_id":"247010469","style":"\u6d41\u884c","del_status":"0","relate_status":"0","toneid":"0","all_rate":"64,128,256","sound_effect":"0","file_duration":0,"has_mv_mobile":1,"versions":"","bitrate_fee":"{\"0\":\"0|0\",\"1\":\"-1|-1\"}","song_id":"266784254","title":"\u7ea2","ting_uid":"232954914","author":"\u51af\u5efa\u5b87","album_id":"266784256","album_title":"\u7ea2","is_first_publish":0,"havehigh":0,"charge":0,"has_mv":1,"learn":0,"song_source":"web","piao_id":"0","korean_bb_song":"1","resource_type_ext":"1","mv_provider":"0100000000","artist_name":"\u51af\u5efa\u5b87"}],"billboard":{"billboard_type":"1","billboard_no":"1871","update_date":"2016-06-28","billboard_songnum":"190","havemore":1,"name":"\u65b0\u6b4c\u699c","comment":"\u8be5\u699c\u5355\u662f\u6839\u636e\u767e\u5ea6\u97f3\u4e50\u5e73\u53f0\u6b4c\u66f2\u6bcf\u65e5\u64ad\u653e\u91cf\u81ea\u52a8\u751f\u6210\u7684\u6570\u636e\u699c\u5355\uff0c\u7edf\u8ba1\u8303\u56f4\u4e3a\u8fd1\u671f\u53d1\u884c\u7684\u6b4c\u66f2\uff0c\u6bcf\u65e5\u66f4\u65b0\u4e00\u6b21","pic_s640":"http:\/\/c.hiphotos.baidu.com\/ting\/pic\/item\/f7246b600c33874495c4d089530fd9f9d62aa0c6.jpg","pic_s444":"http:\/\/d.hiphotos.baidu.com\/ting\/pic\/item\/78310a55b319ebc4845c84eb8026cffc1e17169f.jpg","pic_s260":"http:\/\/b.hiphotos.baidu.com\/ting\/pic\/item\/e850352ac65c1038cb0f3cb0b0119313b07e894b.jpg","pic_s210":"http:\/\/business.cdn.qianqian.com\/qianqian\/pic\/bos_client_c49310115801d43d42a98fdc357f6057.jpg","web_url":"http:\/\/music.baidu.com\/top\/new"},"error_code":22000}


```

打印成功。。。。

当然，该实现比较粗糙，仅作原理的解释


> 通过上面的实现，可以看到在`LoggingInterceptor`中可以获取到`request`对象，那么我们可以通过设置其`header()`，实现全局的`request`头部信息的修改。



该博客中的源码已经共享到[https://github.com/AlexSmille/alex_mahao_sample/tree/master/networklib](https://github.com/AlexSmille/alex_mahao_sample/tree/master/networklib)，有需要请移步。

