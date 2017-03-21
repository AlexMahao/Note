## 手动实现IOC框架，与findViewById说拜拜



### 自序

在`Android`开发中，总要写许多的`findViewById`方法，这无疑是一件非常痛苦的事情。直到接触了`Xutils`框架，发现竟然可以使用注解的方式，优雅的干掉了`findViewById`方法，当时真是惊为天人。后来遇到了`Butterknife`之后，发现不仅能够实现`findViewById`方法，甚至连`setOnClickListener`，`getString()`，`getResource()`方法，都能通过一行注解的方式快速的实现。

当时在使用过程中，感觉这种方式大大的提高了开发的效率，以及编码的舒畅度，自己很有必要实现以下。于是便有了这篇博客，该篇博客主要实现了`findViewById`方法，虽然广度不是很大，但他们的原理都是相同的。



### 如何使用

使用方式和`Butterknife`相似，通过注解`@BindView`标识控件，通过`ViewFinder.inject(this)`实现代码的注入。

```java 
public class MainActivity extends AppCompatActivity {
    // 通过注解绑定控件
    @BindView(R.id.text)
    TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //代码注入
        ViewFinder.inject(this);
        textView.setText("123");
    }
}


```


### 实现原理

实现方式有两种

**通过反射实现**：如果对于反射有深入了解，则应该清楚，我们可以通过反射获取到该注解，并且获取到该注解的值等等一系列的必须量，通过反射我们实现对控件的注入。该方法虽然可以实现，但对效率有着一定的影响。毕竟反射很影响效率。

**通过编译期生成代码实现**：在我们运行java代码的时候，通常先通过`javac`将java文件编译成`.class`文件，然后运行`class`文件，那么我们能不能够在编译时期，根据我们的注解生成`findViewById`等方法，这样我们就能够在运行期查找控件。结果当然是可以的，本例就是使用这种方式。因为虽然其使用的是注解，但在运行期其实质仍是通过`findViewById`方法查找控件，相比于反射来说，大大的提高了性能。

通过编译器生成代码的大致流程如下：

- 编写`Modul`：`ioc-annotation`,该工程主要定义注解`@BindView`用以修饰变量。
- 编写`Modul`:`ioc-compiler`，该工程为最终会打成jar包，主要是在`javac`编译时期根据注解生成注入代码的相关类
- 编写`Modul`：`ioc-api`,该工程主要提供注入的调用方法`ViewFidder.inject()`，调用代码注入的方法。
- 编写`Modul`：`app`,测试工程。


### 代码实现

根据上面的流程，开始实现框架

### **编写`ioc-annotation`模块**

该模块比较简单，就是定义一个注解。如下：

```java 
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface BindView {
    int value();
}

```

该注解主要有两个功能：

- 在`Activity`中修饰变量，用以标识需要`findViewById`的相关控件。
- 在`ioc-compiler`模块中，用以检索和获取需要`findViewById`的控件。

#### **编写`ioc-compiler`模块**

在之前我们提到过，该框架的原理是在编译时期根据我们的要求生成注入的辅助代码，那么如何生成，以何种规则生成，肯定是由我们来定义的。

`javac`命令中，可以在其编译的指定目录放入一个`.jar`文件，当然这个`.jar`文件有特殊的要求（后面再说），这样运行`javac`命令之前，`javac`会调用`jar`，通过这种特性订制一些我们想实现的功能。


那么看一下该模块的关键类`IocProcessor`

```java
@AutoService(Processor.class)
public class IocProcessor extends AbstractProcessor {

    // 文件相关的辅助类，生成JavaSouceCode
    private Filer mFileUtils;

    private Elements mElementUtils;
    // 日志相关
    private Messager mMessager;

    private Types mTypeUtils;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnvironment) {
        super.init(processingEnvironment);
        mTypeUtils = processingEnvironment.getTypeUtils();
        mFileUtils = processingEnvironment.getFiler();
        mElementUtils = processingEnvironment.getElementUtils();
        mMessager = processingEnv.getMessager();
    }

    /**
     * 标示该处理器捕获处理的注解类型
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotationTypes = new LinkedHashSet<>();
        // getCanonicalName 获取规范的名字
        annotationTypes.add(BindView.class.getCanonicalName());
        return annotationTypes;
    }
    /**
     *  指定使用的java版本，一般默认支持返回最新
     * @return
     */
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    private Map<String, ProxyInfo> mProxyMap = new HashMap<>();
    /**
     * 相当于main 函数，处理扫描，评估和处理注解的代码以及生成java文件
     * @param set
     * @param roundEnvironment 用于查询包含特定注解的注解元素
     * @return
     */
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
       	//.....关键处理代码
        return true;
    }
}

```

为了方便浏览，暂时删去了一些代码。

首先在类的声明上，我们添加了一个注解`@AutoService`，该注解是由`Google`提供的用于生成`.jar`包的相关注解。使用该注解需要添加依赖`com.google.auto.service:auto-service:1.0-rc3`，在这里有一个疑问，为什么要使用该注解呢，生成的`jar`包有什么特殊的地方吗？

因为该`jar`包是提供于编译时期的，所以其有特殊的目录形式：

![](ioc1.png)

除了有最基本的类以外，多了`META-INF`的`service`目录，该目录中的文件命名是固定的，其内容很简单

```java 
com.example.IocProcessor
```
就一行内容，标明主要的处理类。而如果我们手动编写`jar`的目录很麻烦，所以`Google`提供了一个`@AutoService`，用以直接实现这种目录形式的`jar`包。


看完注解之后，看类声明，继承`AbstractProcessor`,实现三个方法：

- `getSupportedAnnotationTypes`:获取注解支持的类型，在此，我们支持`BindView`，没有什么疑问
- `getSupportedSourceVersion`:指定使用的java版本，一般默认支持返回最新，默认此写法，没什么问题。
- `process`:处理注解的主要方法，主要在此方法中实现对相关类的处理。

那么看一下`process`的实现：

```java 
 private Map<String, ProxyInfo> mProxyMap = new HashMap<>();
    /**
     * 相当于main 函数，处理扫描，评估和处理注解的代码以及生成java文件
     * @param set
     * @param roundEnvironment 用于查询包含特定注解的注解元素
     * @return
     */
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        mProxyMap.clear();
        // --------------------检索注解，保存包含注解的类以及需要注入的成员变量-------------------------
        // 该方法获取的element是被注解标注的所有元素，所以需要检查
        Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(BindView.class);
        for (Element element : elements) {
            // 检查element的类型
            if (!checkAnnotationValid(element, BindView.class)) {
                return false;
            }
            VariableElement variableElement = ((VariableElement) element);
            TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();
            // 全路径名
            String qualifiedName = typeElement.getQualifiedName().toString();
            // 类信息
            ProxyInfo proxyInfo = mProxyMap.get(qualifiedName);
            if (proxyInfo == null) {
                proxyInfo = new ProxyInfo(mElementUtils, typeElement);
                mProxyMap.put(qualifiedName, proxyInfo);
            }
            // 在类信息中保存需要注入的成员变量和id
            BindView annotation = variableElement.getAnnotation(BindView.class);
            int id = annotation.value();
            proxyInfo.injectVariables.put(id, variableElement);
        }
        // ---------------- 生成对应的类----------------------------
        for (String key : mProxyMap.keySet()) {
            ProxyInfo proxyInfo = mProxyMap.get(key);
            try {
                // processingEnv ： 注解处理环境（工具类），提供很多有用的功能工具类
                JavaFileObject jfo = processingEnv.getFiler().createSourceFile(
                        proxyInfo.getProxyClassFullName(),
                        proxyInfo.getTypeElement());
                Writer writer = jfo.openWriter();
                writer.write(proxyInfo.generateJavaCode());
                writer.flush();
                writer.close();
                //----- 在生成类之后gradle会对代码进行优化
            } catch (IOException e) {
                error(proxyInfo.getTypeElement(),
                        "Unable to write injector for type %s: %s",
                        proxyInfo.getTypeElement(), e.getMessage());
            }

        }
        return true;
    }

```


自上而下分析：

- 首先获取被注解`@BindView`修饰的元素，注意此时获取的元素可能是`TypeElement`，`VariableElement`，`ExecuteableElement`等，不一定确定是成员变量，所以要做一层检查。

检查的方法如下

```java 
 /**
     * 检查查找的元素是否合法
     */
    private boolean checkAnnotationValid(Element annotatedElement, Class clazz) {
        if (annotatedElement.getKind() != ElementKind.FIELD) {
            error(annotatedElement, "%s must be declared on field.", clazz.getSimpleName());
            return false;
        }
        if (ClassValidator.isPrivate(annotatedElement)) {
            error(annotatedElement, "%s() must can not be private.", annotatedElement.getSimpleName());
            return false;
        }
        return true;
    }
    
    // 展示错误信息
    private void error(Element element, String message, Object... args) {
        if (args.length > 0) {
            message = String.format(message, args);
        }
        mMessager.printMessage(Diagnostic.Kind.NOTE, message, element);
    }

```

- 其次：获取到需要生成辅助类的类（含有注解`@BindView`的类），并保存类信息到`ProxyInfo`中，同时保存被注解的成员变量到该`ProxyInfo`中，以`Map`的形式保存，键是注解的值，`value`是被注解的元素。到这里会好奇什么是辅助类，他是什么形式的，我们看一下最终生成的辅助类就会清晰了

```java
// 依赖注入生成的辅助类，最终调用inject()方法，实现注入
public class MainActivity$$Finder implements Finder<MainActivity> {
    public MainActivity$$Finder() {
    }

    public void inject(MainActivity host, Object source, Provider provider) {
		// 依赖注入的具体实现
        host.textView = (TextView)((TextView)provider.findView(source, 2131492945));
    }
}

```

生成的辅助类很清晰，看一眼即可，此时多看会模糊，我们只要确定我们要生成如上格式的类就行了。


- 保存了类以及注解的相关信息之后，便是生成代码，循环`mProxyMap`获取相关类，生成代码，其实就是通过流写文件，关键的实现`writer.write(proxyInfo.generateJavaCode());`，其中调用了`proxyInfo.generateJavaCode()`用以生成代码，看一下实现

```java 
 // 生成java 代码
    public String generateJavaCode() {
        StringBuilder builder = new StringBuilder();
        builder.append("// Generated code. Do not modify!\n");
        builder.append("package ").append(packageName).append(";\n\n");
        builder.append("import com.mahao.ioc_api.*;\n");
        builder.append('\n');
        builder.append("public class ").append(proxyClassName).append(" implements " + ProxyInfo.PROXY + "<" + typeElement.getQualifiedName() + ">");
        builder.append(" {\n");
        // 添加inject()方法
        generateMethods(builder);
        builder.append('\n');
        builder.append("}\n");
        return builder.toString();
    }

    // 生成注入的方法
    private void generateMethods(StringBuilder builder) {
        builder.append("@Override\n ");
        builder.append("public void inject(" + typeElement.getQualifiedName() + " host, Object source, Provider provider) {\n");
        for (int id : injectVariables.keySet()) {
            VariableElement element = injectVariables.get(id);
            String name = element.getSimpleName().toString();
            String type = element.asType().toString();
            builder.append("host." + name).append(" = ");
            builder.append("(" + type + ")(provider.findView( source," + id + "));\n");
        }
        builder.append("  }\n");
    }

```

- 上面的两个方法可能有一点复杂，但逻辑不是很绕。
- 最终我们编写完了在编译期需要用到的`jar`。可以看到其主要目的是为了生成`xxx$$Finder`的辅助类的。

#### **编写`ioc-api`模块**

有了标识，也有了编译时期的`jar`包，最终也生成了`xxx$$Finder`的辅助类，那么久剩下最后的调用了。

根据我们最开始的使用方法`ViewFinder.inject()`可知，肯定要从这里看起。

```java 
public class ViewFinder {

    private static final ActivityProvider PROVIDER_ACTIVITY = new ActivityProvider();
    private static final ViewProvider PROVIDER_VIEW = new ViewProvider();

    /**
     * 从activity 注入
     * @param activity
     */
    public static void inject(Activity activity) {
        inject(activity, activity, PROVIDER_ACTIVITY);
    }
    /**
     * 从View 中查找
     * @param host
     * @param view
     */
    public static void inject(Object host, View view) {
        // for fragment
        inject(host, view, PROVIDER_VIEW);
    }
    // 注入的最终调用
    public static void inject(Object host, Object source, Provider provider) {
        Class<?> clazz = host.getClass();
        String proxyClassFullName = clazz.getName()+"$$Finder";// 根据传入的对象获取辅助类的全路径名
        Class<?> proxyClazz = null;
        try {
            proxyClazz = Class.forName(proxyClassFullName);// 获取辅助类的class
            Finder viewInjector = (Finder) proxyClazz.newInstance(); // 生成辅助类的实例
            viewInjector.inject(host, source, provider); // 调用辅助类的inject()方法
        }catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

可以看到，最终的`inject()`实现方法。因为在生成辅助类的时候，我们是根据类名+`$$finder`的固定写法，所以很容易的通过`class`的形式生成辅助类的实例化，并调用`inject`方法。在这里出现了`Finder`对象，他实际是我们定义的接口

```java 
public interface Finder<T> {
    void inject(T host, Object source, Provider provider);
}
```

辅助类在生成代码时，统一继承该接口，便于调用`inject()`方法（多态）.




注意，此时可以看一下`viewInjector.inject(host, source, provider);`方法，在看一下生成的辅助类

```java 
public class MainActivity$$Finder implements Finder<MainActivity> {
   	
	// 看这里，有没有发现什么
    public void inject(MainActivity host, Object source, Provider provider) {
        host.textView = (TextView)((TextView)provider.findView(source, 2131492945));
    }
}

```

有没有感觉到一种抓住了什么的感觉，再仔细看一下`ViewFinder`类，提供了两个inject()方法，分别是从`Activity`中注入，一种是从`View`中注入，为什么从`View`中注入，我们需要多传入一个`Object`对象呢？

仔细观察发现: 

- `host`其实是我们成员变量所在的类
- `source`：`findViewById`操作的类，即`source.findViewById`
- `provider`: 对`findViewById`的一层封住，便于处理`Activity`和`View`两种情况。

> 最终，整个框架到这里就完全的串联起来了

**最后的最后**，我们需要添加`ioc-compiler`到我们的编译时期，这时需要借助`gradle`的一个脚本`android-apt`，用以达到在编译期添加工程依赖。详细过程不写了，可以看我的`github`上的源码。`github`地址已在文章顶部贴出。



### 总结

通过以上的流程可以发现，我们在编译期生成辅助类用以完成`findViewById`的操作。那么根据这个原理，实现`setOnClickListener`等，基本上和这些事类似的。甚至基于这种思路，我们完全可以做更多的事情。



