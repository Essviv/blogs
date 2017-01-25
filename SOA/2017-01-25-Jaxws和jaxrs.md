---
title: Jaxws和jaxrs
author: essviv
date: 2017-01-25 05:34:24+0800
---

# Jaxws和jaxrs

## jaxws

这是一个api规范，需要提供相应的运行时实现. 不过J2se中提供了jaxws的参考实现（jaxws-ri)

1. @WebService, @WebMethod

2. wsimport自动生成客户端代码

3. 调用服务端的方法

**实现了jaxws规范的框架包括： cxf, axis2**

## jaxrs

用于更方便地创建RESTful服务的api, 它也是一个api规范，也需要提供运行时实现，j2se中并没有提供相应的实现，不过可以自行根据需要添加. jersey是jaxrs的参考实现.

1. @Path, @GET/@PUT/@POST/@DELETE

2. @PathParam, @QueryParam, @HeaderParam, @CookieParam, @MatrixParam

3. @Consumes, @Produces, @ApplicationPath

4. @Context, @NotNull, @Email

**实现了jaxrs规范的框架包括： cxf, jersey**