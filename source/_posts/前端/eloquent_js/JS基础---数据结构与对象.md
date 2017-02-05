---
title: JS基础 - 数据结构与对象
author: essviv
date: 2017-02-04 21:09:00+0800
tags:
	-js
	-Object
---

### 数组

* 数组中的元素可以是不同类型的
* 数组下标以0开始

````javascript
例子：
var arr = [2, 3, 4, '5', function(){}, {}];
for(var obj in arr){
  console.log(arr[obj]);
}
````

### 属性

* 获取属性的方法有两种：
  * obj.pro: 点号后面的属性名必须是有效的属性名
  * obj[pro]: 括号中可以是任何形式的表达式，表达式先经过计算，再被当作对象的属性获取相应的属性信息

````javascript
var obj = {
  name: 'essviv',
  age: 27,
  3: 103
}

var propName = 'age';
console.log(obj.name); //essviv
console.log(obj[propName]); //27
console.log(obj[3]); //103
````

* **数组可以认为是属性全部为数字的对象**
* 值为函数的属性被称为是方法

````javascript
var obj = {
  sayHello: function(name){
    console.log('hello, ' + name);
  }
}

obj.sayHello('essviv'); //hello, essviv
````

* 对象的属性通过key: value的形式指定，尝试访问不存在的属性会返回undefined, 可以通过=进行新增或更新属性操作，delete操作符可以将对象的属性删除

````javascript
obj = {
    name: '熊猫',
    // age: 27
};

console.log(obj.name);

judge(obj, 'age');

obj.age = 27; //赋值操作
judge(obj, 'age');

delete obj.age; //删除属性操作
judge(obj, 'age');

function judge(obj, prop){
    console.log(prop in obj); //判断当前属性是否存在
    console.log(obj[prop]); //直接打印
}
````

