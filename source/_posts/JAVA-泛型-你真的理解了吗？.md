---
title: JAVA 泛型 你真的理解了吗？
tags:
  - java基础
categories:
  - java基础
date: 2019-06-19 16:16:53
---
## 记录原由 

之前虽然泛型一直在使用，但使用的过程中总是没有那么得心应手，有些细节还是过于模糊。究其原因其实是一直都没有系统深入的去理解过，最近花了一点时间去深入的理解了一下java的泛型机制，也希望借这次记录能够彻底的理解java的泛型

## 为什么要有泛型

**泛型的本质是把 类型参数化**，什么是类型参数化？举个例子，我们常用的集合类，要是没有类型参数化的话，我们就得实现装不同类型的集合类，会大大降低代码的可重用性！

例如我们要设计一个装苹果的盘子和一个装香蕉的盘子，要怎么设计呢？非常简单，最简单的方式就是设计两个盘子类，一个是苹果盘子，一个香蕉盘子，就像下面的代码一样

```java
public class ApplePlant {

    private Apple apple;

    public Apple getApple() {
        return apple;
    }

    public void setApple(Apple apple) {
        this.apple = apple;
    }
}

public class BananaPlant {

    private Banana banana;

    public Banana getBanana() {
        return banana;
    }

    public void setBanana(Banana banana) {
        this.banana = banana;
    }
}
```
上面的代码的确能够满足我们的需求，但就像上文所说的，这样的代码降低了代码的可重用性，那么我们要怎么优化呢，聪明的程序员们马上想到了，可以用`Object`来代替盘子里所指代的具体类型，就像下面这样

```java
public class ObjectPlant {
    
    private Object object;

    public Object getObject() {
        return object;
    }

    public void setObject(Object object) {
        this.object = object;
    }
}
```
上面的代码看似完美解决了代码不可重用的问题，但同时也隐含了很多其他问题，例如在我们取数据的时候，取出来的只能是`Object`类型，需要我们强转类型，而往盘子中加入数据的时候，也没有任何验证，有可能往放苹果的盘子中加入了香蕉也说不定，因为编译器并没有做任何限制，只要是`Object`类型，都可以验证通过。

上面的问题在Java 1.5引入了泛型之后得到了完美的解决，下面让我们来看看用泛型如何优雅的解决上述问题
```java
public class Plant <T>{

    private T t;

    public T getT() {
        return t;
    }

    public void setT(T t) {
        this.t = t;
    }
}
```
在上面的代码中，我们定义了一个泛型类，它表示了一个可以`T`类型的篮子，而这个`T`类型需要到我们真正使用的时候才会去指定

例如现在我们要定义两种盘子，可以这样声明
```java
Plant<Apple> applePlant = new Plant<>();
Plant<Banana> bananaPlant = new Plant<>();
Apple apple = new Apple();
Banana banana = new Banana();
applePlant.set(banana); // 编译报错
bananaPlant.set(apple); // 编译报错
```
上面的代码之所以实际类型`new Plant<>()`的`<>`中没有指明具体类型，是因为Java7之后加了类型自动推断，也就不需要两侧都加上泛型类型

可以看到使用了泛型之后，既提高了代码的可重用性，在实际使用时也对类型进行了约束，装苹果的盘子是装不了香蕉的！

不过要注意的是，java的泛型约束是在编译时生效的，一旦编译成了class字节码文件后，一切都打回原形了，泛型信息会被擦除，所以我们把`JAVA`的泛型称为伪泛型，跟`C#`等语言的真泛型有着本质区别，在`C#`中，`List<Integer>`和`List<String>` 就是两种不同的类型

如下面字节码文件的最后一行`invokevirtual`命令的方法描述符`Method setT:(Ljava/lang/Object;)V`，可知泛型类型已经被擦除成了`Object`类型，也就是在字节码层面实际上与我们直接用`Object`来实现是一样的

```
0: new           #3                  // class com/sunshine/common/test/Plant
         3: dup
         4: invokespecial #4                  // Method "<init>":()V
         7: astore_1
         8: aload_1
         9: new           #5                  // class com/sunshine/common/test/Apple
        12: dup
        13: invokespecial #6                  // Method com/sunshine/common/test/Apple."<init>":()V
        16: invokevirtual #7                  // Method setT:(Ljava/lang/Object;)V
```

## 泛型进阶

### 使用 `<T extends ?>` 定义泛型类

头一次看到上面两个符号，是不是有点懵逼？它们在代码里也非常常见，那么它们究竟是表示什么意思呢？
还是通过上文的`plant`盘子来举例吧，举一种情况，我们只希望这个盘子用来装一些水果，而不希望它被用来装肉，那么我们该怎么做呢？
现在的情况是，它既可以实例化成一个装香蕉的盘子，也可以被实例化成一个装肉的盘子
```java
Plant<Apple> applePlant = new Plant<>(); // 
Plant<Meat> meatPlant = new Plant<>(); // 都是Ok的
```

现在让我们来重新设计一个盘子类，通过`<T extends Firut>`来限制泛型，其实如果我们直接用`<T>`的编译成字节码后也会自动编译成`<T extends Object>`

```java
public class FirutPlant <T extends Firut>{

    private T t;

    public T getT() {
        return t;
    }

    public void setT(T t) {
        this.t = t;
    }
}
```

使用
```java
FirutPlant<Apple> appleFirutPlant = new FirutPlant<>(); // OK
FirutPlant<Firut> firutPlant = new FirutPlant<>(); // OK
FirutPlant<Meat> meatFirutPlant = new FirutPlant<>(); // 编译无法通过
```
通过上面的代码可知我们成功的限制了泛型，水果盘子就只能装水果，盘子中的类型只能是水果或则它的子类型。


### `<? extends T> <? super T>` 

注意这里的T是泛型参数，跟上面的<T extends ?> T不一样

看完上面的 `<T extends ?>`，接下来我们再来看`<? extends T> <? super T>`，可能这里你会非常疑惑，哇 这两个有什么区别？其实简单理解的话，`<T extends ?>`是在类文件层面就限制了泛型的范围，而`<? extends T> <? super T>`是在使用泛型的时候再去做限制。明白了这个之后接下来我们就来看下`<? extends T> <? super T>`的使用。

让我们回到`Firut.class`

```java
public class Plant <T>{

    private T t;

    public T getT() {
        return t;
    }

    public void setT(T t) {
        this.t = t;
    }
}
```

我们现在不在定义泛型类的时候做限制，(使用`<T>`，其实这里还是相当于`<T extends Object>`)，而是在使用的是去做限制

下面我们来看下`<? extends T>`的使用

```
Plant<Apple> applePlant = new Plant<Apple>();
applePlant.setT(new Apple());
// 使用 <? extends T>
Plant<? extends Firut> plant = applePlant;

plant.setT(new Apple()); // 无法存 编译报错
plant.setT(new Firut()); // 无法存 编译报错
Firut apple = plant.getT(); // 可以获取
```
可以看到当`<? extends T>`做为接收参数时，因为其必然是`Firut`的子类，这里实例类型是`Apple`，所以`getT()`方法仍然可以使用，因为会进行一次隐式的类型转换（向上转型是安全的），但是`setT()`方法是失效的，这是因为编译器不能确定你一定会往里面添加`Apple`类型，因为你还可以往里面添加`Banana`类型，显然`Banana`类型无法强转`Apple`类型，所以这里的`setT()`方法是失效的

下面我们来看下`<? super T>`的使用
有了上面`<? extends T>`的使用经验，顾明思议，`<? super T>`必然表示`T`的全部父类型，还是再来看下`<? super T>`的使用吧

```
Plant<Firut> firutPlant = new Plant<>();
Plant<? super Apple> plant = firutPlant;


plant.setT(new Apple()); // 可以存成功，不过这里只能存 Apple类型
Apple apple = plant.getT(); // 编译报错

Firut firut = firutPlant.getT();
```
上面可以存成功的原因和上面`<? extends T>`可以取成功的原因一样，也是因为存在一个隐式的向上转型，而获取失败的原因 也很容易理解，因为实际类型肯定是`Apple`的父类（这里是Firut），所以要将`Firut`转成`Apple`显然是不可行的！


###  <?>

`<?>`其实等价与`<? extends Object>`，这样子我相信就比较好理解了，跟`<? extends T>`类似，也是取可以，存被限制

`<T>`一般都是在定义泛型类或则泛型方法时出现，实际使用时都以`?`或具体类型替代`T`

### Type 与 Class 的区别和联系

JDK在1.5之后因为泛型的原因引入了`Type`类型，那么`Type`到底是什么呢，它表示的范围有多大？？ 其实我们可以直接理解把`Class`当成是`Type`的子集，`Class`对应着jdk1.5之前不是泛型的原始类型

`Type`下包含了几种不同的类型

* `Class` 原始类型，我这里直接理解成我们常用的`Class`类型，`Class`是`Type`的直接子类
* `ParameterizedType` 参数化类型类似 `List<String> list` 获取list的类型
* `TypeVariable` 类似 `T t`
* `GenericArrayType` 类似 `List<String>[]` 或则 `List<T> []`
	* `GenericArrayType` 接口只有一个方法`getGenericComponentType`，用户返回`ParameterizedType`或则`TypeVariable`类型
	
* `WildcardType` 类似 `<? super T>` 要得到这个类型，首先需要得到`ParameterizedType`类型，然后通过`getActualTypeArguments`方法获取`WildcardType`类型

说了这么多，还是通过实际的代码来详细解释吧
```java
public class TypeTest {
    
    public void testOne(Plant<Apple> plant, 
                        Plant<Apple> [] plants, 
                        Plant<? extends Apple> applePlant) {   
    }
    public <T> void testOne(T t) {
    }
}
```

写段测试代码
```java
Method [] methods = TypeTest.class.getMethods();
for (Method method : methods) {
    if (method.getName().equals("testOne")) {
        Type [] parameterTypes = method.getGenericParameterTypes();
        System.out.println("methodOne start --------> ");
        for (Type type : parameterTypes) {
            if (type instanceof Class) {
                System.out.println("Class ---- " + type.getTypeName());
            }
            if (type instanceof ParameterizedType) {
                System.out.println("ParameterizedType ---- " + type.getTypeName());
                Type ytpe3[] = ((ParameterizedType) type).getActualTypeArguments();
                for (Type type2 : ytpe3) {
                    if (type2 instanceof WildcardType) {
                        System.out.println("ParameterizedType-WildcardType  ---- " + type2.getTypeName());
                    }
                }
            }
            if (type instanceof GenericArrayType) {
                System.out.println("GenericArrayType ---- " + type.getTypeName());
                Type type2 = ((GenericArrayType) type).getGenericComponentType();
            }
            if (type instanceof TypeVariable) {
                System.out.println("TypeVariable ---- " + type.getTypeName());
            }
        }
    } else if (method.getName().equals("testTwo")) {

        Type [] parameterTypes = method.getGenericParameterTypes();
        System.out.println("methodTwo start --------> ");
        for (Type type : parameterTypes) {
            if (type instanceof TypeVariable) {
                System.out.println("TypeVariable ---- " + type.getTypeName());
            }
        }
    }
}

```

输出

```
methodTwo start --------> 
TypeVariable ---- T
methodOne start --------> 
ParameterizedType ---- com.sunshine.common.test.Plant<com.sunshine.common.test.Apple>
GenericArrayType ---- com.sunshine.common.test.Plant<com.sunshine.common.test.Apple>[]
ParameterizedType ---- com.sunshine.common.test.Plant<? extends com.sunshine.common.test.Apple>
ParameterizedType-WildcardType  ---- ? extends com.sunshine.common.test.Apple
```

虽然我们再前文说过，因为在字节码层面泛型类型会被擦除成`Object`，所以运行时无法拿到泛型类型。那么我们要如何拿到JAVA的泛型类型呢？

JAVA虚拟机由此引入了`Signature`属性，`Signature`属性是方法在字节码层面的特征签名，也就是说`Signature`属性中包含泛型信息，虽然在`Code`属性表中泛型信息确实被擦除了，这也是我们可以通过反射获取到泛型真实类型的原因！





