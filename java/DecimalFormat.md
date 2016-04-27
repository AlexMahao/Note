## DecimalFormat


DecimalFormat 是 NumberFormat 的一个具体子类，用于格式化十进制数字。能够做到日常所需的大部分功能。

### 基本功能

DecimalFormat里面封装了一些对数据最基本的操作。包括对数据三位一组的间隔分组，小数位保留多少位，整数位最多现实多少位能。

```java 
DecimalFormat df = (DecimalFormat) DecimalFormat.getInstance();

		double d = 123456.5555;
		// 默认保留三位小数，并在整数位的时候三位添加一个间隔符
		System.out.println(df.format(d));

		// 最大的小数位数，四舍五入
		df.setMaximumFractionDigits(2);

		// 最小的小数位数，不够的添0
		df.setMinimumFractionDigits(2);
		System.out.println(df.format(d));

		// 最小的整数位数，位数不够补0
		df.setMinimumIntegerDigits(15);

		System.out.println(df.format(d));

		// 最大的整数位数，多余的舍去
		df.setMaximumIntegerDigits(1);

		System.out.println(df.format(d));

		df = (DecimalFormat) DecimalFormat.getInstance();

		// 设置整数位每四个一个分组
		df.setGroupingSize(4);

		System.out.println(df.format(d));

		DecimalFormatSymbols dfs = DecimalFormatSymbols.getInstance();

		// 设置小数点的分隔符
		dfs.setDecimalSeparator('s');
		// 设置每一分组的分隔符
		dfs.setGroupingSeparator('a');

		df.setDecimalFormatSymbols(dfs);

		System.out.println(df.format(d));

		// 取消分组
		df.setGroupingUsed(false);
		System.out.println(df.format(d));
```

结果
```java
123,456.556
123,456.56
000,000,000,123,456.56
6.56
12,3456.556
12a3456s556
123456s556

```



#### 扩展功能

该DecimalFormat 中可以通过#，0，%字符对数字进行特殊的格式化。
- #表示字符安装#位数进行匹配，如果在最后或者整数的最前有0会被舍去。
- 0表示强制匹配，如果位数不够，则会强制补0；
- % 表示将数据转化成百分比数据
```java
		double a = 1.220;
		double b = 11.22;
		double c = 0.22;
		
		DecimalFormat df = (DecimalFormat) DecimalFormat.getInstance();
		
		
		df.applyPattern("00.00%");
		System.out.println(df.format(a));
		System.out.println(df.format(b));
		System.out.println(df.format(c));
		
		df.applyPattern("##.##%");
		System.out.println(df.format(a));
		System.out.println(df.format(b));
		System.out.println(df.format(c));

```

```java 
122.00%
1122.00%
22.00%
122%
1122%
22%

```