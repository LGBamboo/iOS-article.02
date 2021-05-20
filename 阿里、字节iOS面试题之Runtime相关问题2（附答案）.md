# 阿里、字节iOS面试题之Runtime相关问题2（附答案）

## 目录

* [2020 阿里、字节iOS面试题之Runtime相关问题1](https://github.com/LGBamboo/iOS-article.02/blob/main/%E9%98%BF%E9%87%8C%E3%80%81%E5%AD%97%E8%8A%82iOS%E9%9D%A2%E8%AF%95%E9%A2%98%E4%B9%8BRuntime%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%981%EF%BC%88%E9%99%84%E7%AD%94%E6%A1%88%EF%BC%89.md)
* [2020 阿里、字节iOS面试题之Runtime相关问题2](https://github.com/LGBamboo/iOS-article.02/blob/main/%E9%98%BF%E9%87%8C%E3%80%81%E5%AD%97%E8%8A%82iOS%E9%9D%A2%E8%AF%95%E9%A2%98%E4%B9%8BRuntime%E7%9B%B8%E5%85%B3%E9%97%AE%E9%A2%982%EF%BC%88%E9%99%84%E7%AD%94%E6%A1%88%EF%BC%89.md)
* [2020 阿里、字节iOS面试题之Runtime相关问题3](https://www.jianshu.com/p/4e507f9f9f04)

# runtime相关问题之 内存管理

基本内容包括:

*   weak的实现原理？SideTable的结构是什么样的
*   关联对象的应用？系统如何实现关联对象的
*   关联对象的如何进行内存管理的？关联对象如何实现weak属性
*   Autoreleasepool的原理？所使用的的数据结构是什么
*   ARC的实现原理？ARC下对retain, release做了哪些优化
*   ARC下哪些情况会造成内存泄漏

## weak的实现原理？SideTable的结构是什么样的

先说结论:

*   `weak表`其实是一个hash(哈西)表.`Key`是所指对象的地址，`Value`是`weak`指针的地址数组.实现原理是通过新旧表的更新指针方式,对weak对象单独存储于`SideTable`中的`weak_table_t`(类型) `weak_table`表中,通过函数`objc_initWeak()`->`storeWeak()`函数中的新旧`SideTable`(结构体)表来实现
*   `SideTable`是一个结构体，内部主要有引用计数表和弱引用表两个成员，内存存储的其实都是对象的地址和引用计数和weak变量的地址，而不是对象本身的数据,它的结构如下

| 

```
struct SideTable {
     spinlock_t slock;
     RefcountMap refcnts;
     weak_table_t weak_table;
     SideTable() {
         memset(&weak_table, 0, sizeof(weak_table));
 }
     ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
 }
     void lock() { slock.lock(); }
     void unlock() { slock.unlock(); }
     void forceReset() { slock.forceReset(); }
     // Address-ordered lock discipline for a pair of side tables.
     template<HaveOld, HaveNew>
     static void lockTwo(SideTable *lock1,SideTable *lock2);
     template<HaveOld, HaveNew>
     static void unlockTwo(SideTable *lock1, SideTable *lock2);
};
```

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
***

### weak实现原理

实现原理概括分为3个时机

*   1.初始化
*   2.添加引用
*   3.释放

#### 1.初始化时候

`runtime`会调用`objc_initWeak`函数，初始化一个新的`weak`指针指向对象的地址.

我们引入一段测试代码

```
NSObject *obj = [[NSObject alloc] init];
id __weak obj1 = obj;
```

当我们初始化一个weak变量时，`runtime`会调用`NSObject.mm`中的`objc_initWeak()`函数。这个函数在Clang中的声明如下：

```
id objc_initWeak(id *location, id newObj) {
     if (!newObj) { // 查看对象实例是否有效 无效对象直接导致指针释放
         *location = nil;
         return nil;
 }
     // 这里传递了三个 bool 数值 old, new, crash.使用 template 进行常量参数传递是为了优化性能
     return storeWeak<DontHaveOld, DoHaveNew, DontCrashIfDeallocating>
         (location, (objc_object*)newObj);
}
```

可以看出，这个函数仅仅是一个深层函数的调用入口，而一般的入口函数中，都会做一些简单的判断（例如 `objc_msgSend` 中的缓存判断），这里判断了其指针指向的类对象是否有效，无效直接释放，不再往深层调用函数。否则，object将被注册为一个指向value的`__weak`对象。而这事应该是`objc_storeWeak`函数干的.

> 注意： `objc_initWeak`函数有一个前提条件：就是object必须是一个没有被注册为`__weak`对象的有效指针。而value则可以是null，或者指向一个有效的对象.

#### 2.添加引用时

`objc_initWeak`函数会调用 `objc_storeWeak()`函数,`objc_storeWeak()`则会调用`storeWeak()`函数， `storeWeak()`的作用是更新指针指向，创建对应的弱引用表

模板

```
// HaveOld:  true - 变量有值 ,false - 需要被及时清理，当前值可能为 nil
// HaveNew:  true - 需要被分配的新值，当前值可能为nil, false - 不需要分配新值
// CrashIfDeallocating: true - 说明 newObj 已经释放或者 newObj 不支持弱引用，该过程需要暂停,false - 用 nil 替代存储
template <HaveOld haveOld, HaveNew haveNew,CrashIfDeallocating crashIfDeallocating>
```

weak实现函数 **该过程用来更新弱引用指针的指向**.

```
static id 
storeWeak(id *location, objc_object *newObj)
{
    ASSERT(haveOld  ||  haveNew);
    if (!haveNew) ASSERT(newObj == nil);  
    // 初始化 previouslyInitializedClass 指针.
    Class previouslyInitializedClass = nil;
    id oldObj;
    // 声明两个 SideTable,① 新旧散列创建
    SideTable *oldTable;
    SideTable *newTable;
    //获得新值和旧值的锁存位置(用地址作为唯一标示),通过地址来建立索引标志,防止桶重复,下面指向的操作会改变旧值.
    if (haveOld) {
        oldObj = *location;// 更改指针，获得以 oldObj 为索引所存储的值地址
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];// 更改新值指针，获得以 newObj 为索引所存储的值地址
    } else {
        newTable = nil;
    }
    // 加锁操作，防止多线程中竞争冲突
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);
	// 避免线程冲突重处理,location 应该与 oldObj 保持一致，如果不同，说明当前的 location 已经处理过 oldObj 可是又被其他线程所修改
    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }
    // 防止弱引用间死锁,并且通过 +initialize 初始化构造器保证所有弱引用的 isa 非空指向
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();// 获得新对象的 isa 指针
        // 判断 isa 非空且已经初始化
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        { 
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);/ 解锁
            class_initialize(cls, (id)newObj); //如果该类已经完成执行 +initialize 方法是最理想情况,如果该类 +initialize 在线程中,例如 +initialize 正在调用 storeWeak 方法,需要手动对其增加保护策略，并设置 previouslyInitializedClass 指针进行标记
            previouslyInitializedClass = cls;
            goto retry; //重试
        }
    }
    // ② 清除旧值
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }
	 // ③ 分配新值
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        //如果弱引用被释放 weak_register_no_lock 方法返回 nil,在引用计数表中设置若引用标记位
        if (newObj  &&  !newObj->isTaggedPointer()) {
	        //弱引用位初始化操作,引用计数那张散列表的weak引用对象的引用计数中标识为weak引用
            newObj->setWeaklyReferenced_nolock();
        }
        //之前不要设置 location 对象，这里需要更改指针指向
        *location = (id)newObj;
    }
    else {
        // 没有新值，则无需更改
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```

##### SideTable

SideTable就是一个结构体，内部主要有引用计数表和弱引用表两个成员，内存存储的其实都是对象的地址和引用计数和weak变量的地址，而不是对象本身的数据. > 主要用于管理对象的引用计数和 weak 表.

我们来看图

[![](https://upload-images.jianshu.io/upload_images/13277235-8dee1bc3b238d3f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/SideTableStructure.png) 

> 操作系统维护64个SideTable，通过对象的地址位置hash之后模64(就是%64求余数)找到指定的SideTable 每个SideTable维护了一个RefcountMap的引用计数表，key就是对象地址，value就是此对象的引用计数

```
struct SideTable {
    spinlock_t slock; //保证原子操作的自旋锁
    RefcountMap refcnts; //引用计数的 hash 表
    weak_table_t weak_table; //weak 引用全局 hash 表
    ...
};
```

*   slock 防止竞争的自旋锁
*   refcnts 协助对象的 isa 指针的`extra_rc`共同引用计数的变量

##### weak表

弱引用hash表,`weak_table_t`类型的结构体,存储某个实例对象相关的所有弱引用信息. 定义如下:

```
struct weak_table_t {
    weak_entry_t *weak_entries; // 保存了所有指向指定对象的 weak 指针
    size_t    num_entries;		 // 存储空间
    uintptr_t mask;     			// 参与判断引用计数辅助量
    uintptr_t max_hash_displacement;     // hash key 最大偏移值
};
```

这是一个全局弱引用hash表。使用不定类型对象的地址作为`key`，用`weak_entry_t`类型结构体对象作为`value`,其中的`weak_entries` 成员,即为弱引用表入口.

其中`weak_entry_t`是存储在弱引用表中的一个内部结构体，它负责维护和存储指向一个对象的所有弱引用hash表。其定义如下：

```
typedef DisguisedPtr<objc_object *> weak_referrer_t;
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
    ...
};
```

其中`DisguisedPtr`类型的`referent`变量是**对泛型对象的指针的封装**,通过这个`泛型类`来解决内存泄露的问题.

注释中有个很重要的`out_of_line`成员,它代表最低的有效位,当它为0的时候,`weak_referrer_t`成员将扩展为多行静态的`hask table`.

其中`weak_referrer_t` 是一个二维`objc_object`的别名(typedef),通过一个二维指针地址偏移,用下标作hash的`key`,做成了一个弱引用的散列。

那么`weak_entry_t`中的各成员`out_of_line`、`num_refs`、`mask`、`max_hash_displacement` 在有效位未生效的时候有什么作用？

*   `out_of_line`:最低有效位，也是标志位。当标志位 0 时，增加引用表指针纬度。
*   `num_refs`: 引用数值。这里记录弱引用表中引用有效数字，因为弱引用表使用的是静态 hash 结构，所以需要使用变量来记录数目。
*   `mask`:计数辅助量。
*   `max_hash_displacement`:`hash`元素上限阀值。

> 其实 `out_of_line` 的值通常情况下是等于零的，所以弱引用表总是一个`objc_objective`指针二维数组。一维 `objc_objective`指针可构成一张弱引用散列表，通过第三纬度实现了多张散列表，并且表数量为 `WEAK_INLINE_COUNT`.

以上是weak表的实现原理.

#### 3.释放

释放时，调用`clearDeallocating`函数。`clearDeallocating`函数首先根据对象地址获取所有`weak`指针地址的数组，然后遍历这个数组把其中的数据设为`nil`，最后把这个`entry`从`weak`表中删除，最后清理对象的记录.

##### 当weak引用指向的对象被释放时，又是如何去处理weak指针的呢？当释放对象时，其基本流程如下:

*   1.调用`objc_release`
*   2.因为对象的引用计数为0，所以执行`dealloc`
*   3.在dealloc中，调用了`_objc_rootDealloc`函数
*   4.在`_objc_rootDealloc`中，调用了`object_dispose`函数
*   5.调用`objc_destructInstance`
*   6.最后调用`objc_clear_deallocating`

重点看对象被释放时调用的`objc_clear_deallocating`函数。该函数实现如下:

```
void objc_clear_deallocating(id obj)  
{
    ASSERT(obj);
    if (obj->isTaggedPointer()) return;
    obj->clearDeallocating();
}
```

调用了`clearDeallocating()`,点击源码进去追踪发现,它最终是使用了迭代器来取`weak`表的`value`,然后调用`weak_clear_no_lock()`查找对应`value`,将该`weak`指针置空.

`weak_clear_no_lock()`函数的实现如下:

```
void weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }
    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    weak_entry_remove(weak_table, entry);
}
```

`objc_clear_deallocating()`该函数的动作如下：

*   1.从weak表中获取废弃对象的地址为键值的记录
*   2.将包含在记录中的所有附有 weak修饰符变量的地址，赋值为nil
*   3.将weak表中该记录删除
*   4.从引用计数表中删除废弃对象的地址为键值的记录

[参考](https://www.jianshu.com/p/13c4fb1cedea)

## 关联对象的应用？系统如何实现关联对象的

### 关联对象的应用？

一般应用在`category`(分类)中为 当前类 添加关联属性,因为不能直接添加成员变量，但是可以通过runtime的方式间接实现添加成员变量的效果。

当我们在`category`中声明如下代码:

```
@interface ClassA : NSObject (Category)
@property (nonatomic, strong) NSString *property;
@end
```

实际上`@property`这个objc标准库的内建关键字帮我们实现了 setter和 getter,但是在category中并不能帮我们声明成员变量 `property` 我们需要通过runtime提供的两个C函数的api间接实现 动态添加 成员变量`property`.

*   `objc_setAssociatedObject()`
*   `objc_getAssociatedObject()`

```
#import "ClassA+Category.h"
#import <objc/runtime.h>

@implementation ClassA (Category)

- (NSString *) property {
    return objc_getAssociatedObject(self, _cmd);
}

- (void)setProperty:(NSString *)categoryProperty {
    objc_setAssociatedObject(self, @selector(property), categoryProperty, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
``` 


看到上面的关联方法,我们来仔细研究一下下面经常使用的关联属性相关的API

```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
id objc_getAssociatedObject(id object, const void *key);
void objc_removeAssociatedObjects(id object);
```

1.  `objc_setAssociatedObject()`以键值对形式添加关联对象
2.  `objc_getAssociatedObject()`根据 key 获取关联对象
3.  `objc_removeAssociatedObjects()`移除所有关联对象

`objc_setAssociatedObject()`的调用栈

```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
└── SetAssocHook.get()(object, key, value, policy)
    └── void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy)

```

上述调用栈中的`_object_set_associative_reference()`函数实际完成了设置关联对象的任务：

```
void
_object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)
{
     if (!object && !value) return;
    if (object->getIsa()->forbidsAssociatedObjects())
        _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));
    DisguisedPtr<objc_object> disguised{(objc_object *)object};
    ObjcAssociation association{policy, value};
    association.acquireValue();
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());
        if (value) {
            auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{});
            if (refs_result.second) {
                object->setHasAssociatedObjects();
            }
            auto &refs = refs_result.first->second;
            auto result = refs.try_emplace(key, std::move(association));
            if (!result.second) {
                association.swap(result.first->second);
            }
        } else {
            ...
        }
    }
    association.releaseHeldValue();
}
```

省略的很多代码,上述代码中就是应用场景,上面调用的类`AssociationsManager`就是我们下面要讲的系统如何实现关联对象的原理.

### 系统如何实现关联对象的(关联对象实现原理)

实现关联对象技术的核心对象 有如下这么几个:

1.  AssociationsManager
2.  AssociationsHashMap

3.  ObjectAssociationMap
4.  ObjcAssociation

> 其中Map同我们平时使用的字典类似。通过`key`-`value`的形式对应存值.

下面我们通过源码来一探究竟

#### `objc_setAssociatedObject()`函数

runtime源码

```
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
{
    _object_set_associative_reference(object, key, value, policy);
}
```

> 源码调用过程有hook函数,有点长,这里我简化一下,直接调用核心的函数

下面看下`_object_set_associative_reference()`函数的代码实现

```
void _object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)
{
    if (object->getIsa()->forbidsAssociatedObjects())
        _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));
    DisguisedPtr<objc_object> disguised{(objc_object *)object};
    ObjcAssociation association{policy, value}; //4. 我们用到的ObjcAssociation
    association.acquireValue();
    {
        AssociationsManager manager; //1. 我们用到的AssociationsManager
        AssociationsHashMap &associations(manager.get()); //2.我们上面列举的AssociationsHashMap
        if (value) {
            auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{}); //3.我们用到的ObjectAssociationMap
            if (refs_result.second) {
                object->setHasAssociatedObjects();
            }
            auto &refs = refs_result.first->second;
            auto result = refs.try_emplace(key, std::move(association));
            if (!result.second) {
                association.swap(result.first->second);
            }
        } else {
            auto refs_it = associations.find(disguised);
            if (refs_it != associations.end()) {
                auto &refs = refs_it->second;
                auto it = refs.find(key);
                if (it != refs.end()) {
                    association.swap(it->second);
                    refs.erase(it);
                    if (refs.size() == 0) {
                        associations.erase(refs_it);
                    }
                }
            }
        }
    }
    association.releaseHeldValue();
}
```

上述代码可以找到我们实现关联对象技术的核心对象. 下面我们分别介绍一下几个核心对象的内部实现.

##### AssociationsManager

```
typedef DenseMap<const void *, ObjcAssociation> ObjectAssociationMap;
typedef DenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap> AssociationsHashMap;
class AssociationsManager {
    using Storage = ExplicitInitDenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap>;
    static Storage _mapStorage;

public:
    AssociationsManager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }

    AssociationsHashMap &get() {
        return _mapStorage.get();
    }
    static void init() {
        _mapStorage.init();
    }
};
```

`AssociationsManager`内部有一个`get()`函数返回一个`AssociationsHashMap`对象

##### AssociationsHashMap

`AssociationsHashMap` 是`DenseMap`的typedef(可以理解为别名) 只不过它被定义成符合某些`元组`的条件的`DenseMap`类型

实际上 `AssociationsHashMap` 用与保存从对象的 `disguised_ptr_t`到 `ObjectAssociationMap`的映射,这个数据结构保存了当前对象对应的所有关联对象

```
typedef DenseMap<const void *, ObjcAssociation> ObjectAssociationMap;
typedef DenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap> AssociationsHashMap;

```

这里的`ObjectAssociationMap`是另一类型的typedef,里面存着`ObjcAssociation`类型的对象指针的key,value形式.

下面再看下 `ObjcAssociation` ,这是一个C++的类对象,最关键的`ObjcAssociation`包含了`policy`以及`value`.

```
class ObjcAssociation {
    uintptr_t _policy;
    id _value;
public:
    ObjcAssociation(uintptr_t policy, id value) : _policy(policy), _value(value) {}
    ObjcAssociation() : _policy(0), _value(nil) {}
    ObjcAssociation(const ObjcAssociation &other) = default;
    ObjcAssociation &operator=(const ObjcAssociation &other) = default;
    ObjcAssociation(ObjcAssociation &&other) : ObjcAssociation() {
        swap(other);
    }
    inline void swap(ObjcAssociation &other) {
        std::swap(_policy, other._policy);
        std::swap(_value, other._value);
    }
    inline uintptr_t policy() const { return _policy; }
    inline id value() const { return _value; }
    ...
};
```

##### 关联对象在内存中以什么形式存储的？

示例代码

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSObject *obj = [NSObject new];
        objc_setAssociatedObject(obj, @selector(hello), @"Hello", OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return 0;
}
```

这个调用函数`objc_setAssociatedObject(OBJC_ASSOCIATION_RETAIN_NONATOMIC, @"Hello")`在内存中是这样的存储结构

[![](https://upload-images.jianshu.io/upload_images/13277235-d5bf35501e7cf17e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/AssociationOrder.png) 

##### `objc_setAssociatedObject()`

我们回头来详细分解一下`objc_setAssociatedObject()`函数中的真实实现部分,`_object_set_associative_reference()`

这个函数需要传入`(id object, const void *key, id value, uintptr_t policy)`,这么几个参数,我们拿第3个`value`参数来分解.

我们分解为2步

1.  `value != nil` 设置或者更新关联对象的值
2.  `value == nil` 删除一个关联对象.

下面是具体是代码解释 **注意看代码注释!!!**

```
void
_object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)
{
    // 判空
    if (!object && !value) return;

	// 判断本类对象是否允许关联其他对象.如果允许则进入代码块
    if (object->getIsa()->forbidsAssociatedObjects())
        _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));

	// 将被关联的对象封装成DisguisedPtr方便在后边hash表中的管理,它的作用就像是一个指针
    DisguisedPtr<objc_object> disguised{(objc_object *)object};
    // 将需要关联的对象,封装成ObjcAssociation,方便管理
    ObjcAssociation association{policy, value};

    // 处理policy为retain和copy的修饰情况,
    association.acquireValue();

    {
    	// 获取关联对象管理者对象
        AssociationsManager manager;
        // 根据管理者对象获取对应关联表(HashMap)
        AssociationsHashMap &associations(manager.get());

        if (value) {
        	// 如果这个disguised存在于ObjectAssociationMap()中,则替换,如果不存在则初始化后在插入
        	// 这里说明一下,我们关联的对象关系存在于ObjectAssociationMap中,而
        	//	ObjectAssociationMap有多个,所以,这一步是对ObjectAssociationMap的一个管理,下边才是对我们要关联的对象的操作
            auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{});
            // 如果这是此对象第一次被关联
            if (refs_result.second) {
               // 修改isa_t中的has_assoc字段,标记其被关联状态
                object->setHasAssociatedObjects();
            }

            // 这里才是对我们要关联的对象操作
            auto &refs = refs_result.first->second;
            // 想map中插入key value对
            auto result = refs.try_emplace(key, std::move(association));
            // 这里没有看懂,为什么没有第二个就要交换一下..
            if (!result.second) {
                association.swap(result.first->second);
            }
        } else {
        	// value为空, 并且在associations中有记录,则进行擦除操作 
            auto refs_it = associations.find(disguised);
            if (refs_it != associations.end()) {
                auto &refs = refs_it->second;
                auto it = refs.find(key);
                if (it != refs.end()) {
                    association.swap(it->second);
                    refs.erase(it);
                    if (refs.size() == 0) {
                        associations.erase(refs_it);
                    }
                }
            }
        }
    }

    // release the old value (outside of the lock).
    association.releaseHeldValue();
}
```

##### `objc_setAssociatedObject()`函数的作用是什么?

```
inline void
objc_object::setHasAssociatedObjects()
{
    if (isTaggedPointer()) return;

 retry:
    isa_t oldisa = LoadExclusive(&isa.bits);
    isa_t newisa = oldisa;
    if (!newisa.nonpointer  ||  newisa.has_assoc) {
        ClearExclusive(&isa.bits);
        return;
    }
    newisa.has_assoc = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
}
```

它会将`isa`结构体中的标记位`has_assoc`标记为`true`，也就是表示当前对象有关联对象，如下图`isa`中的各个标记位都是干什么的.

[![](https://upload-images.jianshu.io/upload_images/13277235-47b4c390532bf401.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/isa.jpg) 

##### `objc_getAssociatedObject()`

这个函数的调用栈如下

```
id objc_getAssociatedObject(id object, const void *key)
└── id _object_get_associative_reference(id object, const void *key);
```

通过上面我们介绍，理解这个函数相当简单了

```
id
_object_get_associative_reference(id object, const void *key)
{
    ObjcAssociation association{};
    {
        AssociationsManager manager; //1
        AssociationsHashMap &associations(manager.get()); //1
        AssociationsHashMap::iterator i = associations.find((objc_object *)object); //2
        if (i != associations.end()) {
            ObjectAssociationMap &refs = i->second;
            ObjectAssociationMap::iterator j = refs.find(key);
            if (j != refs.end()) {
                association = j->second;
                association.retainReturnedValue();
            }
        }
    }
    return association.autoreleaseReturnedValue();
}
```

1.  通过`AssociationsManager`拿到`AssociationsHashMap`哈西表
2.  通过哈西表寻找关联对象
3.  剩下的就是更新对象是否初次创建等标记 然后返回对象

##### `objc_removeAssociatedObjects()`

调用栈如下:

```
void objc_removeAssociatedObjects(id object)
└── void _object_remove_assocations(id object)
```

代码具体实现

```
void objc_removeAssociatedObjects(id object) 
{
    if (object && object->hasAssociatedObjects()) { 
        _object_remove_assocations(object);
    }
}
```

> check对象是否为nil 且 关联对象是否存在

然后调用实现跟上边的get差不多

```
void
_object_remove_assocations(id object)
{
    ObjectAssociationMap refs{};
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());
        AssociationsHashMap::iterator i = associations.find((objc_object *)object);
        if (i != associations.end()) {
            refs.swap(i->second);
            associations.erase(i);
        }
    }
    // release everything (outside of the lock).
    for (auto &i: refs) {
        i.second.releaseHeldValue();
    }
}
```

通过`AssociationsManager` -> `AssociationsHashMap` -> object 是否存在,如果存在就**擦除**.- > releaseHeldValue()是否对象

#### 小结

关联对象的应用和系统如何实现关联对象的大概顺序如下:
`AssociationsManager`关联对象管理器->`AssociationsHashMap`哈希映射表->`ObjectAssociationMap`关联对象指针->`ObjcAssociation`关联对象

## 关联对象的如何进行内存管理的？关联对象如何实现weak属性?

### 关联对象的如何进行内存管理的？

当我调用关联对象函数`objc_setAssociatedObject()`的时候会调用如下函数：

`_object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)`,这里面有个方法

```
ObjcAssociation association{policy, value};
// retain the new value (if any) outside the lock.
association.acquireValue();
```

这里的 `policy`就是具体绝对内存使用retain还是其它相关的内存枚举.

```
enum {
    OBJC_ASSOCIATION_SETTER_ASSIGN      = 0,
    OBJC_ASSOCIATION_SETTER_RETAIN      = 1,
    OBJC_ASSOCIATION_SETTER_COPY        = 3,            // NOTE:  both bits are set, so we can simply test 1 bit in releaseValue below.
    OBJC_ASSOCIATION_GETTER_READ        = (0 << 8),
    OBJC_ASSOCIATION_GETTER_RETAIN      = (1 << 8),
    OBJC_ASSOCIATION_GETTER_AUTORELEASE = (2 << 8)
};
```

通过 acquireValue()函数判断使用那种内存关键字.

```
inline void acquireValue() {
    if (_value) {
        switch (_policy & 0xFF) {
        case OBJC_ASSOCIATION_SETTER_RETAIN:
            _value = objc_retain(_value);
            break;
        case OBJC_ASSOCIATION_SETTER_COPY:
            _value = ((id(*)(id, SEL))objc_msgSend)(_value, @selector(copy));
            break;
        }
    }
}
```

### 关联对象如何实现weak属性？

首先说一下 这个问题问的非常有技术含量,完全考验iOS开发者对底层了解的程度.

在为NSObject对象绑定 associated object 时可以指定如下依赖关系：

```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0, //弱引用
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, //强引用，非原子操作
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,  //先 copy，然后强引用
    OBJC_ASSOCIATION_RETAIN = 01401, //强引用，原子操作
    OBJC_ASSOCIATION_COPY = 01403 //先 copy，然后强引用，原子操作
};
```

根据上述的枚举我们发现一个很奇怪的问题,这里的枚举中并没有`OBJC_ASSOCIATION_WEAK`这样的选项.

基于上述的代码介绍我们知道`Objective-C`在底层使用`AssociationsManager`统一管理各个对象的 `associated objects`关联对象.然后通过`static key`(一般是一个固定值)去访问对应的`associated object`关联对象.然后在`dealloc`的时候调用`擦除函数`(`associations.erase()`)来解除对这些关联对象的引用:

```
dealloc
    object_dispose
        objc_destructInstance
            _object_remove_assocations  // 移除必要的associated objects
```

也就是说,在`NSObject`对象的内存空间里，并没有为 `associated objects`(关联对象) 分配任何变量.

我们知道weak变量和 assign变量的区别是:weak指向的对象销毁的时候,`Objective-C`会自动帮我们设置`nil`,而`assign`却不能.

这个逻辑是如何实现的呢？

`Runtime`在底层维护一个`weak`表(也就是本文开头讲的`SlideTable`中的`weak_table_t` `weak_tabl`)，每每分配一个`weak`指针并赋值有效对象的地址时，会将对象地址和`weak`指针地址注册到`weak`表中，其中对象地址作为`key`;当对象被废弃时,可根据对象地址快速寻找到指向它的所有`weak` 指针,这些`weak`指针会被赋值`0`(即`nil`）并移出`weak表。

所以,实现`weak`引用(而非`assign`引用)的前提是存在一个`__weak`指针指向到被引用对象的地址,只有这样,当对象被销毁时，指针才能被`runtime`找到然后被设置为`nil`；`NSObject`对象和其`associated object`关联对象的关系，并不存在指针这样的**中间媒介**，因此只存在`OBJC_ASSOCIATION_ASSIGN`选项，而不存在`OBJC_ASSOCIATION_WEAK`选项.

#### 那我们怎么解决为关联对象实现weak属性呢？

可以通过曲线救国的方式声明一个`class`类 持有一个weak的成员变量,然后通过 实例化 我们自定义的class的实例,然后把这个实例作为关联对象即可.

声明封装weak对象的类

```
@interface WeakAssociatedObjectWrapper : NSObject
@property (nonatomic, weak) id object;
@end

@implementation WeakAssociatedObjectWrapper
@end
```

调用

```
@interface UIView (ViewController)
@property (nonatomic, weak) UIViewController *vc;
@end

@implementation UIView (ViewController)
- (void)setVc:(UIViewController *)vc {
    WeakAssociatedObjectWrapper *wrapper = [WeakAssociatedObjectWrapper new];
    wrapper.object = vc;
    objc_setAssociatedObject(self, @selector(vc), wrapper, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
- (UIViewController *)vc {
    WeakAssociatedObjectWrapper *wrapper = objc_getAssociatedObject(self, _cmd);
    return wrapper.object;
}
@end
```

> 看明白没有,曲线救国.代码引入自[Weak Associated Object](https://zhangbuhuai.com/post/weak-associated-object.html)

[关联对象参考](https://draveness.me/ao/)

## Autoreleasepool的原理？所使用的的数据结构是什么？

在ARC下我们使用`@autoreleasepool{}` 关键字 把需要自动管理的代码块圈起来 ,这个过程就是在使用一个`AutoReleasePool`

```
@autoreleasepool {
	 <#statements#> //代码块
}
```

以上代码编译器 最终会把它改写成下面的样子

```
void *context = objc_autoreleasePoolPush();
```

既然有压栈一定就有 出栈操作`objc_autoreleasePoolPop(context)`;

*   `objc_autoreleasePoolPush()`
*   `objc_autoreleasePoolPop()`

这俩函数都是对`AutoreleasePoolPage`的封装,自动释放机制的核心就是这个类

### `AutoreleasePoolPage`

`AutoreleasePoolPage`是个C++的类

[![](https://upload-images.jianshu.io/upload_images/13277235-a112c7bd28593aa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/autoreleasepoolpage.png) 

*   **AutoreleasePool**并没有单独的结构,而是由若干个`AutoreleasePoolPage`以`双向链表`的形式组合成的,根据上图可以看出,这个双向链表有`前驱parent`和`后继child`.
*   **AutoreleasePool**是按`线程`一一对应的(thread 成员变量)
*   **AutoreleasePoolPage**就是自动释放池存储对象的数据结构每个Page占用`4KB`内存，本身的成员变量占用`56`字节，剩下的空间用来存放调用了`autorelease`方法的对象地址,同时将一个哨兵插入到Page中，这个哨兵其实就是一个空地址
*   当一个page被占满以后会新建一个新的`AutoreleasePoolPage`对象,并插入哨兵标记. 具体代码如下:

```
class AutoreleasePoolPage {
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)
#   define POOL_BOUNDARY nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE = 
#if PROTECT_AUTORELEASEPOOL
        PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
        PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
    static size_t const COUNT = SIZE / sizeof(id);
    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};
```

*   `magic` 检查校验完整性的变量
*   `next` 指向新加入的autorelease对象
*   `thread` page当前所在的线程，AutoreleasePool是按线程一一对应的（结构中的thread指针指向当前线程）
*   `parent` 父节点 指向前一个page
*   `child` 子节点 指向下一个page
*   `depth` 链表的深度，节点个数
*   `hiwat` high water mark 数据容纳的一个上限
*   `EMPTY_POOL_PLACEHOLDER` 空池占位
*   `POOL_BOUNDARY` 是一个边界对象 nil,之前的源代码变量名是 `POOL_SENTINEL`哨兵对象,用来区别每个page即每个 AutoreleasePoolPage 边界
*   `PAGE_MAX_SIZE` = 4096, 为什么是4096呢？其实就是虚拟内存每个扇区4096个字节,4K对齐的说法。
*   `COUNT` 一个page里对象数

下面看下工作机制图

[![](https://upload-images.jianshu.io/upload_images/13277235-4d8656f7b69c5dba.gif?imageMogr2/auto-orient/strip)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/autoreleasepoolworkflow.gif) 

> 这张图来自快手同事 周学运,如果大佬看到这张图的话希望能允许授权给我使用哈.

根据上面的示意图我们大概明白, `AutoreleasePoolPage`是以栈的形式存在,并且内部对象通过进栈出栈来对应着`objc_autoreleasePoolPush`和`objc_autoreleasePoolPop`

如果嵌套AutoreleasePool 就是通过`哨兵对象`来标识,每次更新链表的next和`前驱``后继`来完成表的创建销毁.

[![](https://upload-images.jianshu.io/upload_images/13277235-82a0c2edb402845f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200808iOSinterviewAnswers/autoreleasepoolpage1.png) 

当我们对一个对象发送一条`autorelease`消息的时候实际上就是将这个对象加入到当前`AutoreleasePoolPage`的栈顶`next`指针指向的位置

> 这里只拿了一张page举例.

#### 小结

*   自动释放池是有N张`AutoreleasePoolPage`组成,每张page 4K大小, AutoreleasePoolPage是c++的类, AutoreleasePoolPage以双向链表连接起来形成一个自动释放池
*   当对象调用 autorelease 方法时，会将对象加入 AutoreleasePoolPage 的栈中
*   pop 时是传入边界对象(哨兵对象),然后对page 中的对象发送release 的消息

[自动释放池原理](https://www.jianshu.com/p/0afda1f23782) [AutoreleasePool底层实现原理](https://juejin.im/post/6844903609428115470)

## ARC的实现原理？ARC下对retain, release做了哪些优化

ARC自动引用计数,是苹果objc4引入的编译器自动在适当位置 帮助实例对象进行 自动retain后者release的一套机制.

它的实现原理就是在编译层面插入相关代码,帮助补全MRC时代需要开发者手动填写的和管理的对象的相关内存操作的方法.

为了解释清楚具体实现原理 ,我找到一篇有代码示例的文章,从代码编译成汇编过程中 编译器做了很多优化工作. 更新`isa指针`的信息.

[理解 ARC 实现原理](https://juejin.im/post/6844903847622606861#heading-4)

这里有个点需要跟大家说一下, 上文 中我们讲了SlideTable,但是还是有不懂得地方下面我们来通过isa串联起来

isa的组成

```
union isa_t 
{
    Class cls;
    uintptr_t bits;
    struct {
         uintptr_t nonpointer        : 1;//->表示使用优化的isa指针
         uintptr_t has_assoc         : 1;//->是否包含关联对象
         uintptr_t has_cxx_dtor      : 1;//->是否设置了析构函数，如果没有，释放对象更快
         uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000 ->类的指针
         uintptr_t magic             : 6;//->固定值,用于判断是否完成初始化
         uintptr_t weakly_referenced : 1;//->对象是否被弱引用
         uintptr_t deallocating      : 1;//->对象是否正在销毁
         uintptr_t has_sidetable_rc  : 1;//1->在extra_rc存储引用计数将要溢出的时候,借助Sidetable(散列表)存储引用计数,has_sidetable_rc设置成1
        uintptr_t extra_rc          : 19;  //->存储引用计数
    };
};
```

其中`nonpointer`、`weakly_referenced`、`has_sidetable_rc`和`extra_rc`都是 `ARC`有直接关系的成员变量，其他的大多也有涉及到。

### retain,release做了哪些优化

大概可以分为如下

*   TaggedPointer 指针优化
*   !newisa.nonpointer：未优化的 isa 的情况下retain或者release
*   newisa.nonpointer：已优化的 isa ， 这其中又分 extra_rc 溢出区别 我把相关代码站在下面并且把结论输出出来.

| 内存操作 | objc_retain | objc_release |
| --- | --- | --- |
| TaggedPointer | 值存在指针内，直接返回 | 直接返回 false。 |
| !nonpointer | 未优化的`isa`,使用`sidetable_retain()` | 未优化的`isa`执行`sidetable_release` |
| nonpointer | 已优化的`isa`,这其中又分`extra_rc`溢出和未溢出的两种情况 | 已优化的`isa`,分下溢和未下溢两种情况 |


| nonpointer已优化isa的extra_rc | objc_retain | objc_release |
| --- | --- | --- |
| 未溢出时 | `isa.extra_rc`+1 | NA |
| 溢出时 | 将`isa.extra_rc`中一半值转移至`sidetable`中,然后将`isa.has_sidetable_rc`设置为`true`,表示使用了`sidetable`来计算引用次数 | NA |
| 未下溢 | NA | extra_rc-- |
| 下溢 | NA | 从`sidetable`中借位给`extra_rc`达到半满,如果无法借位则说明引用计数归零需要进行释放,其中借位时可能保存失败会不断重试 |

> NA -> non available 不可获得

下面我们看下retain源码

```
ALWAYS_INLINE id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    if (isTaggedPointer()) return (id)this;     // 如果是 TaggedPointer 直接返回
    bool sideTableLocked = false;
    bool transcribeToSideTable = false;
    isa_t oldisa;
    isa_t newisa;
    do {
        transcribeToSideTable = false;
        oldisa = LoadExclusive(&isa.bits);  // 获取 isa
        newisa = oldisa;
        if (slowpath(!newisa.nonpointer)) {
            ClearExclusive(&isa.bits);// 未优化的 isa 部分
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            if (tryRetain) return sidetable_tryRetain() ? (id)this : nil;
            else return sidetable_retain();
        }
        if (slowpath(tryRetain && newisa.deallocating)) { // 正在被释放的处理
            ClearExclusive(&isa.bits);
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            return nil;
        }
        // extra_rc 未溢出时引用计数++
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++
        // extra_rc 溢出
        if (slowpath(carry)) {
            // newisa.extra_rc++ overflowed
            if (!handleOverflow) {
                ClearExclusive(&isa.bits);
                return rootRetain_overflow(tryRetain);   // 重新调用该函数 入参 handleOverflow 为 true
            } 
            // 保留一半引用计数,准备将另一半复制到 side table.
            if (!tryRetain && !sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
        //  更新 isa 值
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));
    if (slowpath(transcribeToSideTable)) {
        sidetable_addExtraRC_nolock(RC_HALF); // 将另一半复制到 side table side table.
    }
    if (slowpath(!tryRetain && sideTableLocked)) sidetable_unlock();
    return (id)this;
}
```

`release`源码

```
ALWAYS_INLINE bool objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    if (isTaggedPointer()) return false;
    bool sideTableLocked = false;
    isa_t oldisa;
    isa_t newisa;
 retry:
    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        if (slowpath(!newisa.nonpointer)) {
            ClearExclusive(&isa.bits);// 未优化 isa
            if (sideTableLocked) sidetable_unlock();
            return sidetable_release(performDealloc);// 入参是否要执行 Dealloc 函数，如果为 true 则执行 SEL_dealloc
        }
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        if (slowpath(carry)) {
            // donot ClearExclusive()
            goto underflow;
        }
        // 更新 isa 值
    } while (slowpath(!StoreReleaseExclusive(&isa.bits, 
                                             oldisa.bits, newisa.bits)));
    if (slowpath(sideTableLocked)) sidetable_unlock();
    return false;
 underflow:
 	// 处理下溢，从 side table 中借位或者释放
    newisa = oldisa;
    if (slowpath(newisa.has_sidetable_rc)) { // 如果使用了 sidetable_rc
        if (!handleUnderflow) {
        	ClearExclusive(&isa.bits);// 调用本函数处理下溢
            return rootRelease_underflow(performDealloc);
        }
        size_t borrowed = sidetable_subExtraRC_nolock(RC_HALF); // 从 sidetable 中借位引用计数给 extra_rc

        if (borrowed > 0) {
		// extra_rc 是计算额外的引用计数，0 即表示被引用一次
            newisa.extra_rc = borrowed - 1;  // redo the original decrement too
            bool stored = StoreReleaseExclusive(&isa.bits, 
                                                oldisa.bits, newisa.bits);                                    
            // 保存失败，恢复现场，重试                                    
            if (!stored) {
                isa_t oldisa2 = LoadExclusive(&isa.bits);
                isa_t newisa2 = oldisa2;
                if (newisa2.nonpointer) {
                    uintptr_t overflow;
                    newisa2.bits = 
                        addc(newisa2.bits, RC_ONE * (borrowed-1), 0, &overflow);
                    if (!overflow) {
                        stored = StoreReleaseExclusive(&isa.bits, oldisa2.bits, 
                                                       newisa2.bits);
                    }
                }
            }
		// 如果还是保存失败，则还回 side table
            if (!stored) {
                sidetable_addExtraRC_nolock(borrowed);
                goto retry;
            }
            sidetable_unlock();
            return false;
        }
        else {
            // Side table is empty after all. Fall-through to the dealloc path.
        }
    }
    // 没有使用 sidetable_rc ，或者 sidetable_rc 计数 == 0 的就直接释放
    // 如果已经是释放中，抛个过度释放错误
    if (slowpath(newisa.deallocating)) {
        ClearExclusive(&isa.bits);
        if (sideTableLocked) sidetable_unlock();
        return overrelease_error();
    }
    // 更新 isa 状态
    newisa.deallocating = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
    if (slowpath(sideTableLocked)) sidetable_unlock();
	// 执行 SEL_dealloc 事件
    __sync_synchronize();
    if (performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return true;
}
```

### 小结

到这里可以知道 引用计数分别保存在`isa.extra_rc`和`sidetable`中，当`isa.extra_rc`溢出时，将一半计数转移至`sidetable`中，而当其下溢时，又会将计数转回。当二者都为空时，会执行释放流程

## ARC下哪些情况会造成内存泄漏

*   block中的循环引用
*   NSTimer的循环引用
*   addObserver的循环引用
*   delegate的强引用
*   大次数循环内存爆涨
*   非OC对象的内存处理（需手动释放）

# 总结

以上就是我们讨论上述一套面试题的 runtime相关问题之 内存管理部分, 感谢各位支持!

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
