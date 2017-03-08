---
title: JS基础 - 表达式
author: essviv
date: 2017-02-03 20:27:00+0800
tags:
- expression
- eloquent js
---

### 变量

变量的声明是通过**var**关键字来完成的，变量可以用来存储值，以供后续使用

变量名区分大小写，可以包含数字、字母及下划线，但**不能以数字开头**.

````javascript
var name = 'essviv';
var one, two = 2, three = 'three';

console.log(name); //'essviv'
console.log(one); //undefined
console.log(two); //2
console.log(three); //three
````

### 函数

函数是JS中比较特殊的一种变量，它将一段代码声明成某个变量，后续可以通过这个变量来引用这个函数，如果要“执行”这个函数的话，只需要在这个变量后增加“()”即可. 

````javascript
var func = function(){
  return 'hello world';
}
console.log(func()); //调用func变量所指向的函数，输出hello world
````

在不同的环境中，预定义了一系列和环境相关的函数：

* alert: 通常在浏览器环境中会定义这个方法
* console.log: 在几乎所有的环境中，都会定义这个方法，事实上，真正的方法名为log
* confirm: 在浏览器环境中通常会定义这个方法，用于向用户展示确认信息
* prompt: 在浏览器环境中通常会定义这个方法，用于获取用户的输入

### 控制流

#### 1. 条件语句

````javascript
语法1: 
if(condition){
  
}

语法2: 
if(condition){
  
}else{
  
}

语法3：
if(condition){
  
}else if(condition_2){
  
}else if(condition_3){
  
}else{
  
}
````

2. 循环语句

   * while…do...

   * do…while...

   * for

````javascript
语法1：
while(condition){
  
}

语法2:
do{
  
}while(condition)
  
语法3:
for(var i = initValue; condition; updateLoop){
  
}
````

3. 跳出循环

   * break
   * continue

4. 多条件分支

   ````javascript
   语法：
   switch(checkValue){
     case 1:
       break;
     case 2:
       break;
     default:
       break;
   }
   ````

### 注释

* 单行注释: 以双斜杠(//)开始
* 多行注释: 以/\*开始， 以*/结尾