#SQL

[字符类型的长度的意义](http://www.cnblogs.com/echo-something/archive/2012/08/26/mysql_int.html)

[浮点数的精度问题](http://www.jb51.net/article/31723.htm)

	计算机中浮点数的实现大都遵从 IEEE754 标准，IEEE754 规定了单精度浮点数和双精度  浮点数两种规格，单精度浮点数用4字节（32bit）表示浮点数，格式是：
	
	1位符号位，8位表示指数，23位表示尾数
	
	双精度浮点数8字节（64bit）表示实数，格式是：
	
	1位符号位 11位表示指数 52位表示尾数
	
	同时，IEEE754标准还对尾数的格式做了规范：d.dddddd...，小数点左面只有1位且不能为零，计算机内部是二进制，因此，尾数小数点左面部分总是1。显然，这个1可以省去，以提高尾数的精度。由上可知，单精度浮点数的尾数是用24bit表示的，双精度浮点数的尾数是用53bit表示的，转换成十进制：
	
	2^24 - 1 = 16777215；  2^53 - 1 = 9007199254740991
	
	由上可见，IEEE754单精度浮点数的有效数字二进制是24位，按十进制来说，是8位；双精度浮点数的有效数字二进制是53位，按十进制来说，是16 位。
	
日期类型: date, time, datetime, timestamp(自动更新时间)

数据库表设计的三范式：

* 1st NF(原子性)
	* Columns with similar content must be eliminated
	* A table must be created for a group of associated data
	* Each data record must be identifiable by means of a primary key
* 2nd NF(非主属性非部分依赖于主属性）
	* Whenever the contents of columns repeat themselves, this means that the table must be divided into subtables.
	* These tables must be linked by foreign keys
* 3rd NF(属性不依赖于其它非主属性）
	* Columns that are not directly related to primary key must be eliminated(that is, must be translated into a table of their own)