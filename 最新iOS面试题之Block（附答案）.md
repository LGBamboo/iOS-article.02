# 最新iOS面试题之Block（附答案）

### 这一篇我们来研究一下objc的block并回答一下面试中的下列问题:

* 1.block的内部实现，结构体是什么样的
* 2.block是类吗，有哪些类型
* 3.一个int变量被 `__block` 修饰与否的区别？block的变量截获
* 4.block在修改NSMutableArray，需不需要添加`__block`
* 5.怎么进行内存管理的
* 6.block可以用strong修饰吗
* 7.解决循环引用时为什么要用`__strong`、`__weak`修饰
* 8.`block`发生`copy`时机
* 9.`Block`访问对象类型的`auto`变量时，在`ARC`和`MRC`下有什么区别

在回答所有问题之前我们需要了解一些block背景相关的知识. 如下:

- 如何查看Block的内部实现,也就是说转换成背后真正的c/c++代码的block是什么样的？以及转换格式或者原理等.
-关于变量的作用域

#### Objective-C 转 C++的方法

下面我写了个示例`TestClass.m`类其中block代码如下

OC代码:

```
@interface TestClass ()
@end

@implementation TestClass
- (void)testMethods {
    void (^blockA)(int a) = ^(int a) {
        NSLog(@"%d",a);
    };
    if (blockA) {
        blockA(1990);
    }
}
@end
```

经过上述转换操作我们在TestClass.cpp中最下面发现如下代码

C++代码

```
// @interface TestClass ()
/* @end */


// @implementation TestClass


struct __TestClass__testMethods_block_impl_0 {
  struct __block_impl impl;
  struct __TestClass__testMethods_block_desc_0* Desc;
  __TestClass__testMethods_block_impl_0(void *fp, struct __TestClass__testMethods_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void __TestClass__testMethods_block_func_0(struct __TestClass__testMethods_block_impl_0 *__cself, int a) {

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_wx_b8tcry0j24dbhr7zlzjq3v340000gn_T_TestClass_ee18d3_mi_0,a);
    }

static struct __TestClass__testMethods_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __TestClass__testMethods_block_desc_0_DATA = { 0, sizeof(struct __TestClass__testMethods_block_impl_0)};

static void _I_TestClass_testMethods(TestClass * self, SEL _cmd) {
    void (*blockA)(int a) = ((void (*)(int))&__TestClass__testMethods_block_impl_0((void *)__TestClass__testMethods_block_func_0, &__TestClass__testMethods_block_desc_0_DATA));
    if (blockA) {
        ((void (*)(__block_impl *, int))((__block_impl *)blockA)->FuncPtr)((__block_impl *)blockA, 1990);
    }
}
```

上面的代码生成是通过如下操作:

打开终端，cd到TestClass.m所在文件夹,使用如下命令
```
clang -rewrite-objc TestClass.m
```

就会在当前文件夹内自动生成对应的TestClass.cpp文件

> 注意: 如果提示clang没有的话 需要安装, 输入如下

```
brew install clang-format
或者
brew link clang-forma
然后输入 下面命令测试是否好使
clang-format --help
```

通过上述代码我们发现Block的其实是一个结构体类型

底层实现 会根据 `__`**类名**`__`**方法名**`_`block`_`impl`_`**下标** (0代表这个方法或者这个类中第0个block 下面如果还有将会 第1个block 第2个...)

```
struct __类名__方法名_block_impl_下标
```

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
***

#### 关于变量的作用域

c语言的函数中可能使用的参数变量种类

*   参数类型
*   自动变量(局部变量)
*   静态变量(静态局部变量)
*   静态全局变量
*   全局变量

由于存储区域特殊,这其中有三种变量是可以在任何时候以任何状态调用的.

*   静态变量
*   静态全局变量
*   全局变量

而其他两种,则是有各自相应的作用域,超过作用域后,会被销毁.

* * *

### 1.block的内部实现，结构体是什么样的

看了上面的背景知识我们来回到一下这个问题

block的内部实现如下:

```
struct __TestClass__testMethods_block_impl_0 {
  struct __block_impl impl; //成员变量
  struct __TestClass__testMethods_block_desc_0* Desc; //desc 结构体声明
  // 构造函数
  // fp 函数指针
  // desc 静态全局变量初始化的 __main_block_desc_ 结构体实例指针
  // flags block 的负载信息(引用计数和类型信息),按位存储.
  __TestClass__testMethods_block_impl_0(void *fp, struct __TestClass__testMethods_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
//将来被调用的block内部的代码：block值被转换为C的函数代码
//这里，*__cself 是指向Block的值的指针，也就相当于是Block的值它自己(相当于C++里的this，
OC里的self)
//__cself 是指向__TestClass__testMethods_block_impl_0结构体实现的指针
//Block结构体就是__TestClass__testMethods_block_impl_0结构体.Block的值就是通过__TestClass__testMethods_block_impl_0构造出来的
static void __TestClass__testMethods_block_func_0(struct __TestClass__testMethods_block_impl_0 *__cself, int a) {
	NSLog((NSString *)&__NSConstantStringImpl__var_folders_wx_b8tcry0j24dbhr7zlzjq3v340000gn_T_TestClass_9f58f7_mi_0,a);
}

static struct __TestClass__testMethods_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __TestClass__testMethods_block_desc_0_DATA = { 0, sizeof(struct __TestClass__testMethods_block_impl_0)};

static void _I_TestClass_testMethods(TestClass * self, SEL _cmd) {
    void (*blockA)(int a) = ((void (*)(int))&__TestClass__testMethods_block_impl_0((void *)__TestClass__testMethods_block_func_0, &__TestClass__testMethods_block_desc_0_DATA));
    if (blockA) {
        ((void (*)(__block_impl *, int))((__block_impl *)blockA)->FuncPtr)((__block_impl *)blockA, 1990);
    }
}
```

可以看得出来`__TestClass__testMethods_block_impl_0`有3个部分组成

*   impl 函数指针指向`__TestClass__testMethods_block_impl_0`

```
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;  //今后版本升级所需的区域
  void *FuncPtr; //函数指针
};
```

*   Desc 指向`__TestClass__testMethods_block_impl_0`的Desc指针,用于描述当前这个block的附加信息的，包括结构体的大小等等信息.

```
static struct __TestClass__testMethods_block_desc_0 {
  size_t reserved; //今后升级版本所需区域
  size_t Block_size; //block的大小
} __TestClass__testMethods_block_desc_0_DATA = { 0, sizeof(struct __TestClass__testMethods_block_impl_0)};

```

*  `__TestClass__testMethods_block_impl_0()`构造函数,也就是该block的具体实现

```
__TestClass__testMethods_block_impl_0(void *fp, struct __TestClass__testMethods_block_desc_0 *desc, int flags=0) {
   impl.isa = &_NSConcreteStackBlock;
   impl.Flags = flags;
   impl.FuncPtr = fp;
   Desc = desc;
}
```

此结构体中

*   isa指针保持这所属类的结构体的实例的指针.
*   `struct __TestClass__testMethods_block_impl_0`相当于Objective-C类对象的结构体
*   `_NSConcreteStackBlock`相当于Block的结构体实例,也就是说**block其实就是Objective-C对于闭包的对象实现**

讲到这里block的内部实现你看懂了吗?结构体是什么样的你记住了吗? 其实看着繁琐 细心观察代码会发现还是比较简单的.

### 2.block是类吗，有哪些类型?

block也算是个类,因为它有isa指针,block.isa的类型包括

*   _NSConcreteGlobalBlock 跟全局变量一样,设置在程序的数据区域(.data)中
*   _NSConcreteStackBlock栈上(前面讲的都是栈上的 block)
*   _NSConcreteMallocBlock 堆上

> 这个isa可以按位运算

### 3.一个int变量被 `__block` 修饰与否的区别？block的变量截获

#### 被`__block` 修饰与否的区别

用一段示例代码来解答这个问题吧:

```
__block int a = 10;
int b = 20;
    
PrintTwoIntBlock block = ^(){
    a -= 10;
    printf("%d, %d\n",a,b);
};
    
block();//0 20
    
a += 20;
b += 30;
    
printf("%d, %d\n",a,b);//20 50
    
block();/10 20
```

通过`__block`修饰`int` `a`,block体中对这个变量的引用是指针拷贝,它会作为block结构体构造参数传入到结构体中且复制这个变量的指针引用，从而达到可以修改变量的作用.

`int` `b`没有被`__block`修饰,block内部对`b`是值copy.所以在block内部修改`b`不影响外部b的变化.

#### block的变量截获

通过如下代码我们来观察要一下变量的捕获

```
blk_t blk;
{
    id array = [NSMutableArray new];
    blk = [^(id object){
        [array addObject:object];
        NSLog(@"array count = %ld",[array count]);
    } copy];
}
blk([NSObject new]);
blk([NSObject new]);
blk([NSObject new]);
```

输出打印

```
block_demo[28963:1629127] array count = 1
block_demo[28963:1629127] array count = 2
block_demo[28963:1629127] array count = 3
```

我们把上面的代码翻译成C++看下

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  id array;//截获的对象
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, id _array, int flags=0) : array(_array) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

在Objc中，C结构体里不能含有被`__strong`修饰的变量，因为编译器不知道应该何时初始化和废弃C结构体。但是Objc的运行时库能够准确把握`Block`从栈复制到堆，以及堆上的block被废弃的时机，在实现上是通过`__TestClass__testMethods_block_copy_0`函数和`__TestClass__testMethods_block_dispose_0`函数进行的

```
static void __TestClass__testMethods_block_copy_0(struct __TestClass__testMethods_block_impl_0*dst, struct __TestClass__testMethods_block_impl_0*src) {
    _Block_object_assign((void*)&dst->array, (void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);
}  
static void __TestClass__testMethods_block_dispose_0(struct __TestClass__testMethods_block_impl_0*src) {
    _Block_object_dispose((void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);
}
```

*   `_Block_object_assign`相当于retain操作,将对象赋值在对象类型的结构体成员变量中.

*   `_Block_object_dispose`相当于release操作.

这两个函数调用的时机是在什么时候呢？

| 函数 | 被调用时机 |
| --- | --- |
| `__TestClass__testMethods_block_copy_0` | 从栈复制到堆时 |
| `__TestClass__testMethods_block_dispose_0` | 堆上的Block被废弃时 |

##### 什么时候栈上的Block会被复制到堆呢？

*   调用block的copy函数时。

*   Block作为函数返回值返回时。

*   将Block赋值给附有`__strong`修饰符id类型的类或者Block类型成员变量时。

*   方法中含有usingBlock的Cocoa框架方法或者GCD的API中传递Block时。

##### 什么时候Block被废弃呢？

*   堆上的Block被释放后,谁都不再持有Block时调用dispose函数。

以上就是变量被block捕获的内容

* * *

### 4.`block`在修改`NSMutableArray`，需不需要添加`__block`

*   如修改`NSMutableArray`的存储内容的话,是不需要添加`__block`修饰的。
*   如修改`NSMutableArray`对象的本身,那必须添加`__block`修饰。

### 5.怎么进行内存管理的?

在上面Block的构造函数`__TestClass__testMethods_block_impl_0`中的isa指针指向的是&_NSConcreteStackBlock，它表示当前的Block位于栈区中.

| block内存操作 | 存储域/存储位置 | copy操作的影响 |
| --- | --- | --- |
| _NSConcreteGlobalBlock | 程序的数据区域 | 什么也不做 |
| _NSConcreteStackBlock | 栈 | 从栈拷贝到堆 |
| _NSConcreteMallocBlock | 堆 | 引用计数增加 |

*   全局Block:`_NSConcreteGlobalBlock`的结构体实例设置在程序的数据存储区，所以可以在程序的任意位置通过指针来访问，它的产生条件:
    *   记述全局变量的地方有block语法时.
    *   block不截获的自动变量.

    > 以上两个条件只要满足一个就可以产生全局Block. [参考](https://juejin.im/post/6844903474312773646#heading-13)

*   栈Block:`_NSConcreteStackBlock`在生成Block以后，如果这个Block不是全局Block,那它就是栈Block,生命周期在其所属的变量作用域内.(也就是说如果销毁取决于所属的变量作用域).如果Block变量和`__block`变量复制到了堆上以后，则不再会受到变量作用域结束的影响了，因为它变成了堆Block.

*   堆Block:`_NSConcreteMallocBlock`将栈block复制到堆以后，block结构体的isa成员变量变成了`_NSConcreteMallocBlock`。

### 6.block可以用strong修饰吗?

在ARC中可以，因为在ARC环境中的block只能在堆内存或全局内存中，因此不涉及到从栈拷贝到堆中的操作.

在MRC中不行,因为要有拷贝过程.如果执行copy用strong的话会crash, `strong`是ARC中引入的关键字.如果使用retain相当于忽视了block的copy过程.

### 7.解决循环引用时为什么要用`__strong`、`__weak`修饰?

首先因为block捕获变量的时候 结构体构造时传入了self,造成了默认的引用关系,所以一般在block外部对操作对象会加上`__weak`,在Block内部使用`__strong`修饰符的对象类型的自动变量，那么当Block从栈复制到堆的时候，该对象就会被Block所持有,但是持有的是我们上面加了`__weak`所以行程了比消此长的链条,刚好能解决block延迟销毁的时候对外部对象生命周期造成的影响.如果不这样做很容易造成循环引用.

### 8.block发生copy时机?

在ARC中,编译器将创建在栈中的block会自动拷贝到堆内存中,而block作为方法或函数的参数传递时,编译器不会做copy操作.

*   调用block的copy函数时。

*   Block作为函数返回值返回时。

*   将Block赋值给附有`__strong`修饰符id类型的类或者Block类型成员变量时。

*   方法中含有usingBlock的Cocoa框架方法或者GCD的API中传递Block时。

### 9.Block访问对象类型的auto变量时，在ARC和MRC下有什么区别?

ARC下会对这个对象强引用，MRC下不会

[详细请参考](https://juejin.im/post/6844903474312773646)

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
