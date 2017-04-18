

### 怎么声明函数

声明函数的方式如下：

```
函数名 () {
    函数体
}
```
或

```
function 函数名 {
    函数体
}
```

可以看到，函数的声明并没有任何传入参数的方式，那么如何传入参数呢，后面再说。

> 注意：函数的声明必须放在调用之前。

编写一个简单的例子：

```
#!/bin/bash

run_fun(){
    echo "I am run function"
}

run_fun
run_fun

```

运行结果如下

```
[root@iZ2zebizp6le568407aeayZ shell]# ./fun1.sh 
I am run function
I am run function

```


### 传递参数

根据方法的声明，发现无法声明传入参数，那么如何传入参数呢。

其实传入参数的获取和之前获取运行脚本是获取参数的方式是相同的，使用如下：

```
#!/bin/bash

run_fun(){
    echo "I am run function $1"
}

run_fun 1
run_fun 2

```
在方法中通过`$1`获取参数，而在使用方法时，在后面添加对应的参数，以空格隔开。

运行结果如下：

```
[root@iZ2zebizp6le568407aeayZ shell]# ./fun2.sh
I am run function 1
I am run function 2
```

具体获取参数的细节，可以看我之前的文章[Linux学习之shell脚本的变量](http://blog.csdn.net/lisdye2/article/details/70148181)。

### 返回值

先看一个例子：

```
#!/bin/bash

run_fun(){
    echo "I am run function"
    return 3
}

run_fun
run_fun

echo $?

```

首先看一下`return 3`这条语句，在shell中，方法的返回值是固定的数值[0,255]。不能返回其他类型的数据。

其次看`$?`，该方式获取最后一次被运行的命令或者函数。

或者我们也可以通过`num=$(run_fun 1 2)`获取返回值，因为shell的`()`括号中，可以使用命令，我们可以把方法当做是命令，后跟参数。

### 变量的作用域

在其他语言中，变量统称分为全局变量和局部变量。shell也是如此。

在shell中定义的变量，默认都是全局变量。

要定义一个局部变量，我们可以在变量的声明赋值时，前加`local`作为标识。


### 重载命令

我们可以使用函数来实现命令的重载，也就是说把函数的名字定义的和命令的名字相同。

看如下例子：

```
#!/bin/bash

ls(){
    command ls -lh
}

ls

```

可以看到我们重载了`ls`命令，注意，在调用命令行命令时，及此函数的实现部分，一定要加上`command`关键字，不然会陷入无限循环。

看一下运行结果：

```
[root@iZ2zebizp6le568407aeayZ shell]# ./fun4.sh
total 48K
-rwxr-xr-x 1 root root 130 Apr 12 22:51 condition_case.sh
-rwxr-xr-x 1 root root 105 Apr 12 22:30 condition.sh
-rwxr-xr-x 1 root root  89 Apr 12 23:06 for.sh
-rwxr-xr-x 1 root root  73 Apr 12 23:17 fun1.sh
-rwxr-xr-x 1 root root 117 Apr 12 23:22 fun2.sh
-rwxr-xr-x 1 root root  95 Apr 12 23:27 fun3.sh
-rwxr-xr-x 1 root root  45 Apr 12 23:37 fun4.sh
-rwxr-xr-x 1 root root 152 Apr 12 10:50 read_variable.sh
-rwxr-xr-x 1 root root  82 Apr 11 23:14 test.sh
-rwxr-xr-x 1 root root 101 Apr 12 11:03 variable_array.sh
-rwxr-xr-x 1 root root 137 Apr 12 10:58 variable_porams.sh
-rwxr-xr-x 1 root root 168 Apr 11 23:57 variable.sh
```

