### 返回目录:[全网各大厂iOS面试题-题集大全](https://github.com/LGBamboo/iOS-Advanced)

# 精选大厂的iOS面试题总结（一）

### iOS面试题目录（一）


> * **[精选大厂的iOS面试题总结（一）](https://github.com/LGBamboo/iOS-article.02/blob/main/%E7%B2%BE%E9%80%89%E5%A4%A7%E5%8E%82%E7%9A%84iOS%E9%9D%A2%E8%AF%95%E9%A2%98%E6%80%BB%E7%BB%93%EF%BC%88%E4%B8%80%EF%BC%89.md)**
> * **[精选大厂的iOS面试题总结（二）](https://github.com/LGBamboo/iOS-article.02/blob/main/%E7%B2%BE%E9%80%89%E5%A4%A7%E5%8E%82%E7%9A%84iOS%E9%9D%A2%E8%AF%95%E9%A2%98%E6%80%BB%E7%BB%93%EF%BC%88%E4%BA%8C%EF%BC%89.md)**

*   1\. iOS内存管理机制
*   2\. NSThread、GCD、NSOperation多线程
*  3\. 输入一个字符串，判断这个字符串是否是有效的IP地址
*   4\. 大数加法怎么实现？
*   5\. 简述KVC和KVO，其中KVO实现原理？
*   6\. Block实现原理；堆上和栈上的数据如何同步？
*   7\. iOS设计模式
*   8\. 多线程有哪些？如何保证多线程中读写分离，加锁方案？
*   9\. 如何删除单链表中一个元素？
*   10\. NSNotificationCenter通知中心的实现原理？
*   11\. 推送如何实现的？
*   12\. SEL的使用和原理？
*   13\. 点击事件如何穿透透明的View?
*   14\. RunLoop的实现原理？（答案待完善）
*   15\. 简述Runtime，发送消息的过程；
*   16\. 简述weak的实现原理；
*   17\. 写一个单例；
*   18\. 如何从字符串中得到一个整数？
*   19\. 数组去重方式；
*   20\. 设计一个数据库；（答案待完善）
*   21\. 实现多个网络请求ABC执行完再执行D
*   22\. 列表页性能优化
*   23\. HTTPS（答案待完善）
*   23\. 音视频相关

## 1\. iOS内存管理机制

iOS内存管理机制的原理是引用计数，当这块内存被创建后，它的引用计数0->1，表示有一个对象或指针持有这块内存，拥有这块内存的所有权，如果这时候有另外一个对象或指针指向这块内存，那么为了表示这个后来的对象或指针对这块内存的所有权，引用计数1->2，之后若有一个对象或指针不再指向这块内存时，引用计数-1，表示这个对象或指针不再拥有这块内存的所有权，当一块内存的引用计数变为0，表示没有任何对象或指针持有这块内存，系统便会立刻释放掉这块内存。

*   alloc、new ：类初始化方法，开辟新的内存空间，引用计数+1；
*   retain ：实例方法，不会开辟新的内存空间，引用计数+1；
*   copy : 实例方法，把一个对象复制到新的内存空间，新的内存空间引用计数+1，旧的不会；其中分为浅拷贝和深拷贝，浅拷贝只是拷贝地址，不会开辟新的内存空间；深拷贝是拷贝内容，会开辟新的内存空间；
*   strong ：强引用； 引用计数+1；
*   release ：实例方法，释放对象；引用计数-1；
*   autorelease : 延迟释放；autoreleasepool自动释放池；当执行完之后引用计数-1；
*   还有是initWithFormat和stringWithFormat 字符串长度大于9时，引用计数+1；
*   assign : 弱引用 ；weak也是弱引用，两者区别：assign不但能作用于对象还能作用于基本数据类型，但是所指向的对象销毁时不会将当前指向对象的指针指向nil，有野指针的生成；weak只能作用于对象，不能作用于基本数据类型，所指向的对象销毁时会将当前指向对象的指针指向nil，防止野指针的生成。

## 2\. NSThread、GCD、NSOperation多线程

*   1、NSThread

> NSThread是封装程度最小最轻量级的，使用更灵活，但要手动管理线程的生命周期、线程同步和线程加锁等，开销较大；

```
[NSThread isMultiThreaded];//BOOL 是否开启了多线程	
[NSThread currentThread];//NSThread 获取当前线程	
[NSThread mainThread];//NSThread 获取主线程	
[NSThread sleepForTimeInterval:1];//线程睡眠1s

```

*   2、GCD

> GCD基于C语言封装的，遵循FIFO

```
dispatch_sync与dispatch_async//同步和异步操作

dispatch_queue_t;//主要有串行和并发两种；
	其中：
	dispatch_queue_create("concurrent_queue", DISPATCH_QUEUE_CONCURRENT)并发；
	dispatch_queue_create("serial_queue", DISPATCH_QUEUE_SERIAL)串行；

dispatch_once_t;//代码只会被执行一次,用于单例
dispatch_after；//延迟操作
dispatch_get_main_queue;//回到主线程操作

//Demo单例
+ (instancetype)sharedInstance {
    static ZZScreenshotsMonitor *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

//Demo：执行顺序
- (void)viewDidLoad {
    [super viewDidLoad];
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"1");
    });

    NSLog(@"2");

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND,0);

    dispatch_sync(queue, ^{
        NSLog(@"3");
    });

    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"4");
    });

    dispatch_async(queue, ^{
        NSLog(@"5");
    });

    NSLog(@"6");

    [self performSelector:@selector(delayMethod) withObject:nil afterDelay:0];

    NSLog(@"8");
}

- (void)delayMethod {
	NSLog(@"7");
}

打印结果：23658147；其中5和8随机调换

```

*   NSOperation

> NSOperation基于GCD封装的，比GCD可控性更强;可以加入操作依赖（addDependency）、设置操作队列最大可并发执行的操作个数（setMaxConcurrentOperationCount）、取消操作（cancel）等,需要使用两个它的实体子类：NSBlockOperation和NSInvocationOperation，或者继承NSOperation自定义子类;NSBlockOperation和NSInvocationOperation用法的主要区别是：前者执行指定的方法，后者执行代码块，相对来说后者更加灵活易用。NSOperation操作配置完成后便可调用start函数在当前线程执行，如果要异步执行避免阻塞当前线程则可以加入NSOperationQueue中异步执行


***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
***

## 3.输入一个字符串，判断这个字符串是否是有效的IP地址

```
+ (BOOL)isValidIP:(NSString *)ipStr {
    if (nil == ipStr) {
        return NO;
    }

    NSArray *ipArray = [ipStr componentsSeparatedByString:@"."];
    if (ipArray.count == 4) {
        for (NSString *ipnumberStr in ipArray) {
			  if ([self isPureInt:ipnumberStr]) {
				  int ipnumber = [ipnumberStr intValue];
	            if (!(ipnumber>=0 && ipnumber<=255)) {
	                return NO;
	            }
			  }            
        }
        return YES;
    }
    return NO;
}
//是否整形
- (BOOL)isPureInt:(NSString*)string {
	NSScanner* scan = [NSScanner scannerWithString:string];
	int val;
	return[scan scanInt:&val] && [scan isAtEnd];
}
//是否只含有数字
- (BOOL)validateNumber:(NSString*)number {
	BOOL res = YES;
	NSCharacterSet* tmpSet = [NSCharacterSet characterSetWithCharactersInString:@"0123456789"];
	int i = 0;
	while (i < number.length) {
	    NSString * string = [number substringWithRange:NSMakeRange(i, 1)];
	    NSRange range = [string rangeOfCharacterFromSet:tmpSet];
	    if (range.length == 0) {
	        res = NO;
	        break;
	    }
	    i++;
	}
	return res;

}

```

## 4.大数加法怎么实现？

> 使用字符串实现；

```
/两个大数相加算法
-(NSString *)addTwoNumberWithOneNumStr:(NSString *)one anotherNumStr:(NSString *)another
{
    int i = 0;
    int j = 0;
    int maxLength = 0;
    int sum = 0;
    int overflow = 0;
    int carryBit = 0;
    NSString *temp1 = @"";
    NSString *temp2 = @"";
    NSString *sums = @"";
    NSString *tempSum = @"";
    int length1 = (int)one.length;
    int length2 = (int)another.length;
    //1.反转字符串
    for (i = length1 - 1; i >= 0 ; i--) {
        NSRange range = NSMakeRange(i, 1);
        temp1 = [temp1 stringByAppendingString:[one substringWithRange:range]];
        NSLog(@"%@",temp1);
    }
    for (j = length2 - 1; j >= 0; j--) {
        NSRange range = NSMakeRange(j, 1);
        temp2 = [temp2 stringByAppendingString:[another substringWithRange:range]];
        NSLog(@"%@",temp2);
    }

    //2.补全缺少位数为0
    maxLength = length1 > length2 ? length1 : length2;
    if (maxLength == length1) {
        for (i = length2; i < length1; i++) {
            temp2 = [temp2 stringByAppendingString:@"0"];
            NSLog(@"i = %d --%@",i,temp2);
        }
    }else{
        for (j = length1; j < length2; j++) {
            temp1 = [temp1 stringByAppendingString:@"0"];
            NSLog(@"j = %d --%@",j,temp1);
        }
    }
    //3.取数做加法
    for (i = 0; i < maxLength; i++) {
        NSRange range = NSMakeRange(i, 1);
        int a = [temp1 substringWithRange:range].intValue;
        int b = [temp2 substringWithRange:range].intValue;
        sum = a + b + carryBit;
        if (sum > 9) {
            if (i == maxLength -1) {
                overflow = 1;
            }
            carryBit = 1;
            sum -= 10;
        }else{
            carryBit = 0;
        }
        tempSum = [tempSum stringByAppendingString:[NSString stringWithFormat:@"%d",sum]];
    }
    if (overflow == 1) {
        tempSum = [tempSum stringByAppendingString:@"1"];
    }
    int sumlength = (int)tempSum.length;
    for (i = sumlength - 1; i >= 0 ; i--) {
        NSRange range = NSMakeRange(i, 1);
        sums = [sums stringByAppendingString:[tempSum substringWithRange:range]];
    }
    NSLog(@"sums = %@",sums);
    return sums;
}

```

## 5.简述KVC和KVO，其中KVO实现原理？

> KVC : 键值编码（Key-Value Coding）,它是一种通过key值访问类属性的机制，而不是通过setter/getter方法访问。其中 KVC 原理：当调用- (void)setValue:(id)value forUndefinedKey:(NSString *)key时，KVC底层的执行机制如下：
> 首先搜索对应属性的setter方法
> 如果没有找到属性的setter方法，则会检查+ (BOOL)accessInstanceVariablesDirectly方法是否返回了YES（该方法默认返回YES）,如果返回了YES, 则KVC机制会搜索类中是否存在该属性的成员变量，也就是_属性名，存在则对该成员变量赋值。搜索成员变量名的顺序是 _key，_isKey，key，isKey。
> 另外我们也可以通过重写+ (BOOL)accessInstanceVariablesDirectly方法返回NO,这个时候KVC机制就会调用 - (void)setValue:(id)value forUndefinedKey:(NSString *)key。
> 如果没有找到成员变量，调用 - (void)setValue:(id)value forUndefinedKey:(NSString *)key。

> KVO : 键值观察者 （Key-Value Observer）: KVO 是观察者模式的一种实现，观察者A监听被观察者B的某个属性，当B的属性发生更改时，A就会收到通知，执行相应的方法。实现原理：基本的原理：当观察某对象A时，KVO机制动态创建一个对象A当前类的子类，并为这个新的子类重写了被观察属性keyPath的setter 方法。setter 方法随后负责通知观察对象属性的改变状况。

> Apple 使用了 isa 混写（isa-swizzling）来实现 KVO 。当观察对象A时，KVO机制动态创建一个新的名为：NSKVONotifying_A 的新类，该类继承自对象A的本类，且 KVO 为 NSKVONotifying_A 重写观察属性的 setter 方法，setter 方法会负责在调用原 setter 方法之前和之后，通知所有观察对象属性值的更改情况。

> 子类setter方法剖析：KVO 的键值观察通知依赖于 NSObject 的两个方法:willChangeValueForKey:和 didChangevlueForKey:，在存取数值的前后分别调用 2 个方法：
> 被观察属性发生改变之前，willChangeValueForKey:被调用，通知系统该 keyPath 的属性值即将变更；当改变发生后， didChangeValueForKey: 被调用，通知系统该 keyPath 的属性值已经变更；之后， observeValueForKey:ofObject:change:context: 也会被调用。且重写观察属性的 setter 方法这种继承方式的注入是在运行时而不是编译时实现的。

> 如果赋值没有通过 setter 方法或者 KVC，而是直接修改属性对应的成员变量，例如：仅调用 _name = @“newName”，这时是不会触发 KVO 机制，更加不会调用回调方法的

## 6.Block实现原理；堆上和栈上的数据如何同步？

> block本质上也是一个oc对象，他内部也有一个isa指针。block是封装了函数调用以及函数调用环境的OC对象。结构体，在栈上的情况, Block中的指针只是指向栈上的__block变量, 而当Block/__block变量被copy到堆上以后, 堆上Block会持有堆上__block变量. 而堆上的Block再次被调用copy时, 只是Block的引用计数+1而已, 而__block变量如果被多个堆上Block持有也只涉及到引用记数的变化. 一旦Block/__block变量的引用计数为0, 就会自动从堆上释放内存.这里Block/__block变量在堆上的内存管理与Objective-C对象完全一致.

```
Block类					原存储域	调用copy效果
_NSConcreteStackBlock	栈			从栈copy到堆
_NSConcreteGlobalBlock	数据域(.data域)	什么也不做
_NSConcreteMallocBlock	堆			引用计数+1

```

## 7.iOS设计模式

*   构造模式

> 构造模式用于将某个业务的属性和行为进行分离，当业务属性越多的时候该模式用起来就越方便。比如：我要自定义一个比较灵活的弹窗，这个弹窗有显示和隐藏、动画的功能，并且弹窗的大小、样式显示的位置都要可以自定义。这样我们就可以使用构造模式，将行为和属性分离出来，弹窗的显示和隐藏就是行为，其他的均为属性，这些属性的构造过程中就可以被定义好.

*   适配器模式；

> 1：何为适配器模式？
> 适配器模式将一个类的接口适配成用户所期待的。一个适配器通常允许因为接口不兼容而不能一起工作的类能够在一起工作，做法是将类自己的接口包裹在一个已存在的类中。
> 2：[如何使用适配器模式？]
> 当你想使用一个已经存在的类，而它的接口不符合你的需求；
> 你想创建一个可以复用的类，该类可以与其他不相关的类或不可预见的类协同工作；
> 你想使用一些已经存在的子类，但是不可能对每一个都进行子类化以匹配它们的接口，对象适配器可以适配它的父亲接口。
> 3：[适配器模式的优缺点？]
> 优点：降低数据层和视图层（对象）的耦合度，使之使用更加广泛，适应复杂多变的变化。
> 缺点：降低了可读性，代码量增加，对于不理解这种模式的人来说比较难看懂。

*   策略模式;

> 1:何为策略模式？策略模式定义了一系列的算法，并将每一个算法封装起来，而且使它们还可以相互替换。策略模式让算法独立于使用它的客户而独立变化。
> 2:如何使用策略模式？
> 在有多种算法相似的情况下，使用 if…else 所带来的复杂和难以维护。
> 如果在一个系统里面有许多类，它们之间的区别仅在于它们的行为，那么使用策略模式可以动态地让一个对象在许多行为中选择一种行为。
> 一个系统需要动态地在几种算法中选择一种。
> 如果一个对象有很多的行为，如果不用恰当的模式，这些行为就只好使用多重的条件选择语句来实现。
> 注意事项：如果一个系统的策略多于四个，就需要考虑使用混合模式，解决策略类膨胀的问题。
> 3:策略模式的优缺点？
> 优点：简化操作，提高代码维护性。算法可以自由切换，避免使用多重条件判断，扩展性良好。
> 缺点：在使用之前就要确定使用某种策略，而不是动态的选择策略。策略类会增多，所有策略类都需要对外暴露。

*   观察者模式;

> 1:[何为观察者模式？]
> 当对象间存在一对多关系时，则使用观察者模式（Observer Pattern）。比如，当一个对象被修改时，则会自动通知它的依赖对象。观察者模式属于行为型模式。
> 2:如何使用观察者模式？
> 一个对象状态改变给其他对象通知的问题，而且要考虑到易用和低耦合，保证高度的协作。一个对象（目标对象）的状态发生改变，所有的依赖对象（观察者对象）都将得到通知，进行广播通知。
> 3:观察者模式的优缺点？
> 优点：观察者和被观察者是抽象耦合的。建立一套触发机制。缺点：如果一个被观察者对象有很多的直接和间接的观察者的话，将所有的观察者都通知到会花费很多时间。如果在观察者和观察目标之间有循环依赖的话，观察目标会触发它们之间进行循环调用，可能导致系统崩溃。观察者模式没有相应的机制让观察者知道所观察的目标对象是怎么发生变化的，而仅仅只是知道观察目标发生了变化。

*   原型/外观模式;

> 1:何为原型/外观模式？
> 原型模式：（Prototype Pattern）用于创建重复的对象，同时又能保证性能。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。这种模式是实现了一个原型接口，该接口用于创建当前对象的克隆。当直接创建对象的代价比较大时，则采用这种模式。
> 外观模式：（Facade Pattern）隐藏系统的复杂性，并向客户端提供了一个客户端可以访问系统的接口。这种类型的设计模式属于结构型模式，它向现有的系统添加一个接口，来隐藏系统的复杂性。这种模式涉及到一个单一的类，该类提供了客户端请求的简化方法和对现有系统类方法的委托调用。
> 2:如何使用原型/外观模式？
> 原型模式：
> 当一个系统应该独立于它的产品创建，构成和表示时。
> 当要实例化的类是在运行时刻指定时，例如，通过动态装载。
> 为了避免创建一个与产品类层次平行的工厂类层次时。
> 当一个类的实例只能有几个不同状态组合中的一种时。建立相应数目的原型并克隆它们可能比每次用合适的状态手工实例化该类更方便一些。
> 外观模式：
> 客户端不需要知道系统内部的复杂联系，整个系统只需提供一个"接待员"即可。
> 定义系统的入口。
> 3:原型/外观模式的优缺点？
> 原型模式：
> 优点：性能提高，逃避构造函数的约束。
> 缺点：
> 配备克隆方法需要对类的功能进行通盘考虑，这对于全新的类不是很难，但对于已有的类不一定很容易。
> 必须实现 Cloneable 接口。
> 逃避构造函数的约束。
> 外观模式
> 优点：减少系统相互依赖、提高灵活性、提高了安全性。
> 缺点：不符合开闭原则，如果要改东西很麻烦，继承重写都不合适。

*   工厂模式;

> 1:何为工厂模式？
> 这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
> 在工厂模式中，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。
> 2:如何使用工厂模式？
> 我们明确地计划不同条件下创建不同实例时。
> 作为一种创建类模式，在任何需要生成复杂对象的地方，都可以使用工厂方法模式。有一点需要注意的地方就是复杂对象适合使用工厂模式，而简单对象，特别是只需要通过 new 就可以完成创建的对象，无需使用工厂模式。如果使用工厂模式，就需要引入一个工厂类，会增加系统的复杂度。
> 3:工厂模式的优缺点？
> 优点：
> 一个调用者想创建一个对象，只要知道其名称就可以了。
> 扩展性高，如果想增加一个产品，只要扩展一个工厂类就可以。
> 屏蔽产品的具体实现，调用者只关心产品的接口。
> 缺点：
> 每次增加一个产品时，都需要增加一个具体类和对象实现工厂，使得系统中类的个数成倍增加，在一定程度上增加了系统的复杂度，同时也增加了系统具体类的依赖。这并不是什么好事。

*   桥接模式;

> 1:何为桥接模式？
> 桥接（Bridge）是用于把抽象化与实现化解耦，使得二者可以独立变化。这种类型的设计模式属于结构型模式，它通过提供抽象化和实现化之间的桥接结构，来实现二者的解耦。
> 这种模式涉及到一个作为桥接的接口，使得实体类的功能独立于接口实现类。这两种类型的类可被结构化改变而互不影响。
> 2:如何使用桥接模式？
> 在有多种可能会变化的情况下，用继承会造成类爆炸问题，扩展起来不灵活。
> 实现系统可能有多个角度分类，每一种角度都可能变化。
> 把这种多角度分类分离出来，让它们独立变化，减少它们之间耦合。
> 3:桥接模式的优缺点？
> 优点 ：抽象和实现的分离、优秀的扩展能力、实现细节对客户透明。
> 缺点：桥接模式的引入会增加系统的理解与设计难度，由于聚合关联关系建立在抽象层，要求开发者针对抽象进行设计与编程。

*   代理模式;

> 1:何为代理模式？
> 在代理模式（Proxy Pattern）中，一个类代表另一个类的功能。这种类型的设计模式属于结构型模式。
> 在代理模式中，我们创建具有现有对象的对象，以便向外界提供功能接口。
> 2:如何使用代理模式？
> 在直接访问对象时带来的问题，比如说：要访问的对象在远程的机器上。在面向对象系统中，有些对象由于某些原因（比如对象创建开销很大，或者某些操作需要安全控制，或者需要进程外的访问），直接访问会给使用者或者系统结构带来很多麻烦，我们可以在访问此对象时加上一个对此对象的访问层。
> 想在访问一个类时做一些控制。
> 3:代理模式的优缺点？
> 优点：
> 职责清晰、高扩展性、智能化。
> 缺点：
> 由于在客户端和真实主题之间增加了代理对象，因此有些类型的代理模式可能会造成请求的处理速度变慢。
> 实现代理模式需要额外的工作，有些代理模式的实现非常复杂。

*   单例模式;

> 1:何为单例模式？
> 这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。
> 注意：
> 单例类只能有一个实例。
> 单例类必须自己创建自己的唯一实例。
> 单例类必须给所有其他对象提供这一实例。
> 2：如何使用单例模式？
> 当您想控制实例数目，节省系统资源的时候。
> 3：单例模式的优缺点？
> 优点：
> 在内存里只有一个实例，减少了内存的开销，尤其是频繁的创建和销毁实例（比如管理学院首页页面缓存）。
> 避免对资源的多重占用比如写文件操作。
> 缺点：
> 没有接口，不能继承，与单一职责原则冲突，一个类应该只关心内部逻辑，而不关心外面怎么样来实例化。

*   备忘录模式;

> 1:何为备忘录模式？
> 备忘录模式（Memento Pattern）保存一个对象的某个状态，以便在适当的时候恢复对象。备忘录模式属于行为型模式。
> 2：如何使用备忘录模式？
> 很多时候我们总是需要记录一个对象的内部状态，这样做的目的就是为了允许用户取消不确定或者错误的操作，能够恢复到他原先的状态，使得他有"后悔药"可吃。
> 3：备忘录模式的优缺点？
> 优点：
> 给用户提供了一种可以恢复状态的机制，可以使用户能够比较方便地回到某个历史的状态。
> 实现了信息的封装，使得用户不需要关心状态的保存细节。
> 缺点：
> 消耗资源。如果类的成员变量过多，势必会占用比较大的资源，而且每一次保存都会消耗一定的内存。

*   生成器模式；

> 1：何为送生成器模式？
> 建造者模式（Builder Pattern）使用多个简单的对象一步一步构建成一个复杂的对象。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
> 2：如何使用生成器模式？
> 主要解决在软件系统中，有时候面临着"一个复杂对象"的创建工作，其通常由各个部分的子对象用一定的算法构成；由于需求的变化，这个复杂对象的各个部分经常面临着剧烈的变化，但是将它们组合在一起的算法却相对稳定。
> 一些基本部件不会变，而其组合经常变化的时候。
> 3：生成器模式的优缺点？
> 优点：
> 建造者独立，易扩展。
> 便于控制细节风险。
> 缺点：
> 产品必须有共同点，范围有限制。
> 如内部变化复杂，会有很多的建造类。

*   命令模式;

> 1:何为命令模式？
> 命令模式（Command Pattern）是一种数据驱动的设计模式，它属于行为型模式。请求以命令的形式包裹在对象中，并传给调用对象。调用对象寻找可以处理该命令的合适的对象，并把该命令传给相应的对象，该对象执行命令。
> 主要解决的问题？
> 在软件系统中，行为请求者与行为实现者通常是一种紧耦合的关系，但某些场合，比如需要对行为进行记录、撤销或重做、事务等处理时，这种无法抵御变化的紧耦合的设计就不太合适。
> 2：如何使用命令模式？
> 在某些场合，比如要对行为进行"记录、撤销/重做、事务"等处理，这种无法抵御变化的紧耦合是不合适的。在这种情况下，如何将"行为请求者"与"行为实现者"解耦？将一组行为抽象为对象，可以实现二者之间的松耦合。
> 3：命令模式的优缺点？
> 优点：降低了系统耦合度，新的命令可以很容易添加到系统中去。
> 缺点：使用命令模式可能会导致某些系统有过多的具体命令类。

## 8.多线程有哪些？如何保证多线程中读写分离，加锁方案？

> NSThread GCD NSOperation

> iOS 实现线程加锁有很多种方式。@synchronized、 NSLock、NSRecursiveLock、NSCondition、NSConditionLock、pthread_mutex、dispatch_semaphore、OSSpinLock、atomic(property) set/ge等等各种方式。

## 9.如何删除单链表中一个元素？

> 先来看看删除的原理：因为数据结构是单链表，要想删除第i个节点，就要找到第i个节点；要想找到第i个节点，就要找到第i-1个节点；要想找到第i-1个节点，就要找到第i-2个节点…于是就要从第一个节点开始找起，一直找到第i-1个节点。如何找？让一个指针从头结点开始移动，一直移动到第i-1个节点为止。这个过程中可以用一个变量j从0开始计数，一直自增到i-1。之后呢？我们把第i-1个节点找到了，就让它的指针域指向第i+1个节点，这样就达到了删除的目的。而第i+1个节点的地址又从第i个节点获得，第i个节点的地址又是第i-1个节点的后继。因此我们可以这样做：先让一个指针指向第i-1个节点的后继，（保存i+1节点的地址），再让i-1节点的后继指向第i个节点的后继，这样就将第i个节点删除了。(p->next=q->next;)

```
Status ListDelete(LinkList *L,int i,ElemType *e){
    int j;
    LinkList p,q;
    p = *L; // 声明一结点p指向链表第一个结点
    j = 1;
    while (p->next && j < i)  /* 遍历寻找第i个元素 */
    {
        p = p->next;
        ++j;
    }
    if (!(p->next) || j > i)
        return ERROR;           /* 第i个元素不存在 */
    q = p->next;
    p->next = q->next;            /* 将q的后继赋值给p的后继 */
    *e = q->data;               /* 将q结点中的数据给e */
    free(q);                    /* 让系统回收此结点，释放内存 */
    return OK;
}

```

## 10.NSNotificationCenter通知中心的实现原理？

> NSNotificationCenter是类似一个广播中心站，使用defaultCenter来获取应用中的通知中心，它可以向应用任何地方发送和接收通知。在通知中心注册观察者，发送者使用通知中心广播时，以NSNotification的name和object来确定需要发送给哪个观察者。为保证观察者能接收到通知，所以应先向通知中心注册观察者，接着再发送通知这样才能在通知中心调度表中查找到相应观察者进行通知。

```
-(void)postNotification:(NSNotification *)notification;
-(void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject;
-(void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject userInfo:(nullable NSDictionary *)aUserInfo;

```

> 发送通知通过name和object来确定来标识观察者,name和object两个参数的规则相同即当通知设置name为kChangeNotifition时，那么只会发送给符合name为kChangeNotifition的观察者，同理object指发送给某个特定对象通知，如果只设置了name，那么只有对应名称的通知会触发。如果同时设置name和object参数时就必须同时符合这两个条件的观察者才能接收到通知。

```
- (void)addObserver:(id)observer selector:(SEL)aSelector name:(nullable NSNotificationName)aName object:(nullable id)anObject;
- (id <NSObject>)addObserverForName:(nullable NSNotificationName)name object:(nullable id)obj queue:(nullable NSOperationQueue *)queue usingBlock:(void (^)(NSNotification *note))block NS_AVAILABLE(10_6, 4_0);

```

> 第一种方式是比较常用的添加Oberver的方式，接到通知时执行aSelector。
> 第二种方式是基于Block来添加观察者，往通知中心的调度表中添加观察者，这个观察者包括一个queue和一个block,并且会返回这个观察者对象。当接到通知时执行block所在的线程为添加观察者时传入的queue参数，queue也可以为nil，那么block就在通知所在的线程同步执行。
> 这里需要注意的是如果使用第二种的方式创建观察者需要弱引用可能引起循环引用的对象,避免内存泄漏。

* * *

> NSNotificatinonCenter实现原理:
> NSNotificatinonCenter是使用观察者模式来实现的用于跨层传递消息，用来降低耦合度。
> NSNotificatinonCenter用来管理通知，将观察者注册到NSNotificatinonCenter的通知调度表中，然后发送通知时利用标识符name和object识别出调度表中的观察者，然后调用相应的观察者的方法，即传递消息（在Objective-C中对象调用方法，就是传递消息，消息有name或者selector，可以接受参数，而且可能有返回值），如果是基于block创建的通知就调用NSNotification的block。

## 11.推送如何实现的？

*   1.由App向iOS设备发送一个注册通知，用户需要同意系统发送推送。
*   2.iOS应用向APNS远程推送服务器发送App的Bundle Id和设备的UDID。
*   3.APNS根据设备的UDID和App的Bundle Id生成deviceToken再发回给App。
*   4.App再将deviceToken发送给远程推送服务器(自己的服务器), 由服务器保存在数据库中。
*   5.当自己的服务器想发送推送时, 在远程推送服务器中输入要发送的消息并选择发给哪些用户的deviceToken，由远程推送服务器发送给APNS。
*   6.APNS根据deviceToken发送给对应的用户。

## 12.SEL的使用和原理？

> SEL 类成员方法的指针
> 可以理解 @selector()就是取类方法的编号,他的行为基本可以等同C语言的中函数指针,只不过C语言中，可以把函数名直接赋给一个函数指针，而Object-C的类不能直接应用函数指针，这样只能做一个@selector语法来取.
> 它的结果是一个SEL类型。这个类型本质是类方法的编号(函数地址)

## 13.点击事件如何穿透透明的View?

```
- (UIView*)hitTest:(CGPoint)point withEvent:(UIEvent *)event{
    UIView *hitView = [super hitTest:point withEvent:event];
    if(hitView == self){
        return nil;
    }
    return hitView;
}

```

## 14.RunLoop的实现原理？（答案待完善）

> RunLoop实际上是一个对象，这个对象在循环中用来处理程序运行过程中出现的各种事件（比如说触摸事件、UI刷新事件、定时器事件、Selector事件）和消息，从而保持程序的持续运行，而且在没有事件处理的时候，会进入睡眠模式，从而节省CPU资源,提高程序性能。

**[参考：深入理解Runloop](https://blog.ibireme.com/2015/05/18/runloop/#base)**


## 15.简述Runtime，发送消息的过程；

> 动态的添加对象的成员变量和方法;动态交换两个方法的实现;拦截并替换方法;在方法上增加额外功能;实现NSCoding的自动归档和解档;实现字典转模型的自动转换;

> clang -rewrite-objc main.m 查看最终生成代码

消息转发：

*   1：动态方法解析

```
+ (BOOL)resolveInstanceMethod:(SEL)selector;
+ (BOOL)resolveClassMethod:(SEL)selector;

```

*   2：备援接收者(重定向)

```
- (id)forwardingTargetForSelector:(SEL)selector;

```

*   3：完整的消息转发(NSInvocation)

```
- (void)forwardInvocation:(NSInvocation *)invocation;

```

## 16.简述weak的实现原理；

> weak 关键字的作用弱引用，所引用对象的计数器不会加一，并在引用对象被释放的时候自动被设置为 nil;
> 
> weak是有Runtime维护的weak表;
> 
> 3.weak释放为nil过程
> weak被释放为nil，需要对对象整个释放过程了解，如下是对象释放的整体流程：
> 1、调用objc_release
> 2、因为对象的引用计数为0，所以执行dealloc
> 3、在dealloc中，调用了_objc_rootDealloc函数
> 4、在_objc_rootDealloc中，调用了object_dispose函数
> 5、调用objc_destructInstance
> 6、最后调用objc_clear_deallocating。
> 
> 对象准备释放时，调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。
> 
> 其实Weak表是一个hash（哈希）表，然后里面的key是指向对象的地址，Value是Weak指针的地址的数组。
> ###总结
> weak是Runtime维护了一个hash(哈希)表，用于存储指向某个对象的所有weak指针。weak表其实是一个hash（哈希）表，Key是所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象指针的地址）数组。

## 17.写一个单例；

```
+ (instancetype)sharedInstance {
    static RMF *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });

    return instance;
}

```

## 18.如何从字符串中得到一个整数？

```
- (BOOL)isPureNumandCharacters:(NSString *)text 
{ 
    for(int i = 0; i < [text length]; ++i) {
        int a = [text characterAtIndex:i]; 
        if ([self isNum:a]){
            continue; 
        } else { 
            return NO; 
        } 
    } 
    return YES; 
}

```

## 19.数组去重方式；

*   数组法

```
for (NSString *item in originalArr) {
	if (![resultArrM containsObject:item]) {
	  [resultArrM addObject:item];
	}
}

```

*   利用NSDictionary

```
for (NSNumber *n in originalArr) {
	[dict setObject:n forKey:n];
}

```

*   NSSet

```
NSSet *set = [NSSet setWithArray:originalArr];

```

## 20.设计一个数据库；（答案待完善）

> 主要是对比一下常用的几种

## 21.实现多个网络请求ABC执行完再执行D

> 方案1：使用group和semaphore
> 方案2：group_enter和group_leave也可以实现
> 下面使用方案1实现例子

```
	dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    dispatch_group_async(group, queue, ^{
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0f * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            //异步执行A
            dispatch_semaphore_signal(semaphore);
        });
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    });

    dispatch_group_async(group, queue, ^{
                dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0f * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
             //异步执行B
            dispatch_semaphore_signal(semaphore);
        });
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    });

    dispatch_group_async(group, queue, ^{
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0f * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
             //异步执行C
            dispatch_semaphore_signal(semaphore);
        });
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    });

    dispatch_group_notify(group, queue, ^{
			 //执行D
    });

```

## 22.列表页性能优化

*   如何检测

> 1）Instruments中：Core Animation；
> 2）FPS：CADisplayLink

*   优化方案

> 1、文本、布局计算，提前计算缓存；
> 2、对象创建；CALayer代替UIView;
> 3、离屏渲染；
> 4、图片解码；

> （离屏渲染是指图层在被显示之前是在当前屏幕缓冲区以外开辟的一个缓冲区进行渲染操作。
> 离屏渲染需要多次切换上下文环境：先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上又需要将上下文环境从离屏切换到当前屏幕，而上下文环境的切换是一项高开销的动作。）
> 
> 1.阴影（UIView.layer.shadowOffset/shadowRadius/…）
> 2.圆角（当 UIView.layer.cornerRadius 和 UIView.layer.maskToBounds 一起使用时）
> 3.图层蒙板
> 4.开启光栅化（shouldRasterize = true）

> 1、使用CAShapeLayer和UIBezierPath设置圆角;
> 2、UIBezierPath和Core Graphics框架画出一个圆角;

```
//1、使用CAShapeLayer和UIBezierPath设置圆角;
UIBezierPath *maskPath = [UIBezierPath bezierPathWithRoundedRect:imageView.bounds byRoundingCorners:UIRectCornerAllCorners cornerRadii:imageView.bounds.size];
CAShapeLayer *maskLayer = [[CAShapeLayer alloc]init];
maskLayer.frame = imageView.bounds;
maskLayer.path = maskPath.CGPath;
imageView.layer.mask = maskLayer;

//2、UIBezierPath和Core Graphics框架画出一个圆角
UIGraphicsBeginImageContextWithOptions(imageView.bounds.size,NO,1.0);
[[UIBezierPath bezierPathWithRoundedRect:imageView.boundscornerRadius:imageView.frame.size.width]addClip];
[imageView drawRect:imageView.bounds];
imageView.image=UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
[self.view addSubview:imageView];

```

## 23.HTTPS（答案待完善）

> HTTP+对称加密和非对称加密+证书认证

## 23.音视频相关

采集视频,音频–》使用iOS原生框架 AVFoundation.framework
视频滤镜处理–》使用iOS原生框架 CoreImage.framework；使用第三方框架 GPUImage.framework

> CoreImage 与 GPUImage 框架比较:
> 在实际项目开发中,开发者更加倾向使用于GPUImage框架.
> 首先它在使用性能上与iOS提供的原生框架,并没有差别;其次它的使用便利性高于iOS原生框架,最后也是最重要的GPUImage框架是开源的.而大家如果想要学习GPUImage框架,建议学习OpenGL ES,其实GPUImage的封装和思维都是基于OpenGL ES.

视频\音频编码压缩
视频: 使用FFmpeg,X264算法把视频原数据YUV/RGB编码成H264
音频: 使用fdk_aac 将音频数据PCM转换成AAC
视频: VideoToolBox框架
音频: AudioToolBox 框架
硬编码
软编码
推流
流媒体协议: RTMP\RTSP\HLS\FLV
视频封装格式: TS\FLV
音频封装格式: Mp3\AAC
推流: 将采集的音频.视频数据通过流媒体协议发送到流媒体服务器
推流技术
流媒体服务器
数据分发
截屏
实时转码
内容检测
拉流
拉流: 从流媒体服务器中获取音频\视频数据
流媒体协议: RTMP\RTSP\HLS\FLV
音视频解码
视频: 使用FFmpeg,X264算法解码
音频: 使用fdk_aac 解码
视频: VideoToolBox框架
音频: AudioToolBox 框架
硬解码
软解码
播放
ijkplayer,kxmovie 都是基于FFmpeg框架封装的
ijkplayer 播放框架
kxmovie 播放框架

### 返回目录:[全网各大厂iOS面试题-题集大全](https://github.com/LGBamboo/iOS-Advanced)

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
