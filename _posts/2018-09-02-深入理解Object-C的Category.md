---
layout: post
title: 深入理解Object-C的Category
date: 2018-09-5 23:34:15.000000000 +09:00
tags: iOS技术
---
参考文档[深入理解Objective-C：Category](https://tech.meituan.com/DiveIntoCategory.html)

上面这篇文章是我看过的最完整的一个对Category的描述。本篇文章只是一个结论性的总结，并对其中的一些概念做说明。

## 1. 前言
category是Objective-C 2.0添加的语言特性，category的主要作用是为已经存在的类添加方法。

app推荐的使用场景
- 可以减少单个文件的体积 
- 可以把不同的功能组织到不同的category里 
- 可以由多个开发者共同完成一个类 
- 可以按需加载想要的category 
- 将私有方法提前声明
 
不过除了apple推荐的使用场景，category的其他几个使用场景：
- 模拟多继承
- 把framework的私有方法公开

补充说明
什么是私有方法提前声明?
Cocoa没有任何真正的私有方法。只要知道对象支持的某个方法的名称，即使该对象所在的类的接口中没有该方法的声明，你也可以调用该方法。不过这么做编译器会报错，但是只要新建一个该类的类别，在类别.h文件中写上原始类该方法的声明，类别.m文件中什么也不写，就可以正常调用私有方法了。这就是传说中的私有方法前向引用。 所以说cocoa没有真正的私有方法。

我们还知道，即使没有引入 Category 的头文件，Category 的方法也会被添加进主类的方法列表里，可以通过 performSelector 的方式使用，导入头文件只是为了通过编译器的静态检查（上面所说的私有方法）。

category如何模拟多继承？
模拟多继承主要是利用可以添加方法，添加属性。既然可以添加属性，也可以添加方法，那么我要的方法与东西，都可以在里面实现。

>
Category实际上就是将这些方法添加到主类的方法列表里的头部。

## 2. category与extension对比

- extension
在**编译器决议**，它是类的一部分。在编译期和头文件的@interface以及实现文件里的@implement一起形成一个完整的类。它伴随类的产生而产生。extension一般用于隐藏类的私有信息。

- category
在**运行期决议**，就category和extension的区别来看，可以推导出，extension可以添加实例变量，而category是无法添加实例变量的。因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部结构，这对编译型语言来说是灾难性的。

## 3. Category的结构与编译
我们知道所有的OC类和对象，在runtime层都是用struct表示的，category也不例外。category结构体如下

```
typedef struct category_t *Category;
typedef struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
} category_t;

```

类的名字，类，实例方法列表，类方法列表，协议列表，属性
从category的定义可以看出category可以添加实例方法，类方法，甚至可以实现协议，添加属性。但无法添加实例变量。
从category的结构体表示可以看出
**在编译时，编译器在DATA段下（静态区）的objc_catlist_section里保存了category的数组。实际上这是个全局数组，所有的编译的category都会放在里面,这个数组供在运行时的加载**
如下是这个全局数组

```
static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [1] __attribute__((used, section ("__DATA, __objc_catlist,regular,no_dead_strip")))= {
&_OBJC_$_CATEGORY_MyClass_$_MyAddition,
};
```

## 4. Category的加载
我们知道，Objective-C的运行是依赖OC的runtime的，而OC的runtime和其他系统库一样，是OS X和iOS通过dyld动态加载的。
当动态加载了OC的runtime时，就会将上面编译的那个全局数组做整理，添加到类的方法列表里。具体结论是

1. 把category的实例方法，协议以及属性添加到类上
2. 把category的类方法和协议添加到类的metaclass上

具体怎么添加的，我们就不看源码了。有兴趣的可以自行查看源码，或者查看参考文档[深入理解Objective-C：Category](https://tech.meituan.com/DiveIntoCategory.html)

需要特别注意的是(特别重要)

1. category的方法没有”完全替换掉“原来已经有的方法，也就是说category和原来类都有的methoda,那么category附加完成后，类的方法列表里会有两个methodA;

2. category的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的category的方法会“覆盖”掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法，就会罢休^_^，殊不知后面可能还有一样名字的方法。还有多个category时，越后的会越在数组的前面。

## 5. Category的+load方法
思考如下两个问题

1. 在类的+load方法调用的时候，我们可以调用category中声明的方法吗?(其实是在思考，category将方法加到主类与load执行的顺序问题)

2. 这些+load方法执行的顺序？（其实是在思考主类与category的执行load的顺序）

```
//Dog+DogCategory.h
@interface Dog (DogCategory)
+ (void)dogJiao;

@end
```
```
//Dog+DogCategory.m
#import "Dog+DogCategory.h"
@implementation Dog (DogCategory)
+ (void)load {
    NSLog(@"%s",__FUNCTION__);
}

+ (void)dogJiao {
      NSLog(@"%s",__FUNCTION__);
}
@end
```
```
//Dog+DogCategory2.h
#import "Dog+DogCategory.h"
@interface Dog (DogCategory2)

@end
```
```
//Dog+DogCategory2
#import "Dog+DogCategory2.h"
@implementation Dog (DogCategory2)
+ (void)load {
    
    NSLog(@"%s",__FUNCTION__);
    
    [self dogJiao];
}
@end
```
我们在DogCategory2中调用了DogCategory中的方法，由此可见，category的方法是在+load执行之前。
+load的执行顺序是先是 类，然后是category，而category的+load方法是按编译顺序。编译顺序在后的+load后执行。因为编译在后，在形成全局的数组时，会被加在数组最前面，这也是为什么，在category有相同的方法时，在执行时会选择后面文件在后面编译的文件。哈哈。

## 6. 怎么调用原来类中被category覆盖掉的方法？
我们已经知道category其实并不是完全替换掉原理的类的同名方法。只是category在方法列表的前面而已。所以我嫩只要顺着方法列表找到最后一个对应名字的方法，就可以调用原来类的方方法。

## 7. category与关联对象
其实category与关联对象没有关系。只是category只能给原有类添加方法，不能添加实例变量，可以用关联对象去解决这个问题罢了。

那么关联对象又是存在什么地方？如何存储？对象销毁的时候如何处理关联对象？
有兴趣的可以去看源码

所有的关联对象都由AssociationsManager管理，而AssociationsManager里面由一个静态AsscociationsHashMap来存储所有的关联对象的。这相当于把所有对象的关联对象存在一个全局的map里面。而map的key就是这个对象的指针地址（任意两个不同对象的指针地址一定是不同的），而这个map的value又是另外一个AssociationsHashMap,里面保存了关联对象的kv值。在对象销毁时，会查看这个对象有没有关联对象。如果有就会完成关联对象的移除工作。

