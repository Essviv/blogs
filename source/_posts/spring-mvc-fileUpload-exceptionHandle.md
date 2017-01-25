---
title: spring-mvc-fileUpload-exceptionHandle
author: essviv
date: 2017-01-25 10:20:54+0800
---

---
layout:		post
author:		essviv
title:		SpringMVC学习(4)
subtitle:	文件上传和异常处理
date:		2016-06-29 18:54:00+0800
tag:		SpringMVC fileUpload exceptionHandle
---

# 文件上传

Spring中对于文件上传的支持是通过MultipartResolver接口来实现的，默认情况下，spring关于文件上传的支持是关闭的，如果需要，需要在上下文中配置相应的multipartResolver对象，并指定它的名字为“multipartResolver”. spring框架中自带了两种multipartResolver的实现， 配置的样例如下：

* CommonsMultipartResolver: 使用这个解析器需要引用o.s.web.multipart包

* StandardServletMultipartResolver: spring内部提供的multipartResolver的标准实现

```
<bean id="multipartResolver"
        class="org.springframework.web.multipart.commons.CommonsMultipartResolver">

    <!-- one of the properties available; the maximum file size in bytes -->
    <property name="maxUploadSize" value="100000"/>

</bean>
```

在配置了multipartResolver之后，后续所有的请求都会被解析，如果判断出该请求是文件上传请求，则会调用multipartResolver的resolveMultipart方法，将request包装成MultipartHttpServletRequest. 

## 在表单中使用multipartResolver

在表单中使用multipartResolver分为两个步骤， 示例代码如下, 可以看出，在控制器的实现中基本没有什么变化，只是将原来的HttpServletRequest包装成了MultipartHttpServletRequest，这步是在请求进入DispatcherServlet的doDispatch方法时被调用的（后续会对DispatcherServlet的doDispatch方法做源码分析)： 

1. 指定表单的enctype为“multipart/form-data”

2. 在控制器相应的处理方法中，通过MultipartFile或者MultipartHttpServletRequest获取相应的请求对象，并获取上传的文件

```
//表单内容
<html>
	<body>
		<form action="/form" enctype="multipart/form-data" method="post">
			<input type="input" name="name" />
			<input type="file" name="file" />
			<input type="submit" name="submit" value="submit"/>
		</form>
	</body>
</html>

//控制器实现
@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(@RequestParam("name") String name,
            @RequestParam("file") MultipartFile file) {

        if (!file.isEmpty()) {
            byte[] bytes = file.getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }

        return "redirect:uploadFailure";
    }

}
```

## 在编码客户端中使用MultipartResolver

通过代码向服务端上传文件时，绝大部分的逻辑和上述通过表单上传是一致的，但是通过代码上传文件时，可以一次性上传多个文件内容，并且上传的文件内容格式可以互不相同，下面的代码展示了这样的上传样例:

```
POST /someUrl
Content-Type: multipart/mixed

--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="meta-data"
Content-Type: application/json; charset=UTF-8
Content-Transfer-Encoding: 8bit

{
	"name": "value"
}
--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="file-data"; filename="file.properties"
Content-Type: text/xml
Content-Transfer-Encoding: 8bit
... File Data ...

//控制器的实现
@PostMapping("/someUrl")
public String onSubmit(@RequestPart("meta-data") MetaData metadata,
        @RequestPart("file-data") MultipartFile file) {

    // ...

}
```

在这种情况下，可以通过@RequestPart或者@RequestParam来指定获取哪部分的数据，并且可以通过HttpMessageConverter自动完成相应的类型转换，其它的逻辑与表单完全一致.

# 异常处理

在springMVC中，当请求在被处理的过程中，如果出现了异常情况，异常情况会交由HandlerExcetionResolver接口处理. 在SpringMVC框架中，可以通过三种方式进行异常处理: 

1. 实现HandlerExceptionResolver接口: 这个接口定义了resolveException方法来处理在发生异常的情况下返回的ModelAndView对象

2. 使用SimpleMappingExceptionResolver实现: 这个实现提供了异常类型到视图类的映射关系

3. 使用@ExceptionHandler注解: 这个注解的作用是当异常产生的时候，调用有这个注解的方法来处理异常. 它可以指定要处理的异常类型，默认为方法参数的异常类型. 

@ExceptionHandler可以只作用于某个Controller类中的方法，也可以通过ControllerAdvise注解作用于多个控制器类. 在需要使用多个HandlerExceptionResolver的场合，可以通过HandlerExceptionResolverComposite类进行组合. 

## @ExceptionHandler注解

在SpringMVC中，使用ExceptionHandlerExceptionResolver来处理@ExceptionHandler注解的异常信息. 它特别适合于不需要返回错误视图，只是简单地返回错误代码及错误说明的场景, 例如在RESTful接口的使用中，就可以使用这个注解对异常信息进行处理. 以下是示例代码， 当请求/exception地址时会抛出运行时异常，这个运行时异常会被@ExceptionHandler注解的方法处理，最终返回响应码为400，响应信息为“Bad Request.”: 

```
@RequestMapping
@Controller
public class ExceptionHandlerController {
    @RequestMapping("/exception")
    public String throwException() {
        throw new RuntimeException();
    }

    @ExceptionHandler(value = RuntimeException.class)
    @ResponseBody
    public String handleException(HttpServletResponse response) {
        response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        return "Bad request.";
    }
}
```

## 标准的spring异常

在Spring进行处理的过程中，可能会产生许多类型的异常信息，使用SimpleMappingExceptionResolver可以将这些异常信息映射成相应的错误视图，但是有的时候可能希望将这些异常信息映射成相应的错误代码, spring中提供了DefaultHandlerExceptionResolver来实现这个功能. 

DefaultHandlerExceptionResolver默认被注册到spring中, 它当一些标准的spring异常类型转换成相应的错误代码，如BindException转化成400， HttpMediaTypeNotSupportedException转化成415等等，具体的对应关系可以查阅参考文献.

## @ResponseStatus

在Spring中自定义异常类时，可以使用@ResponseStatus注解, 它可以指定相应的错误代码和错误信息，当处理器抛出这种类型的异常信息时，spring会自动将它转化成相应的错误代码和错误信息. 在spring内部，使用ResponseStatusExceptionResolver进行这个注解的处理，默认情况下spring就会注册这个解析器

除此之外，Servlet也支持将异常信息通过错误码和异常类型映射成相应的处理器进行处理. 

# 参考文献 

官方文档: [官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mvc-multipart)



