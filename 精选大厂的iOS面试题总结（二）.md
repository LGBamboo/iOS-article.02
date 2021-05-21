# 精选大厂的iOS面试题总结（二）

### 面试题目录（二）

> * **[精选大厂的iOS面试题总结（一）](https://www.jianshu.com/p/deeac4ab2742)**
> * **[精选大厂的iOS面试题总结（二）](https://www.jianshu.com/p/89601ba29684)**

*   1\. 统计一个字符数组中每个字符出现的次数？
*   2\. 实现一个反转二叉树；
*   3\. 如何获取VC上所有的Button?
*   4\. 排序算法有哪些？(答案待完善)
*   5\. self和super区别；
*   6.UIViewController的生命周期；
*   7.UIButton的继承链，如何改变它的点击区域；
*   8.Category
*   9.实现setter方法
*   10.iOS判断一个字符串中是否都是数字
*   11.如何合并两个有序数组？
*   12.GET和POST区别
*   13.面向对象编程的六大原则
*   14.iOS Animation
*   15.为什么必须在主线程中操作UI
*   16.显式和隐式动画的区别
*   17.OC内联函数 inline

## 1.统计一个字符数组中每个字符出现的次数？

```
void main()
{
    char str[20];
    int i,num[256]={0};
    printf("please input string:");
    scanf("%s",str);
    for(i=0;i<strlen(str);i++)
        num[(int)str[i]]++;
    for(i=0;i<256;i++)
        if(num[i]!=0)
            printf("字符%c出现%d次\n",(char)i,num[i]);
}

```

## 2.实现一个反转二叉树；

```
@interface TreeNode : NSObject
@property (nonatomic, assign) NSInteger val;
@property (nonatomic, strong) TreeNode *left;
@property (nonatomic, strong) TreeNode *right;
@end
- (void)exchangeNode:(TreeNode *)node {

    //判断是否存在node节点
    if(node) {
        //交换左右节点
        TreeNode *temp = node.left;
        node.left = node.right;
        node.right = temp;
    }

}

- (TreeNode *)invertTree:(TreeNode *)root
{
    //边界条件 递归结束或输入为空情况
    if(!root) {
       return root;
    }

    //递归左右子树
    [self invertTree:root.left];
    [self invertTree:root.right];
    //交换左右子节点
    [self exchangeNode:root];

    return root;
}

```

>**作为一个开发者，有一个学习的氛围跟一个交流圈子特别重要，这是小编的一个交流群，点击加入群聊 [iOS开发交流](https://jq.qq.com/?_wv=1027&k=5PARXCI) ：937194184；不管你是小白还是大牛欢迎入驻 ，分享BAT等各大厂面试题、面试经验，讨论技术，大家一起交流学习成长！**

![](https://upload-images.jianshu.io/upload_images/17495317-b75b7dd57b9af243.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 3.如何获取VC上所有的Button?

> 递归

## 4.排序算法有哪些？(答案待完善)

> 冒泡、快速、插入、选择、希尔、堆等等

## 5.self和super区别；

> self调用自己方法，super调用父类方法
> 
> self是类，super是预编译指令
> 
> 【self class】和【super class】输出是一样的
> 
> 1.当使用 self 调用方法时，会从当前类的方法列表中开始找，如果没有，就从父类中再找；而当使用 super 时，则从父类的方法列表中开始找，然后调用父类的这个方法。
> 
> 2.当使用 self 调用时，会使用 objc_msgSend 函数： id objc_msgSend(id theReceiver, SEL theSelector, …)。第 一个参数是消息接收者，第二个参数是调用的具体类方法的 selector，后面是 selector 方法的可变参数。以 [self setName:] 为例，编译器会替换成调用 objc_msgSend 的函数调用，其中 theReceiver 是 self，theSelector 是 @selector(setName:)，这个 selector 是从当前 self 的 class 的方法列表开始找的 setName，当找到后把对应的 selector 传递过去。
> 
> 3.当使用 super 调用时，会使用 objc_msgSendSuper 函数：id objc_msgSendSuper(struct objc_super *super, SEL op, …)第一个参数是个objc_super的结构体，第二个参数还是类似上面的类方法的selector,

```
struct objc_super {
      id receiver;
      Class superClass;
};

```

> 当编译器遇到 [super setName:] 时，开始做这几个事：
> 1）构 建 objc_super 的结构体，此时这个结构体的第一个成员变量 receiver 就是 子类，和 self 相同。而第二个成员变量 superClass 就是指父类
> 调用 objc_msgSendSuper 的方法，将这个结构体和 setName 的 sel 传递过去。
> 2）函数里面在做的事情类似这样：从 objc_super 结构体指向的 superClass 的方法列表开始找 setName 的 selector，找到后再以 objc_super->receiver 去调用这个 selector

## 6.UIViewController的生命周期；

> initWithCoder; awakeFromNib; loadView; viewDidLoad; viewWillAppear; viewWillLayoutSubviews; viewDidLayoutSubviews; viewDidAppear; viewWillDisappear; viewDidDisappear; dealloc; didReceiveMemoryWarning

## 7.UIButton的继承链，如何改变它的点击区域；

> UIButton > UIControl > UIView > UIResponder > NSObject

```
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent*)event
{
    CGRect bounds = self.bounds;
    CGFloat width = MAX(100 - bounds.size.width, 0);
    CGFloat height = MAX(100 - bounds.size.height, 0);
    bounds = CGRectInset(bounds, -width/2, -height/2);
    return CGRectContainsPoint(bounds, point);
}

```

## 8.Category

> Build Phases ->Complie Source 中的编译顺序

> 1.属性。Property
> 2.实例变量。Ivar（属性是给成员变量默认添加了setter和getter方法。tips：如果不用@dynamic修饰的话。）
> 3.isa指针。在Objective-C中，任何类的定义都是对象。类和类的实例（对象）没有任何本质上的区别。任何对象都有isa指针。但是分类没有。

> category 它是在运行期决议的。 因为在运行期即编译完成后，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的。

> 使用Runtime技术中的关联对象可以为类别添加属性。
> 其原因是：关联对象都由AssociationsManager管理，AssociationsManager里面是由一个静态AssociationsHashMap来存储所有的关联对象的。这相当于把所有对象的关联对象都存在一个全局map里面。而map的的key是这个对象的指针地址（任意两个不同对象的指针地址一定是不同的），而这个map的value又是另外一个AssociationsHashMap，里面保存了关联对象的kv对。
> 如合清理关联对象？
> runtime的销毁对象函数objc_destructInstance里面会判断这个对象有没有关联对象，如果有，会调用_object_remove_assocations做关联对象的清理工作。（详见Runtime的源码）

> extension在编译期决议;extension可以添加实例变量，而category是无法添加实例变量的（因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的）

## 9.实现setter方法

```
-(void)setName:(NSString *)name{

    if (_name != name) {
        [_name release];
        _name = [name copy];
    }
}

```

## 10.iOS判断一个字符串中是否都是数字

```
第一种方式是使用NSScanner：
1\. 整形判断
- (BOOL)isPureInt:(NSString *)string{
NSScanner* scan = [NSScanner scannerWithString:string]; 
int val; 
return [scan scanInt:&val] && [scan isAtEnd];
}

2.浮点形判断：
- (BOOL)isPureFloat:(NSString *)string{
NSScanner* scan = [NSScanner scannerWithString:string]; 
float val; 
return [scan scanFloat:&val] && [scan isAtEnd];
}
第二种方式是使用循环判断
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
或者 C语言中常用的方式.
- (BOOL)isAllNum:(NSString *)string{
    unichar c;
    for (int i=0; i<string.length; i++) {
        c=[string characterAtIndex:i];
        if (!isdigit(c)) {
            return NO;
        }
    }
    return YES;
}
第三种方式则是使用NSString的trimming方法
- (BOOL)isPureNumandCharacters:(NSString *)string 
{ 
string = [string stringByTrimmingCharactersInSet;[NSCharacterSet decimalDigitCharacterSet]];
if(string.length > 0) 
{
     return NO;
} 
return YES;
}

```

## 11.如何合并两个有序数组？

> 比较相邻的两个元素，类似冒泡排序；

```
	NSMutableArray *A = [NSMutableArray arrayWithObjects:@4,@5,@8,@10,@15, nil];
    NSMutableArray *B = [NSMutableArray arrayWithObjects:@2,@6,@7,@9,@11,@12,@13, nil];
    NSMutableArray *C = [NSMutableArray array];
    int count = (int)A.count+(int)B.count;
    int index = 0;
    for (int i = 0; i < count; i++) {
        if (A[0]<B[0]) {
            [C addObject:A[0]];
            [A removeObject:A[0]];
        }
        else if (B[0]<A[0]) {
            [C addObject:B[0]];
            [B removeObject:B[0]];
        }
        if (A.count==0) {
            [C addObjectsFromArray:B];
            index = i+1;
            return;
        }
        else if (B.count==0) {
            [C addObjectsFromArray:A];
            index = i+1;
            return;
        }
    }
    //(2).
    //时间复杂度
    //T(n) = O(f(n)):用"T(n)"表示，"O"为数学符号，f(n)为同数量级，一般是算法中频度最大的语句频度。
    //时间复杂度:T(n) = O(index);

```

## 12.GET和POST区别

> tcp在传输层，IP在网络层，http在应用层。。。。
> 
> GET产生一个TCP数据包；POST产生两个TCP数据包（对于GET方式的请求，浏览器会把http header和data一并发送出去，服务器响应200（返回数据）；
> 而对于POST，浏览器先发送header，服务器响应100 continue，浏览器再发送data，服务器响应200 ok（返回数据），）。。。
> GET请求在URL中传送的参数是有长度限制的，而POST没有。
> GET比POST更不安全，因为参数直接暴露在URL上，所以不能用来传递敏感信息。
> GET参数通过URL传递，POST放在Request body中。
> GET请求参数会被完整保留在浏览器历史记录里，而POST中的参数不会被保留。
> GET请求只能进行url编码，而POST支持多种编码方式。
> GET请求会被浏览器主动cache，而POST不会，除非手动设置。
> GET产生的URL地址可以被Bookmark，而POST不可以。
> GET在浏览器回退时是无害的，而POST会再次提交请求。

## 13.面向对象编程的六大原则

*   1.单一职责:

> 不要存在多于一个导致类变更的原因。通俗的说，即一个类只负责一项职责。

*   2.里氏替换原则:

> 所有引用基类的地方必须能透明地使用其子类的对象，也就是说子类可以扩展父类的功能，但不能改变父类原有的功能

*   3.依赖倒置:

> 高层模块不应该依赖低层模块，二者都应该依赖其抽象；抽象不应该依赖细节；细节应该依赖抽象。简单的说就是尽量面向接口编程.

*   4.接口隔离:

> 客户端不应该依赖它不需要的接口；一个类对另一个类的依赖应该建立在最小的接口上。接口最小化,过于臃肿的接口依据功能,可以将其拆分为多个接口.

*   5.迪米特法则:

> 一个对象应该对其他对象保持最少的了解,简单的理解就是高内聚,低耦合，一个类尽量减少对其他对象的依赖，并且这个类的方法和属性能用私有的就尽量私有化.

*   6.开闭原则:

> 一个软件实体如类、模块和函数应该对扩展开放，对修改关闭.当软件需求变化时，尽量通过扩展软件实体的行为来实现变化，而不是通过修改已有的代码来实现变化.

## 14.ios Animation

*   Core Animation

> CAAnimation作为虚基类实现了CAMediaTiming协议（其实还实现了CAAction协议）。CAAnimation有三个子类CAAnimationGroup（组动画）、CAPropertyAnimation（属性动画）、CATrasition（渐变动画）。CAAnimation不能直接使用，应该使用它的子类。作为CAPropertyAnimation也有两个子类CABasicAnimation（基础动画）、CAKeyFrameAnimation（关键帧动画）。CAPropertyAnimation也不能直接使用，应该使用两个子类。综上所诉要使用核心动画，可以使用的就是以下四个类（CAAnimationGroup、CATrasition、CABasicAnimation、CAKeyFrameAnimation）。

```
1.CABasicAnimation简单使用
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    UITouch *touch = [touches anyObject];
    CGPoint point = [touch locationInView:self.view];
    CABasicAnimation *positionAnimation = [CABasicAnimation animationWithKeyPath:@"position"];
    positionAnimation.fromValue = [NSValue valueWithCGPoint:self.testLayer.presentationLayer.position];
    positionAnimation.toValue = [NSValue valueWithCGPoint:point];
    positionAnimation.duration = 1.f;//动画时长
    positionAnimation.removedOnCompletion = NO;//是否在完成时移除
    positionAnimation.fillMode = kCAFillModeForwards;//动画结束后是否保持状态
    [self.testLayer addAnimation:positionAnimation forKey:@"positionAnimation"];
}

2\. CATransition（过渡动画）
	CATransition *transition = [CATransition animation];
    transition.startProgress = 0;//开始进度
    transition.endProgress = 1;//结束进度
    transition.type = kCATransitionReveal;//过渡类型
    transition.subtype = kCATransitionFromLeft;//过渡方向
    transition.duration = 1.f;
    UIColor *color = [UIColor colorWithRed:arc4random_uniform(255) / 255.0 green:arc4random_uniform(255) / 255.0 blue:arc4random_uniform(255) / 255.0 alpha:1.f];
    self.testLayer.backgroundColor = color.CGColor;
    [self.testLayer addAnimation:transition forKey:@"transition"];

 3.CAKeyFrameAnimation关键帧动画
 	CAKeyframeAnimation *moveAnimation = [CAKeyframeAnimation animationWithKeyPath:@"position"];
    moveAnimation.path = bezirePath;
    moveAnimation.fillMode = kCAFillModeForwards;
    moveAnimation.removedOnCompletion = NO;
    moveAnimation.duration = 3.f;

 4.AnimationGroup（组动画)
    CAAnimationGroup *groupAnimation = [[CAAnimationGroup alloc] init];
    groupAnimation.animations = @[xScaleAnimation, yScaleAnimation];//将所有动画添加到动画组

```

## 15.为什么必须在主线程中操作UI

> 因为UIKit不是线程安全的。试想下面这几种情况：
> 两个线程同时设置同一个背景图片，那么很有可能因为当前图片被释放了两次而导致应用崩溃。
> 两个线程同时设置同一个UIView的背景颜色，那么很有可能渲染显示的是颜色A，而此时在UIView逻辑树上的背景颜色属性为B。
> 两个线程同时操作view的树形结构：在线程A中for循环遍历并操作当前View的所有subView，然后此时线程B中将某个subView直接删除，这就导致了错乱还可能导致应用崩溃。iOS4之后苹果将大部分绘图的方法和诸如 UIColor 和 UIFont 这样的类改写为了线程安全可用，但是仍然强烈建议讲UI操作保证在主线程中执行。

## 16.显式和隐式动画的区别

> 1、隐式动画一直存在 如需关闭需设置；显式动画是不存在，如需显式 要开启(创建)。
> 2、显式动画是指用户自己通过beginAnimations:context:和commitAnimations创建的动画。
> 隐式动画是指通过UIView的animateWithDuration:animations:方法创建的动画。
> 3.一种为UIView动画,又称隐式动画,动画后frame的数值发生了变化.另一种是CALayer动画,又称显示动画,UIView（隐式动画）可以相应用户交互。

## 17.OC内联函数 inline

> 优点相比于函数:
> 1.inline函数避免了普通函数的,在汇编时必须调用call的缺点:取消了函数的参数压栈，减少了调用的开销,提高效率.所以执行速度确比一般函数的执行速度要快.
> 2)集成了宏的优点,使用时直接用代码替换(像宏一样);

> 优点相比于宏:
> 1.避免了宏的缺点:需要预编译.因为inline内联函数也是函数,不需要预编译.
> 2)编译器在调用一个内联函数时，会首先检查它的参数的类型，保证调用正确。然后进行一系列的相关检查，就像对待任何一个真正的函数一样。这样就消除了它的隐患和局限性。
> 3)可以使用所在类的保护成员及私有成员。

> inline内联函数的说明
> 1.内联函数只是我们向编译器提供的申请,编译器不一定采取inline形式调用函数.
> 2.内联函数不能承载大量的代码.如果内联函数的函数体过大,编译器会自动放弃内联.
> 3.内联函数内不允许使用循环语句或开关语句.
> 4.内联函数的定义须在调用之前.

```
//数组添加非空对象
static inline void photoComponentSafeAddObject(NSMutableArray *array, id object) {
    if (array && object) {
        [array addObject:object];
    }
}

static inline CGSize liteVideoImageSizeScaleWithSize(CGSize scaleSize, CGSize pixelSize) {
    CGFloat width = pixelSize.width;
    CGFloat height = pixelSize.height;
    float verticalRadio   = scaleSize.height*1.0/height;
    float horizontalRadio = scaleSize.width*1.0/width;
    float radio = 1;
    if (verticalRadio < 1 || horizontalRadio < 1) {
        radio = verticalRadio < horizontalRadio ? verticalRadio : horizontalRadio;
    }
    width = width*radio;
    height = height*radio;
    // 返回新的改变大小后的size
    return CGSizeMake(width, height);
}
```
***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
