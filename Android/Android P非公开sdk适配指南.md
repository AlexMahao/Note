## Android P非公开sdk适配指南

> 因为博客中提到的地址需要通过代理才能访问，所以将对应地址下的内容已打包，有需要者可以下载[https://download.csdn.net/download/lisdye2/10513405](https://download.csdn.net/download/lisdye2/10513405)

### 非公开sdk说明

在`P`上，谷歌提出调用非公开sdk限制的说明，但为了各大应用便于逐渐迁移，推出了三个主要的名单。

- 浅灰名单（`light greylist`） : 在`P`上或以上的手机会有对应的日志提示。
- 深黑名单（`dark greylist`）: `targetSDK < P` 和灰名单的效果类似。`target >= P`则应用直接崩溃，将会抛出`NoSuchMethodError/NoSuchFieldException`，和黑名单的行为一样。
- 黑名单 （`blacklist`）： 在9.0上或以上的手机会直接崩溃。

具体的三个名单的查看地址如下[https://android.googlesource.com/platform/prebuilts/runtime/+/master/appcompat](https://android.googlesource.com/platform/prebuilts/runtime/+/master/appcompat).


### 查找应用内的非法调用

#### `Log`的方式

- 全量测试
- 捕获log
- 查询log总的关键日志

关键日志如下

```
Accessing hidden field Landroid/os/Message;->flags:I (light greylist, JNI)

Landroid/app/ActivityThread;->currentActivityThread()Landroid/app/Activity
Thread; (blacklist, reflection)
```

### Google提供的扫描工具

具体地址如下[https://android.googlesource.com/platform/prebuilts/runtime/+/master/appcompat/](https://android.googlesource.com/platform/prebuilts/runtime/+/master/appcompat/).

找到对应veridex-xxx.zip下载并解压，进入到解压后对应的内部目录。

对应的命令如下

```
./appcompat.sh --dex-file=test

```

日志输出如下

```
NOTE: appcompat.sh is still under development. It can report
API uses that do not execute at runtime, and reflection uses
that do not exist. It can also miss on reflection uses.
#1: Linking light greylist Landroid/util/FloatMath;->ceil(F)F use(s):
       Lcom/viewpagerindicator/LinePageIndicator;->measureHeight(I)I
       Lcom/viewpagerindicator/LinePageIndicator;->measureWidth(I)I

#2: Reflection light greylist Landroid/animation/LayoutTransition;->cancel use(s):
       Landroid/support/transition/ViewGroupUtilsApi14;->cancelLayoutTransition(Landroid/animation/LayoutTransition;)V

#3: Reflection light greylist Landroid/content/res/Resources;->mResourcesImpl use(s):
       Landroid/support/v7/app/ResourcesFlusher;->flushNougats(Landroid/content/res/Resources;)Z

#4: Reflection light greylist Landroid/graphics/FontFamily;->abortCreation use(s):
       Landroid/support/v4/graphics/TypefaceCompatApi26Impl;-><clinit>()V

#5: Reflection light greylist Landroid/graphics/FontFamily;->addFontFromAssetManager use(s):
       Landroid/support/v4/graphics/TypefaceCompatApi26Impl;-><clinit>()V

#6: Reflection light greylist Landroid/graphics/FontFamily;->addFontFromBuffer use(s):
       Landroid/support/v4/graphics/TypefaceCompatApi26Impl;-><clinit>()V

...
...



68 hidden API(s) used: 1 linked against, 67 through reflection
       0 in blacklist
       4 in dark greylist
       64 in light greylist
To run an analysis that can give more reflection accesses, 
but could include false positives, pass the --imprecise flag. 

```


### 适配

- 浅灰名单可以暂不做考虑。
- 深灰名单等`targetSDK >= P`时在做考虑。
- 黑名单找对对应方法去替换和避免。