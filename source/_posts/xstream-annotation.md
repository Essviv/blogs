---
layout:		post
title: 		XStream学习(2)
subtitle:	注解
author: 	essviv
date:		2016-06-14 21:15:00+0800
tag:		XStream 注解
---

# 注解

在上一章中，我们学习了如何使用XStream的别名机制来更好的控制输出，但是有时候这种控制略显烦琐，因此XStream也提供了相应的注解支持.

本章主要展示了如何使用XStream提供的注解机制来更简单有效地控制相应的输出. 首先我们先定义消息模式，并输出最基本的XML格式：

```
package com.thoughtworks.xstream;
public class RendezvousMessage {

	private int messageType;
	
	public RendezvousMessage(int messageType) {
		this.messageType = messageType;
	}
}

//输出代码
package com.thoughtworks.xstream;
public class Tutorial {

	public static void main(String[] args) {
		XStream stream = new XStream();
		RendezvousMessage msg = new RendezvousMessage(15);
		System.out.println(stream.toXML(msg));
	}

}

//输出结果
<com.thoughtworks.xstream.RendezvousMessage>
  <messageType>15</messageType>
</com.thoughtworks.xstream.RendezvousMessage>
```

## 类别名和属性别名注解

XStream提供了@XStreamAlias注解，它同时提供了alias方法和aliasField方法提供的功能，但XStream并不会自动读取这个注解，必须通过processAnnotation方法显式地指定XStream读取这个注解.

```
@XStreamAlias("message")
class RendezvousMessage {
	@XStreamAlias("type")
	private int messageType;
	
	public RendezvousMessage(int messageType) {
		this.messageType = messageType;
	}
}

//输出代码
public static void main(String[] args) {
		XStream stream = new XStream();
		xstream.processAnnotations(RendezvousMessage.class);
		RendezvousMessage msg = new RendezvousMessage(15);
		System.out.println(stream.toXML(msg));
	}

//输出结果
<message>
  <type>15</type>
</message>
```

## 隐式集合注解

现在让我们来试试如何使用注解完成隐式集合的功能，首先为消息模式添加列表属性，最终得到的输出结果为:

```
@XStreamAlias("message")
class RendezvousMessage {

	@XStreamAlias("type")
	private int messageType;        
	
	private List<String> content;
	
	public RendezvousMessage(int messageType, String ... content) {
		this.messageType = messageType;
		this.content = Arrays.asList(content);
	}
	
}

//输出结果
<message>
  <type>15</type>
  <content class="java.util.Arrays$ArrayList">
    <a class="string-array">
      <string>firstPart</string>
      <string>secondPart</string>
    </a>
  </content>
</message>
```

这显然不是我们想要的输出结果，**XStream提供了@XStreamImplicit注解来提供和addImplicitCollection对应的功能.**在加上这个注解后，输出的结果变成了: 

```
@XStreamAlias("message")
class RendezvousMessage {
	@XStreamAlias("type")
	private int messageType;

	@XStreamImplicit
	private List<String> content;

	public RendezvousMessage(int messageType, String... content) {
		this.messageType = messageType;
		this.content = Arrays.asList(content);
	}
}

//输出结果
<message>
  <type>15</type>
  <a class="string-array">
    <string>firstPart</string>
    <string>secondPart</string>
  </a>
</message>
```

离期望的输出近了一点，但还不是最终的样子，我们期望那个'a‘不要出现，同时期望能够给每个元素取个名字，比如part. **XStream为XStreamImplicit注解了相应的属性设置，itemFieldName可以用来设计集合中每个元素的名称**，将它设置为part后的输出结果为:

```
@XStreamAlias("message")
class RendezvousMessage {

	@XStreamAlias("type")
	private int messageType;

	@XStreamImplicit(itemFieldName="part")
	private List<String> content;

	public RendezvousMessage(int messageType, String... content) {
		this.messageType = messageType;
		this.content = Arrays.asList(content);
	}

}

//输出结果
<message>
  <type>15</type>
  <part>firstPart</part>
  <part>secondPart</part>
</message>
```

## 变换器注解

首先我们先为消息模型增加个时间字段，并查看下输出的结果: 

```
@XStreamAlias("message")
class RendezvousMessage {

	@XStreamAlias("type")
	private int messageType;

	@XStreamImplicit(itemFieldName="part")
	private List<String> content;
	
	private Date time;

	public RendezvousMessage(int messageType, String... content) {
		this.messageType = messageType;
		this.content = Arrays.asList(content);
	}
}

//输出结果
<message>
  <type>15</type>
  <part>firstPart</part>
  <part>secondPart</part>
  <time>2016-06-14 14:02:08.305 UTC</time>
</message>
```

如果我们希望将time属性输出成系统的毫秒数该怎么办呢？
**XStream为我们提供了@XStreamConverter注解来解决这个问题，它的作用是自定义某个类的转化器.**
另外，@XStreamConverter注解还可以通过注解的属性给相应的转化器输入构造函数，具体可以查看注解的javadoc说明.
以下的例子首先定义了Date类型的转化器DateConverter，并设置time使用该转化器，最后的输出结果为:

```
public class DateConverter implements Converter {
    @Override
    public void marshal(Object source, HierarchicalStreamWriter writer, MarshallingContext context) {
        Date date = (Date) source;
        writer.setValue(String.valueOf(date.getTime()));
    }

    @Override
    public Object unmarshal(HierarchicalStreamReader reader, UnmarshallingContext context) {
        String value = null;

        reader.moveDown();
        value = reader.getValue();
        reader.moveUp();

        return new Date(NumberUtils.toLong(value));
    }

    @Override
    public boolean canConvert(Class type) {
        return type.equals(Date.class);
    }
}

@XStreamConverter(DateConverter.class)
private Date time;

//输出结果
<message>
  <type>15</type>
  <part>firstPart</part>
  <part>secondPart</part>
  <time>1465913336042</time>
</message>
```

## 属性注解

在有些场景下，可能希望将对象的某个字段显示成结点的属性而不是子结点，**XStream提供了@XStreamAsAttribute来实现属性注解的功能.**

```
@XStreamAlias("type")
@XStreamAsAttribute
private int messageType;

//输出结果
<message type="15">
</message>
```

在另一些场景下，可能希望将对象的某个字段作为结点的值,而将其它全部字段作为结点的属性输出. **XStream提供了ToAttributedValueConverter类来实现这个功能，它指定某个字段作为结点的值，其它字段作为属性输出**
但是，这里有个前提，由于结点的属性只能是字符类型，因此那些作为属性输出的字段必须能够转化成字符类型. 如果某个字段不能完成这种转换，必须通过显式指定SingleValueConverter接口实现.

```
@XStreamAlias("message")
@XStreamConverter(value=ToAttributedValueConverter.class, strings={"content"})
class RendezvousMessage {

	@XStreamAlias("type")
	private int messageType;

	//这里的list并不能自动转化成字符类型，因此必须显式地指定转化器，否则输出会报错
	@XStreamConverter(SomeImplementOfSingleValueConverter.class)
	private List<String> content;
	
	@XStreamConverter(value=BooleanConverter.class, booleans={false}, strings={"yes", "no"})
	private boolean important;

	@XStreamConverter(SingleValueCalendarConverter.class)
	private Calendar created = new GregorianCalendar();

	public RendezvousMessage(int messageType, boolean important, String... content) {
		this.messageType = messageType;
		this.important = important;
		this.content = Arrays.asList(content);
	}
}

//输出
<message type="15" important="no" created="1154097812245">This is the message content.</message>
```

## 忽略字段

有些时候，对象的部分属性并不需要输出到XML中，这时候可以通过**XStream提供的@XStreamOmitField注解来完成忽略输出的功能.**

## 自动检测注解

在上面的描述中我们提到，在设置了类的注解后，调用代码中必须显式地调用processAnnotation方法来通知XStream来处理这些注解，这种处理有时候略显麻烦. XStream还提供了自动检测注解的方法autodetectAnnotations. 但是，如果调用了processAnnotation方法，那么自动检测的功能就自动被关闭了.

另外，XStream自动检测还有一些弊端，具体可以查阅“参考文献".

## 参考文献
XStream官方注解文档: [注解文档](http://x-stream.github.io/annotations-tutorial.html)