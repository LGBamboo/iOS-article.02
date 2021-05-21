# 大厂常问iOS面试小集

### 目录 

>**1、SDWebImage原理**
>
>**2、什么是Block？**
>
>**3、RunLoop剖析**

***
# 一、 SDWebImage原理
***

一个为UIImageView提供一个分类来支持远程服务器图片加载的库。

### 功能简介：

```
      1、一个添加了web图片加载和缓存管理的UIImageView分类
      2、一个异步图片下载器
      3、一个异步的内存加磁盘综合存储图片并且自动处理过期图片
      4、支持动态gif图
      5、支持webP格式的图片
      6、后台图片解压处理
      7、确保同样的图片url不会下载多次
      8、确保伪造的图片url不会重复尝试下载
      9、确保主线程不会阻塞
```

### 工作流程

```
1、入口 setImageWithURL:placeholderImage:options: 会先把 placeholderImage 显示，然后 SDWebImageManager 根据 URL 开始处理图片。

2、进入 SDWebImageManager-downloadWithURL:delegate:options:userInfo:，交给 SDImageCache 从缓存查找图片是否已经下载 queryDiskCacheForKey:delegate:userInfo:.

3、先从内存图片缓存查找是否有图片，如果内存中已经有图片缓存，SDImageCacheDelegate 回调 imageCache:didFindImage:forKey:userInfo: 到 SDWebImageManager。

4、SDWebImageManagerDelegate 回调 webImageManager:didFinishWithImage: 到 UIImageView+WebCache 等前端展示图片。

5、如果内存缓存中没有，生成 NSInvocationOperation 添加到队列开始从硬盘查找图片是否已经缓存。

6、根据 URLKey 在硬盘缓存目录下尝试读取图片文件。这一步是在 NSOperation 进行的操作，所以回主线程进行结果回调 notifyDelegate:。

7、如果上一操作从硬盘读取到了图片，将图片添加到内存缓存中（如果空闲内存过小，会先清空内存缓存）。SDImageCacheDelegate 回调 imageCache:didFindImage:forKey:userInfo:。进而回调展示图片。

8、如果从硬盘缓存目录读取不到图片，说明所有缓存都不存在该图片，需要下载图片，回调 imageCache:didNotFindImageForKey:userInfo:。

9、共享或重新生成一个下载器 SDWebImageDownloader 开始下载图片。

10、图片下载由 NSURLConnection 来做，实现相关 delegate 来判断图片下载中、下载完成和下载失败。

11、connection:didReceiveData: 中利用 ImageIO 做了按图片下载进度加载效果。connectionDidFinishLoading: 数据下载完成后交给 SDWebImageDecoder 做图片解码处理。

12、图片解码处理在一个 NSOperationQueue 完成，不会拖慢主线程 UI。如果有需要对下载的图片进行二次处理，最好也在这里完成，效率会好很多。

13、在主线程 notifyDelegateOnMainThreadWithInfo: 宣告解码完成，imageDecoder:didFinishDecodingImage:userInfo: 回调给 SDWebImageDownloader。imageDownloader:didFinishWithImage: 回调给 SDWebImageManager 告知图片下载完成。

14、通知所有的 downloadDelegates 下载完成，回调给需要的地方展示图片。将图片保存到 SDImageCache 中，内存缓存和硬盘缓存同时保存。写文件到硬盘也在以单独 NSInvocationOperation 完成，避免拖慢主线程。

15、SDImageCache 在初始化的时候会注册一些消息通知，在内存警告或退到后台的时候清理内存图片缓存，应用结束的时候清理过期图片。

16、SDWI 也提供了 UIButton+WebCache 和 MKAnnotationView+WebCache，方便使用。

17、SDWebImagePrefetcher 可以预先下载图片，方便后续使用。
```

### 源码分析

#### 主要用到的对象

#### 一、图片下载

**1、 SDWebImageDownloader**
* 1.单例，图片下载器，负责图片异步下载，并对图片加载做了优化处理

* 2.图片的下载操作放在一个NSOperationQueue并发操作队列中，队列默认最大并发数是6

* 3.每个图片对应一些回调（下载进度，完成回调等），回调信息会存在downloader的URLCallbacks（一个字典，key是url地址，value是图片下载回调数组）中，URLCallbacks可能被多个线程访问，所以downloader把下载任务放在一个barrierQueue中，并设置屏障保证同一时间只有一个线程访问URLCallbacks。，在创建回调URLCallbacks的block中创建了一个NSOperation并添加到NSOperationQueue中。

* 4.每个图片下载都是一个operation类，创建后添加到一个队列中，SDWebimage定义了一个协议 SDWebImageOperation作为图片下载操作的基础协议，声明了一个cancel方法，用于取消操作。
```
@protocol SDWebImageOperation <NSObject>
-(void)cancel;
@end
```

* 5.对于图片的下载，SDWebImageDownloaderOperation完全依赖于NSURLConnection类，继承和实现了NSURLConnectionDataDelegate协议的方法

```
connection:didReceiveResponse:
connection:didReceiveData:
connectionDidFinishLoading:
connection:didFailWithError:
connection:willCacheResponse:
connectionShouldUseCredentialStorage:
-connection:willSendRequestForAuthenticationChalleng
-connection:didReceiveData:方法，接受数据，创建一个CGImageSourceRef对象，在首次获取数据时（图片width，height），图片下载完成之前，使用CGImageSourceRef对象创建一个图片对象，经过缩放、解压操作生成一个UIImage对象供回调使用，同时还有下载进度处理。
注：缩放：SDWebImageCompat中SDScaledImageForKey函数
 解压：SDWebImageDecoder文件中decodedImageWithImage

```

**2、SDWebImageDownloaderOption**

* 1.继承自NSOperation类，没有简单实现main方法，而是采用更加灵活的start方法，以便自己管理下载的状态

* 2.start方法中创建了下载使用的NSURLConnections对象，开启了图片的下载，并抛出一个下载开始的通知，

* 3.小结：下载的核心是利用NSURLSession加载数据，每个图片的下载都有一个operation操作来完成，并将这些操作放到一个操作队列中，这样可以实现图片的并发下载。

**3、SDWebImageDecoder（异步对图片进行解码）**

### 二、缓存
减少网络流量，下载完图片后存储到本地，下载再获取同一张图片时，直接从本地获取，提升用户体验，能快速从本地获取呈现给用户。
SDWebImage提供了对图片进行了缓存，主要由SDImageCache完成。该类负责处理内存缓存以及一个可选的磁盘缓存，其中磁盘缓存的写操作是异步的，不会对UI造成影响。

**1、内存缓存及磁盘缓存**
* 1.内存缓存的处理由NSCache对象实现，NSCache类似一个集合的容器，它存储key-value对，类似于nsdictionary类，我们通常使用缓存来临时存储短时间使用但创建昂贵的对象，重用这些对象可以优化新能，同时这些对象对于程序来说不是紧要的，如果内存紧张就会自动释放。

* 2.磁盘缓存的处理使用NSFileManager对象实现，图片存储的位置位于cache文件夹，另外SDImageCache还定义了一个串行队列来异步存储图片。

* 3.SDImageCache提供了大量方法来缓存、获取、移除及清空图片。对于图片的索引，我们通过一个key来索引，在内存中，我们将其作为NSCache的key值，而在磁盘中，我们用这个key值作为图片的文件名，对于一个远程下载的图片其url实作为这个key的最佳选择。

**2、存储图片**
先在内存中放置一份缓存，如果需要缓存到磁盘，将磁盘缓存操作作为一个task放到串行队列中处理，会先检查图片格式是jpeg还是png，将其转换为响应的图片数据，最后吧数据写入磁盘中（文件名是对key值做MD5后的串）

**3、查询图片**
内存和磁盘查询图片API：
```
- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key;
- (UIImage *)imageFromDiskCacheForKey:(NSString *)key;

```
查看本地是否存在key指定的图片，使用一下API：

```
- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock;
```
**4、移除图片**
移除图片API：

```
- (void)removeImageForKey:(NSString *)key;
- (void)removeImageForKey:(NSString *)key withCompletion:(SDWebImageNoParamsBlock)completion;
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk;
- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk withCompletion:(SDWebImageNoParamsBlock)completion;

```

**5、清理图片（磁盘）**

清空磁盘图片可以选择完全清空和部分清空，完全清空就是吧缓存文件夹删除。

```
- (void)clearDisk;
- (void)clearDiskOnCompletion:(SDWebImageNoParamsBlock)completion;
```
部分清理 会根据设置的一些参数移除部分文件，主要有两个指标：文件的缓存有效期（maxCacheAge：默认是1周）和最大缓存空间大小（maxCacheSize：如果所有文件大小大于最大值，会按照文件最后修改时间的逆序，以每次一半的递归来移除哪些过早的文件，知道缓存文件总大小小于最大值），具体代码参考- (void)cleanDiskWithCompletionBlock；

**6、小结**
SDImageCache处理提供以上API，还提供了获取缓存大小，缓存中图片数量等API，
常用的接口和属性：

```
（1）-getSize  ：获得硬盘缓存的大小

（2）-getDiskCount ： 获得硬盘缓存的图片数量

（3）-clearMemory  ： 清理所有内存图片

（4）- removeImageForKey:(NSString *)key  系列的方法 ： 从内存、硬盘按要求指定清除图片

（5）maxMemoryCost  ：  保存在存储器中像素的总和

（6）maxCacheSize  ：  最大缓存大小 以字节为单位。默认没有设置，也就是为0，而清理磁盘缓存的先决条件为self.maxCacheSize > 0，所以0表示无限制。

（7）maxCacheAge ： 在内存缓存保留的最长时间以秒为单位计算，默认是一周

```

### 三、SDWebImageManager

实际使用中并不直接使用SDWebImageDownloader和SDImageCache类对图片进行下载和存储，而是使用SDWebImageManager来管理。包括平常使用UIImageView+WebCache等控件的分类，都是使用SDWebImageManager来处理，该对象内部定义了一个图片下载器（SDWebImageDownloader）和图片缓存（SDImageCache）

```
@interface SDWebImageManager : NSObject

@property (weak, nonatomic) id <SDWebImageManagerDelegate> delegate;

@property (strong, nonatomic, readonly) SDImageCache *imageCache;
@property (strong, nonatomic, readonly) SDWebImageDownloader *imageDownloader;

...

@end
```
SDWebImageManager声明了一个delegate属性，其实是一个id<SDWebImageManagerDelegate>对象，代理声明了两个方法

```
// 控制当图片在缓存中没有找到时，应该下载哪个图片
- (BOOL)imageManager:(SDWebImageManager *)imageManager shouldDownloadImageForURL:(NSURL *)imageURL;

// 允许在图片已经被下载完成且被缓存到磁盘或内存前立即转换
- (UIImage *)imageManager:(SDWebImageManager *)imageManager transformDownloadedImage:(UIImage *)image withURL:(NSURL *)imageURL;
```

这两个方法会在SDWebImageManager的-downloadImageWithURL:options:progress:completed:方法中调用，而这个方法是SDWebImageManager类的核心所在（具体看源码）

SDWebImageManager的几个API：

```
(1）- (void)cancelAll   ： 取消runningOperations中所有的操作，并全部删除

（2）- (BOOL)isRunning  ：检查是否有操作在运行，这里的操作指的是下载和缓存组成的组合操作

（3） - downloadImageWithURL:options:progress:completed:   核心方法

（4）- (BOOL)diskImageExistsForURL:(NSURL *)url  ：指定url的图片是否进行了磁盘缓存

```
### 四、视图扩展

在使用SDWebImage的时候，使用最多的是UIImageView+WebCache中的针对UIImageView的扩展，核心方法是sd_setImageWithURL:placeholderImage:options:progress:completed:，   其使用SDWebImageManager单例对象下载并缓存图片。

除了扩展UIImageView外，SDWebImage还扩展了UIView，UIButton，MKAnnotationView等视图类，具体可以参考源码，除了可以使用扩展的方法下载图片，同时也可以使用SDWebImageManager下载图片。

UIView+WebCacheOperation分类：
把当前view对应的图片操作对象存储起来（通过运行时设置属性），在基类中完成
存储的结构：一个loadOperationKey属性，value是一个字典（字典结构： key：UIImageViewAnimationImages或者UIImageViewImageLoad，value是  operation数组（动态图片）或者对象）

UIButton+WebCache分类
会根据不同的按钮状态，下载的图片根据不同的状态进行设置
imageURLStorageKey:{state:url}

### 五、技术点

* 1.dispatch_barrier_sync函数，用于对操作设置顺序，确保在执行完任务后再确保后续操作。常用于确保线程安全性操作
* 2.NSMutableURLRequest：用于创建一个网络请求对象，可以根据需要来配置请求报头等信息
* 3.NSOperation及NSOperationQueue：操作队列是OC中一种告诫的并发处理方法，基于GCD实现，相对于GCD来说，操作队列的优点是可以取消在任务处理队列中的任务，另外在管理操作间的依赖关系方面容易一些，对SDWebImage中我们看到如何使用依赖将下载顺序设置成后进先出的顺序
* 4.NSURLSession：用于网络请求及相应处理
* 5.开启后台任务
* 6.NSCache类：一个类似于集合的容器，存储key-value对，这一点类似于nsdictionary类，我们通常用使用缓存来临时存储短时间使用但创建昂贵的对象。重用这些对象可以优化性能，因为它们的值不需要重新计算。另外一方面，这些对象对于程序来说不是紧要的，在内存紧张时会被丢弃
* 7.清理缓存图片的策略：特别是最大缓存空间大小的设置。如果所有缓存文件的总大小超过这一大小，则会按照文件最后修改时间的逆序，以每次一半的递归来移除那些过早的文件，直到缓存的实际大小小于我们设置的最大使用空间。
* 8.图片解压操作：这一操作可以查看SDWebImageDecoder.m中+decodedImageWithImage方法的实现。
* 9.对GIF图片的处理
* 10.对WebP图片的处理。

***
# 二、什么是Block？
***

*   **Block是将函数及其执行上下文封装起来的对象。**

比如：

```
NSInteger num = 3;
    NSInteger(^block)(NSInteger) = ^NSInteger(NSInteger n){
        return n*num;
    };

    block(2);

```

通过clang -rewrite-objc WYTest.m命令编译该.m文件，发现该block被编译成这个形式:

```
    NSInteger num = 3;

    NSInteger(*block)(NSInteger) = ((NSInteger (*)(NSInteger))&__WYTest__blockTest_block_impl_0((void *)__WYTest__blockTest_block_func_0, &__WYTest__blockTest_block_desc_0_DATA, num));

    ((NSInteger (*)(__block_impl *, NSInteger))((__block_impl *)block)->FuncPtr)((__block_impl *)block, 2);

```

其中WYTest是文件名，blockTest是方法名，这些可以忽略。
其中__WYTest__blockTest_block_impl_0结构体为

```
struct __WYTest__blockTest_block_impl_0 {
  struct __block_impl impl;
  struct __WYTest__blockTest_block_desc_0* Desc;
  NSInteger num;
  __WYTest__blockTest_block_impl_0(void *fp, struct __WYTest__blockTest_block_desc_0 *desc, NSInteger _num, int flags=0) : num(_num) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

```

__block_impl结构体为

```
struct __block_impl {
  void *isa;//isa指针，所以说Block是对象
  int Flags;
  int Reserved;
  void *FuncPtr;//函数指针
};

```

block内部有isa指针，所以说其本质也是OC对象
block内部则为:

```
static NSInteger __WYTest__blockTest_block_func_0(struct __WYTest__blockTest_block_impl_0 *__cself, NSInteger n) {
  NSInteger num = __cself->num; // bound by copy

        return n*num;
    }

```

所以说 Block是将函数及其执行上下文封装起来的对象
既然block内部封装了函数，那么它同样也有参数和返回值。

#### 二、Block变量截获

**1、局部变量截获 是值截获。 比如:**

```
    NSInteger num = 3;

    NSInteger(^block)(NSInteger) = ^NSInteger(NSInteger n){

        return n*num;
    };

    num = 1;

    NSLog(@"%zd",block(2));

```

这里的输出是6而不是2，原因就是对局部变量num的截获是值截获。
同样，在block里如果修改变量num，也是无效的，甚至编译器会报错。

**2、局部静态变量截获 是指针截获。**

```
   static  NSInteger num = 3;

    NSInteger(^block)(NSInteger) = ^NSInteger(NSInteger n){

        return n*num;
    };

    num = 1;

    NSLog(@"%zd",block(2));

```

输出为2，意味着num = 1这里的修改num值是有效的，即是指针截获。
同样，在block里去修改变量m，也是有效的。

**3、全局变量，静态全局变量截获：不截获,直接取值。**

我们同样用clang编译看下结果。

```
static NSInteger num3 = 300;

NSInteger num4 = 3000;

- (void)blockTest
{
    NSInteger num = 30;

    static NSInteger num2 = 3;

    __block NSInteger num5 = 30000;

    void(^block)(void) = ^{

        NSLog(@"%zd",num);//局部变量

        NSLog(@"%zd",num2);//静态变量

        NSLog(@"%zd",num3);//全局变量

        NSLog(@"%zd",num4);//全局静态变量

        NSLog(@"%zd",num5);//__block修饰变量
    };

    block();
}

```

编译后

```
struct __WYTest__blockTest_block_impl_0 {
  struct __block_impl impl;
  struct __WYTest__blockTest_block_desc_0* Desc;
  NSInteger num;//局部变量
  NSInteger *num2;//静态变量
  __Block_byref_num5_0 *num5; // by ref//__block修饰变量
  __WYTest__blockTest_block_impl_0(void *fp, struct __WYTest__blockTest_block_desc_0 *desc, NSInteger _num, NSInteger *_num2, __Block_byref_num5_0 *_num5, int flags=0) : num(_num), num2(_num2), num5(_num5->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

```

（ impl.isa = &_NSConcreteStackBlock;这里注意到这一句，即说明该block是栈block）
可以看到局部变量被编译成值形式，而静态变量被编成指针形式，全局变量并未截获。而__block修饰的变量也是以指针形式截获的，并且生成了一个新的结构体**对象**：

```
struct __Block_byref_num5_0 {
  void *__isa;
__Block_byref_num5_0 *__forwarding;
 int __flags;
 int __size;
 NSInteger num5;
};

```

该对象有个属性：num5，即我们用__block修饰的变量。
这里__forwarding是指向自身的(栈block)。
一般情况下，如果我们要对block截获的局部变量进行赋值操作需添加__block
修饰符，而对全局变量，静态变量是不需要添加__block修饰符的。
另外，block里访问self或成员变量都会去截获self。

#### 三、Block的几种形式

*   **分为全局Block(_NSConcreteGlobalBlock)、栈Block(_NSConcreteStackBlock)、堆Block(_NSConcreteMallocBlock)三种形式**

    **其中栈Block存储在栈(stack)区，堆Block存储在堆(heap)区，全局Block存储在已初始化数据(.data)区**

**1、不使用外部变量的block是全局block**

比如：

```
    NSLog(@"%@",[^{
        NSLog(@"globalBlock");
    } class]);

```

输出：

```
__NSGlobalBlock__

```

**2、使用外部变量并且未进行copy操作的block是栈block**

比如:

```
  NSInteger num = 10;
    NSLog(@"%@",[^{
        NSLog(@"stackBlock:%zd",num);
    } class]);

```

输出：

```
__NSStackBlock__

```

日常开发常用于这种情况:

```
[self testWithBlock:^{
    NSLog(@"%@",self);
}];

- (void)testWithBlock:(dispatch_block_t)block {
    block();

    NSLog(@"%@",[block class]);
}

```

**3、对栈block进行copy操作，就是堆block，而对全局block进行copy，仍是全局block**

*   比如堆1中的全局进行copy操作，即赋值：

```
void (^globalBlock)(void) = ^{
        NSLog(@"globalBlock");
    };

 NSLog(@"%@",[globalBlock class]);

```

输出：

```
__NSGlobalBlock__

```

仍是全局block

*   而对2中的栈block进行赋值操作：

```
NSInteger num = 10;

void (^mallocBlock)(void) = ^{

        NSLog(@"stackBlock:%zd",num);
    };

NSLog(@"%@",[mallocBlock class]);

```

输出：

```
__NSMallocBlock__

```

对栈blockcopy之后，并不代表着栈block就消失了，左边的mallock是堆block，右边被copy的仍是栈block
比如:

```
[self testWithBlock:^{

    NSLog(@"%@",self);
}];

- (void)testWithBlock:(dispatch_block_t)block
{
    block();

    dispatch_block_t tempBlock = block;

    NSLog(@"%@,%@",[block class],[tempBlock class]);
}

```

输出：

```
__NSStackBlock__,__NSMallocBlock__

```

*   **即如果对栈Block进行copy，将会copy到堆区，对堆Block进行copy，将会增加引用计数，对全局Block进行copy，因为是已经初始化的，所以什么也不做。**

另外，__block变量在copy时，由于__forwarding的存在，栈上的__forwarding指针会指向堆上的__forwarding变量，而堆上的__forwarding指针指向其自身，所以，如果对__block的修改，实际上是在修改堆上的__block变量。

**即__forwarding指针存在的意义就是，无论在任何内存位置， 都可以顺利地访问同一个__block变量。**

*   另外由于block捕获的__block修饰的变量会去持有变量，那么如果用__block修饰self，且self持有block，并且block内部使用到__block修饰的self时，就会造成多循环引用，即self持有block，block 持有__block变量，而__block变量持有self，造成内存泄漏。
    比如:

```
  __block typeof(self) weakSelf = self;

    _testBlock = ^{

        NSLog(@"%@",weakSelf);
    };

    _testBlock();

```

如果要解决这种循环引用，可以主动断开__block变量对self的持有，即在block内部使用完weakself后，将其置为nil，但这种方式有个问题，如果block一直不被调用，那么循环引用将一直存在。
所以，我们最好还是用__weak来修饰self

***
# 三、RunLoop剖析
***

**RunLoop是通过内部维护的`事件循环(Event Loop)`来对`事件/消息进行管理`的一个对象。**

1、没有消息处理时，休眠已避免资源占用，由用户态切换到内核态([CPU-内核态和用户态](https://www.jianshu.com/p/3bb1cdd44ef0))
2、有消息需要处理时，立刻被唤醒，由内核态切换到用户态

**为什么main函数不会退出？**

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

UIApplicationMain内部默认开启了主线程的RunLoop，并执行了一段无限循环的代码（不是简单的for循环或while循环）

```
//无限循环代码模式(伪代码)
int main(int argc, char * argv[]) {        
    BOOL running = YES;
    do {
        // 执行各种任务，处理各种事件
        // ......
    } while (running);

    return 0;
}

```

UIApplicationMain函数一直没有返回，而是不断地接收处理消息以及等待休眠，所以运行程序之后会保持持续运行状态。

#### 二、RunLoop的数据结构

`NSRunLoop(Foundation)`是`CFRunLoop(CoreFoundation)`的封装，提供了面向对象的API
RunLoop 相关的主要涉及五个类：

`CFRunLoop`：RunLoop对象
`CFRunLoopMode`：运行模式
`CFRunLoopSource`：输入源/事件源
`CFRunLoopTimer`：定时源
`CFRunLoopObserver`：观察者

**1、CFRunLoop**

由`pthread`(线程对象，说明RunLoop和线程是一一对应的)、`currentMode`(当前所处的运行模式)、`modes`(多个运行模式的集合)、`commonModes`(模式名称字符串集合)、`commonModelItems`(Observer,Timer,Source集合)构成

**2、CFRunLoopMode**

由name、source0、source1、observers、timers构成

**3、CFRunLoopSource**

分为source0和source1两种

*   `source0`:
    即非基于port的，也就是用户触发的事件。需要手动唤醒线程，将当前线程从内核态切换到用户态
*   `source1`:
    基于port的，包含一个 mach_port 和一个回调，可监听系统端口和通过内核和其他线程发送的消息，能主动唤醒RunLoop，接收分发系统事件。
    具备唤醒线程的能力

**4、CFRunLoopTimer**

基于时间的触发器，基本上说的就是NSTimer。在预设的时间点唤醒RunLoop执行回调。因为它是基于RunLoop的，因此它不是实时的（就是NSTimer 是不准确的。 因为RunLoop只负责分发源的消息。如果线程当前正在处理繁重的任务，就有可能导致Timer本次延时，或者少执行一次）。

**5、CFRunLoopObserver**

监听以下时间点:`CFRunLoopActivity`

*   `kCFRunLoopEntry`
    RunLoop准备启动
*   `kCFRunLoopBeforeTimers`
    RunLoop将要处理一些Timer相关事件
*   `kCFRunLoopBeforeSources`
    RunLoop将要处理一些Source事件
*   `kCFRunLoopBeforeWaiting`
    RunLoop将要进行休眠状态,即将由用户态切换到内核态
*   `kCFRunLoopAfterWaiting`
    RunLoop被唤醒，即从内核态切换到用户态后
*   `kCFRunLoopExit`
    RunLoop退出
*   `kCFRunLoopAllActivities`
    监听所有状态

**6、各数据结构之间的联系**

线程和RunLoop一一对应， RunLoop和Mode是一对多的，Mode和source、timer、observer也是一对多的

![](https://upload-images.jianshu.io/upload_images/17495317-4664eb028a3953a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 三、RunLoop的Mode

关于Mode首先要知道一个RunLoop 对象中可能包含多个Mode，且每次调用 RunLoop 的主函数时，只能指定其中一个 Mode(CurrentMode)。切换 Mode，需要重新指定一个 Mode 。主要是为了分隔开不同的 Source、Timer、Observer，让它们之间互不影响。

![](https://upload-images.jianshu.io/upload_images/17495317-14318d5787a24b75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


当RunLoop运行在Mode1上时，是无法接受处理Mode2或Mode3上的Source、Timer、Observer事件的

总共是有五种`CFRunLoopMode`:

*   `kCFRunLoopDefaultMode`：默认模式，主线程是在这个运行模式下运行

*   `UITrackingRunLoopMode`：跟踪用户交互事件（用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他Mode影响）

*   `UIInitializationRunLoopMode`：在刚启动App时第进入的第一个 Mode，启动完成后就不再使用

*   `GSEventReceiveRunLoopMode`：接受系统内部事件，通常用不到

*   `kCFRunLoopCommonModes`：伪模式，不是一种真正的运行模式，是同步Source/Timer/Observer到多个Mode中的一种解决方案

#### 四、RunLoop的实现机制

![](https://upload-images.jianshu.io/upload_images/17495317-a49b16466c6dd707.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这张图在网上流传比较广。
对于RunLoop而言最核心的事情就是保证线程在没有消息的时候休眠，在有消息时唤醒，以提高程序性能。RunLoop这个机制是依靠系统内核来完成的（苹果操作系统核心组件Darwin中的Mach）。

![](https://upload-images.jianshu.io/upload_images/17495317-50fd6da547027acd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

RunLoop通过`mach_msg()`函数接收、发送消息。它的本质是调用函数`mach_msg_trap()`，相当于是一个系统调用，会触发内核状态切换。在用户态调用 `mach_msg_trap()`时会切换到内核态；内核态中内核实现的`mach_msg()`函数会完成实际的工作。
即基于port的source1，监听端口，端口有消息就会触发回调；而source0，要手动标记为待处理和手动唤醒RunLoop

[Mach消息发送机制](https://www.jianshu.com/p/a764aad31847)
大致逻辑为：
1、通知观察者 RunLoop 即将启动。
2、通知观察者即将要处理Timer事件。
3、通知观察者即将要处理source0事件。
4、处理source0事件。
5、如果基于端口的源(Source1)准备好并处于等待状态，进入步骤9。
6、通知观察者线程即将进入休眠状态。
7、将线程置于休眠状态，由用户态切换到内核态，直到下面的任一事件发生才唤醒线程。

*   一个基于 port 的Source1 的事件(图里应该是source0)。
*   一个 Timer 到时间了。
*   RunLoop 自身的超时时间到了。
*   被其他调用者手动唤醒。

8、通知观察者线程将被唤醒。
9、处理唤醒时收到的事件。

*   如果用户定义的定时器启动，处理定时器事件并重启RunLoop。进入步骤2。
*   如果输入源启动，传递相应的消息。
*   如果RunLoop被显示唤醒而且时间还没超时，重启RunLoop。进入步骤2

10、通知观察者RunLoop结束。

#### 五、RunLoop与NSTimer

一个比较常见的问题：滑动tableView时，定时器还会生效吗？
默认情况下RunLoop运行在`kCFRunLoopDefaultMode`下，而当滑动tableView时，RunLoop切换到`UITrackingRunLoopMode`，而Timer是在`kCFRunLoopDefaultMode`下的，就无法接受处理Timer的事件。
怎么去解决这个问题呢？把Timer添加到`UITrackingRunLoopMode`上并不能解决问题，因为这样在默认情况下就无法接受定时器事件了。
所以我们需要把Timer同时添加到`UITrackingRunLoopMode`和`kCFRunLoopDefaultMode`上。
那么如何把timer同时添加到多个mode上呢？就要用到`NSRunLoopCommonModes`了

```
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

Timer就被添加到多个mode上，这样即使RunLoop由`kCFRunLoopDefaultMode`切换到`UITrackingRunLoopMode`下，也不会影响接收Timer事件

#### 六、RunLoop和线程

*   线程和RunLoop是一一对应的,其映射关系是保存在一个全局的 Dictionary 里
*   自己创建的线程默认是没有开启RunLoop的

**1、怎么创建一个常驻线程？**

1、为当前线程开启一个RunLoop（第一次调用 [NSRunLoop currentRunLoop]方法时实际是会先去创建一个RunLoop）
1、向当前RunLoop中添加一个Port/Source等维持RunLoop的事件循环（如果RunLoop的mode中一个item都没有，RunLoop会退出）
2、启动该RunLoop

```
   @autoreleasepool {
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
```

**2、输出下边代码的执行顺序**

```
 NSLog(@"1");
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSLog(@"2");
    [self performSelector:@selector(test) withObject:nil afterDelay:10];
    NSLog(@"3");
});
NSLog(@"4");
- (void)test
{
    NSLog(@"5");
}
```

答案是1423，test方法并不会执行。
原因是如果是带afterDelay的延时函数，会在内部创建一个 NSTimer，然后添加到当前线程的RunLoop中。也就是如果当前线程没有开启RunLoop，该方法会失效。
那么我们改成:

```
dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"2");
        [[NSRunLoop currentRunLoop] run];
        [self performSelector:@selector(test) withObject:nil afterDelay:10];
        NSLog(@"3");
    });
```

然而test方法依然不执行。
原因是如果RunLoop的mode中一个item都没有，RunLoop会退出。即在调用RunLoop的run方法后，由于其mode中没有添加任何item去维持RunLoop的时间循环，RunLoop随即还是会退出。
所以我们自己启动RunLoop，一定要在添加item后

```
dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"2");
        [self performSelector:@selector(test) withObject:nil afterDelay:10];
        [[NSRunLoop currentRunLoop] run];
        NSLog(@"3");
    });
```

**3、怎样保证子线程数据回来更新UI的时候不打断用户的滑动操作？**

当我们在子请求数据的同时滑动浏览当前页面，如果数据请求成功要切回主线程更新UI，那么就会影响当前正在滑动的体验。
我们就可以将更新UI事件放在主线程的`NSDefaultRunLoopMode`上执行即可，这样就会等用户不再滑动页面，主线程RunLoop由`UITrackingRunLoopMode`切换到`NSDefaultRunLoopMode`时再去更新UI

```
[self performSelectorOnMainThread:@selector(reloadData) withObject:nil waitUntilDone:NO modes:@[NSDefaultRunLoopMode]];
```
***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
