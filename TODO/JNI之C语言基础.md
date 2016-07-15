

### **前言**

JNI （java native interface） java 本地开发接口。

JNI 是一个协议, 有了这个协议可以使Java代码和C/C++代码相互调用.

在学习JNI 之前，很有必要对C语言进行简单的总结和复习。


### **Hello word**

```c
#include<stdio.h>  // 类似于java 的 import  导包 
#include<stdlib.h>   


main(){
       printf("helloword\n"); //输出hellowrd  \n代表换行 
       system("notepad");  //system() 控制台命令   此处代表打开记事本程序 
       system("pause");  //控制台暂停 
       }

```

以`.h`结尾的代表c 的头文件。

- `stdio.h`： standard io  标准的输入输出
- `stdlib.h` ：standard library 标准的函数库，类似于java.lang


### **基本数据类型**

- `char`:字符型，1个字节（8位二进制数）
- `short`：短整型， 2个字节 
- `int`: 整形，4个字节
- `long`：长整型，4个字节
- `float`:单精度浮点型，4个字节
- `double`:双精度浮点型，8个字节

修饰数据类型的关键字：

- `signed`：有符号位。即数值的最高位代表正负，0或者1，0代表负，1代表证。
- `unsigned`:无符号位。

```c
#include<stdio.h>    
#include<stdlib.h>    

main(){    
           
   // sizeof 显示数据的所占的字节大小        
   printf("char占%d个字节\n", sizeof(char));
   printf("int占%d个字节\n", sizeof(int));
   printf("short占%d个字节\n", sizeof(short));
   printf("float占%d个字节\n", sizeof(float));
   printf("long占%d个字节\n", sizeof(long));
   printf("double占%d个字节\n", sizeof(double));
   
   // 测试无符号和有符号 
   signed char c = 128; //显示-128  
   unsigned char c = 128;//  显示128 
   printf("c = %d\n",c);
   
       system("pause"); 
       } 

```

`char`占一个字节

此时分别定义了无符号的`char`和有符号的`char`。

- 有符号数值大小表示：范围为-128~127。[-128从哪里蹦出来的](http://blog.csdn.net/daiyutage/article/details/8575248) 
- 无符号数值大小：0~255。

那么例子中，有符号的很明显超出了范围，所以程序显示的和我们想要的不一样。具体为什么，牵扯到了2进制一大堆。不解释了。


### **输出函数 printf**

`printf("要输出的内容", 变量); `

例如`printf("数值 = %d\n",n);` ，该方法类似于java 中的`String.format("",...)`方法。


- `%d  -  int`
- `%ld – long int`
- `%lld - long long`
- `%hd – 短整型`
- `%c  - char`
- `%f -  float`
- `%lf – double`
- `%u – 无符号数`
- `%x – 十六进制输出 int 或者long int 或者short int`
- `%o -  八进制输出`
- `%s – 字符串`

```java 
  char c='a';
      short s = 123;
      int i = 12345678;
      long l = 1234567890;
      float f = 3.1415;
      double d = 3.1415926; 
      printf("c = %c\n", c); //  c = a 
      printf("s = %hd\n", s); // s = 123
      printf("i = %hd\n",i); // i = 24910  // 短整型显示，会被截取，所以显示错误 
      printf("l = %ld\n",l); //  l =1234567890 
      printf("f = %.4f\n",f);  //默认输出的是6位有效数字的小数 想手动指定 加上.X 代表小数点后保留x位（舍去） 
      printf("d = %.7lf\n",d); // 默认六位时，多余的四舍五入。和.x 区分 
      printf("%#x\n",i);  // 十六进制显示  0xbc614e 
      printf("%#o\n",i); //  八进制显示    057060516
     // char cArray[]={'a','b','c','d','\0'}; // \0 表示结束符。作为字符数组结束的标识 
      char cArray[]="你好"; // c 中没有String 类型，所以只能用数组或者是指针 
      printf("cArray = %s",cArray);

```


### **scanf 输入函数**

`scanf("占位符",内存地址) `.

例如：`scanf("%d", &count);`

```c
  	   int count;
      scanf("%d", &count); //&取地址符 
      printf("班级的人数是%d\n",count);
      char cArray[4];//c的数组不检测下标越界  
      printf("请输入班级的名字:");
      scanf("%s",&cArray);
      printf("班级的人数是%d,班级的名字%s\n",count,cArray);
```

因为输入的时候，需要的是地址，所以c的数组不检测下标越界，如果输入的长度大于了数组长度，会继续往其下面的内存中存入数据，那么很明显，会导致数据显示错误。

```
请输入班级的人数:2
班级的人数是2
请输入班级的名字:abcde
班级的人数是101,班级的名字abcde

```

因为数组输入的长度大于了其分配的内存大小，占用了班级数所分配的内存，导致班级人数显示错误。












