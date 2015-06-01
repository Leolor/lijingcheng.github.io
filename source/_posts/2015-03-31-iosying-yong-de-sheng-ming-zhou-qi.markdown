---
layout: post
title: "iOS应用的生命周期"
date: 2015-03-28 11:23:29 +0800
comments: true
categories: 
---

<br/>
{% img left http://7x00ed.com1.z0.glb.clouddn.com/app-lifecycle.jpeg 160 160 "" "" %}
>**生命周期 - *程序的生命周期是指应用程序从启动到结束整个阶段的全过程.***   
点击iOS应用图标打开程序，系统会首先通过main函数进行相关设置，然后通过runloop保持程序能够始终运行并监听处理分发事件，当没有事件发生时runloop便处于睡眠状态，节省资源。当发生事件后，runloop将事件对象分发给相应视图处理。当用户按下home键，应用会在进入后台后短暂运行，直到被系统挂起。

<br/>
<!-- more -->

#main函数
main函数是app的入口函数   
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
	- 最后一个参数为app首要类的代理类，它负责实际处理UIApplication监听到的应用程序生命周期事件，具体参考"UIApplicationDelegate"
	- main函数会在初始化app后启动主线程及runloop
	
#runloop

runloop能够保持程序始终运行并监听处理分发事件，当没有事件发生时进入休眠状态，有事件发生时系统会将接收到的事件放在一个队列里，然后唤醒runloop依次处理事件，这一特点不仅可以解决线程同步问题也促使我们要把更新UI的代码放在主线程中运行，以免由于多个线程抢占资源产生奇怪的问题。runloop可以接收的事件来自两种不同的来源，分别是输入源和定时源，输入源传递异步事件，事件通常来自于其他线程或程序，如一些UI事件等。定时源则传递同步事件，发生在特定时间或者重复的时间间隔，如NSTimer。runloop每处理完一个事件后便清理自动释放池。

<br/>
每个线程都有自己的runloop, 主线程是默认开启的，子线程需要手动开启，并且在开启后必须至少添加一个事件源，否则runloop在启动后会立即结束。那么什么情况下需要我们在子线程中开启并使用runloop呢？

- 在线程中需要持续监测某个事件。例如使用NSURLConnection异步请求数据，如果没有开启runloop，会因为子线程的结束导致相关delegate方法不会被触发。(如果你使用AFNetworking，那么就不用考虑这个问题了，因为它已经帮你做了)
- 线程间需要持续交互，例如当多个线程之间产生同步问题时，可以根据情况考虑将多个线程定义成多个事件源，然后让它们运行在同一线程的runloop下。
- 使用NSTimer或performSelector系列方法。
	- performSelecter:afterDelay: 会创建timer并添加到当前线程的runloop中。
	- performSelector:onThread: 会创建timer并添加到指定线程的runloop中。

<br/>
不同的事件源会运行在runloop的不同模式中，它们只有在相匹配的情况下才会被处理。

- NSTimer运行在NSDefaultRunLoopMode模式下。
- 列表滚动事件运行在UITrackingRunLoopMode模式下。
- NSRunLoopCommonModes包含NSDefaultRunLoopMode和UITrackingRunLoopMode。  

我们在Cell上放置一个timer，然后在滚动表格时会发现timer没有正常执行，要解决这个问题，只要将timer运行在NSRunLoopCommonModes模式下，便可以在表格静止以及滚动时都能够正常运行。

``` objc
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];//NSTimer并不是以子线程的方式运行，它只是在runloop里注册了一下，runloop会根据timer的设置情况去检测并触发，如果runloop比较忙卡住了也会影响到timer。
```

<br/>
#事件传递&响应者链
当发生触摸操作后，系统会将这一事件封装成UIEvent对象并放到由UIApplication管理的事件队列中，再由runloop接收事件并传递给触摸点所在的视图，该视图即为"hit-test视图"，而查找这一视图的过程就叫做"hit-testing"。hit-testing过程大致如下: 
 
- runloop将接收到的事件分发给UIWindow。
- UIWindow通过"hitTest:withEvent:"方法在视图树中递归查找触摸点所在的视图。
- 当前视图通过hitTest方法调用"pointInside:withEvent:"来判定触摸点是否在当前视图，如果不在hitTest返回nil，在的话则从当前视图的subViews末尾向前遍历，依次向每个subView发送hitTest消息，以此规则一直到某个subView不再返回nil或遍历完成。
- 最终由返回不是nil的视图作为"hit-test视图"。

在这过程中如果视图不具备响应事件的条件(userInteractionEnabled或enabled为NO，hidden=YES或alpha=0)，那么hitTest就不会调用pointInside方法，会直接返回nil，该视图的子视图也就不会被遍历到，如果我们想改变这一点，或者有别的需求需要改变事件传递的规则，那么需要自定义父视图并重写以下方法来控制子视图。
``` objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event;
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event;
```
<br/>
如果想在某种情况下不响应事件，可以在适当时调用UIApplication的以下方法停止和恢复事件接收和分发。
``` objc
- (void)beginIgnoringInteractionEvents
- (void)endIgnoringInteractionEvents
```

<br/>
如果hit-test视图不处理收到的事件，则通过响应者链机制寻找其它响应者来处理。响应链由一系列链接在一起的响应者组成。如果一个响应者不能处理事件，则会将事件沿着响应链传到下一响应者。下一响应者有可能是它的父视图也可能是它所在的ViewController，系统会以此类推一直传递到UIApplication。如果整个过程都没有响应者响应事件，该事件就会被丢弃。否则事件便会停止传递交由响应者处理。

<br/>
#前后台切换
在程序进入后台后仍然能够在短时间里执行一些代码，然后便进入挂起状态，程序在挂起后仍然会驻留在内存中，但是不能执行代码，直到iOS系统内存降低发出警告后才会把相对耗内存的挂起程序清除掉。  

当程序在前后台切换时，系统会调用"UIApplicationDelegate"的相关代理方法并发送通知，我们可以在不同情况下做出不同处理，例如在进入后台时暂停某些操作或存储某些数据，当恢复到前台时再恢复之前的暂停操作或读取之前存储的数据。


