### MultiDex加载流程分析

> 只介绍主要流程，multiDex源码已经上传到[https://github.com/AlexSmille/google-android-support-source-analyze](https://github.com/AlexSmille/google-android-support-source-analyze),包含了`BaseDexClassloader`等一些android源码。


#### 1.判断当前是否需要动态加载类

在`5.0`以上，会自动加载一个apk中所有的`classesX.dex`，但是低于`5.0`时需要手动导入。

具体判断是否需要手动导入如下：

```java
 static boolean isVMMultidexCapable(String versionString) {
        //You can verify which runtime is in use by calling System.getProperty("java.vm.version").
        // If ART is in use, the property's value is "2.0.0" or higher.
        boolean isMultidexCapable = false;
        //2.1.0的时候开始支持默认加载外部classes.dex,根据vm版本号判断

        if (versionString != null) {
            Matcher matcher = Pattern.compile("(\\d+)\\.(\\d+)(\\.\\d+)?").matcher(versionString);
            if (matcher.matches()) {
                try {
                    int major = Integer.parseInt(matcher.group(1));
                    int minor = Integer.parseInt(matcher.group(2));
                    isMultidexCapable = (major > VM_WITH_MULTIDEX_VERSION_MAJOR)
                            || ((major == VM_WITH_MULTIDEX_VERSION_MAJOR)
                                    && (minor >= VM_WITH_MULTIDEX_VERSION_MINOR));
                } catch (NumberFormatException e) {
                    // let isMultidexCapable be false
                }
            }
        }
        return isMultidexCapable;
    }

```

如果已经支持自动导入，则结束流程，否则进入下一步。


#### 2.构造必要参数，调用`doInstallation()`方法


调用`doInstallation()`传入四个参数

- `Context mainContext`：上下文对象。
- `File sourceApk`：安装额apk的存放路径。/data/app/....
- `File dataDir`: apk的数据存放目录。`/data/data/package/`
- `String secondaryFolderName`:存放缓存的目录文件名`secondary-dexes`
- `String prefsKeyPrefix`:
- `boolean reinstallOnPatchRecoverableException`：如果导入失败，是重试还是直接抛出异常。



#### 3.`doInstallation()`方法判断一些必要条件


```java

 	if (installedApk.contains(sourceApk)) {
                // 是否包含要导入的apk,防止重复导入
                return;
            }
            // 将apk添加到集合中
            installedApk.add(sourceApk);

            // 5.0以上不用导入  IS_VM_MULTIDEX_CAPABLE
            if (Build.VERSION.SDK_INT > MAX_SUPPORTED_SDK_VERSION) {

                Log.w(TAG, "MultiDex is not guaranteed to work in SDK version "
                        + Build.VERSION.SDK_INT + ": SDK version higher than "
                        + MAX_SUPPORTED_SDK_VERSION + " should be backed by "
                        + "runtime with built-in multidex capabilty but it's not the "
                        + "case here: java.vm.version=\""
                        + System.getProperty("java.vm.version") + "\"");
            }
            
    }

```

#### 4.构造存放`classesX.dex`的目录`/data/data/package/code_cache/secondary-dexes`

```java
File dexDir = getDexDir(mainContext, dataDir, secondaryFolderName);
```

创建`dexDir`，即存放`dex`的目录。`getDexDir()`代码如下：

```java
private static File getDexDir(Context context, File dataDir, String secondaryFolderName)
            throws IOException {
        // data/data/package/code_cache
        File cache = new File(dataDir, CODE_CACHE_NAME);
        try {
            mkdirChecked(cache);
        } catch (IOException e) {
            /* If we can't emulate code_cache, then store to filesDir. This means abandoning useless
             * files on disk if the device ever updates to android 5+. But since this seems to
             * happen only on some devices running android 2, this should cause no pollution.
             */
            cache = new File(context.getFilesDir(), CODE_CACHE_NAME);
            mkdirChecked(cache);
        }
        // data/data/package/code_cache/secondary-dexes
        File dexDir = new File(cache, secondaryFolderName);
        mkdirChecked(dexDir);
        return dexDir;
    }

```

#### 5.构造`MultiDexExtractor`对象，准备提取`classesX.dex`

```java
// MultiDexExtractor is taking the file lock and keeping it until it is closed.
            // Keep it open during installSecondaryDexes and through forced extraction to ensure no
            // extraction or optimizing dexopt is running in parallel.
            MultiDexExtractor extractor = new MultiDexExtractor(sourceApk, dexDir);

```

#### 6.通过`MultiDexExtractor`提取并存储到`data/data/package/code_cache/secondary-dexes/`目录下

在此步骤中，会将`apk`中的`classesX.dex`提取并存放为`data/data/package/code_cache/secondary-dexes/package--1.apk.classN.zip`，以便后续加载到内存中。

关键方法`MultiDexExtractor.load()`方法


```java
List<? extends File> load(Context context, String prefsKeyPrefix, boolean forceReload)
            throws IOException {
        Log.i(TAG, "MultiDexExtractor.load(" + sourceApk.getPath() + ", " + forceReload + ", " +
                prefsKeyPrefix + ")");

        if (!cacheLock.isValid()) {
            throw new IllegalStateException("MultiDexExtractor was closed");
        }

        List<ExtractedDex> files;
        if (!forceReload && !isModified(context, sourceApk, sourceCrc, prefsKeyPrefix)) {
            // apk信息没有发生变化
            try {
                // 加载已存在的classX.zip
                files = loadExistingExtractions(context, prefsKeyPrefix);
            } catch (IOException ioe) {
                Log.w(TAG, "Failed to reload existing extracted secondary dex files,"
                        + " falling back to fresh extraction", ioe);
                files = performExtractions();
                putStoredApkInfo(context, prefsKeyPrefix, getTimeStamp(sourceApk), sourceCrc,
                        files);
            }
        } else {
            if (forceReload) {
                Log.i(TAG, "Forced extraction must be performed.");
            } else {
                Log.i(TAG, "Detected that extraction must be performed.");
            }
            // 重新从apk中提取classesX.dex
            files = performExtractions();
            putStoredApkInfo(context, prefsKeyPrefix, getTimeStamp(sourceApk), sourceCrc,
                    files);
        }

        Log.i(TAG, "load found " + files.size() + " secondary dex files");
        return files;
    }

```

根据apk是否有新变化，判断是直接读取之前获取的`classX.zip`文件，还是从`apk`中重新提取`classes.dex`。


看一下提取的方法

```java
private List<ExtractedDex> performExtractions() throws IOException {

        final String extractedFilePrefix = sourceApk.getName() + EXTRACTED_NAME_EXT;
        // It is safe to fully clear the dex dir because we own the file lock so no other process is
        // extracting or running optimizing dexopt. It may cause crash of already running
        // applications if for whatever reason we end up extracting again over a valid extraction.
        // 清楚缓存目录
        clearDexDir();

        List<ExtractedDex> files = new ArrayList<ExtractedDex>();
        // 获取apk的zip对象
        final ZipFile apk = new ZipFile(sourceApk);

        int secondaryNumber = 2;
        // 从apk中获取对应classesX.dex
        ZipEntry dexFile = apk.getEntry(DEX_PREFIX + secondaryNumber + DEX_SUFFIX);
        while (dexFile != null) {
            String fileName = extractedFilePrefix + secondaryNumber + EXTRACTED_SUFFIX;
            ExtractedDex extractedFile = new ExtractedDex(dexDir, fileName);
            files.add(extractedFile);

            Log.i(TAG, "Extraction is needed for file " + extractedFile);
            int numAttempts = 0;
            boolean isExtractionSuccessful = false;
            while (numAttempts < MAX_EXTRACT_ATTEMPTS && !isExtractionSuccessful) {
                numAttempts++;

                // Create a zip file (extractedFile) containing only the secondary dex file
                // (dexFile) from the apk.
                // 创建zip文件，并提取.dex文件
                extract(apk, dexFile, extractedFile, extractedFilePrefix);
                // .... 后面的都是一些判断
            }
        }
        return files;
    }

```

此段对应的`log`如下

```
// MultidexExtractor.load()
07-04 16:19:52.296 I/MultiDex( 4387): MultiDexExtractor.load(/data/app/com.spearbothy.android-2.apk, false, )
// 有新信息变化，删除老的缓存文件
07-04 16:19:52.306 I/MultiDex( 4387): Detected that extraction must be performed.
07-04 16:19:52.306 I/MultiDex( 4387): Trying to delete old file /data/data/com.spearbothy.android/code_cache/secondary-dexes/com.spearbothy.android-1.apk.classes2.zip of size 185848
07-04 16:19:52.306 I/MultiDex( 4387): Deleted old file /data/data/com.spearbothy.android/code_cache/secondary-dexes/com.spearbothy.android-1.apk.classes2.zip
07-04 16:19:52.306 I/MultiDex( 4387): Trying to delete old file /data/data/com.spearbothy.android/code_cache/secondary-dexes/com.spearbothy.android-1.apk.classes2.dex of size 675216
07-04 16:19:52.306 I/MultiDex( 4387): Deleted old file /data/data/com.spearbothy.android/code_cache/secondary-dexes/com.spearbothy.android-1.apk.classes2.dex
// 提取apk中的class文件
07-04 16:19:52.386 I/MultiDex( 4387): Extraction is needed for file /data/data/com.spearbothy.android/code_cache/secondary-dexes/com.spearbothy.android-2.apk.classes2.zip
07-04 16:19:52.386 I/MultiDex( 4387): Extracting /data/data/com.spearbothy.android/code_cache/secondary-dexes/tmp-com.spearbothy.android-2.apk.classes762107569.zip
07-04 16:19:56.016 I/MultiDex( 4387): Renaming to /data/data/com.spearbothy.android/code_cache/secondary-dexes/com.spearbothy.android-2.apk.classes2.zip
07-04 16:19:56.016 I/MultiDex( 4387): Extraction succeeded '/data/data/com.spearbothy.android/code_cache/secondary-dexes/com.spearbothy.android-2.apk.classes2.zip': length 3843460 - crc: 28802720

```

#### 7.加载类到`BaseDexClassLoader`中

在第6步的时候，将`.dex`文件提取出来保存到了对应的目录，那么只需要加载就可以了。

```java
// 获取提出来的`dex`文件目录
files = extractor.load(mainContext, prefsKeyPrefix, true);
// 导入
installSecondaryDexes(loader, dexDir, files);
```

```java
private static void installSecondaryDexes(ClassLoader loader, File dexDir,
        List<? extends File> files)
            throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException,
            InvocationTargetException, NoSuchMethodException, IOException, SecurityException,
            ClassNotFoundException, InstantiationException {
        if (!files.isEmpty()) {
            if (Build.VERSION.SDK_INT >= 19) {
                V19.install(loader, files, dexDir);
            } else if (Build.VERSION.SDK_INT >= 14) {
                V14.install(loader, files);
            } else {
                V4.install(loader, files);
            }
        }
    }

```

不同版本，加载的方式不同，所以根据`sdk`版本选择不同的加载方式。因为`5.0`以上不需要加载，所以只需要到`19`即可。


虽然加载的方式不同，但核心的就是将每一个`.zip`转化为`DexPathList.Element`，添加到`DexPathList`的`elements`字段中。


