---
title: 全面解析JS面向对象 原型链
tags:
  - javascript
categories:
  - 前端技术
date: 2017-05-04 14:33:04
---

对`javascript` 这门语言来说，相信大家都不会陌生，但最近发现其实好多人对JS对象的概念很模糊，甚至会惊异于JS竟然也有对象
>其实JS是完全基于对象的语言，我们所能看到的都是基于对象，包括 js 方法其实也是一个对象

但是js的面向对象与像java的面向对象相比,又是有着极大的不同，java，C#完全面向对象的语言 中有**类**这个概念，而js对象则是依靠 构造器`constructor`利用 原型`prototype`构造出来的。

>简单的理解下，类的概念相当于 模板，原型相当于 各个对象组成新的对象

先来总结一下创建对象的3种方式
### 创建对象的三种方式：

1.通过JSON符号创建
```javascript
var dog = {
    name: 'singledog',
    wangwang: function(){
      console.log(this.name+"旺旺！");
    }
  };
  dog.wangwang();// singledog旺旺！
```
2.通过new 关键字创建
```javascript
function Dog(name){
    this.name = name;
  };
  Dog.prototype.wangwang = function() {
    console.log(this.name+"wangwang!!");
  };
  var dog = new Dog("SingleDog:");
  dog.wangwang();//SingleDog:wangwang!!
```

3.通过 ES5新引进的 Object.create() 方法创建

```javascript
var dog = {
    name: 'singledog',
    wangwang: function(){
      console.log(this.name+"旺旺！");
    }
  };
  var jDog = Object.create(dog);//创建一个新的对象 jDog 并继承 dog 对象
  jDog.wangwang();//singledog旺旺！
```

### 原型（prototype）

**prototype（原型）**是js面向对象中非常重要的一个概念，是深入学习js是必须要理解的一个概念
每个方法 都有一个`prototype`属性，我们可以利用这个属性来大做文章
之前介绍了，我们生成一个需要用到 new 关键字，那么这个new 关键字与JAVA中的new一样吗？

答案是完全不一样

可以看一个例子：
```javascript
function Car(name){
    this.name = name
  };
  Car.prototype = {//指定 Car的原型 对象
    driver: function(){
      console.log(this.name+" driver!!");
    }
  };

  var obj = new Object();
  obj.__proto__ = Car.prototype;
  Car.call(obj,"Benz");
  obj.driver(); //Benz driver!!
```
在上面的代码中，我们发现 `var car = new Car(); `这句被我们替换成了这三句：
  ```javascript
var obj = new Object();
  obj.__proto__ = Car.prototype;
  Car.call(obj,"Benz");
```
以上代码中，我们先新建了一个空的`Object `对象 obj，然后将Car的原型赋给了 obj对象的`__proto__`，这样子obj对象就继承了Car的原型对象，最后一步，在obj对象上执行构造方法Car，这样子 obj 对象就相当于拿来了 Car方法中 全部this. 属性或方法 
>这三步其实等于` new` 操作

基于原型，我们可以实现原型链的继承，因为每一个 prototype 都是一个对象，每个对象都隐式的拥用一个 `prototype` 引用 `__proto__` 属性，这样子当调用一个对象的方法的时候，就会顺着`__proto__`属性一层一层往上找，直到找到为止。

说起来似乎很难理解，看个例子帮助大家理解：
```javascript
function Car(){
  };
  Car.prototype = {//指定 Car的原型 对象
    name : 'car',
    driver: function(){
      console.log(this.name+" driver!!");
    }
  };

  function QqCar(){
    this.name = 'QQ'
  };

  QqCar.prototype = new Car(); //指定 QqCar的原型为 new Car()

  var qqCar = new QqCar();//继承了driver 方法 
  qqCar.driver(); // QQ driver!!

  function BigQqCar(){
    this.name = 'BigQQ'
  };

  BigQqCar.prototype = qqCar; //指定 BigQQCar的原型为 qqCar

  var bigQqCar = new BigQqCar();
  bigQqCar.driver = function(){//重写了driver方法
    console.log(this.name+" flying!!");
  };
  bigQqCar.driver();//BigQQ flying!!

  console.log(bigQqCar.__proto__===BigQqCar.prototype); //true 
  console.log(BigQqCar.prototype === qqCar); //true
  console.log(BigQqCar.prototype.__proto__ === QqCar.prototype); //true

  // 以上代码就是一个简单的原型链,每个构造方法生成的对象都有一个 __proto__ 指向 其构造方法的prototype 属性
```
>以上也只是个人对JS面向对象的理解，欢迎留言补充
