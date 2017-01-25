---
title: xstream-converter
author: essviv
date: 2017-01-25 10:20:54+0800
---

---
layout:		post
author:		Essviv
title:		XStream学习(3)
subtitle:	转换器
date:		2016-06-15 21:24:00+0800
tag:		XStream 转换器
---

# 转换器

我们从最简单的例子开始吧,创建一个简单的对象，设置别名，然后输出：

```
package com.thoughtworks.xstream.examples;

public class Person {

        private String name;

        public String getName() {
                return name;
        }

        public void setName(String name) {
                this.name = name;
        }
}

//设置别名
xStream.alias("person", Person.class);

//输出结果
<person>
  <name>Guilherme</name>
</person>
```

## 创建转换器

接下来我们要开始创建Person类的转换器，转换器要实现Converter接口. 这个转换器必须要完成以下三个功能：

* 支持Person类的转换
* 将Person类转换成XML（序列化）
* 将XML转换成Person（反序列化）

以下是Person类转换器的代码，在序列化的过程中，可以使用startNode/endNode来声明新的节点;在反序列化的过程中，可以使用moveDown/moveUp在结点树中遍历；在创建完转换器的代码后，可以将它注册到XStream中，并观察它的输出: 

```
package com.thoughtworks.xstream.examples;

import com.thoughtworks.xstream.converters.Converter;
import com.thoughtworks.xstream.converters.MarshallingContext;
import com.thoughtworks.xstream.converters.UnmarshallingContext;
import com.thoughtworks.xstream.io.HierarchicalStreamReader;
import com.thoughtworks.xstream.io.HierarchicalStreamWriter;

public class PersonConverter implements Converter {

        public boolean canConvert(Class clazz) {
                return clazz.equals(Person.class);
        }

        public void marshal(Object value, HierarchicalStreamWriter writer,
                        MarshallingContext context) {
                Person person = (Person) value;
                writer.startNode("fullname");
                writer.setValue(person.getName());
                writer.endNode();
        }

        public Object unmarshal(HierarchicalStreamReader reader,
                        UnmarshallingContext context) {
                Person person = new Person();
                reader.moveDown();
                person.setName(reader.getValue());
                reader.moveUp();
                return person;
        }
}

//注册转换器
xStream.registerConverter(new PersonConverter());

//输出结果
<person>
  <fullname>Guilherme</fullname>
</person>
```

## 另一种转换器

如果只是想把某个对象转化成字符串，那么有一种简单的实现方式, 实现AbstractSingleValueConverter接口. 

```
package com.thoughtworks.xstream.examples;

import com.thoughtworks.xstream.converters.basic.AbstractSingleValueConverter;

public class PersonConverter extends AbstractSingleValueConverter {

        public boolean canConvert(Class clazz) {
                return clazz.equals(Person.class);
        }

        public Object fromString(String str) {
                Person person = new Person();
                person.setName(string);
                return person;
        }

}

//输出结果
<person>Guilherme</person>
```

## 参考文献
XStream的官方转换器文档: [官方文档](http://x-stream.github.io/converter-tutorial.html)