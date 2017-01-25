# WSDL

WSDL中包含以下几类元素

* 抽象元素：

	* types: WS定义的类型，通过这种方式，WS可以最大限度的实现平台的中立性
	
	* message: WS的消息， 可以理解成传统函数中的输入输出参数
	
	* portType: WS执行的操作, 可以理解成传统函数库的一个模块或一个类， 也可以认为是接口定义


* 具体定义元素： 

	* binding: WS使用的通信协议， 定义消息的格式和通信细节，注意这里只是定义了协议与通信细节，并没有与具体的地址绑定
	
	* service: WS定义的服务，它将之前的绑定与实际的地址相关联，完成服务接口的完整定义.

## WSDL的结构

````
    <definitions>

        <types />

        <message />

        <portType />

        <binding />

        <service />

    </definitions>
````
 
### binding元素

在这个例子中，portType元素把定义了端口的名称， 还定义了四个操作的名称. 相对于传统的函数库来讲，MathInterfce是函数库，而Add是输入参数为AddMessage，而输出参数为AddMessageResponse的函数. 其它的操作与此类似. 而service元素将MathInterface接口绑定到了http://localhost/math/math.asmx 这个地址.

 

## 参考文档 

1. [微软关于WSDL的说明](https://msdn.microsoft.com/en-us/library/ms996486.aspx#understand_topic5)

2. [IBM关于WSDL中绑定类型的说明](https://www.ibm.com/developerworks/library/ws-whichwsdl/)


## 示例代码

1. [http://blog.csdn.net/onlyqi/article/details/7013893](http://blog.csdn.net/onlyqi/article/details/7013893)

2. [cxf、axis2与spring-ws的比较](https://dzone.com/articles/apache-cxf-vs-apache-axis-vs)

3. [客户端自动生成代码调用](https://github.com/Essviv/spring/tree/master/src/main/java/cn/com/webxml)

4. [自行编码调用](https://github.com/Essviv/spring/tree/master/src/main/java/com/cmcc/syw/wsclient)

 
## 疑问

1. 怎么发布？: 1)jaxws:endpoint   2) java-ws发布

2. 怎么获取WSDL: 直接在WS地址的后面加上?wsdl即可

3. 绑定类型与编码类型

4. 客户端方式与SOAP方式的区别

5. JAVA自带API与Axis2, cxf的api使用

