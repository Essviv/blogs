---
layout:		post
author:		Essviv
title:		Spring学习(5)
subtitle:	基于代码的Servlet容器配置
date:		2016-07-02 17:24:00+0800
tag:		springMVC code-based container config
---

# 基于代码的Servlet容器配置

大部分情况下，使用Spring是采用基于配置的方式，通常情况下是通过web.xml文件对容器的行为进行配置. 从servlet3.0开始，spring提供了基于代码的配置，以下是个例子:

```

public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

        ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }

}
```

WebApplicationInitializer的实现会自动被检测到并且被用于初始化容器的配置. 除此之外, spring还提供了一种AbstractDispatcherServletInitializer抽象类来提供更简洁的方式对容器进行初始化, 这也是spring推荐的使用方式. 

使用AbstractDispatcherServletInitializer进行初始化还可以分为使用代码和使用XML配置文件的方式：

**注意：**这里说的是初始化DispatcherServlet的配置，而上面说的是利用代码方式初始化Servlet容器，两者不要混淆. 

```
//使用代码方式配置
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[] { MyWebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }

}


//使用XML方式配置
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        return null;
    }

    @Override
    protected WebApplicationContext createServletApplicationContext() {
        XmlWebApplicationContext cxt = new XmlWebApplicationContext();
        cxt.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
        return cxt;
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }

}
```

另外，AbstractDispatcherServletInitializer抽象类还提供了关于filter，isAsyncSupport等配置的支持，具体可以参阅其文档，这里就不做过多阐述.

查阅AbstractDispatcherServletInitializer的源码可以发现，它是典型的模板模式的实现，将对WebApplicationInitializer的实现进行抽象，但是整个方法的实现过程没有太多变化.

```
    public void onStartup(ServletContext servletContext) throws ServletException {
        super.onStartup(servletContext);

        registerDispatcherServlet(servletContext);
    }

    protected void registerDispatcherServlet(ServletContext servletContext) {
        String servletName = getServletName();
        Assert.hasLength(servletName, "getServletName() may not return empty or null");

        WebApplicationContext servletAppContext = createServletApplicationContext();
        Assert.notNull(servletAppContext,
                "createServletApplicationContext() did not return an application " +
                "context for servlet [" + servletName + "]");

        DispatcherServlet dispatcherServlet = new DispatcherServlet(servletAppContext);
        ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
        Assert.notNull(registration,
                "Failed to register servlet with name '" + servletName + "'." +
                "Check if there is another servlet registered under the same name.");

        registration.setLoadOnStartup(1);
        registration.addMapping(getServletMappings());
        registration.setAsyncSupported(isAsyncSupported());

        Filter[] filters = getServletFilters();
        if (!ObjectUtils.isEmpty(filters)) {
            for (Filter filter : filters) {
                registerServletFilter(servletContext, filter);
            }
        }

        customizeRegistration(registration);
    }
```

## 参考文献

官方文档： [基于代码配置Servlet容器](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#mvc-container-config)