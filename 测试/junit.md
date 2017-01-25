# JUnit

* **Test(timeout, expected)**： 表示该方法为单元测试方法，timeout表示执行的最长时间，expected表示预期方法会抛出的异常
 
* **Before**: 每个测试方法执行前被执行一次
 
* **After**: 每个测试方法执行后被执行一次
 
* **BeforeClass**: 方法必须为static，在类初始化时被执行一次
 
* **AfterClass**： 方法必须为static, 在所有测试方法执行完后被执行一次
 
* **Ignore**： 表示忽略该测试方法
 
* **parameters**: 用于参数化测试， 方法必须返回collection，作为构造函数的参数feeder，每套参数都会被所有的测试方法执行一次


## 参考资料

1. [http://www.ibm.com/developerworks/cn/java/j-lo-junit4/](http://www.ibm.com/developerworks/cn/java/j-lo-junit4/)
