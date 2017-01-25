#Selector

###简单选择器
* 元素选择器  
  * h1
  
* 类选择器  
  * .className
  * h1.className
  
* ID选择器  
  * \#idName
  * h1\#idName

* 属性选择器
  * [propName]: 拥有某个属性的对象
  * h1[propName][propName]: 同时拥有两个属性

* 简单选择器分组
  * h1, p
  * h2, li

###复合选择器
* 兄弟选择器
  * p + p
  * li + li
  
* 后代选择器(包括多代子元素）
  * p em
  * h1 em

* 子元素（直接子元素）
  * p > h1
  * p > em