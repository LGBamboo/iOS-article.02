# 最新iOS面试题之iOS通知机制全面解析

# 简述

本文主要是针对`iOS通知机制`的全面解析，从接口到原理面面俱到。同时也解决了`阿里、字节：一套高效的iOS面试题`中关于通知的问题，相信看完此文再也不怕面试官问我任何通知相关问题了

由于苹果没有对相关源码开放，所以以[GNUStep](https://github.com/gnustep/libs-base)源码为基础进行研究，[GNUStep](https://github.com/gnustep/libs-base)虽然不是苹果官方的源码，但很具有参考意义，根据实现原理来猜测和实践，更重要的还可以学习观察者模式的架构设计

# 问题列表

先把之前的问题列出来，详细读完本文之后，你会找到答案

1.  实现原理（结构设计、通知如何存储的、`name&observer&SEL`之间的关系等）
2.  通知的发送时同步的，还是异步的
3.  `NSNotificationCenter`接受消息和发送消息是在一个线程里吗？如何异步发送消息
4.  `NSNotificationQueue`是异步还是同步发送？在哪个线程响应
5.  `NSNotificationQueue`和`runloop`的关系
6.  如何保证通知接收的线程在主线程
7.  页面销毁时不移除通知会崩溃吗
8.  多次添加同一个通知会是什么结果？多次移除通知呢
9.  下面的方式能接收到通知吗？为什么

```
// 发送通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:@"TestNotification" object:@1];
// 接收通知
[NSNotificationCenter.defaultCenter postNotificationName:@"TestNotification" object:nil];
复制代码
```

# 关键类结构

## NSNotification

用于描述通知的类，一个`NSNotification`对象就包含了一条通知的信息，所以当创建一个通知时通常包含如下属性：

```
@interface NSNotification : NSObject <NSCopying, NSCoding>
...
/* Querying a Notification Object */

- (NSString*) name; // 通知的name
- (id) object; // 携带的对象
- (NSDictionary*) userInfo; // 配置信息

@end
复制代码
```

一般用于发送通知时使用，常用api如下：

```
- (void)postNotification:(NSNotification *)notification;
复制代码
```

## NSNotificationCenter

这是个单例类，负责管理通知的创建和发送，属于最核心的类了。而`NSNotificationCenter`类主要负责三件事

1.  添加通知
2.  发送通知
3.  移除通知

核心`API`如下：

```
// 添加通知
- (void)addObserver:(id)observer selector:(SEL)aSelector name:(nullable NSNotificationName)aName object:(nullable id)anObject;
// 发送通知
- (void)postNotification:(NSNotification *)notification;
- (void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject;
- (void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject userInfo:(nullable NSDictionary *)aUserInfo;
// 删除通知
- (void)removeObserver:(id)observer;

复制代码
```

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
***

## NSNotificationQueue

### 功能介绍

通知队列，用于异步发送消息，这个异步并不是开启线程，而是把通知存到双向链表实现的队列里面，等待某个时机触发时调用`NSNotificationCenter`的发送接口进行发送通知，这么看`NSNotificationQueue`最终还是调用`NSNotificationCenter`进行消息的分发

另外`NSNotificationQueue`是依赖`runloop`的，所以如果线程的`runloop`未开启则无效，至于为什么依赖`runloop`下面会解释

`NSNotificationQueue`主要做了两件事：

1.  添加通知到队列
2.  删除通知

核心`API`如下：

```
// 把通知添加到队列中，NSPostingStyle是个枚举，下面会介绍
- (void)enqueueNotification:(NSNotification *)notification postingStyle:(NSPostingStyle)postingStyle;
// 删除通知，把满足合并条件的通知从队列中删除
- (void)dequeueNotificationsMatching:(NSNotification *)notification coalesceMask:(NSUInteger)coalesceMask;

复制代码
```

### 队列的合并策略和发送时机

把通知添加到队列等待发送，同时提供了一些附加条件供开发者选择，如：什么时候发送通知、如何合并通知等，系统给了如下定义

```
// 表示通知的发送时机
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1, // runloop空闲时发送通知
    NSPostASAP = 2, // 尽快发送，这种情况稍微复杂，这种时机是穿插在每次事件完成期间来做的
    NSPostNow = 3 // 立刻发送或者合并通知完成之后发送
};
// 通知合并的策略，有些时候同名通知只想存在一个，这时候就可以用到它了
typedef NS_OPTIONS(NSUInteger, NSNotificationCoalescing) {
    NSNotificationNoCoalescing = 0, // 默认不合并
    NSNotificationCoalescingOnName = 1, // 只要name相同，就认为是相同通知
    NSNotificationCoalescingOnSender = 2  // object相同
};
复制代码
```

## GSNotificationObserver

这个类是[GNUStep](https://github.com/gnustep/libs-base)源码中定义的，它的作用是代理观察者，主要用来实现接口：`addObserverForName：object: queue: usingBlock:`时用到，即要实现在指定队列回调block，那么`GSNotificationObserver`对象保存了`queue`和`block`信息，并且作为观察者注册到通知中心，等到接收通知时触发了响应方法，并在响应方法中把`block`抛到指定`queue`中执行，定义如下：

```
@implementation GSNotificationObserver
{
	NSOperationQueue *_queue; // 保存传入的队列
	GSNotificationBlock _block; // 保存传入的block
}
- (id) initWithQueue: (NSOperationQueue *)queue 
               block: (GSNotificationBlock)block
{
......初始化操作
}

- (void) dealloc
{
....
}
// 响应接收通知的方法，并在指定队列中执行block
- (void) didReceiveNotification: (NSNotification *)notif
{
	if (_queue != nil)
	{
		GSNotificationBlockOperation *op = [[GSNotificationBlockOperation alloc] 
			initWithNotification: notif block: _block];

		[_queue addOperation: op];
	}
	else
	{
		CALL_BLOCK(_block, notif);
	}
}

@end
复制代码
```

## 存储容器

上面介绍了一些类的功能，但是要想实现通知中心的逻辑必须设计一套合理的存储结构，对于通知的存储基本上围绕下面几个结构体来做（大致了解下，后面章节会用到），后面会详细介绍具体逻辑的

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

复制代码
```

# 注册通知

正式开始“注册通知”的深入研究，注册通知有几个常用方法，但只需要研究典型的一两个就够了，原理都是一样的

目前只介绍`NSNotificationCenter`的注册流程，`NSNotificationQueue`的方式在下面章节单独拎出来解释

## 接口1

### 直接看源码（精简版便于理解）

```
/*
observer：观察者，即通知的接收者
selector：接收到通知时的响应方法
name: 通知name
object：携带对象
*/
- (void) addObserver: (id)observer
	    	selector: (SEL)selector
                name: (NSString*)name 
                object: (id)object {
  // 前置条件判断
  ......

  // 创建一个observation对象，持有观察者和SEL，下面进行的所有逻辑就是为了存储它
  o = obsNew(TABLE, selector, observer);

/*======= case1： 如果name存在 =======*/
  if (name) {
 	//-------- NAMED是个宏，表示名为named字典。以name为key，从named表中获取对应的mapTable
      n = GSIMapNodeForKey(NAMED, (GSIMapKey)(id)name);
      if (n == 0) { // 不存在，则创建 
          m = mapNew(TABLE); // 先取缓存，如果缓存没有则新建一个map
          GSIMapAddPair(NAMED, (GSIMapKey)(id)name, (GSIMapVal)(void*)m);
          ...
	  }
      else { // 存在则把值取出来 赋值给m
          m = (GSIMapTable)n->value.ptr;
	  }
 	//-------- 以object为key，从字典m中取出对应的value，其实value被MapNode的结构包装了一层，这里不追究细节
      n = GSIMapNodeForSimpleKey(m, (GSIMapKey)object);
      if (n == 0) {// 不存在，则创建 
          o->next = ENDOBS;
          GSIMapAddPair(m, (GSIMapKey)object, (GSIMapVal)o);
	  }
      else {
          list = (Observation*)n->value.ptr;
          o->next = list->next;
          list->next = o;
      }
    }
/*======= case2：如果name为空，但object不为空 =======*/
  else if (object) {
  	// 以object为key，从nameless字典中取出对应的value，value是个链表结构
      n = GSIMapNodeForSimpleKey(NAMELESS, (GSIMapKey)object);
      // 不存在则新建链表，并存到map中
      if (n == 0) { 
          o->next = ENDOBS;
          GSIMapAddPair(NAMELESS, (GSIMapKey)object, (GSIMapVal)o);
	  }
      else { // 存在 则把值接到链表的节点上
		...
	  }
    }
/*======= case3：name 和 object 都为空 则存储到wildcard链表中 =======*/
  else {
      o->next = WILDCARD;
      WILDCARD = o;
  }
}
复制代码
```

### 逻辑说明

从上面介绍的`存储容器`中我们了解到`NCTable`结构体中核心的三个变量以及功能：`wildcard`、`named`、`nameless`，在源码中直接用宏定义表示了：`WILDCARD`、`NAMELESS`、`NAMED`，下面逻辑会用到

建议如果看文字说明觉得复杂不好理解，就看看下节介绍的存储关系图

#### case1: 存在`name`（无论object是否存在）

1.  注册通知，如果通知的`name`存在，则以`name`为key从`named`字典中取出值`n`(这个`n`其实被`MapNode`包装了一层，便于理解这里直接认为没有包装)，这个`n`还是个字典，各种判空新建逻辑不讨论
2.  然后以`object`为key，从字典`n`中取出对应的值，这个值就是`Observation`类型(后面简称`obs`)的链表，然后把刚开始创建的`obs`对象`o`存储进去

**数据结构关系图**

这里就回答了上述`问题列表`的问题1的一部分，现在梳理下存储关系

![](https://upload-images.jianshu.io/upload_images/13277235-161480a9b966b17a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果注册通知时传入`name`，那么会是一个双层的存储结构

1.  找到`NCTable`中的`named`表，这个表存储了还有`name`的通知
2.  以`name`作为key，找到`value`，这个`value`依然是一个`map`
3.  `map`的结构是以`object`作为key，`obs`对象为value，这个`obs`对象的结构上面已经解释，主要存储了`observer & SEL`

#### case2: 只存在object

1.  以`object`为key，从`nameless`字典中取出value，此value是个`obs`类型的链表
2.  把创建的`obs`类型的对象`o`存储到链表中

**数据结构关系图**

![](https://upload-images.jianshu.io/upload_images/13277235-6f91d1f3e88e5769.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

只存在`object`时存储只有一层，那就是`object`和`obs`对象之间的映射

#### case3: 没有name和object

这种情况直接把`obs`对象存放在了`Observation  *wildcard`  链表结构中

## 接口2

### 源码

**接口功能：** 此接口实现的功能是在接收到通知时，在指定队列`queue`执行`block`

```
// 这个api使用频率较低，怎么实现在指定队列回调block的，值得研究
- (id) addObserverForName: (NSString *)name 
                   object: (id)object 
                    queue: (NSOperationQueue *)queue 
               usingBlock: (GSNotificationBlock)block
{
	// 创建一个临时观察者
	GSNotificationObserver *observer = 
		[[GSNotificationObserver alloc] initWithQueue: queue block: block];
	// 调用了接口1的注册方法
	[self addObserver: observer 
	         selector: @selector(didReceiveNotification:) 
	             name: name 
	           object: object];

	return observer;
}
复制代码
```

### 逻辑说明

这个接口依赖于`接口1`，只是多了一层代理观察者`GSNotificationObserver`，在`关键类结构`中已经介绍了它，设计思路值得学习

1.  创建一个`GSNotificationObserver`类型的对象`observer`，并把`queue`和`block`保存下来
2.  调用接口1进行通知的注册
3.  接收到通知时会响应`observer`的`didReceiveNotification:`方法，然后在`didReceiveNotification:`中把`block`抛给指定的`queue`去执行

## 小结

1.  从上述介绍可以总结，存储是以`name`和`object`为维度的，即判定是不是同一个通知要从`name`和`object`区分，如果他们都相同则认为是同一个通知，后面包括查找逻辑、删除逻辑都是以这两个为维度的，`问题列表`中的第九题也迎刃而解了
2.  理解数据结构的设计是整个通知机制的核心，其他功能只是在此基础上扩展了一些逻辑
3.  存储过程并没有做去重操作，这也解释了为什么同一个通知注册多次则响应多次

# 发送通知

## 源码

发送通知的核心逻辑比较简单，基本上就是查找和调用响应方法，核心函数如下

```
// 发送通知
- (void) postNotificationName: (NSString*)name
		       object: (id)object
		     userInfo: (NSDictionary*)info
{
// 构造一个GSNotification对象， GSNotification继承了NSNotification
  GSNotification	*notification;
  notification = (id)NSAllocateObject(concrete, 0, NSDefaultMallocZone());
  notification->_name = [name copyWithZone: [self zone]];
  notification->_object = [object retain];
  notification->_info = [info retain];

  // 进行发送操作
  [self _postAndRelease: notification];
}
//发送通知的核心函数，主要做了三件事：查找通知、发送、释放资源
- (void) _postAndRelease: (NSNotification*)notification {
    //step1: 从named、nameless、wildcard表中查找对应的通知
    ...
    //step2：执行发送，即调用performSelector执行响应方法，从这里可以看出是同步的
   	[o->observer performSelector: o->selector
                    withObject: notification];
	//step3: 释放资源
    RELEASE(notification);
}

复制代码
```

## 逻辑说明

其实上述代码注释说的很清晰了，主要做了三件事

1.  通过`name & object` 查找到所有的`obs`对象(保存了`observer`和`sel`)，放到数组中
2.  通过`performSelector：`逐一调用`sel`，这是个同步操作
3.  释放`notification`对象

## 小结

从源码逻辑可以看出发送过程的概述：从三个存储容器中：`named`、`nameless`、`wildcard`去查找对应的`obs`对象，然后通过`performSelector：`逐一调用响应方法，这就完成了发送流程

**核心点：**

1.  同步发送
2.  遍历所有列表，即注册多次通知就会响应多次

# 删除通知

这里源码太长而且基本上都是查找删除逻辑，不一一列举，感兴趣的去下载[源码](https://github.com/gnustep/libs-base)看下吧
**要注意的点：**

1.  查找时仍然以`name`和`object`为维度的，再加上`observer`做区分
2.  因为查找时做了这个链表的遍历，所以删除时会把重复的通知全都删除掉

```
// 删除已经注册的通知
- (void) removeObserver: (id)observer
		   name: (NSString*)name
                 object: (id)object {
  if (name == nil && object == nil && observer == nil)
      return;
      ...
}

- (void) removeObserver: (id)observer
{
  if (observer == nil)
    return;

  [self removeObserver: observer name: nil object: nil];
}
复制代码
```

# 异步通知

上面介绍的`NSNotificationCenter`都是同步发送的，而这里介绍关于`NSNotificationQueue`的异步发送，从线程的角度看并不是真正的异步发送，或可称为延时发送，它是利用了`runloop`的时机来触发的

## 入队

下面为精简版的源码，看源码的注释，基本上能明白大致逻辑

1.  根据`coalesceMask`参数判断是否合并通知
2.  接着根据`postingStyle`参数，判断通知发送的时机，如果不是立即发送则把通知加入到队列中：`_asapQueue`、`_idleQueue`

核心点：

1.  队列是双向链表实现
2.  当postingStyle值是立即发送时，调用的是`NSNotificationCenter`进行发送的，所以`NSNotificationQueue`还是依赖`NSNotificationCenter`进行发送

```
/*
* 把要发送的通知添加到队列，等待发送
* NSPostingStyle 和 coalesceMask在上面的类结构中有介绍
* modes这个就和runloop有关了，指的是runloop的mode
*/ 
- (void) enqueueNotification: (NSNotification*)notification
		postingStyle: (NSPostingStyle)postingStyle
		coalesceMask: (NSUInteger)coalesceMask
		    forModes: (NSArray*)modes
{
	......
  // 判断是否需要合并通知
  if (coalesceMask != NSNotificationNoCoalescing) {
      [self dequeueNotificationsMatching: notification
			    coalesceMask: coalesceMask];
  }
  switch (postingStyle) {
      case NSPostNow: {
      	...
      	// 如果是立马发送，则调用NSNotificationCenter进行发送
	     [_center postNotification: notification];
         break;
	  }
      case NSPostASAP:
      	// 添加到_asapQueue队列，等待发送
		add_to_queue(_asapQueue, notification, modes, _zone);
		break;

      case NSPostWhenIdle:
        // 添加到_idleQueue队列，等待发送
		add_to_queue(_idleQueue, notification, modes, _zone);
		break;
    }
}
复制代码
```

## 发送通知

这里截取了发送通知的核心代码，这个发送通知逻辑如下：

1.  `runloop`触发某个时机，调用`GSPrivateNotifyASAP()`和`GSPrivateNotifyIdle()`方法，这两个方法最终都调用了`notify()`方法
2.  `notify()`所做的事情就是调用`NSNotificationCenter`的`postNotification:`进行发送通知

```
static void notify(NSNotificationCenter *center, 
                   NSNotificationQueueList *list,
                   NSString *mode, NSZone *zone)
{
 	......
    // 循环遍历发送通知
    for (pos = 0; pos < len; pos++)
	{
	  NSNotification	*n = (NSNotification*)ptr[pos];

	  [center postNotification: n];
	  RELEASE(n);
	}
	......	
}
// 发送_asapQueue中的通知
void GSPrivateNotifyASAP(NSString *mode)
{
	notify(item->queue->_center,
	    item->queue->_asapQueue,
	    mode,
	    item->queue->_zone);
}
// 发送_idleQueue中的通知
void GSPrivateNotifyIdle(NSString *mode)
{
    notify(item->queue->_center,
    	item->queue->_idleQueue,
    	mode,
    	item->queue->_zone);
}

复制代码
```

## 小结

对于`NSNotificationQueue`总结如下

1.  依赖`runloop`，所以如果在其他子线程使用`NSNotificationQueue`，需要开启runloop
2.  最终还是通过`NSNotificationCenter`进行发送通知，所以这个角度讲它还是同步的
3.  所谓异步，指的是非实时发送而是在合适的时机发送，并没有开启异步线程

# 主线程响应通知

异步线程发送通知则响应函数也是在异步线程，如果执行UI刷新相关的话就会出问题，那么如何保证在主线程响应通知呢？

其实也是比较常见的问题了，基本上解决方式如下几种：

1.  使用`addObserverForName: object: queue: usingBlock`方法注册通知，指定在`mainqueue`上响应`block`
2.  在主线程注册一个`machPort`，它是用来做线程通信的，当在异步线程收到通知，然后给`machPort`发送消息，这样肯定是在主线程处理的，具体用法去网上资料很多，苹果官网也有

# 总结

本文写的内容比较多，以[GNUStep](https://github.com/gnustep/libs-base)源码为基础进行研究，全面阐述了通知的存储、发送、异步发送等原理，对研究学习有很大帮助

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
***

