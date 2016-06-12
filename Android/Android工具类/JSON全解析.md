
## JSON 全解析

### 什么是JSON

 - JSON 指的是 JavaScript 对象表示法（JavaScript Object Notation）
 - JSON 是轻量级的文本数据交换格式
 - JSON 独立于语言 （单纯的数据格式，不受语言的约束）
 - JSON 具有自我描述性，更易理解

对于JSON的定义以及数据格式，没有什么太多的难点，这里为官网对JSON的定义。从官网描述中可以看出，JSON本身是JavaScript中对象的描述格式，后来得以推广并逐渐取代xml。

#### JSON和XML的比较

相比 XML 的不同之处

- 没有结束标签（类似于键值对的形式）
- 更短（没有结束标签，当然短了）
- 读写的速度更快
- 能够使用内建的 JavaScript eval() 方法进行解析
- 使用数组
- 不使用保留字


### 原生JSON解析

`Android`原生的解析实际上使用的`JSON`的一个官方jar包。对于`JSON`,不需要页面展示，所以使用`intellij idea`进行演示。

在使用之前我们需要下载`org.json`的jar包。对于Android 开发环境不需要下载此jar包。因为Android SDK 中已经默认包含了该jar包。

[json jar 下载地址](https://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22org.json%22%20AND%20a%3A%22json%22)

下载完之后导入即可。

#### JSONObject对象解析

下面看一下数据

```xml 
{
	"user":{
		"name":"alex",
		"age":"18",
		"isMan":true
	}
}

```

有一个user字段，其中包含了该user的一些基本属性。那么如何解析呢？


在解析时，有一个很关键的地方：如果是`{}`包含，则为`JSONObject`对象，如果为`[]`则为`JSONArray`对象。

看到上面的例子，我们看到整个数据为`JSONObject`,其内部包含了一个`user`字段，该字段的值也是一个`JSONObject`对象。

```java 
public class OrgJSONTest {

    public static String json = "{\"user\":{\"name\":\"alex\",\"age\":\"18\",\"isMan\":true}}";


    public static void main(String[] args){
        JSONObject obj = new JSONObject(json);//最外层的JSONObject对象
        JSONObject user = obj.getJSONObject("user");//通过user字段获取其所包含的JSONObject对象
        String name = user.getString("name");//通过name字段获取其所包含的字符串

        System.out.println(name);


    }
}
```

打印结果如下
```java 

alex

```

可以看到获取到了相应的值。

在`JSONObject`对象中封装了`getXXX()`等一系列方法。用以获取字符串，整形等等一系列的值。


对于如上例子，完全解析user对象如下
```java
 		String name = user.getString("name");//通过name字段获取其所包含的字符串
        String age = user.getString("age");
        boolean isMan = user.getBoolean("isMan");

        System.out.println("name:"+name+"\nage:"+age+"\nisMan:"+isMan);
```

结果如下
```java 
name:alex
age:18
isMan:true

```


这种通过`getXXX`的方式，无疑会出现一些问题，我们开始一一尝试。

##### getXXX方法获取的类型不符

- 字符串类型转整形

对于上面的例子，我们可以看到`age`字段虽然其对应的值是双引号括起的字符串，但其实际上是一个整形，那么我们是否能够通过`getInt`获取整形呢。

```java 
	int age = user.getInt("age");
```

```java 
	age:18
```

当然是可以得，同时字符串类型可以转化为布尔类型，整数类型，浮点型等等。但字符串的内容必须符合规范，否则会报异常。如果看其源码可知，其内部实质是调用了对应对象的`parseXXX（）`方法进行转化操作。

```java 
	//getInt源码
  public int getInt(String key) throws JSONException {
        Object object = this.get(key);

        try {
			//关键点，如果是数值类型，则调用intValue（），否则强转成字符串之后调用parserInt方法（）
            return object instanceof Number?((Num，ber)object).intValue():Integer.parseInt((String)object);
        } catch (Exception var4) {
            throw new JSONException("JSONObject[" + quote(key) + "] is not an int.");
        }
    }

```

- 整形等转字符串类型

按照如上的思维逻辑，直接`getString("xxx")`就可以了。但事实正好相反，该方法，如果对应值不是双引号括起的，则会抛出异常。

```java 

 //getString 源码
 public String getString(String key) throws JSONException {
        Object object = this.get(key);
		//直接判断是否是字符串类型，如果不是，则抛出异常
        if(object instanceof String) {
            return (String)object;
        } else {
            throw new JSONException("JSONObject[" + quote(key) + "] not a string.");
        }
    }

```

##### getXXX("") 没有对应的键值	

通过上面的例子，可以得知`getXXX("")`方法是通过字段（键）获取对应的值。那么肯定存在一种情况，及没有键的存在。

```java 
  System.out.println(user.getString("sex"));

```

```java 
Exception in thread "main" org.json.JSONException: JSONObject["sex"] not found.
	at org.json.JSONObject.get(JSONObject.java:471)
	at org.json.JSONObject.getString(JSONObject.java:717)
	at json.OrgJSONTest.main(OrgJSONTest.java:24)
```

果断报异常。

那么怎么办呢：使用optXXX()。后面会讲。

#### JSONArray 解析

该对象用以解析`[]`的对象，及数组对象。用法类似，修改一下数据格式
```java 
{
	"user":[
		{
			"name":"alex",
			"age":"18",
			"isMan":true
		},
		{
			"name":"alex",
			"age":"18",
			"isMan":true
		}
	]

}

```

user对应的值不再是一个`JSONObject`对象，而是一个`JSONArray`对象（数组对象）。该数组对象中包含了多个`JSONObject`。

```java 
 		JSONObject obj = new JSONObject(json);//最外层的JSONObject对象


        JSONArray array = obj.getJSONArray("user");

        for(int i = 0 ; i<array.length();i++){
            JSONObject user = array.getJSONObject(i);//索引值，获取数组中包含的值
            System.out.println(user.getString("name"));

        }
```

```java 
alex
mahao

```

解析过程与`JSONObject`类似，多了一个遍历`JSONArray`的步骤，把`JSONArray`当作JAVA中的数组对待使用。两者使用方式几乎一样。


#### 构造JSON 数据

对于POST请求，传参数时一般都是传入一个json数据。那么如何构造json数据呢，使用字符串拼接可以，但很蛋疼。当然，`JSONObject`等提供了对应的方法。

使用如下：

```java 
 		//外层obj对象
        JSONObject objWrite = new JSONObject();

        //user对象
        JSONObject userWrite = new JSONObject();

        //写入对应属性
        userWrite.put("name","alex");
        userWrite.put("age","18");
        userWrite.put("isMan",true);

        //将user对象写入到外层obj中
        objWrite.put("user",userWrite);
        
        System.out.println(objWrite);

```

```java 
{"user":{"name":"alex","isMan":true,"age":"18"}}
```

#### opt 替代 get

在上面使用中，我们通过`getXXX()`获取相应值。但是，会发现其局限性很多，很容易就抛异常，需要我们`try...catch`去捕获。而`optXXX()`对此进行了优化。


首先看一下他的用法，在看用法之前，我们回忆之前使用get时的两个问题

- 其他类型转字符串类型抛出异常
- 当需要的字段没有时，抛出异常。

看一下opt针对如上问题的解决：

```java 
 JSONObject obj = new JSONObject(json);//最外层的JSONObject对象


        JSONObject user = obj.optJSONObject("user");

        String name = user.optString("name");

        //整形转字符串
        String age = user.optString("age");

        boolean isMan = user.optBoolean("isMan");

        //默认值，如果没有该字段，则会返回默认值
        String sex = user.optString("sex","男");

        System.out.println("name:"+name+"\nage:"+age+"\nisMan:"+isMan+"\nsex:"+sex);

```


```java 
	name:alex
	age:18
	isMan:true
	sex:男
```

通过上面的例子，可以看出通过使用`optString()`可以将整形转化为字符串。而对于`sex`，因为该字段没有，会为其付默认值。解决了抛出异常的问题。

深入进去，看一下他们的源码。

- `optString()`

```java 

 	// optString   默认调用了optString(key, "");	
 	public String optString(String key) {
        return this.optString(key, "");
    }

	//如果是null，返回默认值，否则调用toString方法返回
	public String optString(String key, String defaultValue) {
        Object object = this.opt(key);
        return NULL.equals(object)?defaultValue:object.toString();
    }


```

- optBoolean

```java 
 	public boolean optBoolean(String key) {
        return this.optBoolean(key, false);
    }

	//实质调用get方法，如果抛出异常，则赋默认值
    public boolean optBoolean(String key, boolean defaultValue) {
        try {
            return this.getBoolean(key);
        } catch (Exception var4) {
            return defaultValue;
        }
    }
```


- get()取值不正确会抛出异常，必须用try catch或者throw包起

- opt()取值不正确则会试图进行转化或者输出友好值，不会抛出异常




> 如上，介绍了`Android`原生的解析，但在实际开发中，为了提高效率，往往使用第三方解析类库，而下面我们将进入到第三方类库的使用。



### GSON 解析

Gson解析是google 提供的快速json解析库。其和原生的相比，最大的优点是可以按照`bean`类对数据进行解析。

因为其属于第三方库，所以我们需要导包，如果使用的是`AndroidStudio`，直接搜索`gson`导入即可。

使用的是`eclipse`的话，则[在该地址下载](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.google.code.gson%22)，将jar包加入到工程中。



#### Gson 解析单一实体类对象

还是上面的数据格式

```java 
{
	"user": {
		"name": "alex",
		"age": "18",
		"isMan": true
	}
}
```

对于该数据，我们想要获取的是`user`中的字段值，首先定义`User`实体类，注意：该类名没有要求，随便命名。

```java 

public class User {

    private String name;

    private int age;

    private boolean isMan;

    private String sex;

}
```

有两点需要注意：

- 类中成员变量名一定要和json数据中字段一一对应。
- 多添加了sex字段，测试当获取的字段不存在时的情况。

```java 
 //使用Gson解析实体类对象

        //1， 获取对应实体类对象的字符串，当前为user的值。

        String userJson = new JSONObject(json).getJSONObject("user").toString();

		//userJson = "{\"name\":\"alex\",\"age\":18,\"isMan\":true}";

        //2 , 创建Gson 对象

        Gson gson = new Gson();

        // 3， fromJson 解析
        User user = gson.fromJson(userJson, User.class);


        System.out.println(user);

```

```java 
User{name='alex', age=18, isMan=true, sex='null'}
```
关键方法为`fromJson`，该方法第一个参数为需要解析的`json`数据，第二个参数为需要解析成的目标对象的class。

因为该例子比较特殊，当前数据中`user`中的值不是最外层的json数据，他被包裹起来了，所以我们先把他剥开，获取`user`对应的值。


有时候，返回的数据中字段值和我们前端的命名习惯不同，导致字段无法一一对应。如果我们的数据修改成了如下样式

```java 
{
	"user": {
		"name": "alex",
		"age": "18",
		"is_man": true
	}
}
```

字段值得到了修改，`isMan`-》`is_man`,如果我们实体类也改成该字段名，那么就不符合命名规范了。

在Gson中有如下注解`@SerializedName`

```java 
 	@SerializedName("is_man")
    private boolean isMan;

``` 
这样就解决了问题。

还有一种情况，如果对于该字段，有的地方使用`is_man`,有的地方使用`is_Man`,甚至还有`isMan`。那么我们因为三个字段名的不同建立三个实体类，很是蛋疼。

```java 
  @SerializedName(value = "is_man",alternate = {"is_Man","isMan"})
    private boolean isMan;
```

当然，如果这三个字段同时出现时，会取最后一次出现的对应值进行赋值。


#### Gson 解析数组型实体类对象

对于例子，我们修改一下

```java 
{
	"user": [
		{
			"name": "alex",
			"age": "18",
			"is_man": true
		},
		{
			"name": "mahao",
			"age": "16",
			"is_man": true
		}
	]
}

```

`user`中所对应的值不再是一个`JSONObject`,而是`JSONArray`类型。

```java 
 public static void parserArray(){
        String json = "{\"user\":[{\"name\":\"alex\",\"age\":18,\"isMan\":true},{\"name\":\"mahao\",\"age\":16,\"isMan\":true}]}";

        //1， 获取对应实体类对象的字符串，当前为user的值。

        String userJson = new JSONObject(json).getJSONArray("user").toString();

        //2 ,创建Gson 对象

        Gson gson = new Gson();

        //3, 获取user 数组
        User[] users = gson.fromJson(userJson, User[].class);

        System.out.println(users[1]);
    }

```

最终解析出了`User[]`数组。

平常我们往往使用List存储数据。

第三步改成了如下代码
```java 
 List<User> users = gson.fromJson(userJson, List<User>.class);
```
报错！！！ 因为第二个参数的字节码，实际为List.class ,User被忽略了。 **泛型擦除**

Gson为我们提供了另一个方法解决该问题

```java 
List<User> users = gson.fromJson(userJson,new TypeToken<List<User>>(){}.getType());
        System.out.println(users.get(1));
```


#### 通过Gson 构造json数据

- 根据实体类对象生成json 数据。

```java 
   public static void writeBeanJson(){

        //1 构造gson 对象
        Gson gson = new Gson();


        //2 构造对象

        User user = new User();

        user.setName("lala");

        user.setAge(20);

        // 3 生成json 数据

        String json = gson.toJson(user);

        System.out.println(json);
    }
```

```java 
	{"name":"lala","age":20,"is_man":false}
```

如果实体类中为null或空字符串，则该字段不会被转化。当然也可以通过GsonBuilder设置，后面会提到。


- 自定义json数据。

对于POST请求，需要的json数据自定义比较大。例如登录时，需要传入账号和密码。

```java 
 private static void writeJson() {
        //1 构造gson 对象
        Gson gson = new Gson();

        //2 ,构建map对象
        Map<String,Object> map = new HashMap<String,Object>();
        map.put("username","haha");
        map.put("password",123456);

        //3 生成json数据
        String json = gson.toJson(map);

        System.out.println(json);

    }
```


#### Gson使用扩展

`GsonBuilder` ,通过该类初始化一些`Gson`的基本属性

```java 
Gson gson = new GsonBuilder()
        //序列化null
        .serializeNulls()
        // 设置日期时间格式，另有2个重载方法
        // 在序列化和反序化时均生效
        .setDateFormat("yyyy-MM-dd")
        // 禁此序列化内部类
        .disableInnerClassSerialization()
        //生成不可执行的Json（多了 )]}' 这4个字符）
        .generateNonExecutableJson()
        //禁止转义html标签
        .disableHtmlEscaping()
        //格式化输出
        .setPrettyPrinting()
        .create();

```



### Gson的封装

```java 
/**
 * gson 的基本封装，完成Gson 解析中常用的功能
 * 备注人： Alex_MaHao
 * @date 创建时间：2016年6月6日 下午4:33:25
 */
public class GsonUtil {
	
	private static Gson gson = null;
	
	static {
		if (gson == null) {
			gson = new Gson();
		}
	}

	private GsonUtil() {
		
	}

	/**
	 * 
	 * 对象转化为json 数据
	 * @param object 需要转化的对象
	 * @return
	 */
	public static String GsonString(Object object) {
		String gsonString = null;
		if (gson != null) {
			gsonString = gson.toJson(object);
		}
		return gsonString;
	}

	/**
	 * json 数据转化为实体类对象
	 * 
	 * @param gsonString
	 * @param cls
	 * @return
	 */
	public static <T> T GsonToBean(String gsonString, Class<T> cls) {
		T t = null;
		if (gson != null) {
			t = gson.fromJson(gsonString, cls);
		}
		return t;
	}

	/**
	 * 
	 * Json 数据转化为List集合--集合中为实体类
	 * 
	 * @param gsonString
	 * @param cls
	 * @return
	 */
	public static <T> List<T> GsonToList(String gsonString, Class<T> cls) {
		ArrayList<T> mList = new ArrayList<T>();
		JsonArray array = new JsonParser().parse(gsonString).getAsJsonArray();
		for (final JsonElement elem : array) {
			mList.add(gson.fromJson(elem, cls));
		}
		return mList;

	}

	/**
	 * 将数据转化成List集合--集合中为map
	 * 
	 * @param gsonString
	 * @return
	 */
	public static <T> List<Map<String, T>> GsonToListMaps(String gsonString) {
		List<Map<String, T>> list = null;
		if (gson != null) {
			list = gson.fromJson(gsonString,
					new TypeToken<List<Map<String, T>>>() {
					}.getType());
		}
		return list;
	}

	/**
	 * 将json数据转化成map
	 * 
	 * @param gsonString
	 * @return
	 */
	public static <T> Map<String, T> GsonToMaps(String gsonString) {
		Map<String, T> map = null;
		if (gson != null) {
			map = gson.fromJson(gsonString, new TypeToken<Map<String, T>>() {
			}.getType());
		}
		return map;
	}
}

```

### fastJson

fastJson 是阿里巴巴出的第三方库。该库的使用非常的方便而且强大。但是用的并不多，大家都是用Gson。虽然不知道原因是为什么。

使用fastJson有一个很大的缺陷，就是其命名与Android 原生的解析命名相同，所以在使用中很容易混淆。

#### 解析对象

```java 
   public static void  parserObject(){

        String json = "{\"user\":{\"name\":\"alex\",\"age\":18,\"isMan\":true}}";

        //1， 获取对应实体类对象的字符串，当前为user的值。 该JSONObject 为org包中的。。
        String userJson = new JSONObject(json).getJSONObject("user").toString();

        //2 调用JSON.parserObject 解析
        User user = JSON.parseObject(userJson, User.class);

        System.out.print(user);
    }
```

```java 
User{name='alex', age=18, isMan=true, sex='null'}
```

#### 解析数组

```java 
 public static void parserArray(){
        String json = "{\"user\":[{\"name\":\"alex\",\"age\":18,\"isMan\":true},{\"name\":\"mahao\",\"age\":16,\"isMan\":true}]}";

        //1， 获取对应实体类对象的字符串，当前为user的值。

        String userJson = new JSONObject(json).getJSONArray("user").toString();

        //2 调用JSON.parseArray 解析
        List<User> users = JSON.parseArray(userJson, User.class);

        System.out.println(users.get(1));
        
    }

```

```java 
User{name='mahao', age=16, isMan=true, sex='null'}
```

#### 构造json 数据

``` java
 public static void wirteJson(){

        Map<String,Object> map = new HashMap<String,Object>();
        map.put("username","haha");
        map.put("password",123456);

        String json = JSON.toJSONString(map);

        System.out.println(json);
    }
```

```java 
{"password":123456,"username":"haha"}

```

> 通过上面的例子看出，使用fastJson 更加的简单，方便。主要通过`JSON.parseArray`,`JSON.paseObject`,`JSON.toJSONString`进行数据的解析和生成。


### GsonFormat

在上面的例子中，使用`Gson`，`fastJson`解析数据时，关键的便是实体类。 但有时候实体类非常复杂时，我们比着敲时很容易出错。


在这里使用`GsonFormat`插件，该插件为Android Studio的插件。

File -》 setting -》plugins ->  搜索`GsonFormat`，导入即可。

使用方式如下图：


点击Alt+insert

![](Gsonformat.gif)

