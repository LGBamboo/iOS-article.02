# 大厂常问iOS面试题--Runtime篇

## 1.Category 的实现原理？

*   Category 实际上是 Category_t的结构体，在运行时，新添加的方法，都被以倒序插入到原有方法列表的最前面，所以不同的Category，添加了同一个方法，执行的实际上是最后一个。

*   Category 在刚刚编译完的时候，和原来的类是分开的，只有在程序运行起来后，通过 Runtime ，Category 和原来的类才会合并到一起。

## 2.isa指针的理解，对象的isa指针指向哪里？isa指针有哪两种类型？

*   isa 等价于 is kind of

    实例对象的 isa 指向类对象

    类对象的 isa 指向元类对象

    元类对象的 isa 指向元类的基类

*   isa 有两种类型

    纯指针，指向内存地址

    NON_POINTER_ISA，除了内存地址，还存有一些其他信息

## 3.Objective-C 如何实现多重继承？

Object-c的类没有多继承,只支持单继承,如果要实现多继承的话，可使用如下几种方式间接实现

*   通过组合实现

    A和B组合，作为C类的组件

*   通过协议实现

    C类实现A和B类的协议方法

*   消息转发实现

    forwardInvocation:方法

## 4.runtime 如何实现 weak 属性？

weak 此特质表明该属性定义了一种「非拥有关系」(nonowning relationship)。为这种属性设置新值时，设置方法既不持有新值（新指向的对象），也不释放旧值（原来指向的对象）。

runtime 对注册的类，会进行内存布局，从一个粗粒度的概念上来讲，这时候会有一个 hash 表，这是一个全局表，表中是用 weak 指向的对象内存地址作为 key，用所有指向该对象的 weak 指针表作为 value。当此对象的引用计数为 0 的时候会 dealloc，假如该对象内存地址是 a，那么就会以 a 为 key，在这个 weak 表中搜索，找到所有以 a 为键的 weak 对象，从而设置为 nil。

runtime 如何实现 weak 属性具体流程大致分为 3 步：

*   1、初始化时：runtime 会调用 objc_initWeak 函数，初始化一个新的 weak 指针指向对象的地址。

*   2、添加引用时：objc_initWeak 函数会调用 objc_storeWeak() 函数，objc_storeWeak() 的作用是更新指针指向（指针可能原来指向着其他对象，这时候需要将该 weak 指针与旧对象解除绑定，会调用到 weak_unregister_no_lock），如果指针指向的新对象非空，则创建对应的弱引用表，将 weak 指针与新对象进行绑定，会调用到 weak_register_no_lock。在这个过程中，为了防止多线程中竞争冲突，会有一些锁的操作。

*   3、释放时：调用 clearDeallocating 函数，clearDeallocating 函数首先根据对象地址获取所有 weak 指针地址的数组，然后遍历这个数组把其中的数据设为 nil，最后把这个 entry 从 weak 表中删除，最后清理对象的记录。

## 5.讲一下 OC 的消息机制

*   OC中的方法调用其实都是转成了objc_msgSend函数的调用，给receiver（方法调用者）发送了一条消息（selector方法名）

*   objc_msgSend底层有3大阶段，消息发送（当前类、父类中查找）、动态方法解析、消息转发

## 6.runtime具体应用

*   利用关联对象（AssociatedObject）给分类添加属性

*   遍历类的所有成员变量（修改textfield的占位文字颜色、字典转模型、自动归档解档）

*   交换方法实现（交换系统的方法）

*   利用消息转发机制解决方法找不到的异常问题

*   KVC 字典转模型

## 7.runtime如何通过selector找到对应的IMP地址？

每一个类对象中都一个对象方法列表（对象方法缓存）

*   类方法列表是存放在类对象中isa指针指向的元类对象中（类方法缓存）。

*   方法列表中每个方法结构体中记录着方法的名称,方法实现,以及参数类型，其实selector本质就是方法名称,通过这个方法名称就可以在方法列表中找到对应的方法实现。

*   当我们发送一个消息给一个NSObject对象时，这条消息会在对象的类对象方法列表里查找。

*   当我们发送一个消息给一个类时，这条消息会在类的Meta Class对象的方法列表里查找。

## 8.简述下Objective-C中调用方法的过程

Objective-C是动态语言，每个方法在运行时会被动态转为消息发送，即：objc_msgSend(receiver, selector)，整个过程介绍如下：

*   objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类

*   然后在该类中的方法列表以及其父类方法列表中寻找方法运行

*   如果，在最顶层的父类（一般也就NSObject）中依然找不到相应的方法时，程序在运行时会挂掉并抛出异常unrecognized selector sent to XXX

*   但是在这之前，objc的运行时会给出三次拯救程序崩溃的机会，这三次拯救程序奔溃的说明见问题《什么时候会报unrecognized selector的异常》中的说明。

## 9.load和initialize的区别

两者都会自动调用父类的，不需要super操作，且仅会调用一次（不包括外部显示调用).

*   load和initialize方法都会在实例化对象之前调用，以main函数为分水岭，前者在main函数之前调用，后者在之后调用。这两个方法会被自动调用，不能手动调用它们。

*   load和initialize方法都不用显示的调用父类的方法而是自动调用，即使子类没有initialize方法也会调用父类的方法，而load方法则不会调用父类。

*   load方法通常用来进行Method Swizzle，initialize方法一般用于初始化全局变量或静态变量。

*   load和initialize方法内部使用了锁，因此它们是线程安全的。实现时要尽可能保持简单，避免阻塞线程，不要再使用锁。

## 10.怎么理解Objective-C是动态运行时语言。

*   主要是将数据类型的确定由编译时,推迟到了运行时。这个问题其实浅涉及到两个概念,运行时和多态。

*   简单来说, 运行时机制使我们直到运行时才去决定一个对象的类别,以及调用该类别对象指定方法。

*   多态:不同对象以自己的方式响应相同的消息的能力叫做多态。

*   意思就是假设生物类(life)都拥有一个相同的方法-eat;那人类属于生物,猪也属于生物,都继承了life后,实现各自的eat,但是调用是我们只需调用各自的eat方法。也就是不同的对象以自己的方式响应了相同的消 息(响应了eat这个选择器)。因此也可以说,运行时机制是多态的基础.

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
