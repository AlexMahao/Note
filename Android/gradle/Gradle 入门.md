## Gradle 入门

> 转载请标明出处：
[http://blog.csdn.net/lisdye2/article/details/52173213](http://blog.csdn.net/lisdye2/article/details/52173213)
本文出自:[【Alex_MaHao的博客】](http://blog.csdn.net/lisdye2?viewmode=contents)
项目中的源码已经共享到github，有需要者请移步[【Alex_MaHao的github】](https://github.com/AlexSmille/Android-Gradle-Demo)


### 什么是Gradle
 
公司最近要进行版本迭代。最终决定将IDE 从Eclipse转到了Android Studio。 以前自己的一些demo也是通过android Studio 来实现。但毕竟真正的转到AS ，一些必要的东西还是需要去掌握的。


转到AS，第一个问题便是多渠道打包，分包架构怎么办。看我之前的博客关于热修复等的，需要用到Ant，那么在Android Studio中能够使用Ant 呢。关于这个我也不清楚。。。。。

AS 中引入了Gradle工具，完成App的编译工作。那么什么是Gradle呢。

Gradle和Ant类似，也是一种自动化脚本编译语言。能够实现Android app从源码到打包生成最终apk 的过程。

通过Gradle 我们能实现多渠道打包，自动签名，MultiDex等方便我们开发的一些自动脚本。

在研究这个东西时，走了很多弯路，网上的教程大部分都是从原理说起。

比如：Gradle是一个由Groovy编写的脚本语言。介绍Groovy语言是什么，哦，Groovy是基于Java的扩展性动态语言。开始学Groovy，然后学了半天，还是一知半解。本篇博客不介绍原理，只介绍如何使用它。先学会用，在慢慢的分析，最后到精通。


### AS工程目录结构

通过AS 创建工程时，系统会自动创建一些文件，下面一张图介绍了这些文件的作用。

![](AS.PNG)

漏了一点，补上。。。

![](AS2.PNG)


从工程目录可以看到，涉及到Gradle编译脚本有很多，但我们只需要修改关注的有以下几个文件

工程根目录下的：

- `settings.gradle`
- `build.gradle`

每一个`module`下：

- `build.gradle`


### 自动生成的Gradle脚本解析

**`settings.gradle`**

配置哪些`module`需要加入编译，例如我新创建的工程,存在一个基础的`app module`和一个`lib`（依赖库），那么它的内容为

``` java 
include ':app', ':lib'

```

**`build.gradle`**

在创建工程时，会默认在工程的根目录下创建一个`build.gradle`，该`build.gradle`向所有的`module`提供默认的Gradle 编译配置，作为公有的配置向每一个`module`下的`build.gradle`提供默认配置。

```java 
// buildscript 属于一个script block 。 他的目的是为了配置Gradle编译所需要的依赖，
//      此依赖指的不是我们apk运行所需要的依赖、
buildscript {

    // 添加远程仓库
    repositories {
		// // 声明远程仓库的源
        // 之前版本则是mavenCentral(), jcenter可以理解成是一个新的中央远程仓库，兼容maven中心仓库，而且性能更优
        jcenter()
    }

    // 添加依赖，此依赖从远程仓库中查找
    dependencies {
        classpath 'com.android.tools.build:gradle:2.1.0'
    }
}

// 面向所有工程提供的依赖，在此只声明了远程仓库的地址
allprojects {
    // 添加远程仓库
    repositories {
        jcenter()
    }
}

// 删除本地的gradle，重新下载
task clean(type: Delete) {
    delete rootProject.buildDir
}

```


从代码格式上可以看出：`xxx{}`的形式，学名`script block`。我们把它当做代码块就可以了，不过每个代码块有固定的意义。


根目录下的`build.gradle`自上向下可以分为三个部分：

- 针对Gradle编译所提供的依赖支持。
	- 使用`buildscript`作为声明：该中的配置是为了编译时期为我们的gradle提供相关的库支持
- 针对我们的apk所提供的依赖支持。
	- 从代码可以看出，只是提供了远程仓库的声明，是因为针对于每一个`module`，他们依赖的jar包都不相同，所以由他们`module`下的`build.gradle`声明相应的依赖。
- 重新下载gradle插件。此方法不会调用，当然可以鼠标放在代码处，右键run。我试了一下。把我的gradle删除了又重新下载了。没有翻墙的不要run。。。


**module下的`build.gradle`**

直接上代码

```java 
/**
 * /**
 * The first line in the build configuration applies the Android plugin for
 * Gradle to this build and makes the android {} block available to specify
 * Android-specific build options.
 *
 *  应用一个用于构建当前工程的插件，
 *
 *      apply plugin: 'com.android.application'
 *     提供了一个android标签 block，能够解析android 中的一些属性
 */
apply plugin: 'com.android.application'

android {
    // 编译时期使用的sdk版本
    compileSdkVersion 23
    // 编译工具版本
    buildToolsVersion "23.0.3"

    // 默认的配置
    defaultConfig {
        // pplicationId 是一个为了发布而定义独特的标识符，一般和包名一致。当然可以不一致
        applicationId "com.alex.gradleproject"
        // 最小sdk
        minSdkVersion 15
        // 此标示和compileSdkVersion，他指定的是运行时的sdk
        targetSdkVersion 23
        // 代码版本
        versionCode 1
        // app版本
        versionName "1.0"
    }

    /**
     *  配置多样的构建类型，构建系统默认的有两种类型：debug 和 relsase
     *      debug 类型不会显示的展示在buildType 中但是他已经包含了debug 工具和完成相应的签名
     *
     *      release 默认显示，且添加了混淆的配置，但是签名相关的并没有被默认配置
     *
     */
    buildTypes {
        // 是否运行混淆
        release {
            // 改为true ，表示开启混淆
            minifyEnabled false
            // 混淆的文件地址
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

// 依赖
dependencies {
    // module下的libs 文件夹下的jar文件
    compile fileTree(dir: 'libs', include: ['*.jar'])
    // 测试需要用的jar，不会打包到apk中
    testCompile 'junit:junit:4.12'
    // 打包到apk中的jar
    compile 'com.android.support:appcompat-v7:23.4.0'
    // lib依赖工程
    compile project(':lib')
}

```

这么详细的注释，真心良心啊。


此版本中需要注意两点

- `compileSdkVersion`和`targetSdkVersion`两个定义的区别：
	- `compileSdkVersion`：编译时依赖的sdk，例如我指定为23，那么编译时将以此sdk作为参考，那些类没有，或者过时不可用等。
	- `targetSdkVersion`：该版本指定的是运行时，使用哪个版本的sdk。23版本android加入了运行时权限，但我们的app没有做相关的处理，则可以指定sdk为23以下，那么就不会有运行时权限的相关东西了。

- `dependencies{}`：在工程根目录下的`build.gradle`中，只是声明了工程的远程仓库。而此处便是声明该`module`下所依赖的相关jar。此时的相关jar会从`jcenter()`中进行查找下载。




### 自动签名


**1，生成签名文件，并将文件放入到`module`根目录下，创建key文件夹，放入其中**

**2，添加签名信息的配置，在`module`下的`build.gradle`文件中**

```java 
android {
    
	// ....

    signingConfigs{
        config_release{
            keyAlias 'alex'
            keyPassword '111111'
            storePassword '123456'
            storeFile file('key/key.jks')


        }

    }
	
//.....

}

```


**3，将签名的信息添加到buildTypes的release的block中**

```java 
 buildTypes {
        // 是否运行混淆
        release {
            // 改为true ，表示开启混淆
            minifyEnabled false
            // 混淆的文件地址
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'

            // 添加签名信息
            signingConfig signingConfigs.config_release

        }
    }

```


> 如果我们修改了gradle文件，会提示sync Now 的提示。必须刷新

**4，通过gradle工具编译apk**

在AS 的右侧边栏中存在一个`Gradle`，点击里面就是对应的Gradle对象。这个不能细说。。。找到`：app/tasks/build/assembleRelease`，双击运行。

**5，获取签过名的apk文件**

具体目录在`GradleProject\app\build\outputs\apk`中



### Gradle 多渠道打包

多渠道打包，一些大公司可能会自己进行统计，而大部分的可能使用的是友盟统计，那么就按照友盟统计的例子进行实现。

**1，清单文件中添加友盟的声明**

```xml 
	<!--这里可以添加更多的配置，gradle构建时动态变量${CHANNEL_VALUE}为渠道信息-->
    <meta-data android:name="UMENG_CHANNEL" android:value="${CHANNEL_VALUE}" />
```

`${}`标示中添一个变量，获取这个变量的值 。`${CHANNEL_VALUE} == CHANNEL_VALUE `

Gradle在编译时，会将`.gradle`和清单文件进行合并编译，那么在`.gradle`中定义一个`CHANNEL_VALUE`就实现了不同值得替换。


**2，编写渠道列表的配置文件，在app的根目录下 `channel.properties`**

```java 
#默认渠道
channel.default=paojiao
#全部渠道列表
channel.list=baidu,360,hiapk

```

`#`号代表注释。

**3，编写多渠道打包的脚本`channel.gradle`**

```java 
apply plugin: 'com.android.application'

/**
 *  gradle 的基础是Groovy，Groovy是基于Java的,所以java代码在这里都可以使用
 *
 *  该文件可以通过一定的方式和app下的build.gradle进行合并
 */
// 生成配置对象
Properties properties = new Properties();
// 加载配置文件
properties.load(new FileInputStream(project.rootDir.getAbsolutePath()+"/app/channel.properties"))
// 获取默认的配置渠道包
String defau = properties.getProperty('channel.default')
// 获取渠道列表
String channel = properties.get('channel.list')
// 将渠道列表进行分割成数组，获取相应的渠道包
String[] channelList = channel.split(',')

// 下面的代码会默认加入到app 下的build.gradle中
android{

    /**
     * 为了获得不同的App 版本 ，能够设置一些信息，覆盖defaultConfig{}的一些配置信息
     *
     *  该block　不是可选项，系统不会默认创建他
     *
     *  打不通包时，如果有重复的，优先级如下 ，相同名字下
     *  build variant > build type > product flavor > main source set > library dependencies
     */
    productFlavors{

        // 循环遍历渠道包
        for(String c : channelList){
            // 引用变量  此时 ${c} = c的具体字符串
            "${c}"{
                // 声明了一个变量  类似于键值对  key = CHANNEL_VALUE  value = channel的值
                manifestPlaceholders = [CHANNEL_VALUE: c]
            }

        }
    }


}

```

**4，将编写的渠道脚本加入到app下的`build.gradle中`**

```java 
apply plugin: 'com.android.application'

// 引入channel.gradle
apply from:'channel.gradle'

```

**5，在右侧的Gradle工具栏中运行gradle脚本**

具体运行为`app/tasks/build/assembleRelease`，双击运行。

**6，获取渠道包**

生成的apk的具体目录`GradleProject\app\build\outputs\apk`中

























