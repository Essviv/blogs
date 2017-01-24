# mockito

* 调用自己的方法: doReturn(sth).when(xxx).somemethod()

* 返回列表中的元素: when(obj.method()).thenAnswer(AdditionalAnswers.returnsElementsOf());

* 抛出异常: doThrow().when(someObject).someMethod().

* 创建mock对象的方式: @InjectMock, @Mock

* 在verify的时候

	* 次数可以通过以下的方式指定 ：times()/never()/atLeastOnce()/atLeast()/atMost()

	* 另外，执行的顺序，可以通过InOrder来指定, 也可以通过createStrictMock来创建顺序有关的mock对象
````java
InOrder obj = inorder(obj);

    obj.verify(methodA);

    obj.verify(methodB);
````
	
	* 执行的时间可以通过timeout来指定

* 自定义mock操作可以通过Answer接口来实现


* 部分mock(partially mock)可以通过spy或者doCallRealMethod来实现.
````java
doCallRealMethod().when().someMethod()
````

## 参考文献
教程： [http://www.w3ii.com/en-US/mockito/default.html](http://www.w3ii.com/en-US/mockito/default.html)

部分mock: [http://heipark.iteye.com/blog/1496603](http://heipark.iteye.com/blog/1496603)

中文教程: [http://blog.csdn.net/bboyfeiyu/article/details/52127551](http://blog.csdn.net/bboyfeiyu/article/details/52127551)

## 示例代码
[https://github.com/Essviv/spring/tree/master/src/test/java/com/cmcc/syw/service/impl ](https://github.com/Essviv/spring/tree/master/src/test/java/com/cmcc/syw/service/impl )