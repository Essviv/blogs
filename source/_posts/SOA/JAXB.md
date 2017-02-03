---
title: JAXB
author: essviv
date: 2016-05-23 10:20:54+0800
---

# JAXB

## 架构

![jaxb](https://github.com/Essviv/images/blob/master/jaxb.png?raw=true)

## 标签

* **@XmlRootElement**: 根元素

* **@XmlElement**: 用于setter方法

* **@XmlAttribute**: 表示将这个字段作为元素的属性

* **@XmlType**: propOrder属性的名称

* **@XmlJavaTypeAdapter**: 用于复杂的java类型，或者当Jaxb输出的格式不是理想的输出时，就可以通过自定义adapter的方式定义输出的内容与格式

* **@XmlAccessorType**: 指定用于生成XML的元素有哪些，默认是public_member，可以指定的包括field, property, public_member, none.

## 代码

* **JaxbContext**: 提供了jaxb的API入口，不管是marshaller和unmarshaller，均可以由它产生

* **Marshaller/Unmarshaller**: 可以认为是编码与解码器，由jaxbContext生成，可以设置相应的属性（如格式化输出等）,并且提供了由java对象与xml表达之间的转换操作.

* **XmlAdapter**: 这个接口提供了java类与xml格式输出的自定义输出接口，marshal与unmarshal分别定义了输入输出.
XmlAdapter接口定义了在内存中如何通过valueType来表示boundType. 


## 参考文献 

1. [https://www.javacodegeeks.com/2015/04/%E7%94%A8%E4%BA%8Ejava%E5%92%8Cxml%E7%BB%91%E5%AE%9A%E7%9A%84jaxb%E6%95%99%E7%A8%8B.html](https://www.javacodegeeks.com/2015/04/%E7%94%A8%E4%BA%8Ejava%E5%92%8Cxml%E7%BB%91%E5%AE%9A%E7%9A%84jaxb%E6%95%99%E7%A8%8B.html)

## 示例代码

1. [https://github.com/Essviv/spring/tree/master/src/main/java/com/cmcc/syw/jaxb](https://github.com/Essviv/spring/tree/master/src/main/java/com/cmcc/syw/jaxb)