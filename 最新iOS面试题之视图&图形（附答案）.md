### 返回目录:[全网各大厂iOS面试题-题集大全](https://github.com/LGBamboo/iOS-Advanced)

# 最新iOS面试题之视图&图形（附答案）

本篇我们来讲一下 【iOS面试题的视图&图形】相关的问题.

## 视图&图像相关

主要问题列表如下:

1.  AutoLayout的原理，性能如何
2.  UIView & CALayer的区别
3.  事件响应链
4.  drawrect & layoutsubviews调用时机
5.  UI的刷新原理
6.  隐式动画 & 显示动画区别
7.  什么是离屏渲染
8.  imageName&imageWithContentsOfFile区别
9.  多个相同的图片，会重复加载吗
10.  图片是什么时候解码的，如何优化
11.  图片渲染怎么优化
12.  如果GPU的刷新率超过了iOS屏幕60Hz刷新率是什么现象，怎么解决

### 1.AutoLayout的原理，性能如何?

#### AutoLayout的原理

> 来历 一般大家都会认为Auto Layout这个东西是苹果自己搞出来的，其实不然，早在1997年Alan Borning, Kim Marriott, Peter Stuckey等人就发布了《Solving Linear Arithmetic Constraints for User Interface Applications》论文（[论文地址:http://constraints.cs.washington.edu/solvers/uist97.html](http://constraints.cs.washington.edu/solvers/uist97.html)）提出了在解决布局问题的Cassowary constraint-solving算法实现，并且将代码发布在他们搭建的[Cassowary网站上http://constraints.cs.washington.edu/cassowary/](http://constraints.cs.washington.edu/cassowary/)。后来更多开发者用各种语言来写Cassowary，比如说pybee用python写的https://github.com/pybee/cassowary。自从它发布以来JavaScript，.NET，JAVA，Smalltall和C++都有相应的库。2011年苹果将这个算法运用到了自家的布局引擎中，美其名曰Auto Layout。

论文下载链接比较慢,我下载了一份[Cassowary原文放到了我的博客 大家可以自由下载](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200920UIViewGraphic/Cassowary.pdf).

**AutoLayout的原理就是用Cassowary算法来将布局问题抽象成线性不等式，并分解成多个位置间的约束**
因为多了计算视图大小frame的过程,所以性能肯定没有指定Frame坐标要快.

详细的原理以及高阶原理请参考戴铭老师的文章 [戴铭老师写的 深入剖析Auto Layout，分析iOS各版本新增特性](http://www.starming.com/2015/11/03/deeply-analyse-autolayout/)

#### 性能如何?

下面是[WWDC2018 High Performance Auto Layout](https://developer.apple.com/videos/play/wwdc2018/220/)中对比的iOS12和iOS11下分别使用自动布局的性能对比现场.

[![](https://upload-images.jianshu.io/upload_images/13277235-7f3b7a19e8e4e67b.gif?imageMogr2/auto-orient/strip)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200920UIViewGraphic/HighPerformanceAutoLayoutiOS11iOS12Compare.gif) 

经过实验得出如下图标结论:

[![](https://upload-images.jianshu.io/upload_images/13277235-b9325441a3364189.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200920UIViewGraphic/HighPerformanceAutoLayoutResult.png) 

iOS12之前，视图嵌套的数量对性能的影响是呈指数级增长的，而iOS12优化之后对性能的影响是线性增长，对性能消耗不大。

无论如何优化也肯定不如CGRectFrame那样的设置更加直接,性能更好.

### 2.UIView & CALayer的区别

| 区别 | UIView | CALayer |
| --- | --- | --- |
| 继承父类 | UIView:UIResponder:NSObject | CALayer:NSObject |
| 用途 | 可以处理触摸事件 | 不处理用户的交互,不参与响应事件传递 |
| 两者关系 | 有一个CALayer成员变量 eg: view.layer | 是UIView的成员变量 |
| 分工 | 处理交互层事件并包装各种图形的简单设置 | 底层渲染图形,支持动画 |

### 3.事件响应链

下面这篇文章我已经在前几篇将runloop的时候提了不止一次,前列建议阅读,快手的同事大部分都以这个理解为标准

[iOS触摸事件全家桶](https://www.jianshu.com/p/c294d1bd963d)

### 4\. drawrect & layoutsubviews调用时机

`layoutSubviews:`(相当于layoutSubviews()函数)在以下情况下会被调用：

1.  init初始化不会触发layoutSubviews。
2.  addSubview会触发layoutSubviews。
3.  设置view的Frame会触发layoutSubviews (frame发生变化触发)。
4.  滚动一个UIScrollView会触发layoutSubviews。
5.  旋转Screen会触发父UIView上的layoutSubviews事件。
6.  改变一个UIView大小的时候也会触发父UIView上的layoutSubviews事件。
7.  直接调用setLayoutSubviews。

`drawrect:`(drawrect()函数)在以下情况下会被调用：

1.  `drawrect:`是在UIViewController的`loadView:`和`ViewDidLoad:`方法之后调用.
2.  当我们调用`[UIFont的 sizeToFit]`后,会触发系统自动调用`drawRect:`
3.  当设置UIView的contentMode或者Frame后会立即触发触发系统调用`drawRect:`
4.  直接调用`setNeedsDisplay`设置标记 或`setNeedsDisplayInRect:`的时候会触发`drawRect:`

> 知识点扩充: 当我们操作drawRect方法的时候实际是在操作内存中存放视图的backingStore区域,用于后续图形的渲染操作,如果不理解可以看下[UIView的渲染过程](https://www.jianshu.com/p/a120d6c64d88).

### 5.UI的刷新原理

这个问题我不知道问的是不是iOS离屏渲染过程,我来简单的回到一下这个吧

iOS 的`MainRunloop` 是一个60fps 的回调,也就是说16.7ms(毫秒)会绘制一次屏幕在这过程中要完成以下的工作:

*   view的缓冲区创建
*   view内容的绘制(如果重写了 drawRect)
*   接收和处理系统的触摸事件

我们看到的UI图形实际上是CPU和GPU不断配合工作的结果.经过[UIView的渲染过程](https://www.jianshu.com/p/a120d6c64d88) 后我们的UI会不间断的接收系统图给我们的事件.

由于主线程的runloop 一直在回调,我们的UI就得到了刷新的窗口,是渲染还是处理事件都是因为runloop不断工作的结果.前几篇我们学过 main线程的runloop默认是启动的.因为我们响应交互.

不知道我这样回答是否满足这个问题的答案.如果回答的不对烦请下方评论区留言 告知我将持续改进.

### 6.隐式动画 & 显示动画区别

隐式动画一直存在 如需关闭需设置
显式动画是不存在，如需显式 要开启

只需要观察动画执行完成的结果 比如: 一个简单UIView的frame移动 如果从A点移动到B点 移动完成 回到原始位置就是隐式动画

Core Animation 是显式动画.因为它既可以直接对其layer属性做动画，也可以覆盖默认的图层行为.

### 7.imageName&imageWithContentsOfFile区别

| 区别 | UIView | imageWithContentsOfFile |
| --- | --- | --- |
| 不同点 | 会图片缓存到内存中 | 无缓存 |

### 8.什么是离屏渲染

[![image](https://upload-images.jianshu.io/upload_images/13277235-acd41dc1f938b9bc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://gitee.com/sunyazhou/sunyazhou13.github.io-images/raw/master/20200920UIViewGraphic/CoreAnimationPipeline.jpg) 

[iOS离屏渲染的深入研究](https://zhuanlan.zhihu.com/p/72653360)

### 9.多个相同的图片，会重复加载吗

不会,GPU有 像素点缓存的mask.

### 10.图片是什么时候解码的，如何优化

是加载到内存中,从UIImge->CGImage->CGImageSourceCreateWithData(data) 创建ImageSource变成bitmap位图,这些工作都是CoreAnimation在图片被加载到内存中存在在backingStore里,送给GPU流水线处理之前被解码.

#### 如何优化

自己手动操作图片的编码API

CGImageSource开头的哪些,根据合理利用时机和操作系统资源调整出一套缓存小加载快的库.

参考[PINRemoteImage](https://github.com/pinterest/PINRemoteImage)或者[YYWebImage](https://github.com/ibireme/YYWebImage)开源

### 11.图片渲染怎么优化

可以从阴影,圆角入手.帧率,电量,图片的锯齿等等.

[iOS开发-视图渲染与性能优化](https://www.jianshu.com/p/748f9abafff8)

### 12.如果GPU的刷新率超过了iOS屏幕60Hz刷新率是什么现象，怎么解决

现象是 图形清晰,场景逼真,但是一般arm芯片的GPU 刷新超过60Hz一定会超级费电,手机发热导致降频.FPS降低,因为低能耗电量不足,无法支持GPU高刷新率

解决办法只能用xcode自带工具检测,看渲染过程哪里可以优化.

# 总结

简单回答了一些图形相关的问题,大部分都是iOS离屏渲染,这个地方大家要认真学习.很多资料看起来比较耗时.

### 返回目录:[全网各大厂iOS面试题-题集大全](https://github.com/LGBamboo/iOS-Advanced)

***
### 更多精选大厂 · iOS面试题答案PDF文集

![](https://upload-images.jianshu.io/upload_images/17495317-e01b6f4e054727b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 获取加小编的iOS技术交流圈：**[937 194 184](https://jq.qq.com/?_wv=1027&k=5PARXCI)**，直接获取
