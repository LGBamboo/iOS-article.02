# 最新iOS面试题之Runloop&KVO（附答案）

### 前言

今天这一篇我们来讲一下 Runloop和KVO

本章的主要回答的问题如下:

#### Runloop

*   app如何接收到触摸事件的
*   为什么只有主线程的runloop是开启的
*   为什么只在主线程刷新UI
*   PerformSelector和runloop的关系
*   如何使线程保活

#### KVO

*   实现原理
*   如何手动关闭kvo
*   通过KVC修改属性会触发KVO么
*   哪些情况下使用kvo会崩溃，怎么防护崩溃
*   kvo的优缺点

## Runloop

作为一个合格的iOS开发者必须对runloop有一个更深入的了解,下面我们来回答一下 相关问题

### 1.app如何接收到触摸事件的

回答这个问题前请认真阅读一下 [iOS触摸事件全家桶](https://www.jianshu.com/p/c294d1bd963d)

[![](https://upload-images.jianshu.io/upload_images/13277235-f667a2cf8887ce98.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200902iOSinterviewAnswers/runloop_event_receive.jpg) 

通过上图可以看出整个流程就是 我们app启动默认会通过machPort监听端口的方式 来接受IOHIDEvent 来接收和处理触摸事件.

### 2.为什么只有主线程的runloop是开启的

mian()函数中调用UIApplicationMain，这里会创建一个主线程，用于UI处理，为了让程序可以一直运行并接收事件，所以在主线程中开启一个runloop，让主线程常驻.

### 3.为什么只在主线程刷新UI

我们所有用到的UI都是来自于UIKit这个基础库.因为objc不是一门线程安全的语言所以存在多线程读写不同步的问题,如果使用加锁的方式操作系统开销很大,会耗费大量的系统资源(内存、时间片轮转、cpu处理速度...)，加上上面讲到的系统事件的接收处理都在主线程,如果UI异步线程的话 还会存在 同步处理事件的问题,所以多点触摸手势等一些事件要保持和UI在同一个线程相对是最优解.

另一方面是 屏幕的渲染是 60帧(60Hz/秒), 也就是1秒钟回调60次的频率,(iPad Pro 是120Hz/秒),我们的runloop 理想状态下也会按照时钟周期 回调60次(iPad Pro 120次), 这么高频率的调用是为了 屏幕图像显示能够垂直同步 不卡顿.在异步线程的话是很难保证这个处理过程的同步更新. 即便能保证的话 相对主线程而言 系统资源开销 线程调度等等将会占据大部分资源和在同一个线程只专门干一件事有点得不偿失.

### 4.PerformSelector和runloop的关系

当调用NSObect的 performSelector:相关的时候,内部会创建一个timer定时器添加到当前线程的runloop中,如果当前线程没有启动runloop,则该方法不会被调用.

开发中遇到最多的问题就是这个performSelector: 导致对象的延迟释放,这里开发过程中注意一下,可以用单次的NSTimer替代.

详细可以参考[Runloop与performSelector](https://juejin.im/post/6844903781755256840)

### 5.如何使线程保活？

想要线程保活的话就开启该线程的runloop即可,注意:在NSThread执行的方法中添加while(true){}，这样是模拟runloop的运行原理，结合GCD的信号量，在{}代码块中处理任务.

但是注意 开启runloop的方法要正确

如下代码

```
//测试开启线程
- (void)memoryTest {
    for (int i = 0; i < 100000; ++i) {
        NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run) object:nil];
        [thread start];
        [self performSelector:@selector(stopThread) onThread:thread withObject:nil waitUntilDone:YES];
    }
}
//线程停止
- (void)stopThread {
    CFRunLoopStop(CFRunLoopGetCurrent());
    NSThread *thread = [NSThread currentThread];
    [thread cancel];
}
//运行线程的runloop 注意 意添加的那个空port,否则会出现内存泄露
- (void)run {
    @autoreleasepool {
        NSLog(@"current thread = %@", [NSThread currentThread]);
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        if (!self.emptyPort) {
            self.emptyPort = [NSMachPort port];
        }
        [runLoop addPort:self.emptyPort forMode:NSDefaultRunLoopMode];
        [runLoop runMode:NSRunLoopCommonModes beforeDate:[NSDate distantFuture]];
    }
}
//下列代码用于模拟线程内部做的一些耗时任务
- (void)printSomething {
    NSLog(@"current thread = %@", [NSThread currentThread]);
    [self performSelector:@selector(printSomething) withObject:nil afterDelay:1];
}
//模拟手动点击按钮 让 runloop停掉
- (void)stopButtonDidClicked:(id)sender {
    [self performSelector:@selector(stopRunloop) onThread:self.thread withObject:nil waitUntilDone:YES];
}

- (void)stopRunloop {
    CFRunLoopStop(CFRunLoopGetCurrent());
}
```

详细请参考:[iOS开发深入研究Runloop与线程保活](https://allluckly.cn/%E6%8A%95%E7%A8%BF/tuogao55)

## KVO

在开发过程中我们经常使用KVO,下面解答一下KVO相关的问题.

### KVO的实现原理

通过`runtime`派生子类的方式 复写相关需要KVO监听的属性,在该属性setter之前和之后调用NSObject的监听方法,这样KVO就实现了属性变换前后的回调.

KVO派生的子类具体格式应该是:`NSKVONotifying_+类名`的类 eg: NSKVONotifying_Person

下面示例代码为Person类的name添加KVO的模拟实验

```
- (void)setName:(NSString *)name{
    _NSSetObjectValueAndNotify();
}

void _NSSetObjectValueAndNotify {
    [self willChangeValueForKey:@"name"];
    [super setName:name];
    [self didChangeValueForKey:@"name"];
}

- (void)didChangeValueForKey:(NSString *)key{
    [observe observeValueForKeyPath:key ofObject:self change:nil context:nil];
}
```

问题来了如何动态创建类呢?

```
//动态创建XXCustomClass
Class customClass = objc_allocateClassPair([NSObject class], "XXCustomClass", 0);
// 添加实例变量
class_addIvar(customClass, "age", sizeof(int), 0, "i");
// 动态添加方法
class_addMethod(customClass, @selector(hahahha), (IMP)hahahha, "V@:");

//需要实现的方法
void hahahha(id self, SEL _cmd)
{
    NSLog(@"hahahha====");
}

- (void)hahahha{

}

//最后注册到运行时环境
objc_registerClassPair(customClass);
```

> [V@:表示方法的参数和返回值](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)

具体原理以及自定义实现KVO可以参考[KVO详解及底层实现](https://cloud.tencent.com/developer/article/1136759)

### 如何手动关闭KVO?

被观察的对象复写如下方法 返回`NO`即可关闭KVO

```
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
	return NO;
}
```

如果关闭后还想触发 KVO的话 修改需要手动调用在变量setter的前后 主动调用 `willChangeValueForKey:`和`didChangeValueForKey:`

### 通过KVC修改属性会触发KVO么?

会的

### 哪些情况下使用kvo会崩溃，怎么防护崩溃？

使用不当 会crash,比如:

- 添加和移出不是成对出现且存在多线程添加KVO的情况,经常遇到的crash是移出 - 内存dealloc的时候 或者对象销毁前没有正确移出Observer

如何防护？

1.注意移出对象 匹配
2.内存野指针问题,一定要在对象销毁前移出观察者 3.可以使用第三方库BlockKit添加KVO,blockkit内部会自动移除Observer避免crash.

### KVO的优缺点

优点:

- 方便两个对象间同步状态(keypath)更加方便,一般都是在A类要观察B类的属性的变化.
- 非侵入式的得到某内部对象的状态改变并作出响应.(就是在不改变原来对象类的代码情况下即可做出对该对象的状态变化进行监听)
- 可以嵌入更改前后的两个时机的状态. - 可以通过Keypaths对嵌套对象的监听.

缺点:

- 需要手动移除观察者,不移除容易造成crash.
- 注册和移出成对匹配出现.
- keypath参数的类型String, 如果对象的成员变量被重构而变化字符串不会被编译器识别而报错.
- 实现观察的方式是复写NSObjec的相关KVO的方法,应该更加面向protocol的方式会更好.

## 总结

这一篇我们讲了 runloop和KVO相关的内容,这里面最负责的当属runloop如何处理触摸手势事件.建议认真研读相关链接文章.这样才有一个对runloop更深刻的理解。

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
