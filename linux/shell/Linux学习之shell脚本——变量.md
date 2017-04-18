

### echo：显示内容

在学习变量之前先了解一个命令`echo`，该命令类似于c中的`print`，在控制台打印消息。

输入`echo Hello World`命令，结果如下

```
[root@iZ2zebizp6le568407aeayZ shell]# echo Hello World
Hello World
```
在这里，echo实际上接收了两个参数`Hello`和`World`并显示。

输入`echo 'Hello world'`命令，

```
[root@iZ2zebizp6le568407aeayZ shell]# echo 'Hello World'
Hello World
```

打印虽然一样，但此时`echo`只接收了一个参数。

可以分别在两条命令的`Hello`和`World`中间添加多个空格，以此验证。

### 定义变量

输入命令`vim variable.sh`然后编写脚本内容如下

```
#!/bin/bash

# 定义变量
message='Hello world'

# 打印变量
echo $message
```

在定义变量时，我们不需要添加`$`符号,但要使用此变量，一定要添加`$`符号。


然后保存该脚本文件并添加执行权限。关于创建、保存和添加权限的细节，如果不了解的请看我上一篇博客[]()


运行脚本`./variable.sh`，结果如下：

```
[root@iZ2zebizp6le568407aeayZ shell]# ./variable.sh
Hello world
```

**注意**：在编写的过程中，一些有着格式化习惯的人会在=两边添加空格，如下`message = 'xxx'`，但在实际运行过程中会将message当做命令去执行，报如下错误
``` 
[root@iZ2zebizp6le568407aeayZ shell]# ./variable.sh 
./variable.sh: line 4: message: command not found
```


### 引号的使用

在上面的程序中，编写时使用了`'`单引号，修改脚本程序如下：

```
#!/bin/bash

# 定义变量
message='Hello world'

# 打印变量
echo 'message is $message'
```

修改了`echo`打印的内容，变量的引用放到了`'`单引号中，此时运行脚本，结果如下

```
[root@iZ2zebizp6le568407aeayZ shell]# ./variable.sh 
message is $message
```

What?变量为什么没有被替换呢？

因为在shell中，单引号忽略它括起来的所有特殊符号，也就是说此时的`$`仅仅表示一个字符，没有什么特殊的含义。

遇到这种情况怎么办呢，可以使用双引号替换单引号，因为双引号虽然也忽略大部分的特殊字符，但不包括美元符号`$`、反引号和反斜杠`\`。不忽略美元符号意味着变量可以被识别。那么修改代码如下

```
#!/bin/bash

# 定义变量
message='Hello world'

# 打印变量
echo "message is $message"
```

运行结果如下：

```
[root@iZ2zebizp6le568407aeayZ shell]# ./variable.sh 
message is Hello world
```

除了双引号和单引号，还有反引号，如何使用呢，我们继续修改代码

```
#!/bin/bash

# 定义变量
message='Hello world'

# 打印变量
echo "message is $message"

# 获取目录，反引号括起
dir=`pwd`

# 打印
echo “dir:$dir”

```

又定义了一个变量并赋值，其中值用反引号括住，最后打印一下。运行结果如下

```
[root@iZ2zebizp6le568407aeayZ shell]# ./variable.sh 
message is Hello world
“dir:/home/shell”
```
可以看到使用反括号括起来的值被当做命令去执行，并将执行的值赋给变量。

### read 请求输入

既然是程序，肯定就会有输入的情况，那么如何获取用户输入呢？

运行`vim read_variable.sh`并添加内容

```
#!/bin/bash

read name

echo "Hello $name"
```

添加权限后运行`./read_variable.sh`命令，结果如下

```
[root@iZ2zebizp6le568407aeayZ shell]# ./read_variable.sh 
111  
Hello 111
```
当程序运行之后，会等待，直到我们输入并敲击回车之后，才往下执行。

如果传入多个参数呢，修改脚本如下：

```
#!/bin/bash

read name password

echo "Hello $name --- $password"
```

运行脚本结果如下：

```
[root@iZ2zebizp6le568407aeayZ shell]# ./read_variable.sh 
1321 dadas das312
Hello 1321 --- dadas das312
```
对于一条`read`命令，获取的参数以空格分割，上面的输入中输入了三个参数，那么**多余的参数会累加到最后一个参数上**。


`read`命令同时提供了一些常用的参数：

- `-p`:显示提示信息。
- `-n`:限制字符的数目。如果输入的参数的字符数目已经达到限制的字符数，则会自动往下执行。
- `-t`:限制输入的时间，单位s。当达到时间之后，会自动执行命令。
- `-s`:隐藏输入，类似于常见的Linux输入密码的样式。

举个例子：

编写如下代码：

```
#!/bin/bash

read -p '请输入姓名(最大字符5):' -n 5 -t 5  name

echo "Hello !"

```

为`read`命令分别添加了提示输出的信息，字符的限制，以及限制输入时间。

执行的如下：

```
[root@iZ2zebizp6le568407aeayZ shell]# ./read_variable.sh 
请输入姓名(最大字符5):dsHello  !
```
输入了ds之后，并没有键入回车，程序等待5s之后继续执行，不过没有换行，我们可以在`echo`中加入`\n`。


到了这里，会发现我们输入的参数必须要根据提示一个个的输入，那么我们能不能在使用脚本的时候，直接在后面拼接上参数呢，当然是可以的。

看如下代码：

```
#!/bin/bash

echo "参数的数目：$#"

echo "运行脚本的名称：$0"

echo "第一个参数：$1"

echo "第二个参数：$2"

```

代码中结束的很清楚，如果想要获取第3,4,5...，依次往下累加即可。

运行一下代码：

```
[root@iZ2zebizp6le568407aeayZ shell]# ./variable_porams.sh one two three
参数的数目：3
运行脚本的名称：./variable_porams.sh
第一个参数：one
第二个参数：two

```


### 数组

shell中的数组的使用基本上和一些脚本语言使用上类似。

首先看代码：

```
#!/bin/bash

array=('0' '1' '2')
array[5]='5'
echo ${array[1]}
# 打印所有数组
echo ${array[*]}
```

数组的声明用`()`括起来。数组可以包含任意大小的元素数目，而且数组的元素编号不需要是连续的，可以略过一些格子。

> 注意：shell中数组的下标也是从0开始


运行一下脚本：

```
[root@iZ2zebizp6le568407aeayZ shell]# ./variable_array.sh 
1
0 1 2 5


```