---
title: spring-mvc-controller
author: essviv
date: 2017-01-25 10:20:54+0800
---

---
layout:		post
author:		essviv
title:		SpringMVC学习(2)
subtitle:	控制器
date:		2016-06-21 21:00:00+0800
tag:		SpringMVC
---

# Controller

Controller提供了service接口中定义的业务的访问途径，它接受用户的输入，并将用户请求委托给service层进行处理，最终将处理结果以视图的形式返回给用户。从Spring2.5以来，提供了注解的方式来实现controller，这些注解包括@Controller, @RequestMapping, @RequestParam, @ModelAttribute，使用这些注解，程序中并不需要实现特定的接口，极大地方便了程序的实现。

## @Controller和@RequestMapping

@Controller注解意味着被注解类的角色为**controller**，即MVC中的“C”角色. 配合@RequestMapping注解，SpringMVC会自动扫描这些注解并将相应的请求地址映射到这些类和方法中，举个例子：

```java
@Controller
@RequestMapping("/appointments")
public class AppointmentsController {
    @RequestMapping(method = RequestMethod.GET)
    public Map<String, Appointment> get() {
        return appointmentBook.getAppointmentsForToday();
    }

    @RequestMapping(path = "/{day}", method = RequestMethod.GET)
    public Map<String, Appointment> getForDay(@PathVariable @DateTimeFormat(iso=ISO.DATE) Date day, Model model) {
        return appointmentBook.getAppointmentsForDay(day);
    }

    @RequestMapping(path = "/new", method = RequestMethod.GET)
    public AppointmentForm getNewForm() {
        return new AppointmentForm();
    }

    @RequestMapping(method = RequestMethod.POST)
    public String add(@Valid AppointmentForm appointment, BindingResult result) {
        if (result.hasErrors()) {
            return "appointments/new";
        }
        appointmentBook.addAppointment(appointment);
        return "redirect:/appointments";
    }
}
```

这个代码片断中多次使用了@RequestMapping注解. 

第一次是对类进行注解，它意味着类中所有的处理方法路径都是相对于这个路径而言的，在这个例子中，也就是都是相对于/appointments路径. 

第二个注解出现在get方法上，同时还增加了method参数，这意味着这个方法在原来类注解的基础上，增加了请求方法的限制，这里只能匹配get方法的请求. 

第三个注解出现在getForDay中，它也只能匹配get方法，同时它还使用了URI模板，后续会对URI模板进行深入的探讨. 除此之外，getNewForm方法还对请求路径做了进一步的限制，它只匹配/appointments/new，并且只匹配get方法. 

另外，类层级的@RequestMapping注解并不是必须的，如果类没有被它注解，意味着方法中所有的路径都是绝对路径，而不是相对路径.

在Spring3.1之前，@RequestMapping默认先由DefaultAnnotationHandlerMapping处理，再由DefaultAnnotationHandlerAdapter进一步缩小匹配的范围. 

在Spring3.1以后，spring提供了新的实现，RequestMappingHandlerMapping和RequestMappingHandlerAdapter，处理器的选择由RequestMappingHandlerMapping直接完成.

Spring4.3以后，对@RequestMapping提供了几个变体版本，针对不同的请求方法，提供了相应的mapping注解，比如@GetMapping, @PostMapping, @PutMapping等等，有需要可以查阅相关的文档，这里不做进一步详述.


## URI模板

在使用RequestMapping注解的过程中，可以使用URI模板很方便地获取部分URL地址. 

URI模板中的变量可以通过@PathVariable注解来获取，如下所示, 当请求/owners/fred地址时，ownerId的值就会被设置成fred. 

当@PathVariable注解被用于Map类型的变量时，所有的URI模板变量会自动被填充到map变量中: 

```java
@GetMapping("/owners/{ownerId}")
public String findOwner(@PathVariable String ownerId, Model model) {
    Owner owner = ownerService.findOwner(ownerId);
    model.addAttribute("owner", owner);
    return "displayOwner";
}
```

除了变量，还可以在URI模板中使用正则表达式. 具体的语法为**{varName: regex}**，其中第一部分为定义的变量，第二部分为正则表达式

如下所示，当请求/spring-web/spring-web-3.0.5.jar时，symbolicName会被设置成spring-web, version被映射成3.0.5，而extension被映射成.jar:

```java
@RequestMapping("/spring-web/{symbolicName:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{extension:\\.[a-z]+}")
public void handle(@PathVariable String version, @PathVariable String extension) {
    // ...
}
```

当某个请求地址匹配多个URI模板时，匹配的规则按以下进行:

* 拥有较少URI变量和通配符的URI模板被优先匹配，比如/hotels/{hotel}/*有1个变量和1个通配符，因此它比/hotels/{hotel}/**优先匹配

* 如果变量和通配符数量相等，路径长的被优先匹配，比如/foo/bar*比/foo/*优先匹配

* 如果变量和通配符数量相等且长度相等，拥有较长通配符个数的优先匹配， 比如/hotels/{hotel}比/hotels/*优先匹配

除此之外，还有两条例外:

* /**拥有最低的匹配次序

* 前置的通配符拥有更低的匹配次序，例如/public/path3/{a}/{b}/{c}比/public/**优先匹配

## 矩阵变量

Spring还支持在URI请求中带上相应的变量，这些变量就被称为矩阵变量(matrix variable). 在spring中，矩阵变量可以有以下三种形式：

* /cars;color=red;year=2012
* /cars;color=red,green,blue;year=2012
* /cars;color=red;color=green;color=blue;year=2012

如果要在请求中访问矩阵变量的值，可以使用@MatrixVariable注解来获取. 这个注解还支持指定相应的变量名和矩阵变量名来进一步精确地获取其中的值.

## @RequestMapping注解方法

@RequestMapping注解被用在方法上时，支持许多类型的输入参数，包括以下这些：

|类型|描述|类型转换|
|---|---|---|
|Request和Response对象|可以选择特定的请求和响应类型，比如HttpServletRequest和HttpServletResponse|N/A|
|Session对象|HttpSession类型，表示会话对象|N/A|
|InputStream, OutputStream, Reader, Writer|表示请求和响应的输入输出流对象|N/A|
|HttpMethod|表示当前请求方法对象|N/A|
|注解参数|包括@PathVariable, @MatrixVariable, @RequestBody, @RequestParam, @RequestAttribute, @RequestHeader注解, 请求中的属性或参数都会被自动转化成注解所标注的参数类型|其中@RequestBody注解使用的是HttpMessageConverter接口来进行转换，其它的注解则由
WebDataBinder和Formatter来完成|
|HttpEntity|包括请求参数和请求头信息|类型转换由HttpMessageConverter完成|
|Model, Map, ModelMap|增强视图的模型内容|N/A|

支持的返回类型包括：

|类型|描述|类型转换|
|---|---|---|
|ModelAndView|包括返回的模型和视图对象|N/A|
|Model, Map|返回的模型对象，视图类由默认的RequestToViewNameTranslator来进行解析得到|N/A|
|View|返回的视图对象，模型对象可以由参数中的Model类型的参数来提供|N/A|
|String|返回的逻辑视图，可以由相应的ViewResolver接口解析得到实际的视图|N/A|
|void|这种情况下，响应的内容由程序直接通过Response对象输出，或者由RequestToViewNameTranslator解析得到|N/A|
|HttpEntity|包括响应头和响应消息体的内容|消息体的内容由HttpMessageConverter完成类型转换|
|HttpHeaders|表示当前响应不包含消息体，只有响应头|N/A|

## @RequestBody和@ResponceBody

@RequestBody和@ResponceBody分别代表了请求和响应的消息体，当参数被RequestBody注解，或者响应的对象被ResonseBody注解时，Spring会自动根据convesionService的配置转化成相应的对象或消息体. 

事实上，在spring中消息体到对象或者由对象到响应消息体之间的转换全部由HttpMessageConverter完成，包括RequestBody、ResponseBody以及HttpEntity中的消息体.

RequestMappingHandlerAdapter自动注册了以下几个converters,如果需要可以自行定义和配置converter

* ByteArrayHttpMessageConverter

* StringHttpMessageConverter

* FormHttpMessageConverter

* SourceHttpMessageConverter

## @RestController

当需要在spring中实现RESTful类型的接口时，可以使用@RestController. 它是@Controller和@ResponseBody两个注解的组合，可以很方便地实现RESTful风格的接口

## 方法参数与类型转换

请求中的参数都是字符串类型的，当它们转换成其它类型时，就需要使用类型转换，这些参数包括请求参数，请求头，cookie信息以及path variable变量. 在Spring中，提供了WebDataBinder来实现相应的类型转换. 通过注册Formatter接口的实现来自定义类型转换的过程. 类型转换的过程可以通过两种方式进行：

* 使用@InitBinder注解

* 使用WebBindingInitializer

### @InitBinder注解

使用@InitBinder注解可以将方法定义为类型转换的实现，它定义了将请求参数转换成相应类型的具体方法，通过registerCustomEditor或者addCustomFormatter来实现注册.

它接受的参数同@RequestMapping基本相同，而且不能有返回值. 

这种方式只在声明@InitBinder的controller类中有用. 

```
@Controller
public class MyFormController {

    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));

        //spring4.2后支持的方式
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd"));
    }

    // ...
}
```

### WebBindingInitializer

通过WebBindingInitializer可以将类型转换的配置放到RequestMappingAdapter， 通过修改默认的配置自定义类型转换的过程. 

```
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
    <property name="cacheSeconds" value="0"/>
    <property name="webBindingInitializer">
        <bean class="org.springframework.samples.petclinic.web.ClinicBindingInitializer"/>
    </property>
</bean>
```

## 异步请求处理

从Servlet3.0规范开始就定义了异步请求处理的方法. Spring框架也从3.2开始支持3.0规范. 在Spring中，可以用两种方式来实现对请求的异步处理：

* Callable<T>: 请求到达服务器后，服务器直接返回callable对象，并且servlet的处理线程退出，从而可以接着处理后续进来的请求。后续的处理由threadPoolTaskExecutor中的线程接管，当工作线程处理完成后，结果通过callable回调给servlet容器的线程，再由这个线程返回到客户端. 

* DeferredResult<T>： 整个处理的过程与Callable<T>基本一致，唯一不同的是，Callable的结果由容器管理的工作线程完成后交还给servlet容器响应. 但deferredResult的响应结果则由其它的任意线程进行设置，通过它的setResult方法，任何方法都可以将处理的结果设置到deferredResult中

```
@RequestMapping("query")
@ResponseBody
public Callable<Person> asyn() {
    return () -> {
    	//延迟两秒响应
        Thread.sleep(2000);

        return buildPerson();
    };
}

@RequestMapping("queryDefered")
@ResponseBody
public DeferredResult<Person> asynDeferredResult() {
    DeferredResult<Person> deferredResult = new DeferredResult<>(5000L, buildPerson("Essviv"));

    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            deferredResult.setResult(buildPerson());
        }
    }).start();

    return deferredResult;
}
```

### 异步处理流程

异步请求处理的整个过程包括：

* 控制器返回callable或者deferredResult

* SpringMVC开始在工作线程中处理请求，对于Callable，工作线程由容器管理；对于deferredResult，工作线程由任意线程实现

* DispatcherServer及所有的过滤器线程，但是response仍然打开着，等待进一步响应

* Callable返回相应的响应并把响应重新分发给dispatchServlet

* DispatcherServlet被重新唤起，并将callable的内容输出到response中

在异步处理的时候，如果出现异常，异常处理的方式与同步处理表现一致. 在DeferredResult中，也可以通过setErrorResult来设置异常信息

### 配置方法

那么，在spring中如何配置异步处理请求呢? 简单来讲，分为两步：

* 设置servlet的版本为3.0

* 在servlet及相应的filter配置中加上<async-support>true</async-support>

* 在controller中返回callable或者deferredResult

## 参考文献

Controller的官方文档： [官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mvc-controller)
