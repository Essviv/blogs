---
title: SOAP
author: essviv
date: 2015-11-25 10:20:54+0800
---

# SOAP

SOAP消息包括以下几个部分：

1. Envelope元素： 标识此XML文档为SOAP消息， 必需

2. Heaher元素： 头部信息，可选

3. Body元素： 包含所有的请求和响应信息， 必需

4. Fault元素： 包含针对此消息出错时的错误信息，可选

例如：

````
    <?xml version="1.0" ?>

    <soap:Envelope>

        <soap: Header />

        <soap: Body>

            <soap:Fault></soap:Fault>

        </soap:Body>

    </soap:Envelope>

````

**HTTP+XML=SOAP**