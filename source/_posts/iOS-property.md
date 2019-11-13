---
title: iOS-认识@property
date: 2017-11-16 12:20:29
tags: "iOS"
categories: "iOS"
---

### 前言

关于`@property`基础的一次总结学习

<!--more-->

---
### 属性与成员变量

当我们写下`@property NSObject *foo`时，编译器帮我们做了以下几件事(这个过程也被称为“自动合成(autoSynthesize)“)

* 创建成员变量`_foo`
* 声明foo属性的`setter`、`getter`方法
* 实现foo属性的`setter`、`getter`方法

但是很久之前的GCC编译器时代，声明一个属性，需要分三步书写
```
.h
{
	NSObject *foo;
}
@property NSObject *foo
.m
@synthesize foo;
```
创建`foo`成员变量,`@property`负责声明`setter`/`getter`方法，而`@synthesize`则负责实现这两个方法

后来苹果将编译器改为LLVM，我们就不必再为属性对应声明成员变量，编译器自动创建一个下划线开头的`_foo`成员变量，并且为我们实现`@synthesize foo = _foo`的属性与成员变量的对应，这就解释了为什么我们重写`setter`/`getter`方法时，都是操作的`_foo`成员变量，而不是`foo`。当然，如果你愿意，可以不使用系统自动生成的`_foo`的命名,自己用`@synthesize foo = differentFoo`去指定一下这个成员变量名字。

与自动合成相对应的是`@dynamic`，作用是告诉编译器取消自动执行的上述三个步骤，对应操作与开发者自己实现。最为常见的是程序中实现对`NSUserDefault`读写的管理类，这个类的属性就不需要编译器帮我们自动合成

---

### @property中属性的作用

```
// 无数次不厌其烦的书写是因为烦也得写
@property (nonatomic, assign) NSInteger testTag;
@property (nonatomic, strong) UIButton *btn;
```


#### 原子性:atomic&nonatomic

* atomic:原子性，这个属性是默认的，通过在**setter**中加`@synchronized(self)
`保证数据数据的**读写安全**，需要注意的是，但它**不是线程安全**的
* nonatomic:非原子性，就是不加锁

atomic和nonatomic的区别在于，系统自动生成的getter/setter方法不一样。如果自己写getter/setter，那atomic/nonatomic/retain/assign/copy这些关键字只起提示作用，写不写都一样

**nonatomic**
```
//@property(nonatomic, retain) UITextField *userName;
//系统生成的代码如下：

- (UITextField *) userName {
    return userName;
}

- (void) setUserName:(UITextField *)userName_ {
    [userName_ retain];
    [userName release];
    userName = userName_;
}
```
**atomic**
```
//@property(retain) UITextField *userName;
//系统生成的代码如下：

- (UITextField *) userName {
    UITextField *retval = nil;
    @synchronized(self) {
        retval = [[userName retain] autorelease];
    }
    return retval;
}

- (void) setUserName:(UITextField *)userName_ {
    @synchronized(self) {
      [userName release];
      userName = [userName_ retain];
    }
}
```
atomic通过**加锁@synchronized(self)**来保障读写安全，并且在getter中引用计数同样会 +1，来向调用者保证这个对象会一直存在。假如不这样做，如有另一个线程调setter可能会出现[线程竞态](http://www.jianshu.com/p/358535119e9b)，导致引用计数降到0，原来那个对象就释放掉了

> [参考文章：《爆栈热门iOS问题 atomic和nonatomic有什么区别？》](http://www.jianshu.com/p/7288eacbb1a2)

---

#### 读写权限:readwrite&readonly

* readwrite:可读写属性，默认属性，允许编译器`@synthesize`自动合成
* readonly:只读属性，不生成`setter`

---

#### 方法名定义:getter=/setter=

看一下`UIButton`中的应用，一般都是在`Bool`类型的属性时使用，方便阅读
```
@property(nonatomic,getter=isEnabled) BOOL enabled;                                  // default is YES. if NO, ignores touch events and subclasses may draw differently
@property(nonatomic,getter=isSelected) BOOL selected;                                // default is NO may be used by some subclasses or by application
@property(nonatomic,getter=isHighlighted) BOOL highlighted;  
```
---

#### 内存管理

* assign
* unsafe_unretained
* strong
* retain
* weak
* copy


##### assign

assign用于值类型，如int、float、double和NSInteger，CGFloat等表示单纯的赋值，简单覆盖原值，没有多余的操作


##### unsafe_unretained

同assign一样，但是对象销毁后，属性指针并不等于`nil`而是指向了一块野地址，形成野指针。如果访问，则会出现`BAD_ACCESS`


##### strong/retain

id类型及对象类型的所有权修饰符。strong是在iOS引入ARC的时候引入的关键字，是retain的一个可选的替代。表示成员变量对传入的对象要有所有权关系，即强引用。strong跟retain的意思相同并产生相同的代码，但是语意上strong更能体现对象的关系。作用是在setter中，对新值retain，对旧值release，并返回新值


##### weak

使用范围同strong。区别在于在setter方法中，对传入的对象不进行引用计数加1的操作，即对传入的对象没有所有权。属性所指的对象销毁时，属性也会清空（nil out）


##### copy

会将修饰的结果copy一份，重新保存。所以通常用于修饰**NSString/NSArray/NSDictionary**的属性保护封装性。因为**多态性**，父类指针可以指向子类，如果给一个上述不可变属性赋值了他们的子类，那么在子类发生增删时，同样会影响了当前属性的值，违背了**不可变的初衷**，使程序变的不可控。而mutable类型copy操作后，就会保存当前的值储存，成为不可变的类型**(immutable)**。

我们简单写个例子，检验一下`copy`对属性的影响：

1. 声明两个分别用`strong`及`copy`修饰类型为**可变数组**的属性
2. 声明两个分别用`strong`及`copy`修饰类型为**不可变数组**的属性

```
@property (nonatomic, strong) NSMutableArray *smArr;
@property (nonatomic, copy) NSMutableArray *cmArr;
@property (nonatomic, strong) NSArray *strongImArr;
@property (nonatomic, copy) NSArray *CopyImArr;
```

接下来分别给他们赋值一个**可变数组**的局部变量，并在改变这个局部变量前后打印几个属性的信息

```
// 初始化一个局部可变数组
    NSArray *arr = @[@"1"];
    NSLog(@"original arr address:%p class:%@",arr,[arr class]);
    NSArray *carr = [arr copy];
    NSLog(@"copy arr address:%p class:%@",carr,[carr class]);
    NSMutableArray *marr = [arr mutableCopy];
    NSLog(@"mutable arr address:%p class:%@",marr,[marr class]);

// 将这个可变数组分别赋值给声明的属性并打印信息
    self.cmArr = marr;
    NSLog(@"copy mutable property address:%p class:%@ property:%@",self.cmArr,[self.cmArr class],self.cmArr);
    self.smArr = marr;
    NSLog(@"strong mutable property address:%p class:%@ property:%@",self.smArr,[self.smArr class],self.smArr);
    
    self.strongImArr = marr;
    NSLog(@"strong immutable property address:%p class:%@ property:%@",self.strongImArr,[self.strongImArr class],self.strongImArr);
    self.CopyImArr = marr;
    NSLog(@"copy immutable property address:%p class:%@ property:%@",self.CopyImArr,[self.CopyImArr class],self.CopyImArr);
 
// 改变可变数组并打印信息
    [marr addObject:@"2"];
    NSLog(@"copy mutable property address:%p class:%@ property:%@",self.cmArr,[self.cmArr class],self.cmArr);
    NSLog(@"strong mutable property address:%p class:%@ property:%@",self.smArr,[self.smArr class],self.smArr);
    NSLog(@"strong immutable property address:%p class:%@ property:%@",self.strongImArr,[self.strongImArr class],self.strongImArr);
    NSLog(@"copy immutable property address:%p class:%@ property:%@",self.CopyImArr,[self.CopyImArr class],self.CopyImArr);
```

打印结果

```
original arr address:0x600000017f50 class:__NSSingleObjectArrayI
copy arr address:0x600000017f50 class:__NSSingleObjectArrayI
mutable arr address:0x60c000250680 class:__NSArrayM

copy mutable property address:0x608000201de0 class:__NSSingleObjectArrayI property:(
    1
)
strong mutable property address:0x60c000250680 class:__NSArrayM property:(
    1
)
strong immutable property address:0x60c000250680 class:__NSArrayM property:(
    1
)
copy immutable property address:0x608000201dd0 class:__NSSingleObjectArrayI property:(
    1
)

// 改变局部变量后的打印结果
copy mutable property address:0x608000201de0 class:__NSSingleObjectArrayI property:(
    1
)
strong mutable property address:0x60c000250680 class:__NSArrayM property:(
    1,
    2
)
strong immutable property address:0x60c000250680 class:__NSArrayM property:(
    1,
    2
)
copy immutable property address:0x608000201dd0 class:__NSSingleObjectArrayI property:(
    1
)
```

可以看到，`copy`将原本是**可变数组的**的属性`self.cmArr`的类型变成了**__NSSingleObjectArrayI**不可变，同时对比内存地址可以发现，编译器开辟了一块新的内存空间用来存储copy的值。所以当我们如果写下类似`@property (nonatomic, copy) NSMutableArray *foo`时，需要考虑一下这样声明是否有意义。

同样有趣的是，我们本来声明为**不可变数组**的`self.strongImArr`却因为**多态**(父类指针指向子类)，指向了一个**可变数组**，并在`marr`改变时，同样也发生了变化。这与开发者声明它为**不可变**时的用意可能会出现出入，对比最后一行打印的结果，可能`copy`会更贴近开发者的真正想法。所以这就是很多文章提倡用`copy`修饰**NSString/NSArray/NSDictionary**属性的用意所在。

---

在认识**@property**后，接下来，我们会继续[iOS-探究@property]()，学习更多**@property**的底层知识

简书地址：[iOS-认识@property](http://www.jianshu.com/p/56cef77a55d8)


