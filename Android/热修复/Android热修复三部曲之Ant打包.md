## Android 热修复三部曲之基本的Ant打包

热修复从2015年开始，逐渐的被推广开来，现在已经是比较热门的技术。

当Android发布的Apk中，因为有个bug，导致程序一直崩溃。如果此时发布版本，时间间隔太短，则会导致用户的使用繁琐，导致用户的流逝。而热修复达到的目的便是在不发布版本的情况下，动态修改其中包含bug的类，实现替换，达到修复bug 的目的。


现在市面上有一些开源的热修复，例如androidFix等等。对于这些开源框架不做解释，网上也有类似的文章。在这里，我们从0开始，自己编写脚本与代码实现热修复。

在使用热修复之前，需要一些必备的基础，这些知识是最终结果的铺垫。网上也有一些例子等，但只是一知半解，按照一些例子去最终实现，需要踩很多的坑。别问我为什么知道，我一路踩过来的。。。

本系列文章将分为三篇博客：

- Android热修复三部曲之基本的Ant打包脚本
- Android热修复三部曲之MultiDex分包架构的实现
- Android热修复三部曲之动态加载dex 

该篇博客将给予Eclipse进行设计实现


### 什么是Ant

Ant是一个将软件编译、测试、部署等步骤联系在一起加以自动化的一个工具。而对于Android来说，我们的项目最终打包成Apk，就是通过Ant进行构建的。不过，平常这些工作Eclipse已经帮我们实现了。

在安装的SDK的tools/ant目录下，有默认实现的`build.xml`文件，如果有兴趣的可以去进行研究，不过代码比较多。

对于Android 中，Ant 最重要的便是编写`build.xml`文件，该文件确定执行的具体流程，当然，平常我们之所以没有是因为上面提到的缺省的`build.xml`文件。

既然`build.xml`确定了编译流程，那么对于一个我们编写好的工程，是怎么最后成我们的apk文件，这个流程我们需要先明确。


### Android 打包流程

Android 打包流程主要分为以下几个步骤

**1.初始化配置**

该配置主要包括删除之前的临时文件，生成临时文件的目录等等。

**2.编译生成 R.java文件**

Android 中R文件是最重要的java 文件，他沟通了资源文件与java 代码之间的相互调用。

使用SDK中提供的aapt.exe工具，对工程中的资源文件进行编译，生成R文件，aapt.exe位于SDK 目录/build-tools\22.0.1目录下。在这里可能会有疑问，`Eclipse`中，我们的工程已经自动生成了R文件。但是，这只是`Eclipse`展示的效果，而实际的工程目录中，并没有R.java 文件的生成。

在这里有一个坑，便是依赖工程的存在。如果我们的工程依赖了别的工程，例如最常见的`appcompat_v7`,那么生成R.java 时，势必要进行相互的联系，在生成R.java文件时，需要注意。

**3.编译工程中的 java 代码**

java 基础好的知道，在平常运行中，都会讲.java文件编译为.class 文件，而此步的目的便是将src目录下的.java文件以及R.java 文件，编译成.class文件.

此时注意的是:如果有依赖工程，则依赖工程中的java代码也需要编译。

使用javac 命令

**4.对.class 文件进行编译生成.dex文件**

如果反编译过apk，会发现里面存在着一个classes.dex文件，该文件便是我们java代码的最终存在形式，此步骤的目的便是将上一步.class 文件整体编译为.dex文件。

使用dx.bat 命令。

**5.压缩资源文件**

对图片，布局等res文件夹下的资源文件进行压缩，如果反编译apk，会发现总有一个.arsc文件，那边是资源文件。

此时注意的是需要将依赖工程的资源也需要一起压缩。

使用命令 aapt.exe。

**6.打出未签名的包**

将如上产生的.arsc资源文件与.dex文件一起打包，生成apk文件，此时apk未签名。

在这里，我遇到一个大坑，我以为此时生成的未签名的apk已经能够运行了，但是，运行之后提示解析包出错。实际上，他和Eclipse中生成的未签名包不一样。

apk为什么要签名，我们了解的最多的，一个凭证。安装是会检验是否是正版。但其实，他还有另一层作用，对apk中的文件进行签名，保证资源的不可修改，如果修改之后，需要重新签名。如果没有该步，则会导致解析失败。

通过这可以看出，实际上`Eclipse`生成的无签名的apk应该也是有签名的，不过不是我们指定的签名而已。（我猜的。。。）

使用命令 apkBuilder ，当然此命令在高版本已删除。我们可以通过另一种方式实现。

**7.对Apk进行签名的**

这个应该不用多说了，网上也有很多的签名工具。这里通过`jarsigner.exe`进行签名，该命令是jdk所提供的命令。


### Ant脚本基础知识

在Ant脚本中有两个关键标签

- `<project>`:跟节点，作为整个工程的基础。
- `<target>`:一个又一个流程。类似上面所说的步骤。



首先，在工程的根目录下：建立`build.xml`,该目录与清单文件统计，同时添加`<project>`节点。


```xml
<project
    name="multiDex"
    default="a"

	
	<target name="b"
		>

	</target>

	<target name="a"
		depends="b">

	</target>

</project>
```

- `name`:此构建脚本的工程名。随便起。
- `default`: 默认执行那个`<target>`

在这里他的实际执行顺序为：

- 先通过`project`的`default`属性找到`a`.
- 发现`a`的执行依赖`b`,所以找到`b`
- 最终，执行`b`之后执行`a`。



### 编写Ant 自定义打包流程

该流程将按照如上的步骤进行讲解。最后会贴出所有代码。该工程打包时，依赖`appcompat_v7`库。


**1.初始化配置**

```xml 
   <!-- 代表当前目录 -->
      <property
        name="project-dir"
        value="." />
      
      <!-- v7工程目录 -->
      <property
          name="appcompat-dir"
          value="D:\eclipse\code\appcompat_v7"/>
      
      
    <target name="init">
        <!-- 提示信息，类似printf(); -->
        <echo>Init</echo>
        <!-- 删除文件，创建文件目录 -->
        <delete dir="${project-dir}\bin"/>
        <delete dir="${project-dir}\gen"/>
        <mkdir  dir ="${project-dir}\bin"/>
        <mkdir dir = "${project-dir}\gen"/>
        <mkdir dir = "${project-dir}\bin/classes"/>
    </target>

```

- `<property>`:该节点的作用表示声明一个别名。类似于变量名。
- `<echo>`： 打印提示信息。类似`printf()`。
- `<delete>` ： 删除文件或文件目录及其下的所有文件。
- `<mkdir>` : 创建文件目录

该步骤，删除之前编译时生成的gen目录（存放R.java）和bin目录（临时文件目录）。并创建新的目录。同时创建`bin\classes`目录，用以存放后面步骤生成的.class文件。


**2.编译生成 R.java文件**

```xml
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
          value="${appcompat-dir}\AndroidManifest.xml"
          />

    <target name="GenR"
        depends="init">
        <!-- 生成工程的R.java 文件 -->
        <echo>gen project  R.java </echo>
        <exec executable="${tools.aapt}" failonerror="true">
            <arg value="package"/>
            <arg value="-m"/>
            <arg value = "-J"/>
            <arg value="${project-dir}\gen"/>
            <arg value = "-M"/>
            <arg value="${manifest}"/>
            <arg value="-S"/>
            <arg value="${project-dir}\res"/>
            <arg value="-S"/>
            <arg value="${appcompat-dir}\res"/>
            <arg value="-I"/>
            <arg value="${android-jar}"  />
            <arg value="--auto-add-overlay"/>          
        </exec>
         <!-- 生成依赖库的R.java 文件 -->
         <echo>gen appcompat  R.java </echo>
        <exec executable="${tools.aapt}" failonerror="true">
            <arg value="package"/>
            <arg value="-m"/>
            <arg value = "-J"/>
            <arg value="${project-dir}\gen"/>
            <arg value = "-M"/>
            <arg value="${appcompat.manifest}"/>
            <arg value="-S"/>
            <arg value="${project-dir}\res"/>
            <arg value="-S"/>
            <arg value="${appcompat-dir}\res"/>
            <arg value="-I"/>
            <arg value="${android-jar}"  />
            <arg value="--auto-add-overlay"/>          
        </exec>
    </target> 
```

在执行命令时：按照顺许分别代表的意思：

- `-m`:使生成的包的目录存放在`-J`参数指定的目录
- `-J`:指定生成的.java 存放的目录.
- `-M`:指定清单文件的路径，之所以需要清单文件的目的是为了获取到包名。
- `-S`：res存放的目录。
- `-I`：某个版本平台的android.jar的路径
- `--auto-add-overlay`：覆盖资源，必须加上。


可以看到，分别对工程和依赖库进行编译。因为工程依赖了v7库，两者中资源有调用，所以在添加`res`存放的目录时，必须都添加。

**3.编译工程中的 java 代码**

上一步中，在bin目录中生成了R.java 文件。此步便是生成.class 文件保存到bin/classes目录中。


```xml 
   <!-- 编译工程中的src 中的代码
    		因为我配置了java 环境变量，所以直接使用了javac
     -->
      <target name="compile" 
        depends="GenR"
        >
         <echo>Compiling java source code...</echo>
         <!-- 编译依赖库的java文件 -->
         <javac encoding="UTF-8" destdir="${project-dir}/bin/classes" bootclasspath="${android-jar}">
             <src path="${appcompat-dir}/src"/>
             <src path="gen"/>
 			<classpath>
                <fileset dir="${appcompat-dir}/libs" includes="*.jar" /><!-- 第三方jar包需要引用，用于辅助编译 -->
            </classpath>             
         </javac>
         
         <!-- 编译工程的java文件 -->
         <javac encoding="UTF-8" destdir="${project-dir}/bin/classes" bootclasspath="${android-jar}">
             <src path="${project-dir}/src"/>
             <src path="gen"/>
 			<classpath>
                <fileset dir="${project-dir}/libs" includes="*.jar" /><!-- 第三方jar包需要引用，用于辅助编译 -->
            </classpath>  
            <classpath>
                <fileset dir="${appcompat-dir}/libs" includes="*.jar" /><!-- 第三方jar包需要引用，用于辅助编译 -->
            </classpath>           
         </javac>
    </target>

```
挑几个重点说： 

`destdir`：生成的.class的存放目录,保存在当前工程目录下bin/classes 目录下。

需要对依赖库和当前工程分别编译，生成.class文件。

`<classpath>`中包含了编译时需要的一些jar包，因为jar包后面可以直接打入.dex中，所以在此时作为一个辅助编译的作用。

对于当前工程，编译时也需要依赖`appcompat_v7`中的jar包，因为使用到了里面的类。


**4.对.class 文件进行编译生成.dex文件**

```xml 
   <!-- 将工程中的源码打成dex -->
    
     <property
        name="tools.dx"
        value="${platform-tools-folder}\dx.bat" />
    
    <target name="dex" depends="compile">
        <echo> build dex</echo>
        <exec executable="${tools.dx}" failonerror="true">
            <arg value="--dex"/>
            <arg value="--output=${project-dir}/bin/classes.dex"/>
            <arg value="${project-dir}/bin/classes"/>
            <arg value="${project-dir}/libs"/>
             <arg value="${appcompat-dir}/libs"/>
        </exec>
        
    </target>

```

将bin/classes，工程的jar，依赖库的jar统一打到bin/classes.dex中。

**5.压缩资源文件**

```xml 
 <!--  打包资源文件 -->
    <target name="build-res-and-assets"
        depends="dex"
        >
        <echo> build-res-and-assets </echo>
        <!-- 打包资源文件 -->
        <exec executable="${tools.aapt}" failonerror="true">
            <arg value="package"/>
            <arg value="-f"/> <!--  资源覆盖重写 -->
            <arg value="-M"/> <!-- 指定清单文件 -->
            <arg value="${manifest}"/>
            <arg value="-S"/> <!-- 指定资源路径 -->
            <arg value="res"/>
            <arg value="-S"/>
            <arg value="${appcompat-dir}/res"/>
            <arg value="-A"/>
            <arg value="assets"/>
            <arg value="-I"/>
            <arg value="${android-jar}"  />
            <arg value="-F" /><!-- 输出资源压缩包 -->
            <arg value="${project-dir}/bin/resources.arsc" />
            <arg value="--auto-add-overlay"/>     
        </exec>
    </target>

```

将工程和依赖库中的`res`，`assets`文件统一压缩到`bin/resources.arsc`中。

**6.打出未签名的包**

```xml 
 <!-- 打包 -->
    <target
        name="package" depends="build-res-and-assets"
        >
        <echo> package unsign  </echo>
          <java
            classname="com.android.sdklib.build.ApkBuilderMain"
            classpath="${sdk-folder}/tools/lib/sdklib.jar" >

            <arg value="${project-dir}/bin/unsign.apk" /> <!-- 未签名apk的生成路径  -->

            <arg value="-u" /> <!-- 创建一个未签名的包 -->

            <arg value="-z" />  <!-- 需要添加的压缩包 -->

            <arg value="${project-dir}/bin/resources.arsc" />

            <arg value="-f" /> <!-- 需要添加的文件-->

            <arg value="bin/classes.dex" />
        </java>
        
    </target>


```

打成的包是未签名的包。他和最终的apk的唯一区别，通过解压会发现未签名的包中没有`META-INF`文件，该文件主要负责解析，以及文件的完整性等。



**7.对Apk进行签名的**
```xml 
    <!-- jdk 路径 -->
      <property
        name="jdk-folder"
        value="C:\Program Files\Java\jdk1.7.0_79" />
      <!-- 签名文件路径 -->
      <property
        name="tools.jarsigner"
        value="${jdk-folder}\bin\jarsigner.exe" />
      
    <target
        name="sign-apk"
        depends="package" >
        <echo>
  			Sign apk
        </echo>
        <exec
            executable="${tools.jarsigner}"
            failonerror="true" >
            <arg value="-keystore" />
            <arg value="${project-dir}/my.keystore" />
            <arg value="-storepass" />
            <arg value="123456" /> <!--仓库密码  -->
            <arg value="-keypass" />
            <arg value="123456" /> <!-- 秘钥的密码 -->
            <arg value="-signedjar" />
            <arg value="${project-dir}/bin/sign.apk" />
            <arg value="${project-dir}/bin/unsign.apk" />
            <arg value="ant_test" /> <!-- 秘钥的别名 -->
        </exec>
    </target>
```

### 完整的Ant 编译脚本代码

```xml 

<project
    name="multiDex"
    default="sign-apk"
    >
      <!-- 代表当前目录 -->
      <property
        name="project-dir"
        value="." />
      
      <!-- v7工程目录 -->
      <property
          name="appcompat-dir"
          value="D:\eclipse\code\appcompat_v7"/>
      
      
    <target name="init">
        <!-- 提示信息，类似printf(); -->
        <echo>Init</echo>
        <!-- 删除文件，创建文件目录 -->
        <delete dir="${project-dir}\bin"/>
        <delete dir="${project-dir}\gen"/>
        <mkdir  dir ="${project-dir}\bin"/>
        <mkdir dir = "${project-dir}\gen"/>
        <mkdir dir = "${project-dir}\bin/classes"/>
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
          value="${appcompat-dir}\AndroidManifest.xml"
          />
     
    <target name="GenR"
        depends="init">
        <!-- 生成工程的R.java 文件 -->
        <echo>gen project  R.java </echo>
        <exec executable="${tools.aapt}" failonerror="true">
            <arg value="package"/>
            <arg value="-m"/>
            <arg value = "-J"/>
            <arg value="${project-dir}\gen"/>
            <arg value = "-M"/>
            <arg value="${manifest}"/>
            <arg value="-S"/>
            <arg value="${project-dir}\res"/>
            <arg value="-S"/>
            <arg value="${appcompat-dir}\res"/>
            <arg value="-I"/>
            <arg value="${android-jar}"  />
            <arg value="--auto-add-overlay"/>          
        </exec>
         <!-- 生成依赖库的R.java 文件 -->
         <echo>gen appcompat  R.java </echo>
        <exec executable="${tools.aapt}" failonerror="true">
            <arg value="package"/>
            <arg value="-m"/>
            <arg value = "-J"/>
            <arg value="${project-dir}\gen"/>
            <arg value = "-M"/>
            <arg value="${appcompat.manifest}"/>
            <arg value="-S"/>
            <arg value="${project-dir}\res"/>
            <arg value="-S"/>
            <arg value="${appcompat-dir}\res"/>
            <arg value="-I"/>
            <arg value="${android-jar}"  />
            <arg value="--auto-add-overlay"/>          
        </exec>
    </target>
    
    <!-- 编译工程中的src 中的代码
    		因为我配置了java 环境变量，所以直接使用了javac
     -->
    <target name="compile" 
        depends="GenR"
        >
         <echo>Compiling java source code...</echo>
         <!-- 编译依赖库的java文件 -->
         <javac encoding="UTF-8" destdir="${project-dir}/bin/classes" bootclasspath="${android-jar}">
             <src path="${appcompat-dir}/src"/>
             <src path="gen"/>
 			<classpath>
                <fileset dir="${appcompat-dir}/libs" includes="*.jar" /><!-- 第三方jar包需要引用，用于辅助编译 -->
            </classpath>             
         </javac>
         
         <!-- 编译工程的java文件 -->
         <javac encoding="UTF-8" destdir="${project-dir}/bin/classes" bootclasspath="${android-jar}">
             <src path="${project-dir}/src"/>
             <src path="gen"/>
 			<classpath>
                <fileset dir="${project-dir}/libs" includes="*.jar" /><!-- 第三方jar包需要引用，用于辅助编译 -->
            </classpath>  
            <classpath>
                <fileset dir="${appcompat-dir}/libs" includes="*.jar" /><!-- 第三方jar包需要引用，用于辅助编译 -->
            </classpath>           
         </javac>
    </target>
    
    <!-- 将工程中的源码打成dex -->
    
     <property
        name="tools.dx"
        value="${platform-tools-folder}\dx.bat" />
    
    <target name="dex" depends="compile">
        <echo> build dex</echo>
        <exec executable="${tools.dx}" failonerror="true">
            <arg value="--dex"/>
            <arg value="--output=${project-dir}/bin/classes.dex"/>
            <arg value="${project-dir}/bin/classes"/>
            <arg value="${project-dir}/libs"/>
             <arg value="${appcompat-dir}/libs"/>
        </exec>
        
    </target>
    
    <!--  打包资源文件 -->
    <target name="build-res-and-assets"
        depends="dex"
        >
        <echo> build-res-and-assets </echo>
        <!-- 打包资源文件 -->
        <exec executable="${tools.aapt}" failonerror="true">
            <arg value="package"/>
            <arg value="-f"/> <!--  资源覆盖重写 -->
            <arg value="-M"/> <!-- 指定清单文件 -->
            <arg value="${manifest}"/>
            <arg value="-S"/> <!-- 指定资源路径 -->
            <arg value="res"/>
            <arg value="-S"/>
            <arg value="${appcompat-dir}/res"/>
            <arg value="-A"/>
            <arg value="assets"/>
            <arg value="-I"/>
            <arg value="${android-jar}"  />
            <arg value="-F" /><!-- 输出资源压缩包 -->
            <arg value="${project-dir}/bin/resources.arsc" />
            <arg value="--auto-add-overlay"/>     
        </exec>
    </target>
    
    <!-- 打包 -->
    <target
        name="package" depends="build-res-and-assets"
        >
        <echo> package unsign  </echo>
          <java
            classname="com.android.sdklib.build.ApkBuilderMain"
            classpath="${sdk-folder}/tools/lib/sdklib.jar" >

            <arg value="${project-dir}/bin/unsign.apk" /> <!-- 未签名apk的生成路径  -->

            <arg value="-u" /> <!-- 创建一个未签名的包 -->

            <arg value="-z" />  <!-- 需要添加的压缩包 -->

            <arg value="${project-dir}/bin/resources.arsc" />

            <arg value="-f" /> <!-- 需要添加的文件-->

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
      
    <target
        name="sign-apk"
        depends="package" >
        <echo>
  			Sign apk
        </echo>
        <exec
            executable="${tools.jarsigner}"
            failonerror="true" >
            <arg value="-keystore" />
            <arg value="${project-dir}/my.keystore" />
            <arg value="-storepass" />
            <arg value="123456" /> <!--仓库密码  -->
            <arg value="-keypass" />
            <arg value="123456" /> <!-- 秘钥的密码 -->
            <arg value="-signedjar" />
            <arg value="${project-dir}/bin/sign.apk" />
            <arg value="${project-dir}/bin/unsign.apk" />
            <arg value="ant_test" /> <!-- 秘钥的别名 -->
        </exec>
    </target>
</project>

```