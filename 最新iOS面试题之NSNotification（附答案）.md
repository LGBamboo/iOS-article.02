### 返回目录:[全网各大厂iOS面试题-题集大全](https://github.com/LGBamboo/iOS-Advanced)

# 最新iOS面试题之NSNotification（附答案）

### 主要内容包含如下:

*   实现原理（结构设计、通知如何存储的、name&observer&SEL之间的关系等）
*   通知的发送时同步的，还是异步的
*   NSNotificationCenter接受消息和发送消息是在一个线程里吗？如何异步发送消息
*   NSNotificationQueue是异步还是同步发送？在哪个线程响应
*   NSNotificationQueue和runloop的关系
*   如何保证通知接收的线程在主线程
*   页面销毁时不移除通知会崩溃吗
*   多次添加同一个通知会是什么结果？多次移除通知呢
*   下面的方式能接收到通知吗？为什么

```
// 发送通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:@"TestNotification" object:@1];
// 接收通知
[NSNotificationCenter.defaultCenter postNotificationName:@"TestNotification" object:nil];

```

在解释这些内容之前 强烈建议认真研读一下这篇 [一文全解iOS通知机制(经典收藏)](https://www.jianshu.com/p/5952c0a3fc62)文章 了解一下大概 所有的问题就迎刃而解了.

## 实现原理（结构设计、通知如何存储的、name&observer&SEL之间的关系等

首先通知中心结构大概分为如下几个类

*   `NSNotification` 通知的模型 name、object、userinfo.
*   `NSNotificationCenter`通知中心 负责发送`NSNotification`
*   `NSNotificationQueue`通知队列 负责在某些时机触发 调用`NSNotificationCenter`通知中心 `post`通知

通知是结构体通过双向链表进行数据存储

```
// 根容器，NSNotificationCenter持有
typedef struct NCTbl {
  Observation		*wildcard;	/* 链表结构，保存既没有name也没有object的通知 */
  GSIMapTable		nameless;	/* 存储没有name但是有object的通知	*/
  GSIMapTable		named;		/* 存储带有name的通知，不管有没有object	*/
    ...
} NCTable;

// Observation 存储观察者和响应结构体，基本的存储单元
typedef	struct	Obs {
  id		observer;	/* 观察者，接收通知的对象	*/
  SEL		selector;	/* 响应方法		*/
  struct Obs	*next;		/* Next item in linked list.	*/
  ...
} Observation;
```

主要是以`key` `value`的形式存储,这里需要重点强调一下 通知以 `name`和`object`两个纬度来存储相关通知内容,也就是我们添加通知的时候传入的两个不同的方法.

[![](https://upload-images.jianshu.io/upload_images/13277235-d1cdd2ef99a5c864.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200901iOSinterviewAnswers/NCTable.jpg) 
[![](https://upload-images.jianshu.io/upload_images/13277235-b25d70e69f5cb196.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200901iOSinterviewAnswers/NCTable2.jpg) 

简单理解`name`&`observer`&`SEL`之间的关系就是`name`作为`key`, `observer`作为观察者对象,当合适时机触发就会调用`observer`的`SEL`.这基本很简单,如果觉得我说的不准确可以看下文章开头的文章.

## 通知的发送时同步的，还是异步的

同步发送.因为要调用消息转发.所谓异步，指的是**非实时发送**而是**在合适的时机发送**，并没有开启异步线程.

## NSNotificationCenter接受消息和发送消息是在一个线程里吗？如何异步发送消息

是的, 异步线程发送通知则响应函数也是在异步线程.

异步发送通知可以开启异步线程发送即可.

## NSNotificationQueue是异步还是同步发送？在哪个线程响应

```
// 表示通知的发送时机
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1, // runloop空闲时发送通知
    NSPostASAP = 2, // 尽快发送，这种时机是穿插在每次事件完成期间来做的
    NSPostNow = 3 // 立刻发送或者合并通知完成之后发送
};
```

|  | NSPostWhenIdle | NSPostASAP | NSPostNow |
| --- | --- | --- | --- |
| NSPostingStyle | 异步发送 | 异步发送 | 同步发送 |

`NSNotificationCenter`都是同步发送的，而这里介绍关于`NSNotificationQueue`的异步发送，从线程的角度看并不是真正的异步发送，或可称为**延时发送**，它是利用了`runloop`的时机来触发的.

异步线程发送通知则响应函数也是在异步线程,主线程发送则在主线程.

## NSNotificationQueue和runloop的关系

`NSNotificationQueue`依赖`runloop`. 因为通知队列要在runloop回调的某个时机调用通知中心发送通知.从下面的枚举值就能看出来

```
// 表示通知的发送时机
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1, // runloop空闲时发送通知
    NSPostASAP = 2, // 尽快发送，这种时机是穿插在每次事件完成期间来做的
    NSPostNow = 3 // 立刻发送或者合并通知完成之后发送
};
```

## 如何保证通知接收的线程在主线程

如果想在主线程响应异步通知的话可以用如下两种方式

1.系统接受通知的API指定队列

```
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block
```

2.`NSMachPort`的方式 通过在主线程的runloop中添加machPort，设置这个port的delegate，通过这个Port其他线程可以跟主线程通信，在这个port的代理回调中执行的代码肯定在主线程中运行，所以，在这里调用NSNotificationCenter发送通知即可

## 页面销毁时不移除通知会崩溃吗?

iOS9.0之前，会crash，原因：通知中心对观察者的引用是unsafe_unretained，导致当观察者释放的时候，观察者的指针值并不为nil，出现野指针.

iOS9.0之后，不会crash，原因：通知中心对观察者的引用是weak。

## 多次添加同一个通知会是什么结果？多次移除通知呢

多次添加同一个通知，会导致发送一次这个通知的时候，响应多次通知回调。 多次移除通知不会产生crash。

## 下面的方式能接收到通知吗？为什么

```
// 发送通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:@"TestNotification" object:@1];
// 接收通知
[NSNotificationCenter.defaultCenter postNotificationName:@"TestNotification" object:nil];
```

不能

首先我们看下通知中心存储通知观察者的结构

```
// 根容器，NSNotificationCenter持有
typedef struct NCTbl {
  Observation  *wildcard;    /* 链表结构，保存既没有name也没有object的通知 */
  GSIMapTable nameless;    /* 存储没有name但是有object的通知    */
  GSIMapTable named;        /* 存储带有name的通知，不管有没有object    */
    ...
} NCTable;

// Observation 存储观察者和响应结构体，基本的存储单元
typedef	struct Obs {
  id observer;    /* 观察者，接收通知的对象    */
  SEL selector;    /* 响应方法        */
  struct Obs *next;        /* Next item in linked list.    */
  ...
} Observation;
```

`nameless`与`named`的具体数据结构如下:

[![](https://upload-images.jianshu.io/upload_images/13277235-439098374929aec2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200901iOSinterviewAnswers/NCTable.jpg) 
[![](https://upload-images.jianshu.io/upload_images/13277235-03d732d55b6025d4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200901iOSinterviewAnswers/NCTable2.jpg) 

当添加通知监听的时候，我们传入了`name`和`object`，所以，观察者的存储链表是这样的：

`named`表：`key(name)` : `value`->`key(object)` : `value(Observation)`

因此在发送通知的时候，如果只传入`name`而并没有传入`object`，是找不到`Observation`的，也就不能执行观察者回调.

# 总结

今天又重新认识了iOS中的通知中心,希望大家经常温故而知新. 

### 返回目录:[全网各大厂iOS面试题-题集大全](https://github.com/LGBamboo/iOS-Advanced)

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
