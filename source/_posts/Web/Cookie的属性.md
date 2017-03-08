---
title: Cookie的属性
author: essviv
date: 2016-08-26 10:20:54+0800
---

# Cookie的属性

Cookie在WEB开发中具有重要的意义，它有以下的属性，现解析如下：

 

1. secure： 当secure=true时，表示该cookie只能通过安全通道（即https）传输给服务器端， 当通道为http时，这个cookie不会由浏览器传输回服务器端; 通过这个标识，能保证cookie不会经由不安全的通道发往服务器端，意即在传输的过程中是保证安全的；但是，在浏览器端仍然可以看到cookie的值，如果需要也可以对cookie进行加密处理

2. path: 表示在访问哪些路径时，客户端必须把这个cookie传输回服务端. 如果设置了cookie的path属性为/contextPath/A， 那么当访问/contextPath/A以及所有它的子目录时，浏览器都会自动带上这个cookie; 相反地，如果访问的是/contextPath/B, 那么浏览器就不会带上这个cookie

3. domain: 功能和path类似，但它指定的是域名范围的地址

4. expire: 设置cookie的有效期