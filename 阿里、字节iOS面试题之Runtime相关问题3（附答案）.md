# 阿里、字节iOS面试题之Runtime相关问题3（附答案）

### 目录

* [阿里、字节iOS面试题之Runtime相关问题1](https://github.com/LGBamboo/iOS-article.02/blob/main/%E9%98%BF%E9%87%8C%E3%80%81%E5%AD%97%E8%8A%82iOS%E9%9D%A2%E8%AF%95%E9%A2%98%E4%B9%8BRuntime%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%981%EF%BC%88%E9%99%84%E7%AD%94%E6%A1%88%EF%BC%89.md)
* [阿里、字节iOS面试题之Runtime相关问题2](https://github.com/LGBamboo/iOS-article.02/blob/main/%E9%98%BF%E9%87%8C%E3%80%81%E5%AD%97%E8%8A%82iOS%E9%9D%A2%E8%AF%95%E9%A2%98%E4%B9%8BRuntime%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%982%EF%BC%88%E9%99%84%E7%AD%94%E6%A1%88%EF%BC%89.md)
* [阿里、字节iOS面试题之Runtime相关问题3](https://github.com/LGBamboo/iOS-article.02/blob/main/%E9%98%BF%E9%87%8C%E3%80%81%E5%AD%97%E8%8A%82iOS%E9%9D%A2%E8%AF%95%E9%A2%98%E4%B9%8BRuntime%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%983%EF%BC%88%E9%99%84%E7%AD%94%E6%A1%88%EF%BC%89.md)


# runtime相关问题之内存部分的关联属性或者hook相关的Method Swizzle

经过前两期内容 我们这期来讲一下 内存部分的剩余问题 主要包含如下:

1.  `Method Swizzle`注意事项
2.  属性修饰符atomic的内部实现是怎么样的?能保证线程安全吗
3.  iOS 中内省的几个方法有哪些？内部实现原理是什么
4.  `class`、`objc_getClass`、`object_getclass` 方法有什么区别?

## `Method Swizzle`注意事项

1.  **需要注意的是交换方法实现后的副作用**, `method_exchangeImplementations()`.交换方法函数最终会以`objc_msgSend()`方式调用,副作用主要集中在第一个参数 如下示例

```
objc_msgSend(payment, @selector(quantity))
```

 方法交换后再去调用quantity方法将有可能会crash.解决这种副作用的方式是使用`method_setImplementation()`来替换原来的交换方式,这样才最为合理, 具体原理请参照 [Objc 黑科技 - Method Swizzle 的一些注意事项](https://www.ctolib.com/topics-103098.html)

2.  **避免交换父类方法**

    如果当前类没有实现被交换的方法且父类实现了,此时父类的实现会被交换,若此父类的多个继承者都在交换时会引起多次交换导致混乱,同时调用父类方法有可能因为找不到方法签名而crash.
    所以交换前都应该check能否为当前类添加被交换的函数的新的实现IMP,这个过程大概分为3步骤

    *   `class_addMethod` check能否添加方法

```
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)

```

> 给类cls的SEL添加一个实现IMP, 返回YES则表明类cls并未实现此方法，返回NO则表明类已实现了此方法。注意：添加成功与否，完全由该类本身来决定，与父类有无该方法无关。

*   `class_replaceMethod` 替换类cls的SEL的函数实现为imp

```
class_replaceMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp, 
                 const char * _Nullable types)

```

*   `method_exchangeImplementations` 最终方法交换

```
method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2)

```

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
***

3.  交换方法应在+load方法

这个前面讲消息转发的时候讲过,+load不是消息转发的方式实现的且在运行时初始化过程中类被加载的时候调用,而且父类,当前类,category,子类等 都会调用一次.所以这里最适合写方法交换的hook(Method Swizzle).

4.  交换的分类方法应该添加自定义前缀,避免冲突

    这个毫无疑问,方法名称一样的时候会出现,分类的方法会覆盖类中同名的方法.

[method swizzling你应该注意的点](https://blog.csdn.net/weixin_34168700/article/details/88762738)

## 属性修饰符atomic的内部实现是怎么样的?能保证线程安全吗?

### atomic内部实现

```
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    ...
    id *slot = (id*) ((char*)self + offset);
    if (!atomic) return *slot;  
    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();
    return objc_autoreleaseReturnValue(value);
}
```

```
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    ...
    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }
    objc_release(oldValue);
}
```

`property` 的 `atomic` 是采用 `spinlock_t`自旋锁实现的.

### 能保证线程安全吗?

`atomic`通过这种方法.在运行时仅仅是保证了`set`,`get`方法的原子性.所以使用atomic并不能保证线程安全。

## iOS 中内省的几个方法有哪些？内部实现原理是什么?

首先要明白一个名词 `introspection` 反省,内省的意思,在iOS开发中我们会称它为反射.

内省方法 例如常用的`NSObject`中的`isKindOfClass:` 通过实例对象判断`class`这就是一种内省方法或者叫反射方法,但我认为`NSClassFromString()`这个应该也算一种反射方法.

### iOS 中内省的几个方法

我们从NSObject.h中看下吧

```
- (BOOL)isKindOfClass:(Class)aClass; //判断是否是这个类或者这个类的子类的实例
- (BOOL)isMemberOfClass:(Class)aClass; //判断是否是这个类的实例
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;  //判断是否遵守某个协议
+ (BOOL)conformsToProtocol:(Protocol *)protocol; //判断某个类是否遵守某个协议
- (BOOL)respondsToSelector:(SEL)aSelector;  //判读实例是否有这样方法
+ (BOOL)instancesRespondToSelector:(SEL)aSelector; //判断类是否有这个方法
...
```

### 内部实现原理

1.`isKindOfClass:`

```
+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = self->ISA(); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}
	
- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```

类方法是通过ISA()函数拿到指向元类的存储isa指针数据的地址bit位按位与上相关掩码的方式判断当前是否是某个类的子类.
实例方法是通过`objc_object::getIsa()`函数通过存储的`tag_ext`表形式拿到isa对于的class来取出class平check来实现的.

2.`isMemberOfClass:`

```
+ (BOOL)isMemberOfClass:(Class)cls {
    return self->ISA() == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}
```

这俩方法非常简单直接 拿到isa指针对比

3.`conformsToProtocol:`

```
+ (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = self; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}

- (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}
```

两个方法最终还是去isa->data()->protocols 拿到相关协议然后判断是否存在相关协议 如下代码：

```
BOOL class_conformsToProtocol(Class cls, Protocol *proto_gen)
{
    protocol_t *proto = newprotocol(proto_gen);  
    if (!cls) return NO;
    if (!proto_gen) return NO;
    mutex_locker_t lock(runtimeLock);
    checkIsKnownClass(cls);
    ASSERT(cls->isRealized())
    for (const auto& proto_ref : cls->data()->protocols) {
        protocol_t *p = remapProtocol(proto_ref);
        if (p == proto || protocol_conformsToProtocol_nolock(p, proto)) {
            return YES;
        }
    }
    return NO;
}
```

> 这里可以清晰的看到for循环 取出相关protocol指针 然后通过指针和传入的参数生成的`proto`对比

4.`respondsToSelector:`

```
+ (BOOL)respondsToSelector:(SEL)sel {
    return class_respondsToSelector_inst(self, sel, self->ISA());
}

- (BOOL)respondsToSelector:(SEL)sel {
    return class_respondsToSelector_inst(self, sel, [self class]);
}
```

这个源码比较麻烦 我简单叙述一下吧 实际上调用栈比较深就是一直寻找到当前实例能响应哪些方法,当前类没有就去父类,父类没有则直到元类.

```
respondsToSelector:
	|__ class_respondsToSelector_inst()
		|__ lookUpImpOrNil()
			|__ lookUpImpOrForward()
				返回IMP结果
```

这就是整个消息转发的过程 就不在这里赘述了.感兴趣回看一下[第二章](https://www.jianshu.com/p/f2331034f0ab) 消息转发部分

我上述列举了一些常用的内省方法,其它的都方法基本没什么特别之处都是拿到isa各种操作内部的获取相关属性的函数返回结.

## `class`、`objc_getClass`、`object_getclass` 方法有什么区别?

我用xcode随便建了一个demo 打印一下viewcontrooller的内容

```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    Class cls1 = [self class];
    Class cls2 = object_getClass(cls1);
    Class cls3 = objc_getClass(object_getClassName([self class]));
    NSLog(@"%p",cls1);
    NSLog(@"%p",cls2);
    NSLog(@"%p",cls3);
}
@end
```

输出

```
2020-08-31 16:15:48.150285+0800 ClassDemo[5582:55836] 0x10205b3b0
2020-08-31 16:15:48.150456+0800 ClassDemo[5582:55836] 0x10205b3d8
2020-08-31 16:15:48.150575+0800 ClassDemo[5582:55836] 0x10205b3b0
```

我简单列举了一张表格

|  | `class` | `object_getclass()` | `objc_getClass()` |
| --- | --- | --- | --- |
| 传入参数 | N/a | id类型 | 类名的字符串 |
| 操作对象 | obj | 这个id的isa指针所指向的Class | 这个类的类对象 |
| 实例对象时 | 和`object_getclass()`一致 | 和`class`一致 | N/a |
| 类对象/元类对象时 | 返回的消息对象本身 | 返回的是下一个对象 | N/a |

> 原因：因为class返回的是self，而object_getClass返回的是isa指向的对象

# 总结

以上就是"一套高效的iOS面试题之runtime相关问题3"中的内存剩余部分,问题答案虽然简短 但是每道题都问的非常到位,值得一看！

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
