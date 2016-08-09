## Android热修复三部曲之动态加载补丁.dex文件

> 转载请标明出处：
[http://blog.csdn.net/lisdye2/article/details/52049857](http://blog.csdn.net/lisdye2/article/details/52049857)
本文出自:[【Alex_MaHao的博客】](http://blog.csdn.net/lisdye2?viewmode=contents)
项目中的源码已经共享到github，有需要者请移步[【Alex_MaHao的github】](https://github.com/AlexSmille/Android-hot-fix)

该篇作为Andriod热修复三部曲的最后一篇，本篇基于前两篇

- [**Android 热修复三部曲之基本的Ant打包脚本**](http://blog.csdn.net/lisdye2/article/details/52049857)
- [**Android热修复三部曲之MultiDex 分包架构**](http://blog.csdn.net/lisdye2/article/details/52094693)

在之前的博客中，我们将`.java`文件打成了三个`.dex`文件
- `classes.dex`:程序必须启动的类，保证没问题的（`Application`,`MainActivity`）
- `classes2.dex`:业务逻辑的类，如果出问题了可以动态替换。
- `classes3.dex`:jar包的类，基本上不会出现问题。

那么我们实现热修复的原理：

- 修改好代码之后打出无bug版的`classes2.dex`。
- 从服务器下载到移动端。
- 动态加载`classes2.dex`，实现覆盖。

### **如何动态加载.dex文件**

在java中，`ClassLoader`负责加载`.class`文件。那么在`Android`中同样存在这样的类即`BaseDexClassLoader`.

因为其实存在于系统源码中的，我们可以看他的源码，有一个关键字段

```java 
/**
 * Base class for common functionality between various dex-based
 * {@link ClassLoader} implementations.
 */
public class BaseDexClassLoader extends ClassLoader {
    private final DexPathList pathList; // 关键字段

    /**
     * Constructs an instance.
     *
     * @param dexPath the list of jar/apk files containing classes and
     * resources, delimited by {@code File.pathSeparator}, which
     * defaults to {@code ":"} on Android
     * @param optimizedDirectory directory where optimized dex files
     * should be written; may be {@code null}
     * @param libraryPath the list of directories containing native
     * libraries, delimited by {@code File.pathSeparator}; may be
     * {@code null}
     * @param parent the parent class loader
     */
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }

	//...
}

```
贴了一部分代码，在`BaseDexClassLoader`中有着很关键的字段`DexPathList`，从命名上就能看出，其是存放`.dex`文件的集合。看一下他的源码

```java 
/*package*/ final class DexPathList {
    private static final String DEX_SUFFIX = ".dex";
    private static final String JAR_SUFFIX = ".jar";
    private static final String ZIP_SUFFIX = ".zip";
    private static final String APK_SUFFIX = ".apk";

    /** class definition context */
    private final ClassLoader definingContext;

    /**
     * List of dex/resource (class path) elements.
     * Should be called pathElements, but the Facebook app uses reflection
     * to modify 'dexElements' (http://b/7726934).
     */
    private final Element[] dexElements; // 关键字段

```

在`DexPathList`中，存在字段`dexElements`，该字段便是存放加载的`.dex`文件的字段。

那么从上面我们可以看出，我们实现动态加载的流程：

- **生成补丁包的`BaseDexClassLoader`类**
- **获取到`BaseDexClassLoader`中的`DexPathList`字段的`dexElements`**
- **将补丁包的`dexElements`和本身的`dexElements`合并为一个新的数组（补丁包的放在前面）**
- **将新合并的`dexElements`设置到系统的`dexElements`中**

### 代码实现

在布局文件中，添加两个按钮

```xml 
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:orientation="vertical"
    android:layout_height="match_parent" >

    <Button 
        android:text="inject"
        android:layout_height="wrap_content"
        android:layout_width="wrap_content"
        android:onClick="inject"/>
    
    <Button
        android:onClick="test"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/hello_world" />

</LinearLayout>
```

两个按钮：分别是热修复，和测试热修复是否成功的。

```java

/**
 * 测试类 
 * @author alex_mahao
 *
 */
public class Test {

	public static void show(Context context){
		Toast.makeText(context, "1", Toast.LENGTH_SHORT).show();
	}
	
}

```

测试类，修改toast的值。

```java 
public class MainActivity extends AppCompatActivity {

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
	}

	public void inject(View view) {

		// 无bug的classes2.dex文件存放地址
		String sourceFile = Environment.getExternalStorageDirectory().getAbsolutePath() + File.separator
				+ "classes2.dex";

		// 系统的私有目录
		String targetFile = this.getDir("odex", Context.MODE_PRIVATE).getAbsolutePath() + File.separator
				+ "classes2.dex";

		try {
			// 复制文件到私有目录
			FileUtils.copyFile(sourceFile, targetFile);
			
			// 加载.dex文件
			FixDexUtils.loadFixDex(this.getApplication());
			
		} catch (IOException e) {
			e.printStackTrace();
		}

		

	}

	public void test(View view) {
		Test.show(this);

	}
}

```

在这里省略了从服务器下载的过程，直接把补丁放在了系统根目录下。

从代码上可以看到，第一步是复制操作。因为android 系统的原因，如果加载`.dex`文件，必须放到私有目录odex下。

然后开始加载odex目录下的`.dex`文件。`FixDexUtils.loadFixDex(this.getApplication());`


直接上源码

```java 
/**
 * 动态加载补丁包
 * @author alex_mahao
 *
 */
public class FixDexUtils {

	private static HashSet<File> loadedDex = new HashSet<File>();

	static {
		loadedDex.clear();
	}


	public static void loadFixDex(Context context) {
		// 获取到系统的odex 目录
		File fileDir = context.getDir("odex", Context.MODE_PRIVATE);
		File[] listFiles = fileDir.listFiles();

		for (File file : listFiles) {
			if (file.getName().endsWith(".dex")) {
				// 存储该目录下的.dex文件（补丁）
				loadedDex.add(file);
			}
		}
		
		doDexInject(context, fileDir);

	}

	private static void doDexInject(Context context, File fileDir) {
		// .dex 的加载需要一个临时目录
		String optimizeDir = fileDir.getAbsolutePath() + File.separator + "opt_dex";
		File fopt = new File(optimizeDir);
		if (!fopt.exists())
			fopt.mkdirs();
		// 根据.dex 文件创建对应的DexClassLoader 类
		for (File file : loadedDex) {
			DexClassLoader classLoader = new DexClassLoader(file.getAbsolutePath(), fopt.getAbsolutePath(), null,
					context.getClassLoader());
			//注入
			inject(classLoader, context);

		}
	}

	private static void inject(DexClassLoader classLoader, Context context) {
		
		// 获取到系统的DexClassLoader 类
		PathClassLoader pathLoader = (PathClassLoader) context.getClassLoader();
		try {
			// 分别获取到补丁的dexElements和系统的dexElements
			Object dexElements = combineArray(getDexElements(getPathList(classLoader)),
					getDexElements(getPathList(pathLoader)));
			// 获取到系统的pathList 对象
			Object pathList = getPathList(pathLoader);
			// 设置系统的dexElements 的值
			setField(pathList, pathList.getClass(), "dexElements", dexElements);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 通过反射设置字段值
	 */
	private static void setField(Object obj, Class<?> cl, String field, Object value)
			throws NoSuchFieldException, IllegalArgumentException, IllegalAccessException {

		Field localField = cl.getDeclaredField(field);
		localField.setAccessible(true);
		localField.set(obj, value);
	}

	/**
	 * 通过反射获取 BaseDexClassLoader中的PathList对象
	 */
	private static Object getPathList(Object baseDexClassLoader)
			throws IllegalArgumentException, NoSuchFieldException, IllegalAccessException, ClassNotFoundException {
		return getField(baseDexClassLoader, Class.forName("dalvik.system.BaseDexClassLoader"), "pathList");
	}

	/**
	 * 通过反射获取指定字段的值
	 */
	private static Object getField(Object obj, Class<?> cl, String field)
			throws NoSuchFieldException, IllegalArgumentException, IllegalAccessException {
		Field localField = cl.getDeclaredField(field);
		localField.setAccessible(true);
		return localField.get(obj);
	}

	/**
	 * 通过反射获取DexPathList中dexElements
	 */
	private static Object getDexElements(Object paramObject)
			throws IllegalArgumentException, NoSuchFieldException, IllegalAccessException {
		return getField(paramObject, paramObject.getClass(), "dexElements");
	}

	/**
	 * 合并两个数组
	 * @param arrayLhs
	 * @param arrayRhs
	 * @return
	 */
	private static Object combineArray(Object arrayLhs, Object arrayRhs) {
		Class<?> localClass = arrayLhs.getClass().getComponentType();
		int i = Array.getLength(arrayLhs);
		int j = i + Array.getLength(arrayRhs);
		Object result = Array.newInstance(localClass, j);
		for (int k = 0; k < j; ++k) {
			if (k < i) {
				Array.set(result, k, Array.get(arrayLhs, k));
			} else {
				Array.set(result, k, Array.get(arrayRhs, k - i));
			}
		}
		return result;
	}
}

```

这样，热修复就实现了。


完结！！！