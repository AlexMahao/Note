## Android热修复三部曲之MultiDex 分包架构

> 转载请标明出处：
[http://blog.csdn.net/lisdye2/article/details/52049857](http://blog.csdn.net/lisdye2/article/details/52049857)
本文出自:[【Alex_MaHao的博客】](http://blog.csdn.net/lisdye2?viewmode=contents)
项目中的源码已经共享到github，有需要者请移步[【Alex_MaHao的github】](https://github.com/AlexSmille/Android-hot-fix)


本篇博客基于上一篇博客，[Android 热修复三部曲之基本的Ant打包脚本](http://blog.csdn.net/lisdye2/article/details/52049857)

在上一篇博客中，讲解了使用Ant打包的流程，也编写了相应的脚本代码。但是忘了说明怎么运行了。有两种方式：

- 在Eclipse的build.xml 中，右键run as 即可。
- 通过命令行形式进入到当前工程目录下，输入命令`ant 工程名`。


上一篇漏下的已经补充。下面开始今天博客的主要内容。

该篇博客主要分为三个部分：

- **什么是分包架构**
- **分包架构的好处**
- **怎么实现分包**

### 什么是分包架构

如果反编译过apk的同学知道，在apk中存在这一个`classes.dex`的文件，该文件主要是存放了经过编译后的java代码。那么我们通过一定的手段，在使用`dx.bat`打包时，将`.class`文件打成多个`.dex`文件，便是分包。

### 分包架构的好处

分包有两个好处：

**1.解决65536的错误**

我们知道Android 应用的方法数不能超过65536个。也就是一个`.dex`文件，起内包含的方法数不能超过65536。这是因为其使用了一个整形来建立相应方法的索引。

**2.实现程序的热修复**

我们将程序分模块打成了多个`.dex`，如果某一个`.dex`出现了问题，我们可以对这个出现问题的`.dex`重新编译出无bug的`.dex`。然后从服务器下载之后，动态的将新的`.dex`加入到内存中，覆盖之前的有问题的`.dex`，变实现了不发版本的`bug`的修复。

### 实现方式

在上一篇博客中，分析了打包的流程，再来复习一下：

**1.初始化配置**

**2.编译生成 R.java文件**

**3.编译工程中的 java 代码**

**4.对.class 文件进行编译生成.dex文件**

**5.压缩资源文件**

**6.打出未签名的包**

**7.对Apk进行签名的**

那么我们在第4步的时候做出修改，根据我们的需求打出多个`.dex`。在第6步，打出未签名包时，因为其命令中，只能添加一个`.dex`，所以需要使用`aapt`命令，将其余的`.dex`文件放入其中。


开始实现代码：

**在第3步完成之后，修改代码如下**

```xml  
<target
        name="dex"
        depends="compile" >

        <echo>
            Generate multi-dex...
        </echo>
        <!-- 分为三个部分  jar  一部分，本地 -->

        <exec
            executable="${tools.dx}"
            failonerror="true" >

            <arg value="--dex" />

            <arg value="--multi-dex" />

            <arg value="--set-max-idx-number=10000" />

            <arg value="--minimal-main-dex" />

            <arg value="--main-dex-list" />

            <arg value="main-dex-rule.txt" />

            <arg value="--output=bin" />

            <arg value="bin/classes" />
        </exec>

        <echo>
            Generate dex...  
        </echo>

        <exec
            executable="${tools.dx}"
            failonerror="true" >

            <arg value="--dex" />

            <arg value="--output=bin/classes3.dex" />

            <arg value="${project-dir}/libs" />
            <!-- classes文件位置 -->

            <arg value="${appcompat-dir}\libs" />
        </exec>
    </target>

```

分别执行了两次dx命令，第一次，将`src`中的类和R文件，打包成了`classes.dex`和`classes2.dex`。`main-dex-rule.txt`文件中表示那些类会打入到主的`.dex`文件中（`classes.dex`）。

**main-dex-rule.txt**文件中就添加了两个.class。

```java 
com/alex_mahao/multidex/MainActivity.class
com/alex_mahao/multidex/MyApp.class
```

第二次将jar包打成一个单独的`classes3.dex`文件。




在这里，分成了三个`.dex`。
- 程序的主要类，如`Application`，`MainActivity`等打入到`classes.dex`，该程序要保证不会出错，一般就打入几个必要的类。
- 业务逻辑类。这些可能会出错的类，以便于修复。
- jar 包。


之所以将业务逻辑类和jar包类分开，是因为jar包基本上不会变，而如果热修复时加上jar包一起替换，加大了内存的负担。


在执行完第6步操作之后，开始将`classes2.dex`和`classes3.dex`导入到包中。

复制`.dex`文件。

``` xml 
    <target
        name="copy_dex"
        depends="package" >

        <echo message="copy dex..." />

        <copy todir="${project-dir}" >

            <fileset dir="bin" >
                <include name="classes*.dex" />
            </fileset>
        </copy>
    </target>

```

拷贝文件到项目的根目录下面，因为我们的脚本是在根目录下面，这样在运行aapt的时候，可以直接操作dex文件了.


开始导入
```xml 

<target
        name="add-subdex-toapk"
        depends="copy_dex" >

        <echo message="Add Subdex to Apk ..." />

        <foreach
            param="dir.name"
            target="aapt-add-dex" >

            <path>
                <fileset
                    dir="bin"
                    includes="classes*.dex" />
            </path>
        </foreach>
    </target>

    <!-- 使用aapt命令添加dex文件 -->

    <target name="aapt-add-dex" >

        <echo message="${dir.name}" />
        <!-- 使用正则表达式获取classes的文件名 -->

        <propertyregex
            casesensitive="false"
            input="${dir.name}"
            property="dexfile"
            regexp="classes(.*).dex"
            select="\0" />
        <!-- 这里不需要添加classes.dex文件 -->

        <if>

            <equals
                arg1="${dexfile}"
                arg2="classes.dex" />

            <then>

                <echo>
			${dexfile} is not handle

                </echo>
            </then>

            <else>

                <echo>
					${dexfile} is handle

                </echo>

                <exec
                    executable="${tools.aapt}"
                    failonerror="true" >

                    <arg value="add" />

                    <arg value="bin/unsign.apk" />

                    <arg value="${dexfile}" />
                </exec>
            </else>
        </if>

        <delete file="${project-dir}\${dexfile}" />
    </target>

```

在这里因为用到了if等条件判断语句，所以需要往我们的ant工具中手动添加一个jar包。该jar包的下载地址：[http://download.csdn.net/detail/lisdye2/9592706](http://download.csdn.net/detail/lisdye2/9592706) 下载此jar包之后，将此jar包添加到`D:\apache-ant-1.9.7\lib`目录下即可。


最后，附上完整的`build.xml`代码。

```xml 
<project
    name="multiDex"
    default="sign-apk" >

    <taskdef
        classpath="D:\apache-ant-1.9.7\lib\ant-contrib-1.0b3.jar"
        resource="net/sf/antcontrib/antlib.xml" />

    <!-- 代表当前目录 -->

    <property
        name="project-dir"
        value="." />

    <!-- v7工程目录 -->

    <property
        name="appcompat-dir"
        value="D:\eclipse\code\appcompat_v7" />

    <target name="init" >

        <!-- 提示信息，类似printf(); -->

        <echo>
Init




        </echo>
        <!-- 删除文件，创建文件目录 -->

        <delete dir="${project-dir}\bin" />

        <delete dir="${project-dir}\gen" />

        <mkdir dir="${project-dir}\bin" />

        <mkdir dir="${project-dir}\gen" />

        <mkdir dir="${project-dir}\bin/classes" />
    </target>

    <!-- 生成R.java 文件。 -->


    <!-- Sdk 的根目录 -->

    <property
        name="sdk-folder"
        value="D:\eclipse\sdk" />

    <!-- 编译工具的目录 -->

    <property
        name="platform-tools-folder"
        value="${sdk-folder}\build-tools\22.0.1" />

    <!-- aapt 命令详细的目录 -->

    <property
        name="tools.aapt"
        value="${platform-tools-folder}\aapt.exe" />

    <property
        name="platform-folder"
        value="${sdk-folder}\platforms\android-22" />

    <!-- android.jar 的路径 -->

    <property
        name="android-jar"
        value="${platform-folder}\android.jar" />

    <!-- 当前工程的清单文件 -->

    <property
        name="manifest"
        value="${project-dir}\AndroidManifest.xml" />

    <!-- 依赖库的清单文件 -->

    <property
        name="appcompat.manifest"
        value="${appcompat-dir}\AndroidManifest.xml" />

    <target
        name="GenR"
        depends="init" >

        <!-- 生成工程的R.java 文件 -->

        <echo>
gen project  R.java 




        </echo>

        <exec
            executable="${tools.aapt}"
            failonerror="true" >

            <arg value="package" />

            <arg value="-m" />

            <arg value="-J" />

            <arg value="${project-dir}\gen" />

            <arg value="-M" />

            <arg value="${manifest}" />

            <arg value="-S" />

            <arg value="${project-dir}\res" />

            <arg value="-S" />

            <arg value="${appcompat-dir}\res" />

            <arg value="-I" />

            <arg value="${android-jar}" />

            <arg value="--auto-add-overlay" />
        </exec>
        <!-- 生成依赖库的R.java 文件 -->

        <echo>
gen appcompat  R.java 




        </echo>

        <exec
            executable="${tools.aapt}"
            failonerror="true" >

            <arg value="package" />

            <arg value="-m" />

            <arg value="-J" />

            <arg value="${project-dir}\gen" />

            <arg value="-M" />

            <arg value="${appcompat.manifest}" />

            <arg value="-S" />

            <arg value="${project-dir}\res" />

            <arg value="-S" />

            <arg value="${appcompat-dir}\res" />

            <arg value="-I" />

            <arg value="${android-jar}" />

            <arg value="--auto-add-overlay" />
        </exec>
    </target>

    <!--
         编译工程中的src 中的代码
    		因为我配置了java 环境变量，所以直接使用了javac
    -->

    <target
        name="compile"
        depends="GenR" >

        <echo>
		Compiling java source code...
        </echo>
        <!-- 编译依赖库的java文件 -->

        <javac
            bootclasspath="${android-jar}"
            destdir="${project-dir}/bin/classes"
            encoding="UTF-8" >

            <src path="${appcompat-dir}/src" />

            <src path="gen" />

            <classpath>

                <fileset
                    dir="${appcompat-dir}/libs"
                    includes="*.jar" />
                <!-- 第三方jar包需要引用，用于辅助编译 -->
            </classpath>
        </javac>

        <!-- 编译工程的java文件 -->

        <javac
            bootclasspath="${android-jar}"
            destdir="${project-dir}/bin/classes"
            encoding="UTF-8" >

            <src path="${project-dir}/src" />

            <src path="gen" />

            <classpath>

                <fileset
                    dir="${project-dir}/libs"
                    includes="*.jar" />
                <!-- 第三方jar包需要引用，用于辅助编译 -->
            </classpath>

            <classpath>

                <fileset
                    dir="${appcompat-dir}/libs"
                    includes="*.jar" />
                <!-- 第三方jar包需要引用，用于辅助编译 -->
            </classpath>
        </javac>
    </target>

    <!-- 将工程中的源码打成dex -->

    <property
        name="tools.dx"
        value="${platform-tools-folder}\dx.bat" />

    <target
        name="dex"
        depends="compile" >

        <echo>
            Generate multi-dex...



        </echo>
        <!-- 分为三个部分  jar  一部分，本地 -->

        <exec
            executable="${tools.dx}"
            failonerror="true" >

            <arg value="--dex" />

            <arg value="--multi-dex" />

            <arg value="--set-max-idx-number=10000" />

            <arg value="--minimal-main-dex" />

            <arg value="--main-dex-list" />

            <arg value="main-dex-rule.txt" />

            <arg value="--output=bin" />

            <arg value="bin/classes" />
        </exec>

        <echo>
            Generate dex...  
        </echo>

        <exec
            executable="${tools.dx}"
            failonerror="true" >

            <arg value="--dex" />

            <arg value="--output=bin/classes3.dex" />

            <arg value="${project-dir}/libs" />
            <!-- classes文件位置 -->

            <arg value="${appcompat-dir}\libs" />
        </exec>
    </target>

    <!-- 打包资源文件 -->

    <target
        name="build-res-and-assets"
        depends="dex" >

        <echo>
 	build-res-and-assets 




        </echo>
        <!-- 打包资源文件 -->

        <exec
            executable="${tools.aapt}"
            failonerror="true" >

            <arg value="package" />

            <arg value="-f" />
            <!-- 资源覆盖重写 -->

            <arg value="-M" />
            <!-- 指定清单文件 -->

            <arg value="${manifest}" />

            <arg value="-S" />
            <!-- 指定资源路径 -->

            <arg value="res" />

            <arg value="-S" />

            <arg value="${appcompat-dir}/res" />

            <arg value="-A" />

            <arg value="assets" />

            <arg value="-I" />

            <arg value="${android-jar}" />

            <arg value="-F" />
            <!-- 输出资源压缩包 -->

            <arg value="${project-dir}/bin/resources.arsc" />

            <arg value="--auto-add-overlay" />
        </exec>
    </target>

    <!-- 打包 -->

    <target
        name="package"
        depends="build-res-and-assets" >

        <echo>
 		package unsign  

        </echo>

        <java
            classname="com.android.sdklib.build.ApkBuilderMain"
            classpath="${sdk-folder}/tools/lib/sdklib.jar" >

            <arg value="${project-dir}/bin/unsign.apk" />
            <!-- 未签名apk的生成路径 -->

            <arg value="-u" />
            <!-- 创建一个未签名的包 -->

            <arg value="-z" />
            <!-- 需要添加的压缩包 -->

            <arg value="${project-dir}/bin/resources.arsc" />

            <arg value="-f" />
            <!-- 需要添加的文件 -->

            <arg value="bin/classes.dex" />
        </java>
    </target>

    <!-- jdk 路径 -->

    <property
        name="jdk-folder"
        value="C:\Program Files\Java\jdk1.7.0_79" />
    <!-- 签名文件路径 -->

    <property
        name="tools.jarsigner"
        value="${jdk-folder}\bin\jarsigner.exe" />

  

    <!-- 拷贝文件到项目的根目录下面，因为我们的脚本是在根目录下面，这样在运行aapt的时候，可以直接操作dex文件了 -->

    <target
        name="copy_dex"
        depends="package" >

        <echo message="copy dex..." />

        <copy todir="${project-dir}" >

            <fileset dir="bin" >

                <include name="classes*.dex" />
            </fileset>
        </copy>
    </target>

    <target
        name="add-subdex-toapk"
        depends="copy_dex" >

        <echo message="Add Subdex to Apk ..." />

        <foreach
            param="dir.name"
            target="aapt-add-dex" >

            <path>
                <fileset
                    dir="bin"
                    includes="classes*.dex" />
            </path>
        </foreach>
    </target>

    <!-- 使用aapt命令添加dex文件 -->

    <target name="aapt-add-dex" >

        <echo message="${dir.name}" />
        <!-- 使用正则表达式获取classes的文件名 -->

        <propertyregex
            casesensitive="false"
            input="${dir.name}"
            property="dexfile"
            regexp="classes(.*).dex"
            select="\0" />
        <!-- 这里不需要添加classes.dex文件 -->

        <if>

            <equals
                arg1="${dexfile}"
                arg2="classes.dex" />

            <then>

                <echo>
			${dexfile} is not handle

                </echo>
            </then>

            <else>

                <echo>
					${dexfile} is handle

                </echo>

                <exec
                    executable="${tools.aapt}"
                    failonerror="true" >

                    <arg value="add" />

                    <arg value="bin/unsign.apk" />

                    <arg value="${dexfile}" />
                </exec>
            </else>
        </if>

        <delete file="${project-dir}\${dexfile}" />
    </target>

    
      <target
        name="sign-apk"
        depends="add-subdex-toapk" >

        <echo>
  			Sign apk
        </echo>

        <exec
            executable="${tools.jarsigner}"
            failonerror="true" >

            <arg value="-keystore" />

            <arg value="${project-dir}/my.keystore" />

            <arg value="-storepass" />

            <arg value="123456" />
            <!-- 仓库密码 -->

            <arg value="-keypass" />

            <arg value="123456" />
            <!-- 秘钥的密码 -->

            <arg value="-signedjar" />

            <arg value="${project-dir}/bin/sign.apk" />

            <arg value="${project-dir}/bin/unsign.apk" />

            <arg value="ant_test" />
            <!-- 秘钥的别名 -->
        </exec>
    </target>
    
</project>

```