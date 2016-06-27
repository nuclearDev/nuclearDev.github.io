---
layout: post
title:  "黑魔法 - Method Swizzling"
date:   2016-05-04
excerpt: "method swizzling可以在不知道一个类的源码以及实现原理的情况下，不需要通过继承或者重写来改变这个类的功能，被誉为黑魔法"
tag:
- iOS 
- method swizzling 
- runtime 
comments: true
---

## 简介

在OC的runtime中，当一个消息被发送给一个对象，就有一个方法会被调用，这意味着一个给定的Selector对应的method可以在runtime中发生改变。利用这一特性可以在不知道一个类的源码以及实现原理的情况下，不需要通过继承或者重写来改变这个类的功能。与继承和重写相比，被改变的功能适用于这个类以及它的子类。这个黑魔法就叫`method swizzling`。

## 实现原理

`Method Swizzling`是改变一个Selector的实际实现的技术。通过这一技术，我们可以在运行时通过修改类的分发表中selector对应的函数，来修改方法的实现。

一个类中的方法列表包含了一系列的Selector映射列表，当给定一个method时，动态消息系统会在这个列表中寻找对应的实现方法。这些实现方法被存储在一个叫做`IMP`的方法指针中，它有一个属性:`id (IMP *)(id, SEL, ...)`

这里可能有点绕，method，Selector，message总觉得差不多。这里需要了解下这几个的概念，否则下面讲的可能就会迷迷糊糊的了。

* Selector

>a Selector is the name of a method.

  Selector是一个方法的名称，例如咱们都非常熟悉的`alloc, init, release, dictionaryWithObjectsAndKeys:, setObject:forKey:`在开发过程中，指定按钮的点击事件时常用到的`@selector(doSomething:)`,就是指定了方法名。

* message：

 >a message is a selector and the arguments you are sending with it.

   message就是包含有参数的Selector，例如`[dictionary setObject:obj forKey:key],`；这里的Selector就是`setObject:forKey:`

* method

>a method is a combination of a selector and an implementation (and accompanying metadata).

  method是Selector和implementation的结合

还有：

* implementation

>the actual executable code of a method. Its type at runtime is an IMP, and it's really just a function pointer.

  好吧，这个概念应该比较不会混淆，不过需要注意的是，implementation在runtime中就是一个函数指针

**一个类维护一个运行时可接收的消息分发表；分发表中的每个入口是一个方法(Method)，其中key是一个特定名称，即选择器(SEL)，其对应一个实现(IMP)，即指向底层C函数的指针。**

## 简单使用

在NSString中，有三个selector：`lowercaseString`, `uppercaseString`, `capitalizedString`分别对应三个不同的`IMP`:


![NSString's selector](https://raw.githubusercontent.com/nuclearDev/nuclearDev.github.io/master/_image/swizzling1.jpeg)

使用`method swizzling`可以在这个列表中添加一个新的selector，或者交换两个selector对应的指针，之后这个表格是长这样的：

![增加交换指针](https://github.com/nuclearDev/nuclearDev.github.io/raw/master/_image/swizzling2.jpeg)

##### 交换两个selector对应的指针

交换两个指针需要用到的方法有：

```
//交换两个method的对应的implementation，也就是IMP
void method_exchangeImplementation(Method m1, Method m2)

//获取一个类中给定的selector对应的method
Method class_getInstanceMethod(Class aClass, SEL aSelector)
```

使用这两个函数就能交换对应的指针了：

```
//获取两个selector对应的method
Method originalMehtod = class_getInstanceMthod([NSString class],@selector(lowercaseString) );
Method swappedMethod = class_getInstanceMethod([NSString class], @selector(uppercaseString));

//交换两个method
god method_exchangeImplementation(originalMethod,swappedMethod);
```

这么做之后，当你调用NSString的`lowercaseString`方法执行的功能会变成`uppercaseString`所实现的功能。

##### 添加一个selector

在实际开发过程中，交换两个方法的功能用处不大，但是添加一个功能就不同了。
例如现在想要在每次调用`lowercaseString`方法的时候加一个log，说明我调用了这个方法，可以在写一个NSString的category，并在这个类中添加这个功能：

```
@interface NSString (NSStringLowerCaseLog)
- (NSString *)myLowerCaseString;
@end


@implementation NSString (NSStringLowerCaseLog)
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Method originalMethod = class_getInstanceMethod([NSString class], @selector(lowercaseString));
        Method swapteMethod = class_getInstanceMethod([NSString class], @selector(myLowercaseString));
        method_exchangeImplementations(originalMethod, swapteMethod);
    });
}

- (NSString *)myLowercaseString {
    //相当于调用了之前的'lowercaseString'方法
    NSString *lowercase = [self myLowercaseString];
    NSLog(@"%@ => %@",self, lowercase);
    return lowercase;
}
@end
```

## 注意事项：

#### 1. Swizzling应该总是在+load中执行

在Objective-C中，运行时会自动调用每个类的两个方法。+load会在类初始加载时调用，+initialize会在第一次调用类的类方法或实例方法之前被调用。这两个方法是可选的，且只有在实现了它们时才会被调用。由于method swizzling会影响到类的全局状态，因此要尽量避免在并发处理中出现竞争的情况。+load能保证在类的初始化过程中被加载，并保证这种改变应用级别的行为的一致性。相比之下，+initialize在其执行时不提供这种保证—事实上，如果在应用中没为给这个类发送消息，则它可能永远不会被调用。

#### 2. Swizzling应该总是在dispatch_once中执行

与上面相同，因为swizzling会改变全局状态，所以我们需要在运行时采取一些预防措施。原子性就是这样一种措施，它确保代码只被执行一次，不管有多少个线程。GCD的dispatch_once可以确保这种行为，我们应该将其作为method swizzling的最佳实践。

## 参考

[Objective-C Runtime 运行时之四：Method Swizzling](http://southpeak.github.io/blog/2014/11/06/objective-c-runtime-yun-xing-shi-zhi-si-:method-swizzling/)

[Method Swizzling](http://nshipster.com/method-swizzling/)

[stackoverflow: What's the difference between a method and a selector?](http://stackoverflow.com/questions/5608476/whats-the-difference-between-a-method-and-a-selector)