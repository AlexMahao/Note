## Android 混淆打包

Android Studio和Eclipse虽然是两个不同的工具，混淆的使用虽然不同，但规则相同。

### Eclipse混淆

在eclipse中，文件根目录中有如下两个文件`projiect.properties`和`proguard-project.txt`。
开启混淆打包只需要在`projiect.properties`中，被注释的有如下一句话
```java 
proguard.config=${sdk.dir}/tools/proguard/proguard-android.txt:proguard-project.txt
```
把他注释即可。

### Android Studio 开启混淆

在build.gradle中修改`minifyEnabled`改为true
即可
```java 
 buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
```

### 规则研究

在开启混淆中，`eclipse`和`androidStudio`中都有一个文件`proguard-android.txt`，这是混淆的一个默认文件，该默认文件为android 提供的一个默认规则，如果我们的工程没有引入第三方库等，那么很简单的就能混淆了。

该文件的位置位于`sdk\tools\proguard\proguard-android`

看一下这个文件的一些配置属性。

- `-dontusemixedcaseclassnames`：不使用大小写形式的混淆名
- `-dontskipnonpubliclibraryclasses`:不跳过library的非public的类
- `-dontoptimize`：不进行优化，优化可能会在某些手机上无法运行。
- `-dontpreverify`：不净行预校验，该校验是java平台上的，对android没啥用处
- `-keepattributes *Annotation*`：对注解中的参数进行保留
- `-keep public class com.google.vending.licensing.ILicensingService  -keep public class com.android.vending.licensing.ILicensingService`:不混淆上面两个类，接入google服务中是使用。

```java 
-keepclasseswithmembernames class * {
    native <methods>;
}
```
- 不混淆包含native方法的类的类名以及native方法名

```java 
-keepclassmembers public class * extends android.view.View {
   void set*(***);
   *** get*();
}
```
- 不混淆任何view子类的get和set方法。

```java 
-keepclassmembers class * extends android.app.Activity {
   public void *(android.view.View);
}
```
-保留Activity中的返回值为void，传入参数为view的方法。因为在xml中存在的onClick属性

```java 
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
```
- 保持枚举类中的属性不被混淆

```java 
-keepclassmembers class * implements android.os.Parcelable {
  public static final android.os.Parcelable$Creator CREATOR;
}

```
- 保持实现Parcelable中的CREATOR字段值不被修改

```java 
-keepclassmembers class **.R$* {
    public static <fields>;
}
```

- 不混淆R文件中的所有静态字段，我们都知道R文件是通过字段来记录每个资源的id的，字段名要是被混淆了，id也就找不着了。

- `-dontwarn android.support.**`对support中的包的东西不警告。因为，当我们指定的目标版本较低时，一些高版本的东西无法使用，但我们无需担心这些问题，support中都有相应的判断。

```java 
# Understand the @Keep support annotation.
-keep class android.support.annotation.Keep

-keep @android.support.annotation.Keep class * {*;}

-keepclasseswithmembers class * {
    @android.support.annotation.Keep <methods>;
}

-keepclasseswithmembers class * {
    @android.support.annotation.Keep <fields>;
}

-keepclasseswithmembers class * {
    @android.support.annotation.Keep <init>(...);
}

```
- 这一堆不知道是干什么的。


### 常用关键字

- `-keep`关键字
	- `keep`：包留类和类中的成员，防止他们被混淆
	- `keepnames`:保留类和类中的成员防止被混淆，但成员如果没有被引用将被删除
	- `keepclassmembers` :只保留类中的成员，防止被混淆和移除。
	- `keepclassmembernames`:只保留类中的成员，但如果成员没有被引用将被删除。
	- `keepclasseswithmembers`:如果当前类中包含指定的方法，则保留类和类成员，否则将被混淆。
	- `keepclasseswithmembernames`:如果当前类中包含指定的方法，则保留类和类成员，如果类成员没有被引用，则会被移除。

- `-dontwarn`:忽视警告。
- `-optimizationpasses 5`：代码混淆压缩比，在0~7之间，默认为5，一般不做修改。
- `-keepattributes Signature`：避免混淆泛型。
- `-keepattributes SourceFile,LineNumberTable`：抛出异常时保留代码行号

### 对于App混淆的一些配置

- Java反射用到的类不能被混淆
Gson,fastJson等解析json数据时，就是根据实体类的映射关系，他需要和成员变量名一一对应。如果我们将实体类放到一个包中，那么久很简单。
```java 
- keep com.entity.**;
``` 


```java 
Can't read [C:\appstart\MDWApp2\libs\V0.2.1.jar] (Can't process class [cn/soundtooth/spush_sdk/event/DeviceStateBroadcastReceiver.class] (Unknown verification type [255] in stack map frame))
```