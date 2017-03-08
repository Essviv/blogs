---
layout:		post
title: 		XStream学习(1)
subtitle:	别名
author: 	essviv
date:		2016-06-14 21:15:00+0800
tag:		XStream 别名
---

# XStream 

## 概览

XStream是用来将对象与XML进行互相转换的第三方库. 它的主要特点包括：

* 简单易用. 
* 无需映射. 
* 性能. 
* 简洁的XML. 
* ...
* 错误信息的输出.
* 支持与JSON格式的转换.

## 别名

假设我们定义了以下的XML格式, 需要通过XStream进行读写：

```
<blog author="Guilherme Silveira">
  <entry>
    <title>first</title>
    <description>My first blog entry.</description>
  </entry>
  <entry>
    <title>tutorial</title>
    <description>
        Today we have developed a nice alias tutorial. Tell your friends! NOW!
    </description>
  </entry>
</blog>
```

首先，我们需要根据XML格式定义以下两个对象模型:

```
package com.thoughtworks.xstream;

public class Blog {
        private Author writer;
        private List entries = new ArrayList();

        public Blog(Author writer) {
                this.writer = writer;
        }

        public void add(Entry entry) {
                entries.add(entry);
        }

        public List getContent() {
                return entries;
        }
}

package com.thoughtworks.xstream;

public class Author {
        private String name;
        public Author(String name) {
                this.name = name;
        }
        public String getName() {
                return name;
        }
}

package com.thoughtworks.xstream;

public class Entry {
        private String title, description;
        public Entry(String title, String description) {
                this.title = title;
                this.description = description;
        }
}

```

接着就可以通过XStream来进行XML格式的输出：

```
public static void main(String[] args) {
        Blog teamBlog = new Blog(new Author("Guilherme Silveira"));
        teamBlog.add(new Entry("first","My first blog entry."));
        teamBlog.add(new Entry("tutorial",
                "Today we have developed a nice alias tutorial. Tell your friends! NOW!"));

        XStream xstream = new XStream();
        System.out.println(xstream.toXML(teamBlog));

}

```

最终我们得到的输出是这样的, 似乎看上去和我们想要的结果有点不一样：

```
<com.thoughtworks.xstream.Blog>
  <writer>
    <name>Guilherme Silveira</name>
  </writer>
  <entries>
    <com.thoughtworks.xstream.Entry>
      <title>first</title>
      <description>My first blog entry.</description>
    </com.thoughtworks.xstream.Entry>
    <com.thoughtworks.xstream.Entry>
      <title>tutorial</title>
      <description>
        Today we have developed a nice alias tutorial. Tell your friends! NOW!
      </description>
    </com.thoughtworks.xstream.Entry>
  </entries>
</com.thoughtworks.xstream.Blog>
```

## 类别名

首先我们需要把类似于'com.thoughtworks.xstream.Blog'这样的包名给去掉，**XStream给我们提供了alias方法来实现类的别名功能**:

```
xstream.alias("blog", Blog.class);
xstream.alias("entry", Entry.class);
```

现在再看来看看我们的输出, 现在它变成了以下这样，离我们期望的结果似乎近了一点：

```
<blog>
  <writer>
    <name>Guilherme Silveira</name>
  </writer>
  <entries>
    <entry>
      <title>first</title>
      <description>My first blog entry.</description>
    </entry>
    <entry>
      <title>tutorial</title>
      <description>
        Today we have developed a nice alias tutorial. Tell your friends! NOW!
      </description>
    </entry>
  </entries>
</blog>
```

## 属性别名

现在我们希望把writer属性改成author，同样地，**XStream提供了aliasField来实现类属性的别名设置**, 现在的输出变成了：

```
xstream.aliasField("author", Blog.class, "writer");

<blog>
  <author>
    <name>Guilherme Silveira</name>
  </author>
  <entries>
    <entry>
      <title>first</title>
      <description>My first blog entry.</description>
    </entry>
    <entry>
      <title>tutorial</title>
      <description>
        Today we have developed a nice alias tutorial. Tell your friends! NOW!
      </description>
    </entry>
  </entries>
</blog>
```

## 隐式集合

再来看看entries的输出，XStream引入了“隐式集合”的概念，当遇到不需要显示集合根结点的情况时，可以将它映射成隐式集合.
在上述的例子中，entries是个列表，默认输出时，会将entries作为集合的根结点，有时候这种输出并不是我们期望的，那么如何将它去掉呢？
**XStream中提供了addImplicitCollection方法来解决这个问题**， 再次输出的结果为：

```
xstream.addImplicitCollection(Blog.class, "entries");

<blog>
  <author>
    <name>Guilherme Silveira</name>
  </author>
  <entry>
    <title>first</title>
    <description>My first blog entry.</description>
  </entry>
  <entry>
    <title>tutorial</title>
    <description>
        Today we have developed a nice alias tutorial. Tell your friends! NOW!
    </description>
  </entry>
</blog>
```

## 属性别名

有时候我们可能会希望将类的某个字段输出成结点的属性，而不是它的子结点，比如在上述的例子中，可以将writer字段输出成属性. **XStream提供了useAttributeFor方法将类的字段作为结点属性输出**

```
stream.useAttributeFor(Blog.class, "writer");
xstream.aliasField("author", Blog.class, "writer");
```

但是，如果要将writer输出成结点的属性，还需要完成一个转换. 那就是Author类如何转换成String值，因为结点的属性值只能是字符类型. **XStream提供了SingleValueConverter接口来实现类与字符类型之间的转换功能**. 因此只要能提供一个converter能将author转化成字符类型，就可以顺利将writer字段作为结点的属性输出. 定义完转化器之后，将它注册到XStream中，就可以得到相应的输出: 

```
class AuthorConverter implements SingleValueConverter {

        public String toString(Object obj) {
                return ((Author) obj).getName();
        }

        public Object fromString(String name) {
                return new Author(name);
        }

        public boolean canConvert(Class type) {
                return type.equals(Author.class);
        }
}

//将新定义的转化器注册到XStream中
xstream.registerConverter(new AuthorConverter());

//输出
<blog author="Guilherme Silveira">
  <entry>
    <title>first</title>
    <description>My first blog entry.</description>
  </entry>
  <entry>
    <title>tutorial</title>
    <description>
        Today we have developed a nice alias tutorial. Tell your friends! NOW!
    </description>
  </entry>
</blog>
```

## 包名别名

有的时候我们可以希望为包名设置别名，**XStream提供了aliasPackage来设置包名的别名**

```
xstream.aliasPackage("my.company", "org.thoughtworks");

//输出
<my.company.xstream.Blog>
  <author>
    <name>Guilherme Silveira</name>
  </author>
  <entries>
    <my.company.xstream.Entry>
      <title>first</title>
      <description>My first blog entry.</description>
    </my.company.xstream.Entry>
    <my.company.xstream.Entry>
      <title>tutorial</title>
      <description>
        Today we have developed a nice alias tutorial. Tell your friends! NOW!
      </description>
    </my.company.xstream.Entry>
  </entries>
</my.company.xstream.Blog>
```

## 参考文献

XStream别名官方文档: [别名](http://x-stream.github.io/alias-tutorial.html)