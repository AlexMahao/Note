
### 什么是shell

shell作为linux系统的一层外壳，向用户提供使用操作系统的接口（命令）。它是命令语言、命令解释语言、程序设计语言的统称。

- shell 是用户和linux之间的接口程序，如果把linux内核当做一个球体，shell就是围绕内核的外层，当从shell向linux传递命令时，内核会做出相应的反应。
- shell是一个命令语言解释器，他拥有自己内建的shell命令集，shell也能被系统中其他应用所调用。用户在终端输入的命令都是由shell先解释然后传给内核。
- shell是一个具备特殊功能的程序，介于使用者和UNI/linux核心程序之间的一个接口。


### shell的分类

在使用linux时，我们通常使用终端命令行输入一些命令来执行操作。而终端命令行最终解释并执行命令便是通过shell解释再交给内核执行的。虽然终端命令行看上去样式都是一样，但也分好多种，对应不同的shell，不同的shell对应的终端命令行的功能也有所不同。

一下列出主流的几种shell:

- sh:Bourne Shell的缩写。可以说是所有shell的祖先。
- bash:Bourne Again Shell 的缩写。bash是sh的进阶版本。bash是目前大多是Liunx发行版和Mac OS X操作系统的默认shell.
- ksh： Korn shell，一般在收费的UNIX中比较常见。
- csh： C shell ， 此shell的语法有点类似C语言。
- tcsh： Tenex C shell ， csh的优化版。
- zsh： Z shell，比较新的一个shell，集bash，ksh和tcsh各家之大成。

> bash 已经是大多数Linux发行版的默认shell。

### shell 可以做什么

- 管理命令行。
- 记住之前在终端中输入的命令，通过方向上键回退到之前执行过的命令。
- 用Tab键自动补全我们要输入的命令。
- 控制进程。
- 重定向命令。
- 定义别名。

### 通过vim编写Shell脚本

shell脚本类似于windows的批处理，就是将一个个命令通过一定的逻辑组装，写到对应的文本中统一执行。

执行命令

```shell
vim test.sh
```

如果没有Vim基础的可以先看一下vim的相关使用。

运行命令之后，显示如下界面

![](shell1.png)

此时点击`i`切换到编辑模式进行输入，输入完之后的脚本如下
```
#!/bin/bash

# 显示当前目录的文件
ls

```

首先第一行指定用那种shell来解释脚本，因为不同的shell用法不同。`/bin/bash`是bash程序在大多数linux中存放的路径。如果当前系统默认的shell是bash，不指定也不会有问题。


shell脚本中通过#开头标识这一行是注释。

添加了一个简单的`ls`命令，显示目录文件的命令。

因为当前我们处于Vim的编辑模式，点击ESC退出到交互模式。输入`:wq`保存并退出。

如上操作，我们就编写了一个简单的shell脚本。


### 如何运行脚本

首先通过输入`ls -l`命令查看编写的shell脚本文件的信息

```
-rw-r--r-- 1 root root 46 Apr 11 22:57 test.sh
```

在展示的信息的最前方`-rw-r--r-`表示文件的权限，根据此时的权限信息发现此时的文件是没有运行权限的，所以要添加运行权限，输入命令

```
chmod +x test.sh 
```

再输入`ls -l`查看文件信息

```
-rwxr-xr-x 1 root root 46 Apr 11 22:57 test.sh
```

可以发现此时文件已经有了运行权限。

那么我们可以输入`./test.sh`运行此脚本，结果如下

```
[root@iZ2zebizp6le568407aeayZ shell]# ./test.sh
test.sh
```
可以看到脚本已经执行成功。这个脚本其实就是打印了当前目录下的所有文件，因为我这里只有一个脚本文件，所以显示如下。

当然也可以通过`sh test.sh`执行脚本。

可能会有疑惑，这么麻烦的步骤，我们直接输入`ls`命令不就可以了吗，但是，如果我们要做一些列复杂的操作，那么在脚本中编写好一个又一个的命令，而不需要一个个的输入了。


### 调试脚本

在此输入命令`vim test.sh`并按下`i`键在此编写脚本如下

```
#!/bin/bash

# 显示当前目录的文件
ls

# 显示当前所在的目录
pwd

```

并按照上面所述的vim的退出并保存方式操作。

再次运行命令`./test.sh`

```
[root@iZ2zebizp6le568407aeayZ shell]# ./test.sh
test.sh
/home/shell
```
运行没有什么问题，我们可以输入

```
bash -x test.sh
```
运行调试模式，运行结果如下：

```
[root@iZ2zebizp6le568407aeayZ shell]# bash -x test.sh
+ ls
test.sh
+ pwd
/home/shell

```
可以看到将运行的命令也打印了出来。



