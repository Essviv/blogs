---
title: JS基础 - 对象的生命周期
author: essviv
date: 2017-02-07 08:56:00+0800
tags:
	- JS
	- Object
---

* 当函数作为对象的方法被调用时，**this**变量指向调用这个方法的对象

````javascript
例1: obj.method(); //this指向obj

例2:
	var obj = {
		speak: function(){

		}
	};

	var func = obj.speak;
	func(); //this指向全局变量，而不是obj
````

### Prototypes

关于原型的作用，以下这段话做了很好的阐述，它是作为对象属性的“后备”来源的**另一个对象**

当请求对象的属性时，首先检查对象本身是否定义了这个属性，如果没有找到，则从对象的原型中查找，如果对象的原型中也没有定义相应的属性，则从原型的原型查找，自下而上形成了”原型链". 原型链的顶端是Object.prototype对象

> A prototype is another object which used as fallback source of properties

**prototype属性的含义**

在每个对象中，都有两个属性，一个叫\__proto\__, 一个叫prototype, 在刚接触原型的时候，经常会搞混这两个属性含义，这里做个澄清：

* \__proto\__: 这个属性是对象在查找属性时访问的原型对象，也就是”原型链”中的对象，在通过new操作符创建对象时，会把对象的prototype复制给\__proto\__属性, 通过Object.getPrototypeOf方法返回的也是这个属性.

* prototype: 这个属性是在创建新的对象时，新对象的\__proto\__属性的来源

> \__proto\__ is the actual object that is used in the lookup chain to resolve methods, etc. prototype is the object that is used to build \__proto\__ when you create new object with new.

````javascript
var car = new Car();
console.log(car.__proto__ == Car.prototype); //true
console.log(car.prototype === undefined); //true
console.log(Object.getPrototypeOf(car) == car.__proto__); //true
console.log(Object.getPrototypeOf(car) == car.prototype); //false
````
### Constructor

在JS中, 使用new操作符来调用方法的时候，就把这个方法当作是新建对象的构造器. 关于构造器，有以下几个事实需要说明：

* 构造器自动获得一个名为prototype的属性，默认情况下它是一个空对象，这个空对象是从Object.prototype继承来的

* 使用该构造器创建的所有对象都将以这个空对象为prototype

* 即 constructor.prototype == Object.getPrototypeOf(instance)

````javascript
//Car是普通的方法
function Car(){

}

var car = new Car(); //这里把Car方法当作是构造函数来使用

//测试constructor
console.log(car.constructor == Car); //true
console.log(Car.prototype == Object.getPrototypeOf(car)); //true
console.log(Object.getPrototypeOf(Car) == Function.prototype); //true
````

### 原型干涉

从原型的定义可以看出，可以在原型中增加相应的属性，从而所有实例都可以自动获取到这个属性，绝大部分情况下，这是我们期望得到的结果，但有些时候，这可能会给我们造成困扰. 如下代码所示：
````javascript
Car.prototype.wheelCount = 4;
car.wheelCount = 5;

console.log(car.wheelCount); //5
console.log(new Car().wheelCount); //4

//原型干涉
function put(event, data){
  map[event]  = data;
}

function printAll(){
    for(var obj in map){
        console.log(obj);
    }
}

var map = {};
put('pizza', 'dfsadf');
put('touched', 'toudsafd');

printAll();

Object.prototype.nonsense = 'hi';
printAll();

console.log('nonsense' in map); //true
console.log('toString' in map); //true

delete Object.prototype.nonsense;
````

从以上代码执行的结果可以看出，虽然我们并没有定义nonsense和toString属性，但通过Object的原型继承得到了这两个属性，并且，虽然toString属性没有被列举出来，但通过in操作的结果也可以看出，这个属性也被继承了. 事实上， JS中将属性分了**enumerable**和**nonenumerable**两大类.默认情况下，通过赋值操作新增的属性都是enumerable的，而Object.prototype中定义的属性都是nonenumerable的，这也是为什么toString没有被列举出来的原因. 如果需要定义nonenumerable的属性，可以通过Object.defineProperty方法来完成，如下所示：

````javascript
//通过defineProperty方法来声明属性
Object.defineProperty(Object.prototype, 'hiddenNonse', {enumerable: false, value: 'hi'});
printAll();
console.log('hiddenNonse' in map); //true
console.log(map.hasOwnProperty('hiddenNonse')); //false
delete Object.prototype.hiddenNonse;
````
