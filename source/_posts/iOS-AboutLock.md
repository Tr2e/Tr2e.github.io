---
title: iOS-关于锁的总结
date: 2019-11-13 22:18:18
tags: "iOS"
categories: "iOS"
---

## 前言

对于iOS中各种锁的学习总结，供日后查阅

<!--more-->

## 引子

日常开发中，`@property (nonatomic, strong) *foo`是我们不厌其烦的使用频率最高的声明方式，也很清楚`atomic`和`nonatomic`属性的区别，这里再复习一下这两个关键字：

* `atomic`:原子性，这个属性是默认的，通过在`setter`、`getter`中加锁保证数据的读写安全
* `nonatomic`:非原子性，就是不加锁。优点是速度优于使用`atomic`，大多数场景不会出现问题

作为编译器标识符，`@property`的作用是帮助我们快速生成*成员变量及其getter/setter*方法，并通过属性关键字，帮助我们管理内存及安全等繁杂的事务，那么`atomic`是如何帮助我们保证成员变量的读写安全呢？下面我们看一段代码：

```
//@property(retain) UITextField *userName;
// 示例代码如下：
- (UITextField *) userName {
    UITextField *retval = nil;
    @synchronized(self) {
        retval = [userName retain];
        _userName = retval;
    }
    return retval;
}
- (void) setUserName:(UITextField *)userName {
    @synchronized(self) {
      [_userName release];
      _userName = [userName retain];
    }
}
```
我们可以很容易的看出，编译器是通过加锁，来保证当前成员变量`_userName`的读写安全，不至于生成脏数据，这便是`atomic`背后，编译器帮我们做的事情。事实上，如果深究下去编译器帮我们加了什么锁，其实并非`@synchronized(object)`

> *自旋锁*不会使线程状态发生切换，一直处于用户态，即线程一直都是active；不会使线程进入阻塞状态，减少了不必要的上下文切换，执行速度快
> *非自旋锁*在获取不到锁的时候会进入阻塞状态，从而进入内核态，当获取到锁的时候需要从内核态恢复，需要线程上下文切换,影响锁的性能

为什么`atomic`会做为默认属性，我们不难看出，苹果这么设计是想告诉我们，很多情况下，效率换安全是值得的

## 如何使用锁

下面一段简单代码，考虑一下输出结果

```
- (void)unlockTest {
    NSMutableString *string = [@"Mike" mutableCopy];
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        [string appendString:@"-Locked"];
        NSLog(@"%@",string);
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [string appendString:@"-JailBreaked"];
        NSLog(@"%@",string);
    });
}  
```
书写这样一段代码，是想在不同线程中在改变变量后，使用这个变量

控制台输出：
```
2019-11-11 16:52:43.019128+0800 DiscoverLock_iOS[89763:11225442] Mike-Locked-JailBreaked
2019-11-11 16:52:43.019128+0800 DiscoverLock_iOS[89763:11225441] Mike-Locked-JailBreaked
```
这显然不是想要的结果，如何保证我们在不同线程中使用的变量，都是我们希望的值呢？答案之一，就是加锁

### <NSLocking>
OC为我们提供了四种遵循<NSLocking>的类，分别是`NSLock`/`NSCondtionLock`/`NSRecursiveLock`/`NSCondition`，满足面向对象编程的需求
```
@protocol NSLocking

- (void)lock;// 阻塞线程，线程休眠
- (void)unlock;

@end
```
**加锁的基本流程： 【加锁】->【操作】->【解锁】**
以上提到的4个类，均可以实现这个基础功能，下文中不再赘述
```
- (void)lockedTest {
    NSMutableString *string = [@"Mike" mutableCopy];
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        // [锁 lock];
        [string appendString:@"-Locked"];
        NSLog(@"%@",string);
        // [锁 unlock];

    });
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // [锁 lock];
        [string appendString:@"-JailBreaked"];
        NSLog(@"%@",string);
        // [锁 unlock];
    });
}
```
控制台输出：
```
DiscoverLock_iOS[90562:11303793] Mike-Locked
DiscoverLock_iOS[90562:11303799] Mike-Locked-JailBreaked
```
```
DiscoverLock_iOS[90562:11303793] Mike-JailBreaked
DiscoverLock_iOS[90562:11303799] Mike-JailBreaked-Locked
```

这里的输出，结果不太一样，侧面说明了`DISPATCH_QUEUE_PRIORITY`并不能保证线程的执行顺序，如果要明确执行顺序，属于线程同步的范畴，本文不展开讨论，只会在**NSConditionLock**部分简单示例如何使用该类做到同步


#### NSLock
* `- (BOOL)tryLock;`:尝试加锁，如果失败返回NO，不会阻塞线程
* `- (BOOL)lockBeforeDate:(NSDate *)limit;`:指定时间前尝试加锁，如果失败返回NO，到时间前阻塞线程

示例代码：
```
- (void)lockTest {

    NSMutableString *string = [@"Mike" mutableCopy];
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
    LOCK(
         [string appendString:@"-Locked"];
         NSLog(@"%@",string);
         sleep(5);
        )
    });

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    TRYLOCK(
            [string appendString:@"-UnLock"];
            NSLog(@"%@",string);
            sleep(3);
        )
    });

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
    TRYLOCKINDURATION(2,
                      [string appendString:@"-Ending"];
                      NSLog(@"%@",string);
                      );
    NSLog(@"-=-=-=-=-");
    });
}
```
控制台输出：
```
2019-11-11 19:54:08.807763+0800 DiscoverLock_iOS[92986:11465678] Mike-Locked
2019-11-11 19:54:08.807763+0800 DiscoverLock_iOS[92986:11465679] TryLock-NO
2019-11-11 19:54:08.807889+0800 DiscoverLock_iOS[92986:11465679] Mike-Locked-UnLock
2019-11-11 19:54:10.810165+0800 DiscoverLock_iOS[92986:11465677] TryLockBefore-NO
2019-11-11 19:54:10.810523+0800 DiscoverLock_iOS[92986:11465677] Mike-Locked-UnLock-Ending
2019-11-11 19:54:10.810810+0800 DiscoverLock_iOS[92986:11465677] -=-=-=-=-
```
通过上面示例代码输出可以看到，`- (BOOL)tryLock;`并不会阻塞线程，在尝试加锁失败时，立即返回了**NO**,但是`- (BOOL)lockBeforeDate:(NSDate *)limit;`则在时间到之前阻塞了线程操作，在等待相应时间后，返回了**NO**，并执行了下一句打印，很明显是在等待期间阻塞了线程

上面代码中用到的几个宏定义，建议以后使用锁时，尽量保持头脑清醒或者干脆定义一些便利方法，保证【上锁】-【解锁】的成对出现，避免线程阻塞或死锁的情况
```
#define LOCK(...) \
[_lock lock]; \
__VA_ARGS__; \
[_lock unlock]; \

#define TRYLOCK(...) \
BOOL locked = [_lock tryLock]; \
NSLog(@"%@",locked?@"TryLock-YES":@"TryLock-NO"); \
__VA_ARGS__; \
if (locked) [_lock unlock]; \

#define TRYLOCKINDURATION(duration,...) \
BOOL locked = [_lock lockBeforeDate:[NSDate dateWithTimeIntervalSinceNow:duration]]; \
NSLog(@"%@",locked?@"TryLockBefore-YES":@"TryLockBefore-NO"); \
__VA_ARGS__; \
if (locked) [_lock unlock]; \
```

#### NSConditionLock
* `- (instancetype)initWithCondition:(NSInteger)condition NS_DESIGNATED_INITIALIZER;`:便利构造方法，传入条件锁的初始值
* `@property (readonly) NSInteger condition;`:当前条件锁的值
* `- (void)lockWhenCondition:(NSInteger)condition;`:当锁的条件值与传入值相等时，执行接下来的操作，否则阻塞线程
* `- (BOOL)tryLock;`:尝试加锁，如果失败返回NO，不会阻塞线程
* `- (BOOL)tryLockWhenCondition:(NSInteger)condition;`:尝试加锁，当锁的条件值与传入值相等，则加锁成功，否则失败返回NO，不会阻塞线程
* `- (void)unlockWithCondition:(NSInteger)condition;`:解锁操作，同时变更锁的条件值为传入值
* `- (BOOL)lockBeforeDate:(NSDate *)limit;`:指定时间前尝试加锁，如果失败返回NO，到时间前阻塞线程
* `- (BOOL)lockWhenCondition:(NSInteger)condition beforeDate:(NSDate *)limit;`:指定时间前尝试加锁，当锁的条件值与传入值相等，则加锁成功返回YES，否则失败返回NO，到时间前阻塞线程

`NSConditionLock`和`NSLock`方法类似，多了一个`condition`属性，以及每个操作都多了一个关于condition属性的方法，`- (void)lockWhenCondition:(NSInteger)condition;`只有condition参数与初始化时候的condition相等，lock才能正确进行加锁操作。而`- (void)unlockWithCondition:(NSInteger)condition;`并不是当条件值符合条件时才解锁，而是解锁之后,修改当前锁的条件值
假如不使用condition相关的方法，`NSConditionLock`同`NSLock`并无二致

上文中我们提到了线程同步问题，这里一起看一下下面这段代码
```
- (void)conditionLockUnordered {
    NSMutableString *conditionString = [[NSMutableString alloc] init];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [conditionString appendString:@"-1-"];
        NSLog(@">>> 1 %@ threadInfo:%@<<<",conditionString,[NSThread currentThread]);
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [conditionString appendString:@"-2-"];
        NSLog(@">>> 2 %@ threadInfo:%@<<<",conditionString,[NSThread currentThread]);
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [conditionString appendString:@"-3-"];
        NSLog(@">>> 3 %@ threadInfo:%@<<<",conditionString,[NSThread currentThread]);
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [conditionString appendString:@"-4-"];
        NSLog(@">>> 4 %@ threadInfo:%@<<<",conditionString,[NSThread currentThread]);
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [conditionString appendString:@"-5-"];
        NSLog(@">>> 5 %@ threadInfo:%@<<<",conditionString,[NSThread currentThread]);
    });
}
```
控制台输出：
```
2019-11-11 20:34:16.875479+0800 DiscoverLock_iOS[93895:11551560] >>> 2 -1--2--4--3- threadInfo:<NSThread: 0x600003905640>{number = 4, name = (null)}<<<
2019-11-11 20:34:16.875525+0800 DiscoverLock_iOS[93895:11551562] >>> 3 -1--2--4--3- threadInfo:<NSThread: 0x600003903680>{number = 6, name = (null)}<<<
2019-11-11 20:34:16.875530+0800 DiscoverLock_iOS[93895:11551561] >>> 1 -1--2- threadInfo:<NSThread: 0x600003908bc0>{number = 3, name = (null)}<<<
2019-11-11 20:34:16.875543+0800 DiscoverLock_iOS[93895:11551559] >>> 4 -1--2--4--3- threadInfo:<NSThread: 0x6000039175c0>{number = 5, name = (null)}<<<
2019-11-11 20:34:16.875628+0800 DiscoverLock_iOS[93895:11551560] >>> 5 -1--2--4--3--5- threadInfo:<NSThread: 0x600003905640>{number = 4, name = (null)}<<<
```
依然是混乱状态，上文中`NSLock`部分已经通过加锁，控制了读写的稳定性，那么如果我们想要按照标号依次执行，该如何操作？

熟悉`GCD`的小伙伴会说这还不简单，`dispatch_barrier`解千愁，当然这么写没问题，但是这里多说一嘴，`dispatch_barrier`只能针对同一个并发队列起作用，注意正确初始化的姿势`dispatch_queue_t thread = dispatch_queue_create("barrier", DISPATCH_QUEUE_CONCURRENT);`,而不是干啥都是一句`dispatch_get_global_queue(0,0)`,如果使用Global_Queue,这个barrier就同普通的`dispatch_async`没什么区别了

我们要是想在不同线程搞定顺序这个事儿，怎么办呢？这个时候`NSConditionLock`自带的条件方法，便能帮你实现这个功能，具体看下面的示例代码
```
- (void)conditionLockOrdered {
    // NSConditionLock
    NSInteger conditionTag = 0;
    _conditionLock = [[NSConditionLock alloc] initWithCondition:conditionTag];
    
    NSMutableString *conditionString = [[NSMutableString alloc] init];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        NSLog(@">>> handle 1 <<<");
        [_conditionLock lockWhenCondition:conditionTag];
        [conditionString appendString:@"-1-"];
        NSLog(@">>> 1 %@ threadInfo:%@<<<",conditionString,[NSThread currentThread]);
        [_conditionLock unlockWithCondition:1];
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        NSLog(@">>> handle 2 <<<");
        [_conditionLock lockWhenCondition:1];
        [conditionString appendString:@"-2-"];
        NSLog(@">>> 2 %@ threadInfo:%@<<<",conditionString,[NSThread currentThread]);
        [_conditionLock unlockWithCondition:2];
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        NSLog(@">>> handle 3 <<<");
        [_conditionLock lockWhenCondition:2];
        [conditionString appendString:@"-3-"];
        NSLog(@">>> 3 %@ threadInfo:%@<<<",conditionString,[NSThread currentThread]);
        [_conditionLock unlockWithCondition:3];
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSLog(@">>> handle 4 <<<");
        [_conditionLock lockWhenCondition:3];
        [conditionString appendString:@"-4-"];
        NSLog(@">>> 4 %@ threadInfo:%@<<<",conditionString,[NSThread currentThread]);
        [_conditionLock unlockWithCondition:4];
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSLog(@">>> handle 5 <<<");
        [_conditionLock lockWhenCondition:4];
        [conditionString appendString:@"-5-"];
        NSLog(@">>> 5 %@ threadInfo:%@<<<",conditionString,[NSThread currentThread]);
        [_conditionLock  unlock];
        NSLog(@"-=-=-=-=-=-=-");
    });
    NSLog(@"🍺");
}
```
控制台输出：
```
2019-11-11 20:53:58.237847+0800 DiscoverLock_iOS[94374:11586439] 🍺
2019-11-11 20:53:58.237862+0800 DiscoverLock_iOS[94374:11586488] >>> handle 1 <<<
2019-11-11 20:53:58.237877+0800 DiscoverLock_iOS[94374:11586489] >>> handle 3 <<<
2019-11-11 20:53:58.237868+0800 DiscoverLock_iOS[94374:11586490] >>> handle 2 <<<
2019-11-11 20:53:58.237887+0800 DiscoverLock_iOS[94374:11586491] >>> handle 4 <<<
2019-11-11 20:53:58.237892+0800 DiscoverLock_iOS[94374:11586495] >>> handle 5 <<<
2019-11-11 20:53:58.238111+0800 DiscoverLock_iOS[94374:11586488] >>> 1 -1- threadInfo:<NSThread: 0x6000014c3380>{number = 3, name = (null)}<<<
2019-11-11 20:53:58.238488+0800 DiscoverLock_iOS[94374:11586490] >>> 2 -1--2- threadInfo:<NSThread: 0x6000014dac40>{number = 4, name = (null)}<<<
2019-11-11 20:53:58.238605+0800 DiscoverLock_iOS[94374:11586489] >>> 3 -1--2--3- threadInfo:<NSThread: 0x6000014daf00>{number = 5, name = (null)}<<<
2019-11-11 20:53:58.239269+0800 DiscoverLock_iOS[94374:11586491] >>> 4 -1--2--3--4- threadInfo:<NSThread: 0x6000014c6740>{number = 6, name = (null)}<<<
2019-11-11 20:53:58.239410+0800 DiscoverLock_iOS[94374:11586495] >>> 5 -1--2--3--4--5- threadInfo:<NSThread: 0x6000014c3480>{number = 7, name = (null)}<<<
2019-11-11 20:53:58.239552+0800 DiscoverLock_iOS[94374:11586495] -=-=-=-=-=-=-
```
可以看到，不同的线程，虽然被调度的时机不同，但是因为`NSConditionLock`的存在，后续对数据具体的操作，我们预想的顺序得到了保证。这种用法笔者并认为在任务耗时较少的情况下没有明显问题的，但是假如存在长时间的耗时操作，还是建议使用`dispatch_barrier`，因为这样不会占用过多资源

#### NSRecursiveLock
* `- (BOOL)tryLock;`:尝试加锁，如果失败返回NO，不会阻塞线程
* `- (BOOL)lockBeforeDate:(NSDate *)limit;`:指定时间前尝试加锁，如果失败返回NO，到时间前阻塞线程
Api同`NSLock`完全一样，区别在于`NSRecursiveLock（递归锁）`可以在同一线程中重复加锁而不死锁，它会记录【上锁】和【解锁】的次数，当这两个值平衡时，才会释放锁，其他线程才可以上锁成功

先看下一段代码，会存在什么问题：
```
@property (nonatomic, assign) NSInteger recursiveNum;// 5
- (void)test_unrecursiveLock {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self recursiveTest];
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        self.recursiveNum = 7;
        NSLog(@">>> changed %ld <<<",self.recursiveNum);
    });
}

- (void)recursiveTest {
    if (self.recursiveNum > 0) {
        self.recursiveNum -= 1;
        NSLog(@">>> %ld <<<",self.recursiveNum);
        [self recursiveTest];
    }
}
```
控制台输出：
```
2019-11-11 21:27:13.451703+0800 DiscoverLock_iOS[95105:11645279] >>> 4 <<<
2019-11-11 21:27:13.451709+0800 DiscoverLock_iOS[95105:11645277] >>> changed 7 <<<
2019-11-11 21:27:13.451812+0800 DiscoverLock_iOS[95105:11645279] >>> 6 <<<
2019-11-11 21:27:13.451883+0800 DiscoverLock_iOS[95105:11645279] >>> 5 <<<
2019-11-11 21:27:13.451940+0800 DiscoverLock_iOS[95105:11645279] >>> 4 <<<
2019-11-11 21:27:13.452004+0800 DiscoverLock_iOS[95105:11645279] >>> 3 <<<
2019-11-11 21:27:13.452068+0800 DiscoverLock_iOS[95105:11645279] >>> 2 <<<
2019-11-11 21:27:13.452130+0800 DiscoverLock_iOS[95105:11645279] >>> 1 <<<
2019-11-11 21:27:13.452241+0800 DiscoverLock_iOS[95105:11645279] >>> 0 <<<
```
同时存在两个线程，对已知的recursiveNum的值进行写操作，其中一个线程使用递归调用，对该值进行了操作，但是同时另一个线程改变了这个值，在不加锁的情况下，这种操作问题很多，如果递归中含有重要的逻辑处理，竞态可能导致整个逻辑执行完的结果大概率是错误的。

如何规避这种竞态导致的不必要的错误，首先我们想到的是加锁，但是如果递归加锁的话，线程会重复加锁，导致死锁。所以这时候必须使用**递归锁**来解决这个问题
```
- (void)test_unrecursiveLock {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self recursiveTest];
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        [_recursiveLock lock];// 递归锁
        self.recursiveNum = 7;
        NSLog(@">>> changed %ld <<<",self.recursiveNum);
        [_recursiveLock unlock];// 解锁
    });
}
- (void)recursiveTest {
    [_recursiveLock lock];// 递归锁
    if (self.recursiveNum > 0) {
        self.recursiveNum -= 1;
        NSLog(@">>> %ld <<<",self.recursiveNum);
        [self recursiveTest];
    }
    [_recursiveLock unlock];// 解锁
}
```
控制台输出：
```
2019-11-11 21:34:44.422337+0800 DiscoverLock_iOS[95341:11655990] >>> 4 <<<
2019-11-11 21:34:44.422442+0800 DiscoverLock_iOS[95341:11655990] >>> 3 <<<
2019-11-11 21:34:44.422511+0800 DiscoverLock_iOS[95341:11655990] >>> 2 <<<
2019-11-11 21:34:44.422583+0800 DiscoverLock_iOS[95341:11655990] >>> 1 <<<
2019-11-11 21:34:44.422645+0800 DiscoverLock_iOS[95341:11655990] >>> 0 <<<
2019-11-11 21:34:44.422747+0800 DiscoverLock_iOS[95341:11655992] >>> changed 7 <<<

------

2019-11-11 21:37:11.238448+0800 DiscoverLock_iOS[95396:11662426] >>> changed 7 <<<
2019-11-11 21:37:11.238635+0800 DiscoverLock_iOS[95396:11662423] >>> 6 <<<
2019-11-11 21:37:11.238793+0800 DiscoverLock_iOS[95396:11662423] >>> 5 <<<
2019-11-11 21:37:11.238930+0800 DiscoverLock_iOS[95396:11662423] >>> 4 <<<
2019-11-11 21:37:11.239093+0800 DiscoverLock_iOS[95396:11662423] >>> 3 <<<
2019-11-11 21:37:11.239293+0800 DiscoverLock_iOS[95396:11662423] >>> 2 <<<
2019-11-11 21:37:11.239844+0800 DiscoverLock_iOS[95396:11662423] >>> 1 <<<
2019-11-11 21:37:11.239976+0800 DiscoverLock_iOS[95396:11662423] >>> 0 <<<
```
虽然存在两种输出结果，但是我们的递归操作的逻辑，可以完全不受干扰，如果需要控制顺序，（敲黑板）要怎么做呢？

#### NSCondition
* `- (void)wait;`:当前线程立即进入休眠状态
* `- (BOOL)waitUntilDate:(NSDate *)limit;`:当前线程立即进入休眠状态，limit时间后唤醒
* `- (void)signal;`:唤醒wait后进入休眠的单条线程
* `- (void)broadcast;`:唤醒wait后进入休眠的所有线程，调度

有些情况需要协调线程之间的执行。例如，一个线程可能需要等待其他线程返回结果，这个时候`NSCondition`可能是个好选择

为了能体现NSCondition的作用，我们举一个可能并不是很恰当的**生产者-消费者**的例子：
**我们现在有一条柔性生产线，限定每个批次只能生产3件商品，耗时6s，同时开放网络购买平台让大家抢购拼团，订单式销售，三人成团，现在有三位天选之子*Tom/Mike/Lily*从全球千万人中脱颖而出，成功成团。为了增强可玩性，活动是从开启的一刻起，同时开始生产和抢购，3件库存销售完成后，再次进行同时进行生产和抢购活动**

代码示例如下：

```
@interface Producer : NSObject
@property (nonatomic, assign) BOOL shouldProduce;
@property (nonatomic, strong) NSString *itemName;
@property (nonatomic, strong) NSCondition *condition;
@property (nonatomic, strong) NSMutableArray *collector;

- (instancetype)initWithConditon:(NSCondition *)condition collector:(NSMutableArray *)collector;
- (void)produce;
@end

@implementation Producer

- (instancetype)initWithConditon:(NSCondition *)condition collector:(NSMutableArray *)collector{
    
    self = [super init];
    if (self) {
        self.condition = condition;
        self.collector = collector;
        self.shouldProduce = NO;
        self.itemName = nil;
    }
    return self;
}

-(void)produce{
    self.shouldProduce = YES;
    while (self.shouldProduce) {
        NSLog(@"准备生产");
        [self.condition lock];
        NSLog(@"- p lock -");
        if (self.collector.count > 0 ) {
            NSLog(@"- p - wait");
            [self.condition wait];
        }
        NSLog(@"开始生产");
        [self.collector addObject:@"商品1"];
        [self.collector addObject:@"商品2"];
        [self.collector addObject:@"商品3"];
        NSLog(@"生产:商品1/商品2/商品3");
        sleep(6);
        NSLog(@"生产结束");
        [self.condition broadcast];
        NSLog(@"- p signal -");
        [self.condition unlock];
        NSLog(@"- p unlock -");
    }
    NSLog(@"-结束生产-");
}

@end

@interface Consumer : NSObject
@property (nonatomic, assign) BOOL shouldConsumer;
@property (nonatomic, strong) NSCondition *condition;
@property (nonatomic, strong) NSMutableArray *collector;
@property (nonatomic,   copy) NSString *itemName;
- (instancetype)initWithConditon:(NSCondition *)condition collector:(NSMutableArray *)collector name:(NSString *)name;
- (void)consumer;
@end

@implementation Consumer
- (instancetype)initWithConditon:(NSCondition *)condition collector:(NSMutableArray *)collector name:(NSString *)name{
    self = [super init];
    if (self) {
        self.condition = condition;
        self.collector = collector;
        self.shouldConsumer = NO;
        self.itemName = name;
    }
    return self;
}

-(void)consumer{
    self.shouldConsumer = YES;
    while (self.shouldConsumer) {
        NSLog(@"%@-准备购买",self.itemName);
        [self.condition lock];
        NSLog(@"- c:%@ lock -",self.itemName);
        if (self.collector.count == 0 ) {
            NSLog(@"- c:%@ wait -",self.itemName);
            [self.condition wait];
        }
        NSString *item = [self.collector objectAtIndex:0];
        NSLog(@"%@-买入:%@",self.itemName,item);
        [self.collector removeObjectAtIndex:0];
        sleep(2);
        [self.condition signal];
        NSLog(@"- c:%@ signal -",self.itemName);
        [self.condition unlock];
        NSLog(@"- c:%@ unlock -",self.itemName);
    }
    NSLog(@"-%@结束购买-",self.itemName);
}
@end

// 调用
{
    NSMutableArray *pipeline = [NSMutableArray array];
    NSCondition *condition = [NSCondition new];
    
    Producer *p = [[Producer alloc] initWithConditon:condition collector:pipeline];
    Consumer *c = [[Consumer alloc] initWithConditon:condition collector:pipeline name:@"Tom"];
    Consumer *c1 = [[Consumer alloc] initWithConditon:condition collector:pipeline name:@"Mike"];
    Consumer *c2 = [[Consumer alloc] initWithConditon:condition collector:pipeline name:@"Lily"];
    [[[NSThread alloc] initWithTarget:c selector:@selector(consumer) object:c] start];
    [[[NSThread alloc] initWithTarget:c1 selector:@selector(consumer) object:c] start];
    [[[NSThread alloc] initWithTarget:c2 selector:@selector(consumer) object:c] start];
    [[[NSThread alloc] initWithTarget:p selector:@selector(produce) object:p] start];
    

    sleep(15);
    NSLog(@"<----------------->");
    p.shouldProduce = NO;
    c.shouldConsumer = NO;
    c1.shouldConsumer = NO;
    c2.shouldConsumer = NO;
}
```
部分控制台输出：
```
2019-11-12 17:04:03.662926+0800 DiscoverLock_iOS[7110:12246052] Mike-准备购买
2019-11-12 17:04:03.662916+0800 DiscoverLock_iOS[7110:12246051] Tom-准备购买
2019-11-12 17:04:03.662990+0800 DiscoverLock_iOS[7110:12246053] Lily-准备购买
2019-11-12 17:04:03.663005+0800 DiscoverLock_iOS[7110:12246054] 准备生产
2019-11-12 17:04:03.663083+0800 DiscoverLock_iOS[7110:12246053] - c:Lily lock -
2019-11-12 17:04:03.663144+0800 DiscoverLock_iOS[7110:12246053] - c:Lily wait -
2019-11-12 17:04:03.663254+0800 DiscoverLock_iOS[7110:12246052] - c:Mike lock -
2019-11-12 17:04:03.663439+0800 DiscoverLock_iOS[7110:12246052] - c:Mike wait -
2019-11-12 17:04:03.663805+0800 DiscoverLock_iOS[7110:12246051] - c:Tom lock -
2019-11-12 17:04:03.663903+0800 DiscoverLock_iOS[7110:12246051] - c:Tom wait -
2019-11-12 17:04:03.664126+0800 DiscoverLock_iOS[7110:12246054] - p lock -
2019-11-12 17:04:03.664297+0800 DiscoverLock_iOS[7110:12246054] 开始生产
2019-11-12 17:04:03.664433+0800 DiscoverLock_iOS[7110:12246054] 生产:商品1/商品2/商品3
2019-11-12 17:04:09.669735+0800 DiscoverLock_iOS[7110:12246054] 生产结束
```
基于多线程并发的工作原理，通过上面的部分打印结果，也很容易得到这个结论。**由于不符合购买条件**，*Lily*/*Mike*/*Tom*都只能选择`wait`，这个时候，生产者获取到锁并执行生产代码，在生产完成后，`broadcast`或者`signal`告诉其他线程，可以唤醒线程并继续执行消费者相关代码。
`NSCondition`相较于`NSConditionLock`的不同点在于他依赖的是外部值，能够满足更多复杂需求场景。
假如将上述代码中生产者的`broadcast`替换成`signal`后发现，在当前这种特定场景下，这两个方法的作用似乎并没有什么区别。而且感兴趣的同学，可以使用上述代码多运行几次，看看是否能够得出同笔者一样的猜测：
1. NSCondition会自身通过队列管理协同任务的调度
2. wait的任务依次入等待队列
3. 未wait的任务根据获得锁的顺序依次入执行队列
4. wait任务的等待队列会在执行队列执行完后依次执行并入执行队列
4. 第一次调度顺序确定后，后续任务的执行，按照执行队列缓存依次出列执行
这里仅做猜想，具体实现可能并非如此，待大佬指点迷津或有机会鶸笔者自行研究

### OSSpinLock
看了`NSCondition`这么个复杂的东西，我们看点轻松的，`OSSpinLock`是苹果在**iOS10**之前提供的自旋锁方案，但是存在优先级反转的问题，被苹果废弃，以前源码中使用`OSSpinLock`的地方，都被苹果替换成了`pthread_mutex`

![被废弃的OSSpinLock.png](https://upload-images.jianshu.io/upload_images/1742463-2acd56eaea585c78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![官方备注.png](https://upload-images.jianshu.io/upload_images/1742463-ae8eb41b371e5c62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### os_unfair_lock
`os_unfair_lock`是**iOS10**以后新增的低级别加锁方案，本质是**互斥锁**，这里需要注意，目前很多文章认为他是作为替代`OSSpinLock`的方案就是自旋锁是有问题的
* `void os_unfair_lock_lock(os_unfair_lock_t lock);`:加锁
* `bool os_unfair_lock_trylock(os_unfair_lock_t lock);`:尝试加锁，成功返回true，失败返回false
* `void os_unfair_lock_unlock(os_unfair_lock_t lock);`:解锁
* `void os_unfair_lock_assert_owner(os_unfair_lock_t lock);`:如果当前线程未持有指定的锁，则触发断言
* `void os_unfair_lock_assert_not_owner(os_unfair_lock_t lock);`:如果当前线程持有指定的锁，则触发断言

各方法同常见的锁没太大差别，可以看下方法注释，只是需要注意一下初始化方式
```
{
    os_unfair_lock_t unfairLock;
    unfairLock = &(OS_UNFAIR_LOCK_INIT);
}
```

### @synchronize(object)
`@synchronized(object)`指令使用传入的对象作为该锁的唯一标识，只有当标识相同时，才满足互斥
`@synchronized(object)`指令实现锁的优点就是我们不需要在代码中显式的创建锁对象，便可以实现锁的机制，而且不用担心忘记解锁
使用方法极其常见，不做示例了

### dispatch_semaphore
* `dispatch_semaphore_t dispatch_semaphore_create(long value);`:创建信号量，传入初始值
* `long dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout);`:当信号<=0时，根据传入的时间阻塞线程；如果信号>0则不阻塞线程，并对信号-1处理
* `long dispatch_semaphore_signal(dispatch_semaphore_t dsema);`:对信号+1处理

`GCD`为我们提供的**信号量**也是常用的加锁方式，常见用法是初始化信号值为1

```
{
    dispatch_semaphore_t lock = dispatch_semaphore_create(1);

    dispatch_semaphore_wait(lock,DISPATCH_TIME_FOREVER);
    // 操作
    dispatch_semaphare_signal(lock);
}
```

常规操作大家都知道，有常规操作，那么一定也有非常规操作，可以看一下`AFNetwork`给我们的示范

```
- (NSArray *)tasksForKeyPath:(NSString *)keyPath {
    __block NSArray *tasks = nil;
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        if ([keyPath isEqualToString:NSStringFromSelector(@selector(dataTasks))]) {
            tasks = dataTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(uploadTasks))]) {
            tasks = uploadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(downloadTasks))]) {
            tasks = downloadTasks;
        } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(tasks))]) {
            tasks = [@[dataTasks, uploadTasks, downloadTasks] valueForKeyPath:@"@unionOfArrays.self"];
        }

        dispatch_semaphore_signal(semaphore);
    }];

    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

    return tasks;
}
```

在`AFURLSessionManager`中，初始化使用`dispatch_semaphore_create(0)`，在`return tasks;`前调用`dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);`阻塞线程，待block将目标值赋值后，执行`dispatch_semaphore_signal(semaphore);`,此时tasks已经有值，线程被唤醒后正常返回。很秀

### pthread_mutex
C语言下的互斥锁方案，是<NSLocking>协议下四个类的底层

锁常用函数：
* `pthread_mutex_init`:动态初始化互斥量
* `PTHREAD_MUTEX_INITIALIZER`:静态创建互斥量
* `pthread_mutex_lock`:给一个互斥量加锁
* `pthread_mutex_trylock`:加锁，如果失败不阻塞
* `pthread_mutex_unlock`:解锁
* `pthread_mutex_destroy`:销毁锁

参数配置函数：
* `pthread_mutexattr_init`:初始化参数
* `pthread_mutexattr_settype`:设置类型
* `pthread_mutexattr_setpshared`:设置作用域
* `pthread_mutexattr_destroy`:销毁参数

条件常见函数：
* `pthread_cond_init`:动态初始化条件量
* `PTHREAD_COND_INITIALIZER`:静态创建条件量
* `pthread_cond_wait`:传入条件量及锁
* `pthread_cond_signal`:唤醒单条线程并加锁
* `pthread_cond_broadcast`:广播唤醒所有线程
* `pthread_cond_destroy`:销毁条件


**以上函数都是有返回值的，需要注意的是，若成功则返回0，否则返回错误编号，不是我们习惯中的成功YES失败NO**

锁类型：
* `PTHREAD_MUTEX_NORMAL`:缺省值，这种类型的互斥锁不会自动检测死锁。如果一个线程试图对一个互斥锁重复锁定，将会引起这个线程的死锁。如果试图解锁一个由别的线程锁定的互斥锁会引发不可预料的结果。如果一个线程试图解锁已经被解锁的互斥锁也会引发不可预料的结果
* `PTHREAD_MUTEX_ERRORCHECK`:这种类型的互斥锁会自动检测死锁。如果一个线程试图对一个互斥锁重复锁定，将会返回一个错误代码。如果试图解锁一个由别的线程锁定的互斥锁将会返回一个错误代码。如果一个线程试图解锁已经被解锁的互斥锁也将会返回一个错误代码
* `PTHREAD_MUTEX_RECURSIVE`:如果一个线程对这种类型的互斥锁重复上锁，不会引起死锁，一个线程对这类互斥锁的多次重复上锁必须由这个线程来重复相同数量的解锁，这样才能解开这个互斥锁，别的线程才能得到这个互斥锁。如果试图解锁一个由别的线程锁定的互斥锁将会返回一个错误代码。如果一个线程试图解锁已经被解锁的互斥锁也将会返回一个错误代码。这种类型的互斥锁只能是进程私有的（作用域属性PTHREAD_PROCESS_PRIVATE）
* `PTHREAD_MUTEX_DEFAULT`:就是NORMAL类型

锁作用域：
* `PTHREAD_PROCESS_PRIVATE`:缺省值，作用域为进程内
* `PTHREAD_PROCESS_SHARED`:作用域为进程间

使用示例：
```
static pthread_mutex_t c_lock;
- (void)testPthread_mutex {
    pthread_mutexattr_t c_lockAttr;
    pthread_mutexattr_init(&c_lockAttr);
    pthread_mutexattr_settype(&c_lockAttr, PTHREAD_MUTEX_RECURSIVE);
    pthread_mutexattr_setpshared(&c_lockAttr, PTHREAD_PROCESS_PRIVATE);
    
    pthread_mutex_init(&c_lock, &c_lockAttr);
    pthread_mutexattr_destroy(&c_lockAttr);

    pthread_t thread1;
    pthread_create(&thread1, NULL, _thread1, NULL);
    
    pthread_t thread2;
    pthread_create(&thread2, NULL, _thread2, NULL);
}

void *_thread1() {
    pthread_mutex_lock(&c_lock);
    printf("thread 1\n");
    pthread_mutex_unlock(&c_lock);
    return 0;
}

void *_thread2() {
    pthread_mutex_lock(&c_lock);
    printf("thread 2 busy\n");
    sleep(3);
    printf("thread 2\n");
    pthread_mutex_unlock(&c_lock);
    return 0;
}
```

## 使用锁的注意点
1. 互斥量需要时间来加锁和解锁。锁住较少互斥量的程序通常运行得更快。所以，互斥量应该尽量少，够用即可，每个互斥量保护的区域应则尽量大。

2. 互斥量的本质是串行执行。如果很多线程需要领繁地加锁同一个互斥量，
则线程的大部分时间就会在等待，这对性能是有害的。如果互斥量保护的数据(或代码)包含彼此无关的片段，则可以特大的互斥量分解为几个小的互斥量来提高性能。这样，任意时刻需要小互斥量的线程减少，线程等待时间就会减少。所以，互斥量应该足够多(到有意义的地步)，每个互斥量保护的区域则应尽量的少。

## 参考文档
> [Posix互斥量pthread_mutex_t](https://www.xuebuyuan.com/3121962.html)
> [iOS 常见知识点（三）：Lock](https://www.jianshu.com/p/ddbe44064ca4)
> [不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)
> [How does @synchronized lock/unlock in Objective-C?](https://stackoverflow.com/questions/1215330/how-does-synchronized-lock-unlock-in-objective-c/6047218#6047218)
> [[爆栈热门 iOS 问题] atomic 和 nonatomic 有什么区别？](https://www.jianshu.com/p/7288eacbb1a2)
> [《高性能iOS应用开发中文版》]()
