# 阿里、字节iOS面试题之Runtime相关问题1（附答案）

### 目录

* [2020 阿里、字节iOS面试题之Runtime相关问题1](https://www.jianshu.com/p/7f94db2e5928)
* [2020 阿里、字节iOS面试题之Runtime相关问题2](https://www.jianshu.com/p/f2331034f0ab)
* [2020 阿里、字节iOS面试题之Runtime相关问题3](https://www.jianshu.com/p/4e507f9f9f04)


# 面试题的结构分类和细化

*   **runtime相关问题**
    1.  runtime结构模型
    2.  内存管理
    3.  关联属性或者hook相关的Method Swizzle

*   **NSNotification相关**
    1.  参考GNUStep源码
    2.  NSNotification实现原理 相关

*   **Runloop & KVO**
    1.  runloop
    2.  KVO

*   **Block**
    1.  Block实现原理和注意事项相关

*   **多线程**
    1.  GCD相关和一些多线程概念

*   **视图&图像相关**
    1.  视图UI布局方案
    2.  视图渲染相关

*   **性能优化**

*   **开发证书**

*   **架构设计**
    1.  各种设计模式
    2.  自己的设计

*   **其他问题**
    1.  方法调用和切面编程等

*   **系统基础知识**

*   **数据结构与算法**

## runtime相关问题

[objc-runtime源码地址](https://github.com/RetVal/objc-runtime)
[objc4官方源码地址](https://opensource.apple.com/tarballs/objc4/)

### 结构模型

#### 介绍下runtime的内存模型（isa、对象、类、metaclass、结构体的存储信息等）

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
***

##### 对象

OC中的对象指向的是一个`objc_object`指针类型，`typedef struct objc_object *id;`从它的结构体中可以看出，它包括一个isa指针，指向的是这个对象的类对象,一个对象实例就是通过这个isa找到它自己的Class，而这个Class中存储的就是这个实例的方法列表、属性列表、成员变量列表等相关信息的。

```
/// Represents an instance of a class.
struct objc_object {
 Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;
```

这个objc_object 的实现比较长 在这里[查看](https://github.com/RetVal/objc-runtime/blob/master/runtime/objc-private.h)

#### 类

在OC中的类是用Class来表示的，实际上它指向的是一个`objc_class`的指针类型，`typedef struct objc_class *Class;`
对应的结构体如下：

```
struct objc_class {
 Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
 Class _Nullable super_class                              OBJC2_UNAVAILABLE;
 const char * _Nonnull name                               OBJC2_UNAVAILABLE;
 long version                                             OBJC2_UNAVAILABLE;
 long info                                                OBJC2_UNAVAILABLE;
 long instance_size                                       OBJC2_UNAVAILABLE;
 struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
 struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
 struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
 struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

}
```

##### class和 object 小结

从结构体中定义的变量可知，OC的`Class`类型包括如下

数据（即：元数据`metadata`）：`super_class`（父类类对象）;
name（类对象的名称）;
version、info（版本和相关信息）;
instance_size（实例内存大小）;
ivars（实例变量列表）；
methodLists（方法列表）；
cache（缓存）；
protocols（实现的协议列表）;
当然也包括一个isa指针，这说明Class也是一个对象类型，所以我们称之为类对象， 这里的isa指向的是元类对象（metaclass），元类中保存了创建类对象（Class）的类方法的全部信息。

[![Objective-C的对象原型继承链](https://upload-images.jianshu.io/upload_images/13277235-36a257745b6ff11c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/class_inherit.png) 

从图中可知，最终的基类`NSObject`的元类对象`isa`指向的是自己本身，从而形成一个闭环。
元类（`Meta Class`）：是一个类对象的类，即：Class的类，这里保存了类方法等相关信息。
我们再看一下类对象中存储的方法、属性、成员变量等信息的结构体
`objc_ivar_list`：存储了类的成员变量，
可以通过`object_getIvar`或`class_copyIvarList`获取；
另外这两个方法是用来获取类的属性列表的`class_getProperty`和`class_copyPropertyList`，属性和成员变量是有区别的。

```
struct objc_ivar {
 char * _Nullable ivar_name                               OBJC2_UNAVAILABLE;
 char * _Nullable ivar_type                               OBJC2_UNAVAILABLE;
 int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
 int space                                                OBJC2_UNAVAILABLE;
#endif
}                                                            OBJC2_UNAVAILABLE;

struct objc_ivar_list {
 int ivar_count                                           OBJC2_UNAVAILABLE;
#ifdef __LP64__
 int space                                                OBJC2_UNAVAILABLE;
#endif
 /* variable length structure */
 struct objc_ivar ivar_list[1]                            OBJC2_UNAVAILABLE;
}
```

`objc_method_list`：存储了类的方法列表，可以通过`class_copyMethodList`获取。

结构体如下:

```
struct objc_method {
 SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
 char * _Nullable method_types                            OBJC2_UNAVAILABLE;
 IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;

struct objc_method_list {
 struct objc_method_list * _Nullable obsolete             OBJC2_UNAVAILABLE;

 int method_count                                         OBJC2_UNAVAILABLE;
#ifdef __LP64__
 int space                                                OBJC2_UNAVAILABLE;
#endif
 /* variable length structure */
 struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
}
```

`objc_protocol_list`：储存了类的协议列表，可以通过`class_copyProtocolList`获取。

结构体如下：

```
struct objc_protocol_list {
 struct objc_protocol_list * _Nullable next;
 long count;
 __unsafe_unretained Protocol * _Nullable list[1];
};
```
此问题参考[介绍下runtime的内存模型（isa、对象、类、metaclass、结构体的存储信息等）](https://developer.aliyun.com/ask/282811)

#### 为什么要设计metaclass?

先说结论: 为了更好的**复用传递消息**.metaclass只是需要**实现复用消息传递**为目的工具.而Objective-C所有的类默认都是同一个MetaClass(通过isa指针最终指向metaclass). 因为Objective-C的特性基本上是照搬的Smalltalk,Smalltalk中的MetaClass的设计是Smalltalk-80加入的.所以Objective-C也就有了metaclass的设计.

> 本质上因为Smalltalk的面向对象的亮点是它的**消息发送机制**.

回答这个问题之前我们先回看一下上边的Objective-C的对象原型继承链[![Objective-C的对象原型继承链](https://upload-images.jianshu.io/upload_images/13277235-d7fd4cd6cffceb0f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/class_inherit2.jpg) 

通过上图我们明白如下 重点内容:

*   **实例的实例方法函数存在类结构体中**
*   **类方法函数存在metaclass结构体中**

而Objective-C的方法调用（消息）就会根据对象去找isa指针指向的Class对象中的方法列表找到对应的方法。 > isa 指向的类就是我们创建实例的类型.

通过[为什么Objective-C中有MetaClass这个设计？](https://www.jianshu.com/p/c1793bc2ca13)文章我们了解到一个十分重要的概念,python和**Objective-C不太一样的是,并不是每一个类都有一个MetaClass,而是Objective-C所有的类默认都是同一个MetaClass.**

##### Smalltalk中的metaclass

Smalltalk，被公认为历史上第二个面向对象的语言，其亮点是它的**消息发送机制**。
Smalltalk中的MetaClass的设计是Smalltalk-80加入的。而之前的Smalltalk-76，并不是每个类有一个MetaClass，而是所有类的isa指针都指向一个特殊的类，叫做Class(这种设计之后也被Java借鉴了）。
而每个类都有自己MetaClass的设计，加入的原因是，因为Smalltalk里面，类是对象，而对象就可以响应消息，那么类的消息的响应的方法就应该由类的类去存储，而每个MetaClass就持有每个类的类方法。

###### 每个MetaClass的isa指针指向什么？

如果MetaClass再有MetaClass，那么这个关系将无穷无尽。Smalltalk里的解决方案是，指向同一个叫MetaClass的类。

###### MetaClass的isa指针指向什么？

指向他的实例，也就是实例的isa指向MetaClass，同时MetaClassisa指向实例，相互指着。

那么Smalltalk的继承关系，其实和Objective-C的很像了（后面有class的是前者的MetaClass）。

[![](https://upload-images.jianshu.io/upload_images/13277235-5ea051d1aa043b59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/class_inherit2_smaltalk.png) 

###### 这时候产生了一个重要的问题，假如去掉MetaClass，把类方法放到也类里面是否可行？

这个问题，我思索许久，发现其实是一个对面向对象的哲学思想问题，要对这个问题下结论，不得不重新讲讲面向对象

##### 从Smalltalk重新认识面向对象

以前谈到面向对象，总会提到，面向对象三特征：封装、继承、多态。但其实，面向对象中也分流派，如C++这种来自Simula的设计思想的，更注重的是类的划分，因为方法调用是静态的。而如Objective-C这种借鉴Smalltalk的，更注重的是消息传递，是动态响应消息。

而面向对象三种特征，更基于的是类的划分而提出的。

这两种思想最大的不同，我认为是自上而下和自下而上的思考方式。

*   类的划分，要求类的设计者是以一个很高的层次去设计这个类，提取出类的特性和本质，进行类的构建。知道类型才可以去发送消息给对象。
*   消息传递，要求的是类的设计者以消息为起点去构建类，也就是对外界的变化进行响应，而不关心自身的类型，设计接口。尝试理解消息，无法处理则进行特殊处理。 在此不讨论两种方式的优劣之分，而着重讲讲Smalltalk这种设计。

消息传递对于面向对象的设计，其实在于给出一种对消息的解决方案。而面向对象优点之一的复用，在这种设计里，更多在于复用解决方案，而不是单纯的类本身。这种思想就如设计组件一般，关心接口，关心组合而非类本身。其实之所以有MetaClass这种设计，我的理解并不是先有MetaClass，而是在万物都是对象的Smalltalk里，向对象发送消息的基本解决方案是统一的，希望复用的。而实例和类之间用的这一套通过isa指针指向的Class单例中存储方法列表和查询方法的解决方案的流程，是应该在类上复用的，而MetaClass就顺理成章出现罢了。

##### 为什么要设计metaclass小结

###### 回到一开始那个问题，为什么要设计MetaClass，去掉把类方法放到类里面行不行？

我的理解是，可以，但不Smalltalk。这样的设计是C++那种自上而下的设计方式，类方法也是类的一种特征描述。而Smalltalk的精髓正在于消息传递，复用消息传递才是根本目的，而MetaClass只不过是因此需要的一个工具罢了。

参考[为什么Objective-C中有MetaClass这个设计？](https://www.jianshu.com/p/c1793bc2ca13)

#### **class_copyIvarList()** & **class_copyPropertyList()**区别

先说结论:

*   **class_copyIvarList()** 能获取到所有的成员变量,包括 花括号内的变量(`.h`和`.m`都包括).
*   **class_copyPropertyList()** 只能获取到 以`@property`关键字 声明的中属性(`.h`和`.m`都包括)

区别:

*   `class_copyIvarList()`获取默认是带下划线的变量
*   `class_copyPropertyList()`获取默认是不带下划线的变量名称.

> 但是以上两个方法都只能获取到当前类的属性和变量（也就是说获取不到父类的属性和变量）

* * *

举例说明:

我们声明一个`ClassA` 通过 调试代码实现

```
#import <Foundation/Foundation.h>
#import <objc/runtime.h>

@interface ClassA : NSObject {
 int _a;
 int _b;
 int _c;
 CGFloat d; //不推荐这样写
}

@property (nonatomic, strong) NSArray          *arrayA;
@property (nonatomic, copy  ) NSString         *stringA;
@property (nonatomic, assign) dispatch_queue_t testQueue;

@end

@implementation ClassA
@end
```

如果是通过`class_copyIvarList()`函数获取则打印如下结果.

```
--- class_copyIvarList ↓↓↓---
_a
_b
_c
d
_arrayA
_stringA
_testQueue
--------------END----------------
```

如果是通过`class_copyPropertyList()`函数获取则打印如下结果.

```
--- class_copyPropertyList ↓↓↓---
arrayA
stringA
testQueue
--------------END----------------
```

debug代码如下:

```
- (void)printIvarOrProperty {
 NSLog(@"--- class_copyPropertyList ↓↓↓---");
 ClassA *classA = [[ClassA alloc] init];
 unsigned int propertyCount;
 objc_property_t *result = class_copyPropertyList(object_getClass(classA), &propertyCount);
 for (unsigned int i = 0; i < propertyCount; i++) {
     objc_property_t objc_property_name = result[i];
     NSLog(@"%@",[NSString stringWithFormat:@"%s", property_getName(objc_property_name)]);
 }
 free(result);
 NSLog(@"--------------END----------------");
 NSLog(@"--- class_copyIvarList ↓↓↓---");
 Ivar *iv = class_copyIvarList(object_getClass(classA), &propertyCount);
 for (unsigned int i = 0; i < propertyCount; i++) {
     Ivar ivar = iv[i];
     NSLog(@"%@",[NSString stringWithFormat:@"%s", ivar_getName(ivar)]);
 }
 free(iv);
 NSLog(@"--------------END----------------");
}
```

以上[demo点击这里下载](https://github.com/sunyazhou13/IvarAndPropertyDemo)

* * *

下面我们看下[objc的源码](https://github.com/sunyazhou13/objc-runtime)

以下代码位于`objc-runtime-new.mm`中

```
/***********************************************************************
* class_copyPropertyList. Returns a heap block containing the 
* properties declared in the class, or nil if the class 
* declares no properties. Caller must free the block.
* Does not copy any superclass's properties.
* Locking: read-locks runtimeLock
**********************************************************************/
objc_property_t *
class_copyPropertyList(Class cls, unsigned int *outCount)
{
 if (!cls) {
     if (outCount) *outCount = 0;
     return nil;
 }

 mutex_locker_t lock(runtimeLock);

 checkIsKnownClass(cls);
 ASSERT(cls->isRealized());

 auto rw = cls->data();

 property_t **result = nil;
 unsigned int count = rw->properties.count();
 if (count > 0) {
     result = (property_t **)malloc((count + 1) * sizeof(property_t *));

     count = 0;
     for (auto& prop : rw->properties) {
         result[count++] = &prop;
 }
     result[count] = nil;
 }

 if (outCount) *outCount = count;
 return (objc_property_t *)result;
}
```

通过源码我们可以看到

```
auto rw = cls->data();
rw->properties; //通过rw直接拿到properties
```

通过rw直接拿到properties,然后便利拿出想要的 以`@property`关键字 声明变量名称.

`properties`详细内容 还请异步运行时源码看下这里篇幅限制就不啰嗦了.

* * *

```
/***********************************************************************
* class_copyIvarList
* fixme
* Locking: read-locks runtimeLock
**********************************************************************/
Ivar *
class_copyIvarList(Class cls, unsigned int *outCount)
{
 const ivar_list_t *ivars;
 Ivar *result = nil;
 unsigned int count = 0;

 if (!cls) {
     if (outCount) *outCount = 0;
     return nil;
 }

 mutex_locker_t lock(runtimeLock);

 ASSERT(cls->isRealized());

 if ((ivars = cls->data()->ro->ivars)  &&  ivars->count) {
     result = (Ivar *)malloc((ivars->count+1) * sizeof(Ivar));

     for (auto& ivar : *ivars) {
         if (!ivar.offset) continue;  // anonymous bitfield
         result[count++] = &ivar;
 }
     result[count] = nil;
 }

 if (outCount) *outCount = count;
 return result;
}
```
这里就一个关键点

```
ivars = cls->data()->ro->ivars
```

拿到ivars.

由于这两者拿到的成员不一样所以两个API就会有区别.

#### `class_rw_t` 和 `class_ro_t` 的区别

先说结论:

*   两个结构体都存放着当前类的属性、实例变量、方法、协议等.
*   `class_ro_t`存放的是编译期间就确定的.
*   而`class_rw_t`是在runtime时才确定，它会先将`class_ro_t`的内容拷贝过去，然后再将当前类的分类的这些属性、方法等拷贝到其中。所以可以说`class_rw_t`是`class_ro_t`的超集，当然实际访问类的方法、属性等也都是访问的`class_rw_t`中的内容.

* * *

##### 下面我来深入了解两者具体是什么

首先我们需要了解它俩的由来,在`objc_class`我们知道有一个成员变量叫`isa`,我们这里要介绍的是`objc_class`的另一成员变量`bits`.

`objc_class`的结构如下:

[![objc_class的结构](https://upload-images.jianshu.io/upload_images/13277235-aa0b11f641fa554d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/objc_class_struct.png) 

`bits` 用来存储类的属性，方法，协议等信息。它是一个`class_data_bits_t`类型

`class_data_bits_t` 如下:
 
```
struct class_data_bits_t {
     uintptr_t bits;
     // method here
}
```

这个结构体只有一个`64bit`的成员变量`bits`，先来看看这`64bit`分别存放的什么信息：

[![](https://upload-images.jianshu.io/upload_images/13277235-0645dd043991b95d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/objc_class_bits.png) 

*   `is_swift` : 第一个bit，判断类是否是Swift类
*   `has_default_rr` ：第二个bit，判断当前类或者父类含有默认的`retain/release/autorelease/retainCount/_tryRetain/_isDeallocating/retainWeakReference/allowsWeakReference` 方法
*   `require_raw_isa` ：第三个bit， 判断当前类的实例是否需要`raw_isa`
*   `data` : 第4-48位，存放一个指向class_rw_t结构体的指针，该结构体包含了该类的属性，方法，协议等信息。至于为何只用44bit来存放地址

##### `class_rw_t` 和`class_ro_t`

先来看看两个结构体的内部成员变量

```
struct class_rw_t {
     uint32_t flags;
     uint32_t version;

     const class_ro_t *ro;

     method_array_t methods;
     property_array_t properties;
     protocol_array_t protocols;

     Class firstSubclass;
     Class nextSiblingClass;
};
```

```
struct class_ro_t {
     uint32_t flags;
     uint32_t instanceStart;
     uint32_t instanceSize;
     uint32_t reserved;

     const uint8_t * ivarLayout;

     const char * name;
     method_list_t * baseMethodList;
     protocol_list_t * baseProtocols;
     const ivar_list_t * ivars;

     const uint8_t * weakIvarLayout;
     property_list_t *baseProperties;
};
```

`class_rw_t`结构体内有一个指向`class_ro_t`结构体的指针.

每个类都对应有一个`class_ro_t`结构体和一个`class_rw_t`结构体。在编译期间，`class_ro_t`结构体就已经确定，`objc_class`中的`bits`的`data`部分存放着该结构体的地址。在`runtime`运行之后，具体说来是在运行`runtime`的`realizeClass` 方法时，会生成`class_rw_t`结构体，该结构体包含了`class_ro_t`，并且更新`data`部分，换成`class_rw_t`结构体的地址。

用两张图来说明这个过程：

类的`realizeClass`运行之前：
[![](https://upload-images.jianshu.io/upload_images/13277235-5540d513f9d517e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/before_bits.png) 

类的`realizeClass`运行之后：

[![](https://upload-images.jianshu.io/upload_images/13277235-a5e0786aaace5131.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/after_bits.png) 

细看两个结构体的成员变量会发现很多相同的地方，他们都存放着当前类的属性、实例变量、方法、协议等等。区别在于：`class_ro_t`存放的是编译期间就确定的；而`class_rw_t`是在`runtime`时才确定，它会先将`class_ro_t`的内容拷贝过去，然后再将当前类的分类的这些属性、方法等拷贝到其中。所以可以说`class_rw_t`是`class_ro_t`的超集，当然实际访问类的方法、属性等也都是访问的`class_rw_t`中的内容

属性(property)存放在`class_rw_t`中，实例变量(ivar)存放在`class_ro_t`中。

详细内容请 参考资料[Objective-C runtime - 属性与方法](http://vanney9.com/2017/06/05/objective-c-runtime-property-method/)

#### category如何被加载的,两个category的load方法的加载顺序，两个category的同名方法的加载顺序

结论:

1.  category 是 这样 `realizeClass` -> `methodizeClass()` -> `attachCategories()` 一步步被加载的.
2.  主类与分类的加载顺序是:**主类优先于分类加载,无关编译顺序**.
3.  分类间的加载顺序取决于编译的顺序:**编译在前则先加载,编译在后则后加载**.

* * *

##### category如何被加载的

我在运行时的源码 `objc-runtime-new.mm`中找到如下:

```
static Class realizeClassWithoutSwift(Class cls, Class previously)
{
         ...
         // Attach categories  被加载
         methodizeClass(cls, previously);
         return cls;
}
```

`realizeClass` -> `methodizeClass()` -> `attachCategories()`

核心是在methodizeClass()函数中实现的.

```
static void methodizeClass(Class cls)
{
     runtimeLock.assertLocked();
     bool isMeta = cls->isMetaClass();
     auto rw = cls->data();
     auto ro = rw->ro;
     ...
     property_list_t *proplist = ro->baseProperties;
     if (proplist) {
         rw->properties.attachLists(&proplist, 1);
     }
     ...
     // Attach categories.
     category_list *cats =unattachedCategoriesForClass(cls, true /*realizing*/);
     attachCategories(cls, cats, false /*don't flush caches*/);
     ... 
     if (cats) free(cats);

}
```

通过上述代码我们发现`ro->baseProperties;` , baseProperties 在前，category 在后,

```
property_list_t *proplist = ro->baseProperties;
if (proplist) {
  rw->properties.attachLists(&proplist, 1);
}
```
但决定顺序的是 rw->`properties.attachLists ()`这个方法.

```
/// category 被附加进去
void attachLists(List* const * addedLists, uint32_t addedCount) {
     if (addedCount == 0) return;
     if (hasArray()) {
         // many lists -> many lists
         uint32_t oldCount = array()->count;
         uint32_t newCount = oldCount +addedCount;
         setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
         array()->count = newCount;
 // 将旧内容移动偏移量 addedCount 然后将 addedLists copy 到起始位置
 /*
     struct array_t {
         uint32_t count;
         List* lists[0];
         };
 */
     memmove(array()->lists + addedCount,array()->lists, 
             oldCount * sizeof(array()->lists[0]));
     memcpy(array()->lists, addedLists, 
             addedCount * sizeof(array()->lists[0]));
 }
 else if (!list  &&  addedCount == 1) {
     // 0 lists -> 1 list
     list = addedLists[0];
 } 
 else {
     // 1 list -> many lists
     List* oldList = list;
     uint32_t oldCount = oldList ? 1 : 0;
     uint32_t newCount = oldCount + addedCount;
     setArray((array_t*)malloc(array_t::byteSize(newCount)));
     array()->count = newCount;
     if (oldList) array()->lists[addedCount] = oldList;
     memcpy(array()->lists, addedLists, 
     addedCount * sizeof(array()->lists[0]));
 }
}
```

所以 category 的属性总是在前面的，baseClass的属性被往后偏移了。

##### 两个category的load方法的加载顺序

```
A class’s +load method is called after all of its superclasses’ +load methods.
一个类的+load方法在其父类的+load方法后调用

A category +load method is called after the class’s own +load method.
一个Category的+load方法在被其扩展的类的自有+load方法后调用
```

结论: 主类与分类的加载顺序是:**主类优先于分类加载,无关编译顺序**.

##### 两个category的同名方法的加载顺序

应用程序 image 镜像加载到内存中时， `Category` 解析的过程，注意下面的 `while(i--)` 循环 这里倒序将 `category` 中的协议 方法 属性添加到了`rw = cls->data()`中的 `methods/properties/protocols`中。

```
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
     if (!cats) return;
     if (PrintReplacedMethods) printReplacements(cls, cats);

     bool isMeta = cls->isMetaClass();

     // fixme rearrange to remove these intermediate allocations
     method_list_t **mlists = (method_list_t **)
         malloc(cats->count * sizeof(*mlists));
     property_list_t **proplists = (property_list_t **)
         malloc(cats->count * sizeof(*proplists));
     protocol_list_t **protolists = (protocol_list_t **)
         malloc(cats->count * sizeof(*protolists));

 // Count backwards through cats to get newest categories first
 int mcount = 0;
 int propcount = 0;
 int protocount = 0;
 int i = cats->count;
 bool fromBundle = NO;
 while (i--) {
     auto& entry = cats->list[i];

     method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
     if (mlist) {
         mlists[mcount++] = mlist;
         fromBundle |= entry.hi->isBundle();
     }

     property_list_t *proplist = 
         entry.cat->propertiesForMeta(isMeta, entry.hi);
     if (proplist) {
         proplists[propcount++] = proplist;
     }

     protocol_list_t *protolist = entry.cat->protocols;
     if (protolist) {
         protolists[protocount++] = protolist;
     }
 }
 auto rw = cls->data();

 // 注意下面的代码，上面采用倒叙遍历方式，所以后编译的 category 会先add到数组的前部
 prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
 rw->methods.attachLists(mlists, mcount);
 free(mlists);
 if (flush_caches  &&  mcount > 0) flushCaches(cls);

 rw->properties.attachLists(proplists, propcount);
 free(proplists);

 rw->protocols.attachLists(protolists, protocount);
 free(protolists);
}
```

所以结论是:分类间的加载顺序取决于编译的顺序:编译在前则先加载,编译在后则后加载

这个问题网上有很多例子 就不多在这举例了.

#### `category` & `extension`区别，能给NSObject添加Extension吗，结果如何

##### `category`

*   运行时添加分类属性/协议/方法
*   分类添加的方法会“覆盖”原类方法，因为方法查找的话是从头至尾，一旦查找到了就停止了
*   同名分类方法谁生效取决于编译顺序，image 读取的信息是倒叙的，所以编译越靠后的越先读入
*   名字相同的分类会引起编译报错；

##### `extension`

*   编译时决议
*   只以声明的形式存在，多数情况下就存在于 .m 文件中；
*   不能为系统类添加扩展

可以给类添加成员变量，但是是私有的 可以給类添加方法，但是是私有的 添加的属性和方法是类的一部分，在编译期就决定的。在编译器和头文件的@interface和实现文件里的@implement一起形成了一个完整的类。 伴随着类的产生而产生，也随着类的消失而消失

> **必须有类的源码才可以给类添加extension**!!!

##### `category` & `extension`区别

*   Category的小括号中有名字,而Extension没有;
*   Category只能扩充方法,不能扩充成员变量和属性;
*   如果Category声明了声明了一个属性,那么Category只会生成这个属性的set,get方法的声明,也就不是会实现.所以对于系统一些类，如nsstring，就无法添加类扩展 不能给NSObject添加Extension，因为在extension中添加的方法或属性必须在源类的文件的.m文件中实现才可以，即：你必须有一个类的源码才能添加一个类的`extension`

##### 能给NSObject添加Extension吗，结果如何?

不能 因为没有NSObject的.m源码文件.

> 如果能的话那应该不叫Extension.或者我们自己通过运行时的api自己造一套ExtensionDIY.结果就是你用的根本不能称为`Extension`,而是api调用而已.

#### 消息转发机制，消息转发机制和其他语言的消息机制优劣对比

> 前言: 了解消息转发之前我们有必要了解一些Objectivce-C中的消息传递机制

##### 消息传递机制

在Objectivce-C中,我们通过`实例变量(对象)`或者`类方法名`调用一个方法,那么我们实际上是在发送一条消息

```
id returnValue = [someObject messageName:parameter];  //实例调用方式
id returnValue = [ClassA messageName:parameter];  //类调用方式
```

上述`someObject`和`ClassA`是接受者(receiver)，`messageName:`是选择器(`selector`),选择器和参数合起来称为消息(`message`)。编译器看到此消息后，将其转换为一条标准的c语言函数调用，所调用的函数乃是消息传递机制中的核心函数：`objc_msgSend()`。

```
void objc_msgSend(id self, SEL cmd, ...)
```
第一个参数代表接受者，第二个参数代表选择子，后续参数就是消息中的那些参数 编译器会把刚才的那个例子中的消息转换为如下函数：

```
id returnValue = objc_msgSend(someObject, @selector(messageName:),parameter);
id returnValue = objc_msgSend(ClassA, @selector(messageName:),parameter);
```

`objc_msgSend()`函数会依据接受者与选择器的类型来调用适当的方法.为来完成此操作，该方法需要在接受者所属的类中搜寻其“方法列表”(也就是上文我们说的`class_ro_t`中的method_list)。找到则跳到现实代码，否则，就沿着继承体系继续向上查找，如果还没有则执行消息转发操作。对于其他的“边界情况”，则需要交由Objective-c运行环境的另一些函数来处理：

```
objc_msgSend_stret  //待发送的消息返回结构体时
objc_msgSend_fpret  //消息返回的是浮点型
objc_msgSendSuper   //如果要给超类发送消息
```

##### 消息转发机制

结合上边的消息传递机制,在Objective-C中如果给一个对象发送一条它无法处理的消息，就会进入下图描述的消息转发(Message Forwarding)流程

[![](https://upload-images.jianshu.io/upload_images/13277235-1d51c512eecb61a5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200721iOSinterviewAnswers/methodforward.jpg) 

在objc中消息转发需要经历3个阶段 `resolveInstanceMethod` -> `forwardingTargetForSelectoer` -> `forwardInvocation` ->`消息未能处理`。

*   第一阶段:**动态方法解析(Dynamic Method Resolution)**也就是在所属的类中先征询接受者,看其是否能动态加方法，来处理当前这个**未知选择器**
*   第二阶段:**替换消息接收者快速转发**
*   第三阶段:**完全消息转发机制**

##### 第一阶段:**动态方法解析(Dynamic Method Resolution)**

对象在受到无法解读的消息后，首先将调用其所属类的下列类方法:

```
+ (BOOL)resolveClassMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
+ (BOOL)resolveInstanceMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
```

> 这俩方法在NSObject.h中

返回一个`Boolean`类型，表示这个类是否能新增一个实例方法以处理选择器.

在 消息转发过程中,我们可以使用`resolveInstanceMethod:`动态的将一个方法添加到一个类中.

例下面示例代码:

```
@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
 if (aSEL == @selector(resolveThisMethodDynamically)) {
 class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
 return YES;
 }
 return [super resolveInstanceMethod:aSEL];
}
@end
```

这里我们用到一个运行时函数`class_addMethod()`.

```
{
 if (!cls) return NO;

 mutex_locker_t lock(runtimeLock);
 return ! addMethod(cls, name, imp, types ?: "", NO);
}
```

*   `class_addMethod()`最后一个参数叫做`types`，是一个描述方法的参数类型的字符串.
*   `v`代表`void`
*   `@`代表对象或者说`id类型`
*   `:`(这个冒号)代表方法选择器SEL

具体代表什么不是我们瞎写的,得按照苹果的这个标准 [Objective-C Runtime Programming Guide->Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)

上面的`dynamicMethodIMP`，返回值是`void`，两个入参分别是`id`和`SEL`，所以描述这个方法的参数类型的字符串就是`v@:`

这个阶段的意义是为一个类动态提供方法实现,严格来说，还没进入消息转发流程。

`resolveInstanceMethod:` 控制这下面两个方法是否会被调用

*   `respondsToSelector:`

*   `instancesRespondToSelector:`

> 也就是说，如果`resolveInstanceMethod:`返回了`YES`，那么`respondsToSelector:`和`instancesRespondToSelector:`都会返回`YES`.

##### 第二阶段：替换消息接收者(快速转发)

如果第一阶段中`resolveInstanceMethod:`返回NO,就会调用`forwardingTargetForSelector:`询问是否把消息转发给另一个对象.消息的接收者就改变了。

```
- (id)forwardingTargetForSelector:(SEL)aSelector {
 return someOtherObject;
}
```

##### 第三阶段：完全消息转发机制

如果第二阶段的`forwardingTargetForSelector:`返回了`nil`，这就进入了所谓完全消息转发的机制。

首先调用`methodSignatureForSelector:`为要转发的消息返回正确的签名：

```
- (void)forwardInvocation:(NSInvocation *)anInvocation {
     NSLog(@"forwardInvocation");
     SomeOtherObject *someOtherObject = [SomeOtherObject new];
     if ([someOtherObject respondsToSelector:[anInvocation selector]]) {
         [anInvocation invokeWithTarget:someOtherObject];
     } else {
         [super forwardInvocation:anInvocation];
     }
}
```

上面代码是将消息转发给其他对象，其实这与第二阶段中示例代码做的事情是一样的。区别就在于这个阶段会有一个`NSInvocation`对象。`NSInvocation`是一个用来存储和转发消息的对象。它包含了一个Objective-C消息的所有元素：一个target，一个selector，参数和返回值。每个元素都可以被直接设置。

> `NSInvocation`可以简单理解为一个对象把我们用到 selector方法和对象都存储了一下,然后哪个是指向我们需要调用的指针对象.

所以不同与第二阶段，在这个阶段你可以：

*   把消息存储，在你觉得合适的时机转发出去，或者不处理这个消息。
*   修改消息的target，selector，参数等
*   多次转发这个消息，转发给多个对象

显然在这个阶段，你可以对一个OC消息做更多的事情

* * *

##### 消息转发机制和其他语言的消息机制优劣对比

这个目前没有深入其它编程语言的运行时层面,比如C的底层或者C++的底层或者Java的底层消息传递

#### 在方法调用的时候，方法查询-> 动态解析-> 消息转发 之前做了什么

Objective-C 实例对象执行方法步骤

1.  获取 receiver 对应的类 Class
2.  在 Class 缓存列表中(就是`objc_class`里的`cache_t`到`class_ro_t`的方法list)根据选择子`selector`查找`IMP`
3.  若缓存中没有找到，则在方法列表中继续查找.
4.  若方法列表没有，则从父类查找，重复以上步骤.
5.  若最终没有找到，则进行消息转发操作.

*   方法查询之前 要知道 receiver和 selector.主要是要明确我们是哪个实例调用了哪个方法.

*   动态解析解析之前要 在所属的类中先征询接受者,看其是否能动态加方法，来处理当前这个未知选择器.
*   消息转发 之前 要询问是否把消息转发给另一个对象.

> 如果更深入的而理解 那应该是 objc_msgSend() 为啥是汇编实现的,上面的那些方法 调用之前 汇编的哪些指令被执行

这里找到两篇文章可以参考一下
[深入了解Objective-C消息发送与转发过程](https://chipengliu.github.io/2019/06/02/objc-msgSend-forward/)
[汇编语言编写的，其中具体过程细节](https://chipengliu.github.io/2019/04/07/objc-msg-armd64/)

#### `IMP`、`SEL`、`Method`的区别和使用场景

*   `IMP` : 是方法的具体实现(指针)

*   `SEL` :方法名称

*   `Method`:是objc_method类型指针，它是一个结构体 ,如下:

```
 struct objc_method {
     SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
     char * _Nullable method_types                            OBJC2_UNAVAILABLE;
     IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
     }
     ``` 

    使用场景

    * 例如 Button添加Target和Selector的时候.或者 实现类的`swizzle`的时候会用到，通过`class_getInstanceMethod(class, SEL)`来获取类的方法`Method`，其中用到了SEL作为方法名

    * 例如 给类动态添加方法，此时我们需要调用class_addMethod(Class, SEL, IMP, types)，该方法需要我们传递一个方法的实现函数IMP，例如:

    ``` objc
    static void funcName(id receiver, SEL cmd, 方法参数...) {
     // 方法具体的实现 
    }
```

> SEL相当于 方法的类型 关键字.

#### `load`、`initialize`方法的区别什么？在继承关系中他们有什么区别

在Objective-C的类被加载和初始化的时候, 类 是 可以收到 方法回调的.


```
- (void)load;
- (void)initialize;
```

##### `+load`

`+ load`方法是在这个文件(就是你复写的子类化的class)被程序装载时调用,只要是在Xcode `Compile Sources`中出现的文件总是会被装载，这与这个类是否被用到无关，因此+load方法总是在`main()`函数之前调用.

调用时机比较早，运行环境有不确定因素。具体说来，在iOS上通常就是App启动时进行加载，但当load调用的时候，并不能保证所有类都加载完成且可用，必要时还要自己负责做auto release处理。

> 补充上面一点，对于有依赖关系的两个库中，被依赖的类的+load会优先调用。但在一个库之内，父、子类、类别之间调用有顺序，不同类之间调用顺序是不确定的。

*   关于继承：对于一个类而言，没有+load方法实现就不会调用，不会考虑对NSObject的继承，就是不会沿用父类的+load。
*   父类和本类的调用：父类的方法优先于子类的方法。一个类的+load方法不用写明`[super load]`，父类就会收到调用。
*   本类和Category的调用：本类的方法优先于类别(Category)中的方法。Category的+load也会收到调用，但顺序上在本类的+load调用之后。
*   不会直接触发initialize的调用。

#### `+initialize`

`+initialize`方法是在类或它的子类收到第一条消息之前被调用的，这里所指的消息包括实例方法和类方法的调用，并且只会调用一次。`initialize`方法实际上是一种惰性(lazy load)调用，也就是说如果一个类一直没被用到，那它的initialize方法也不会被调用，这一点有利于节约资源.

runtime 使用了发送消息 `objc_msgSend` 的方式对 `+initialize` 方法进行调用。也就是说 `+initialize` 方法的调用与普通方法的调用是一样的，走的都是`发送消息的流程`。换言之，如果子类没有实现 +initialize 方法，那么继承自父类的实现会被调用；如果一个类的分类实现了 `+initialize`方法，那么就会对这个类中的实现造成覆盖(override)。

*   initialize的自然调用是在第一次主动使用当前类的时候。
*   在initialize方法收到调用时，运行环境基本健全。
*   关于继承：和load不同，即使子类不实现initialize方法，会把父类的实现继承过来调用一遍，就是会沿用父类的+initialize。（沿用父类的方法中，self还是指子类）
*   父类和本类的调用：子类的+initialize将要调用时会激发父类调用的+initialize方法，所以也不需要在子类写明[super initialize]。(本着除主动调用外，只会调用一次的原则，如果父类的+initialize方法调用过了，则不会再调用)
*   本类和Category的调用：Category中的+initialize方法会覆盖本类的方法，只执行一个Category的+initialize方法。

下面是我整理的一个表格希望对解释这俩方法有帮助:


|   | + load | + initialize |
--- | :--: | :--: |
| 调用方式 | 直接使用函数内存地址 | objc_msgSend()方式 |
| 调用时机 | 被程序装载时调用main()函数之前,就是被添加到runtime时 | 在本类或它的子类收到第一条消息之前被调用 |
| 是否被系统单次调用(除主动调用外) | 是 | 是 |
| 运行时环境是否稳定 | 不确定 | 稳定 |
| 线程是否安全 | 默认是安全的(已加锁) | 安全(已加锁 ) |
| 特性 | 由于非`objc_msgSend()`方式调用就使得 +load 方法拥有了一个非常有趣的特性，那就是子类、父类和分类中的 +load 方法的实现是被区别对待的。也就是说如果子类没有实现 +load 方法，那么当它被加载时 runtime 是不会去调用父类的 +load 方法的。同理，当一个类和它的分类都实现了 +load 方法时，两个方法都会被调用 | +initialize 方法的调用与普通方法的调用是一样的，如果子类没有实现 +initialize 方法，那么继承自父类的实现会被调用；如果一个类的分类实现了 +initialize 方法，那么就会对这个类中的实现造成覆盖 |

参考[类方法load和initialize的区别](https://cloud.tencent.com/developer/article/1355957)

##### 在继承关系中他们有什么区别

super的方法会成功调用，但是这是多余的，因为runtime会自动对父类的+load方法进行调用，而+initialize则会随子类自动激发父类的方法（如Apple文档中所言）不需要显示调用。另一方面，如果父类中的方法用到的self（像示例中的方法），其指代的依然是类自身，而不是父类

#### 说说消息转发机制的优劣

优点:

*   利用消息转发机制可以无代码侵入的实现多重代理，让不同对象可以同时代理同个回调，然后在各自负责的区域进行相应的处理，降低了代码的耦合程度。

*   使用 @synthesize 可以为 @property 自动生成 getter 和 setter 方法（现 Xcode 版本中，会自动生成），而 @dynamic 则是告诉编译器，不用生成 getter 和 setter 方法。当使用 @dynamic 时，我们可以使用消息转发机制，来动态添加 getter 和 setter 方法。当然你也用其他的方法来实现。

缺点:

*   Objective-C本身不支持多继承，这是因为消息机制名称查找发生在运行时而非编译时，很难解决多个基类可能导致的二义性问题，但是可以通过消息转发机制在内部创建多个功能的对象，把不能实现的功能给转发到其他对象上去，这样就做出来一种多继承的假象。转发和继承相似，可用于为OC编程添加一些多继承的效果，一个对象把消息转发出去，就好像他把另一个对象中放法接过来或者“继承”一样。消息转发弥补了objc不支持多继承的性质，也避免了因为多继承导致单个类变得臃肿复杂。

# 总结

本篇讲述的面试题中的**runtime相关问题**之**结构模型**部分。

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
