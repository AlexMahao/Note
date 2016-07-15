
## Dagger2 依赖注入

在上一篇博客中总结了依赖注入的原理与简单实现。[ 依赖注入的原理 ](http://blog.csdn.net/lisdye2/article/details/51887402)

依赖注入就是将调用者需要的另一个对象实例不在调用者内部实现，而是通过一定的方式从外部传入实例，解决了各个类之间的耦合。


那么这个外部，到底指的是哪里，如果指的是另一个类，那么，另一个类内部不就耦合了。能不能有一种方式，将这些构造的对象放到一个容器中，具体需要哪个实例时，就从这个容器中取就行了。那么，类的实例和使用就不在有联系了，而是通过一个容器将他们联系起来。实现了解耦。这个容器，便是`Dagger2`。


`Dagger2`是Google出的依赖注入框架。肯定有小伙伴疑问，为什么会有个 2 呢。该框架是基于`square`开发的`dagger`基础上开发的。


`Dagger2`的原理是在编译期生成相应的依赖注入代码。这也是和其他依赖注入框架不同的地方，其他框架是在运行时期反射获取注解内容，影响了运行效率。


### **导入Dagger2**

使用`Dagger2`之前需要一些配置，该配置是在`Android Studio`中进行操作。


在工程的`build.gradle`文件中添加`android-apt`插件（该插件会后面介绍）

```java 
buildscript {
   
	....

    dependencies {
		
        classpath 'com.android.tools.build:gradle:2.1.0'
		// 添加android-apt 插件
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
}

```

在app的中的`build.gradle`文件中添加配置

```java 
apply plugin: 'com.android.application'
// 应用插件
apply plugin: 'com.neenbedankt.android-apt'

android {
    compileSdkVersion 23
    buildToolsVersion "23.0.2"

    defaultConfig {
        applicationId "com.mahao.alex.architecture"
        minSdkVersion 15
        targetSdkVersion 23
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}


dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:23.3.0'

	// dagger 2 的配置
    compile 'com.google.dagger:dagger:2.4'
    apt 'com.google.dagger:dagger-compiler:2.4'
    compile 'org.glassfish:javax.annotation:10.0-b28'// 添加java 注解库
}

```


以上两个配置就可以了。


`android-apt`是`Gradle`编译器的插件，根据其官方文档，主要两个目的：

- 编译时使用该工具，最终打包时不会将该插件打入到apk中。

- 能够根据设置的源路径，在编译时期生成相应代码。


在导入类库时，

```java 
	compile 'com.google.dagger:dagger:2.4'
    apt 'com.google.dagger:dagger-compiler:2.4'

```

`dagger`是主要的工具类库。`dagger-compiler`为编译时期生成代码等相关的类库。


在`android-apt`的文档中，也推荐使用这种方式。因为，编译时期生成代码的类库在运行期并不需要，那么将其分为两个库，（运行类库`dagger`）和（编译器生成代码类库（`dagger-compiler`）），那么在打包时，就不需要将`dagger-compiler`打入其中（用不到），减小APK 的大小。



### **Dagger2的简单使用**

一个东西需要先会用，然后才能看它的原理。该篇博客的目的主要是讲解如何使用。后面会有专门的分析源码的博客。


在之前的分析中，通过`Dagger2`的目的是将程序分为三个部分。
- 实例化部分：对象的实例化。类似于容器，将类的实例放在容器里。
- 调用者：需要实例化对象的类。
- 沟通桥梁：利用`Dagger2`中的一些API 将两者联系。


先看实例化部分（容器），在此处是`Module`。

```java 

@Module   //提供依赖对象的实例
public class MainModule {

    @Provides // 关键字，标明该方法提供依赖对象
    Person providerPerson(){
        //提供Person对象
        return new Person();
    }


}

```

沟通部分`Component`

```java 
@Component(modules = MainModule.class)  // 作为桥梁，沟通调用者和依赖对象库
public interface MainComponent {

    //定义注入的方法
    void inject(MainActivity activity);

}

```

使用者`Actvity`中调用。

```java 
public class MainActivity extends AppCompatActivity{

    @Inject   //标明需要注入的对象
    Person person;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // 构造桥梁对象
        MainComponent component = DaggerMainComponent.builder().mainModule(new MainModule()).build();

        //注入
        component.inject(this);

    }
}

```

看一下`Person`类

```java 
public class Person {

    public Person(){
        Log.i("dagger","person create!!!");
    }


}

```

最后结果不在演示。其过程如下：

- 创建`Component`(桥梁)，并调用注入方法。

```java 
		// 构造桥梁对象
        MainComponent component = DaggerMainComponent.builder().mainModule(new MainModule()).build();

        //注入
        component.inject(this);
```

- 查找当前类中带有`@Inject`的成员变量。

```java 
 	@Inject   //标明需要注入的对象
    Person person;

```
- 根据成员变量的类型从`Module`中查找哪个有`@Provides`注解的方法返回值为当类型。

```java 
 	@Provides // 关键字，标明该方法提供依赖对象
    Person providerPerson(){
        //提供Person对象
        return new Person();
    }

```



在使用过程出现了很多注解：

- `@Module`:作为实例对象的容器。
- `@Provides`:标注能够提供实例化对象的方法。
- `@Component`:作为桥梁，注入对象的通道。
- `@Inject`：需要注入的方法


如上使用有一种变通，修改`MainModule`和`Person`类。

```java 
@Module   //提供依赖对象的实例
public class MainModule {

/*
    @Provides // 关键字，标明该方法提供依赖对象
    Person providerPerson(){
        //提供Person对象
        Log.i("dagger"," from Module");
        return new Person();
    }

*/

}

```
```java 
public class Person {

    @Inject  // 添加注解关键字
    public Person(){
        Log.i("dagger","person create!!!");
    }

}

```

将`Module`中的`providePerson()`方法注释，在`Person`中添加`@Inject`注解，依然能够实现。

逻辑如下：
- 先判断`Module`中是否有提供该对象实例化的方法。
- 如果有则返回。结束。
- 如果没有，则查找该类的构造方法，是否有带有`@Inject`的方法。如过存在，则返回。



### **@Singleton  单例注解**

假如，对于同一个对象，我们需要注入两次，如下方式

```java 
 public class MainActivity extends AppCompatActivity{

    @Inject
    Person person;

    @Inject
    Person person2;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // 构造桥梁对象
        MainComponent component = DaggerMainComponent.builder().mainModule(new MainModule()).build();

        //注入
        component.inject(this);

        // 打印两个对象的地址
        Log.i("dagger","person = "+ person.toString()+"; person2 = "+ person2.toString());
    }
}

```

看一下结果：

```java 
person = com.mahao.alex.architecture.dagger2.Person@430d1620; person2 = com.mahao.alex.architecture.dagger2.Person@430d17c8

```

可见两个对象不一致。也就是说创建了两个对象。


可以在提供实例化对象的方法上添加`@Singleton`注解

```java 
 @Provides // 关键字，标明该方法提供依赖对象
    @Singleton
    Person providerPerson(){

        return new Person();
    }

```

同时，对于`MainComponent`也需要添加注解，不添加会无法编译

```java 
@Singleton
@Component(modules = MainModule.class)  // 作为桥梁，沟通调用者和依赖对象库
public interface MainComponent {
    //定义注入的方法
    void inject(MainActivity activity);

}
```

此时在Log,会发现两个对象的地址一样，可见是同一个对象。

```java 
person = com.mahao.alex.architecture.dagger2.Person@4310f898; person2 = com.mahao.alex.architecture.dagger2.Person@4310f898
```

那么不同的`Activity`之间，能否保持单例呢？

创建一个新的`Activity`，代码如下：

```java 
public class Main2Actvity extends AppCompatActivity {

    @Inject
    Person person;


    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // 构造桥梁对象
        MainComponent component = DaggerMainComponent.builder().mainModule(new MainModule()).build();

        //注入
        component.inject(this);

        Log.i("dagger","person = "+ person.toString());
    }
}

```

结果如下：

```java 
 person create!!!
 person = com.mahao.alex.architecture.dagger2.Person@4310f898; person2 = com.mahao.alex.architecture.dagger2.Person@4310f898
 person create!!!
 person = com.mahao.alex.architecture.dagger2.Person@43130058
```

可见，`@Singleton`只对一个`Component`有效。


###  **需要参数的实例化对象**

`Person`的构造方法发生了变化，需要传入一个`Context`，代码如下：

```java 
public class Person {

    private Context mContext;

    public Person(Context context){
        mContext = context;
        Log.i("dagger","create");
    }

}

```

这样的话，我们需要修改`MainModule`

```java 

@Module   //提供依赖对象的实例
public class MainModule {

    private Context mContext;

    public MainModule(Context context){
        mContext = context;
    }


    @Provides
    Context providesContext(){
        // 提供上下文对象
        return mContext;
    }

    @Provides // 关键字，标明该方法提供依赖对象
    @Singleton
    Person providerPerson(Context context){

        return new Person(context);
    }

}

```

- 修改`providerPerson`方法，传入`Context`对象。
- 添加`providesContext()`,用以提供`Context`对象。


看一下使用

```java 
 // 构造桥梁对象
        MainComponent component = DaggerMainComponent.builder().mainModule(new MainModule(this)).build();

        //注入
        component.inject(this);

```

逻辑：

- 根据`@Inject`注解，查找需要依赖注入的对象。
- 从`MainModule`中找到








- `Module`: 提供依赖注入所需要的对象。
- `Component`：作为注入的桥梁，沟通实际对象

