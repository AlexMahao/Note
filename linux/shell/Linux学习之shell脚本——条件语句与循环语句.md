
### if条件语句

和其他语言类比，我们只需要了解`if`，`if...else`，`if..else if...else...`三种形式在shell中的使用。


在`shell`中，最基本的`if`的语法如下

```
if [ 条件 ]
then 
    符合条件的执行逻辑
fi
``` 

或
```
if [ 条件 ]; then
    符合条件的执行逻辑
fi
```

两种的区别在于`if`和`then`是否写在一行，如果写在一行使用`;`隔开。

`fi`表示`if`语句的结束，及代码范围。


> 注意：`[]`中的条件两边都有一个空格。

`if...else`的在shell中的写法如下：

```
if [ 条件 ]
then
    符合条件执行的逻辑
else
    不符合条件执行的逻辑
fi
```

`if...else if...else`在shell中的写法：

```
if [ 条件 1 ]
then
    做 1 的事情
elif [ 条件 2 ]
then
    做 2 的事情
elif [ 条件 3 ]
then
    做 3 的事情
else
    做其他事情
fi
```

最后看一个例子：

```
#!/bin/bash

if [ $1 = "one" ]
then
  echo "1"
elif [ $1 = "two" ]
then
  echo "2"
else
  echo "3"
fi 

```

在`if`判断的条件里，使用的是`=`号，这有区别于其他编程语言，但同样`shell`可以使用`==`号。注意`=`号两边有空格，如果不加空格，会认为是赋值操作。

很简单，就是判断输入，显示不同的结果，看一下执行结果

```
[root@iZ2zebizp6le568407aeayZ shell]# ./condition.sh  two
2
```

### 条件测试

`if`中关键的便是判断条件，那么可以做哪些条件判断呢？

**字符串**

- `$string1 = $string2`:两个字符串是否相等。
- `$string1 != $string2`:两个字符串是否不等。
- `-z $string`:字符串是否为null。
- `-n $string`:字符串是否不为null。

**数字**

- `$num1 -eq $num2`：判断两个数值是否相等，区别于`=`。`=`号是判断字符串是否相等。
- `$num1 -ne $num2`:判断两个数值是否不等。
- `$num1 -lt $num2`:判断num1是否小于num2。
- `$num1 -le $num2`:判断num1是否小于等于num2。
- `$num1 -gt $num2`:判断num1是否大于num2。
- `$num1 -ge $num2`:判断num1是否大于等于num2。

**测试文件**

- `-e $file`：文件是否存在。
- `-d $file`：文件是否是一个目录。
- `-f $file`：文件是否是一个文件。
- `-L $file`：文件是否是一个符号连接文件。
- `-r $file`：文件是否是可读的。
- `-w $file`：文件是否是可写的。
- `-x $file`：文件是否是可执行的。

**与，或，非判断条件**

- `&&`: 逻辑与。
- `||`: 逻辑或。
- `!` : 逻辑非。

注意：他们的使用方式不同。

与和或是以`[]`为一个整体，如下

```
if [ 条件 ] && [ 条件 ]
then 
    符合条件的执行逻辑
fi
```

而非的使用方式如下

```
if [ ! 条件 ]
then 
    符合条件的执行逻辑
fi
```


### case 多条件选择

在最初的`if`中编写了例子

```
if [ $1 = "one" ]
then
  echo "1"
elif [ $1 = "two" ]
then
  echo "2"
else
  echo "3"
fi 
```

可以将其修改为`case`语句：修改之后的如下

```
#!/bin/bash

case $1 in
    "one")
        echo "1"
        ;;
    "two")
        echo "2"
        ;;
    *)
	echo "3"
	;;
esac

```

`case $1 in`：类似于其他语言的`switch(xxx)`一样。

`"one")`：匹配项，类似于`case X:`。

`echo "1"`：符合匹配项执行的逻辑。

`;;`： 类似于`break;`结束。

`*)`：类似于`default`。

`esac`：`case`语句的结束标记。

> 注意：匹配项可以用正则表达式进行匹配。


看一下运行的结果

```
[root@iZ2zebizp6le568407aeayZ shell]# ./condition_case.sh two
2

```

### while 循环

`while`循环的语法如下：

```
while [ 条件 ]
do
    做某些事
done
```

或

```
while [ 条件 ]; do
    做某些事
done
```

> 注意：条件为真是，才会做do之后的逻辑，为什么强调这个呢，因为shell中有一个`until`语法。和其正好相反。

### until循环

`while`循环表示如果条件为真，则执行do中的逻辑，而`until`和`while`正好相反，虽然语法类似


```
until [ 条件 ]
do
    做某些事
done
```

或

```
until [ 条件 ]; do
    做某些事
done
```

但是，其是当条件为false是，才会走`do`之后的逻辑。


### for循环

`for`循环的语法如下：

```
for 变量 in '值1' '值2' '值3' ... '值n'
do
    做某些事
done
```
当然我们可以在`in`之后不用谢一大串，而是用变量去代替。

```
#!/bin/bash

fileList=`ls`

for file in $fileList
do
  echo "file found : $file"
done

```

首先通过`ls`命令查找当前目录下的所有文件。

其次通过`for`循环遍历变量，并打印。

运行的结果如下：

```
[root@iZ2zebizp6le568407aeayZ shell]# ./for.sh 
file found : condition_case.sh
file found : condition.sh
file found : for.sh
file found : read_variable.sh
file found : test.sh
file found : variable_array.sh
file found : variable_porams.sh
file found : variable.sh
```