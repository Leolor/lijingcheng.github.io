---
layout: post
title: "iOS应用的生命周期"
date: 2015-03-28 11:23:29 +0800
comments: true
categories: 
---


{% img left http://7x00ed.com1.z0.glb.clouddn.com/app-lifecycle.jpeg 160 160 "" "" %}
>**生命周期 - *程序的生命周期是指应用程序从启动到结束整个阶段的全过程.***   
点击iOS应用图标打开程序，系统会首先通过main函数进行相关设置，然后通过runloop保持程序能够始终运行并监听处理分发事件，系统依靠事件传递及响应者链机制将事件对象交给相应视图处理，当没有事件发生时runloop便处于睡眠状态，节省资源。当用户按下home键，应用会在后台短暂运行后被系统挂起。

<br/>
<!-- more -->

#main函数
main函数是app的入口函数，所以我们要先了解一下   
``` objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```  
  
- @autoreleasepool 作为app主线程的自动释放池，用来管理所有主线程中标记为自动释放的对象。
- UIApplicationMain 根据传入参数以及info.plist文件来初始化app  
	- argc和argv参数在iOS应用中用不到
	- 第三个参数为app的首要类名，用来监听并管理应用的生命周期，默认使用UIApplication
	- 最后一个参数为app首要类的代理类，它负责实际处理UIApplication监听到的应用程序生命周期事件，具体参考代理类实现的这个协议"UIApplicationDelegate"
	- main函数会在初始化app后启动主线程及runloop
	
#runloop

runloop是事件接收和分发机制的一个实现，当没有事件发生时runloop会进入休眠状态，有事件发生时它就被唤醒并处理事件，在处理完事件后还会清理一下自动释放池。runloop可以接收的事件来自两种不同的来源，分别是输入源和定时源，输入源传递异步事件，事件通常来自于其他线程或程序，如一些UI事件等。定时源则传递同步事件，发生在特定时间或者重复的时间间隔。iOS系统将接收到的事件放在一个队列里，然后依次交给runloop处理分发，这样可以解决线程同步问题，这一特点也促使我们要把更新UI的代码放在主线程中运行，以免由于多个线程抢占资源产生奇怪的问题。每个线程都有自己的runloop, 主线程是默认开启的，子线程需要手动开启，并且在开启后必须至少添加一个事件源，否则runloop在启动后会立即结束。  

<br/>
那么什么情况下需要我们在子线程中开启并使用runloop呢？

- 在线程中需要持续监测某个事件。例如使用NSURLConnection异步请求数据，如果没有开启runloop，会因为该子线程已经结束，所以相关delegate方法不会被触发。如果你使用AFNetworking，那么就不用考虑这个问题了，因为它已经帮你做了
- 线程间需要持续交互，例如当遇到多个线程之间发生同步问题时，可以将多个线程定义成多个事件源，然后让他们运行在同一线程的runloop下，解决线程同步问题
- 使用NSTimer或performSelector方法

<br/>
runloop有几种特定模式，不同的事件运行在不同的模式中，只有和模式相匹配的源才会被监视并允许他们传递事件消息。  

- NSTimer运行在NSDefaultRunLoopMode模式下
- 列表滚动事件运行在UITrackingRunLoopMode模式下
- NSRunLoopCommonModes包含NSDefaultRunLoopMode和UITrackingRunLoopMode。  

当我们在UITableViewCell上放置一个timer，并且在滚动表格时，timer就会停止工作。解决办法只要将timer对象放置于NSRunLoopCommonModes模式下，这样timer不仅可以在表格滚动时运行，也可以在表格停止滚动时运行。

``` objc
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];//NSTimer并不是以子线程的方式运行，它只是在runloop里注册了一下，runloop会根据timer的设置情况去检测并触发，如果runloop比较忙卡住了也会影响到timer。
```

<br/>
当用户点击界面上的一个按钮后，系统将这个事件交给runloop，runloop再交给相应对象去处理这个事件，那么系统是怎么找到按钮，并且调用相关事件方法呢？

<br/>
#响应者链
响应链由一系列链接在一起的响应者组成。一般情况下一条响应链开始于第一响应者，结束于application对象。如果一个响应者不能处理事件，则会将事件沿着响应链传到下一响应者。下一响应者有可能是它的父视图也可能是第一响应者所在的ViewController，系统会以此类推，一直向上传递到UIWindow和UIApplication。当最后到达UIApplication对象时，如果整个过程都没有响应者响应这个事件，该事件就被丢弃。一般情况下，在响应者链中只要有对象处理事件，事件就停止传递。

当用户点击button后，系统会将相关UIEvent对象放到由UIApplication管理的事件队列中，再由runloop接收事件并分发出去。通常先将事件分发给UIWindow，UIWindow会调用"hitTest:withEvent:"方法寻找最合适的子视图来处理触摸事件，它会调用"pointInside:withEvent:"方法来判定触摸点是否在当前窗口，这时pointInside方法会返回YES，UIWindow会调用子视图的hitTest方法(如果有多个子视图，会按子视图的添加顺序，由后向前选择)。子视图会继续通过pointInside方法来判断触摸点是否在当前窗口，如果仍然返回YES，则会继续调用子视图的hitTest方法，如果返回NO，则hitTest方法返回nil，这时当前视图会向其它子视图继续hitTest，如果每个子视图都返回nil，则由当前返回不是nil的视图作为我们希望处理该事件的视图，也称为** hit-test视图 **，查找这一视图的过程就叫做** hit-testing **。在这过程中如果视图不具备响应事件的条件(userInteractionEnabled或enabled为NO，hidden=YES或alpha=0)，那么hitTest就不会调用自己的pointInside方法，会直接返回nil，该视图的子视图也不会被遍历到，也不能够响应该事件，如果希望它的子视图能够响应事件，那么需要重写这个视图的hitTest方法，让它在某种情况下直接返回某个子视图。
- 如果要对hit-test视图的事件响应过程进行控制，那么需要自定义它的父视图并重写以下方法
``` objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event;
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event;
```

如果想让整个程序在某种情况下不响应事件，可以调用UIApplication的以下方法停止和恢复事件接收和分发。
``` objc
- (void)beginIgnoringInteractionEvents
- (void)endIgnoringInteractionEvents
```

<br/>
#前后台切换
当应用程序在前台和后台之间切换时，系统会调用UIApplicationDelegate的相关代理方法并发送相关通知，你可以通过这些通知来修改你的应用程序的行为，例如可以在进入后台时暂停某些操作或存储某些数据，当恢复到前台时再读取这些数据或恢复之前的暂停事件。

程序进入后台后仍然能够在短时间里执行代码，随后便会进入挂起状态，这时程序仍然在内存中，但是不能执行代码，当系统内存低时，系统就把挂起的程序清除掉，为前台程序提供更多的内存。


